# Tdarr AV1 VAAPI Flow

<img width="841" height="841" alt="image" src="https://github.com/user-attachments/assets/6f05b181-d42f-41f9-b035-e0be1abb6791" />


A Tdarr flow that transcodes video to AV1 using Intel GPU hardware encoding via VAAPI. Handles HDR10, HDR10+, and Dolby Vision (Profiles 4, 5, 7, and 8) correctly, skipping files that should not be processed.

> **Tested on:** Intel Arc Pro B70 (Battlemage BMG-G31, 32GB VRAM) running on Unraid with the `xe` driver.

---

## Prerequisites

### 1. Intel GPU with VAAPI support

Your Tdarr node container must have access to the Intel render device. Verify it exists on your host:

```bash
ls /dev/dri/
```

You should see a `renderD128` (or similar) entry. Pass it through to your container:

```yaml
devices:
  - /dev/dri:/dev/dri
```

### 2. nichols89ben's DoVi Processing plugins

This flow depends on three local plugins from [nichols89ben/Tdarr_DoVi_Processing](https://github.com/nichols89ben/Tdarr_DoVi_Processing):

- `checkHDRType`
- `checkHdrFallback`
- `checkDoViProfile`

Install them before importing this flow, otherwise the HDR/DoVi detection nodes will fail. Follow the installation instructions in that repo — the plugins go into your local FlowPlugins directory, **not** the community folder.

### 3. `av1_vaapi` encoder support

Your ffmpeg build must support `av1_vaapi`. Verify with:

```bash
ffmpeg -encoders | grep av1_vaapi
```

---

## Flow Variables

Five user variables must be configured in your Tdarr library before running this flow.

Go to **Libraries → [your library] → Options → Flow Variables** and add:

| Variable | Description | Example Value |
|---|---|---|
| `vaapiDevice` | Path to your Intel render device | `/dev/dri/renderD128` |
| `av1Quality` | AV1 encoding quality. Lower = better quality, larger file. | `24` |
| `isTest` | Set to `true` to move output to `testDirectory` instead of replacing the original. Set to `false` for normal operation. | `false` |
| `testDirectory` | Directory to move output files to when `isTest` is `true`. Only used in test mode. | `/mnt/media/temp` |
| `checkFileSize` | Set to `true` to skip the post-encode file size comparison. Set to `false` to keep it enabled (output larger than input will not replace original). | `false` |

> **Note:** If `isTest` is `false`, the `testDirectory` variable is still required by the flow but its value will not be used. Set it to any valid path to avoid an empty variable warning.

---

## What This Flow Does

### Files that are skipped (no action taken):
- Files under 700MB
- Files already encoded as AV1 in an MKV container
- Dolby Vision Profile 5 or 7
- Dolby Vision files with no HDR10 fallback metadata

### Files that are processed:
- Any video over 700MB that is not already AV1+MKV, passes the HDR/DoVi checks, and has a supported HDR type

### Processing path:
1. **File size check** — skips if under 700MB
2. **HDR type detection** — identifies SDR, HDR10, HDR10+, or Dolby Vision
3. **Dolby Vision profile check** — skips unsupported profiles (5, 7) and files with no HDR10 fallback
4. **AV1 codec check** — if already AV1, checks container; if MKV, skips; if not MKV, remuxes to MKV
5. **Encoding** — non-AV1 files are encoded via `av1_vaapi` at the configured quality
   - HDR files: preserves BT.2020 color metadata (`p010le`, `smpte2084`, `bt2020nc`)
   - SDR files: encodes to `p010le`
6. **Stream reorder** — prioritizes English and French audio/subtitle tracks
7. **Post-encode size check** — if `checkFileSize` is `false`, keeps original if output is larger
8. **Output** — replaces original file, or moves to test directory if `isTest` is `true`

---

## Encoding Settings

| Setting | Value | Notes |
|---|---|---|
| Encoder | `av1_vaapi` | Intel VAAPI hardware encoder |
| Quality | Configurable via `av1Quality` | Passed as `-global_quality`. Lower = better quality. |
| Preset | Disabled | `veryslow` is not a valid option for `av1_vaapi`. Quality is controlled solely via the quality value. |
| HDR pixel format | `p010le` | Required for VAAPI 10-bit output |
| SDR pixel format | `p010le` | Encoded as 10-bit even for SDR sources |
| Audio | Copied | No re-encoding |
| Subtitles | Copied | No re-encoding |
| Stream order | `eng, fre` | Edit the **Reorder Streams** node directly to change language priority |

---

## Importing the Flow

1. In Tdarr, go to **Flows → Import Flow**
2. Paste the contents of `AV1_Flow.json`
3. Configure all five flow variables in your library as described above
4. Run on a test file with `isTest` set to `true` before applying to your full library

---

## Known Limitations

- **Intel-only.** This flow uses `av1_vaapi` and will fail on nodes without Intel GPU VAAPI support. There is no AMD or NVIDIA fallback — the flow will explicitly fail if `av1_vaapi` is not detected.
- **No QSV support.** Intel Battlemage (Arc B-series) does not support QSV for AV1. VAAPI is the correct path for this generation.
- **DV Profile 5 and 7 are skipped.** These profiles cannot be safely transcoded to AV1 without specialized tooling beyond this flow's scope. Use nichols89ben's flow to handle those files first if needed.

---

## Credits

HDR and Dolby Vision detection plugins by [nichols89ben](https://github.com/nichols89ben/Tdarr_DoVi_Processing), originally based on work by [andrasmaroy](https://github.com/andrasmaroy/Tdarr_Plugins_DoVi).
