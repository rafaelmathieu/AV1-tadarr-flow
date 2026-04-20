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

> **Important:** This flow hardcodes `/dev/dri/renderD128`. If your render node is at a different path (e.g., `renderD129`), you must update it manually in two nodes after importing:
> - **Custom Arguments HDR** → `inputArguments`
> - **Custom Arguments SDR** → `inputArguments`

### 2. nichols89ben's DoVi Processing plugins

This flow depends on three local plugins from [nichols89ben/Tdarr_DoVi_Processing](https://github.com/nichols89ben/Tdarr_DoVi_Processing):

- `checkHDRType`
- `checkHdrFallback`
- `checkDoViProfile`

Install them before importing this flow, otherwise the HDR/DoVi detection nodes will fail. Follow the installation instructions in that repo — the plugins go into your local FlowPlugins directory, **not** the community folder.

### 3. `av1_vaapi` encoder support

Your ffmpeg build must support `av1_vaapi`. This is included in most recent builds but verify with:

```bash
ffmpeg -encoders | grep av1_vaapi
```

---

## Flow Variables

Two user variables must be configured in your Tdarr library before running this flow.

Go to **Libraries → [your library] → Options → Flow Variables** and add:

| Variable | Description | Example Value |
|---|---|---|
| `checkFileSize` | Set to `true` to skip the post-encode file size comparison. Set to `false` (default) to keep it enabled. | `false` |
| `testDirectory` | Directory to move output files to when running in test mode (`tempFile = true`). Only used if you enable test mode. | `/mnt/media/temp` |

If `testDirectory` is not set and test mode is triggered, the Move To Directory node will fail with an empty path.

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
1. File size check — skip if under 700MB
2. HDR type detection — identifies SDR, HDR10, HDR10+, or Dolby Vision
3. For Dolby Vision: checks profile and HDR10 fallback — skips unsupported profiles
4. AV1 codec check — if already AV1, checks container; if MKV, skips; if not MKV, remuxes
5. For non-AV1 files: encodes to AV1 via `av1_vaapi` at quality 24
   - HDR files: preserves BT.2020 color metadata (`p010le`, `smpte2084`, `bt2020nc`)
   - SDR files: encodes to `p010le`
6. Reorders audio/subtitle streams (English and French prioritized)
7. Post-encode file size comparison — keeps original if output is larger
8. Replaces original file or moves to test directory

---

## Quality and Encoding Settings

| Setting | Value | Notes |
|---|---|---|
| Encoder | `av1_vaapi` | Intel VAAPI hardware encoder |
| Quality | `24` | Global quality via `-global_quality`. Lower = better quality, larger file. |
| Preset | Disabled | `veryslow` is not a valid option for `av1_vaapi`. Quality is controlled solely via the quality value above. |
| HDR pixel format | `p010le` | Required for VAAPI 10-bit output |
| SDR pixel format | `p010le` | Encoded as 10-bit even for SDR sources |
| Audio | Copied | No re-encoding |
| Subtitles | Copied | No re-encoding |
| Stream order | `eng, fre` | English and French prioritized; edit the Reorder Streams node for other languages |

To adjust quality, edit the **Set Video Encoder** node after importing and change `ffmpegQuality` from `24` to your preferred value.

---

## Importing the Flow

1. In Tdarr, go to **Flows → Import Flow**
2. Paste the contents of `AV1_Flow.json`
3. Verify the two nodes below show your correct render device path:
   - **Custom Arguments HDR** → `inputArguments`
   - **Custom Arguments SDR** → `inputArguments`
4. Configure the flow variables in your library as described above
5. Run on a test file before applying to your library

---

## Known Limitations

- **Intel-only.** This flow uses `av1_vaapi` and will fail on nodes without Intel GPU VAAPI support. There is no AMD or NVIDIA fallback path — the flow will explicitly fail if `av1_vaapi` is not detected.
- **No QSV support.** Intel Battlemage (Arc B-series) does not support QSV for AV1. VAAPI is the correct path for this generation.
- **DV Profile 5 and 7 are skipped.** These profiles cannot be safely transcoded to AV1 without specialized tooling beyond this flow's scope. Use nichols89ben's flow to handle those files first if needed.

---

## Credits

HDR and Dolby Vision detection plugins by [nichols89ben](https://github.com/nichols89ben/Tdarr_DoVi_Processing), originally based on work by [andrasmaroy](https://github.com/andrasmaroy/Tdarr_Plugins_DoVi).
