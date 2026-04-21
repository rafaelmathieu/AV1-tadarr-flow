# Tdarr AV1 VAAPI Flow

<img width="836" height="845" alt="image" src="https://github.com/user-attachments/assets/06dc64fe-61c1-4770-87c1-ee18d876cdac" />


# Tdarr AV1 VAAPI Flow

A Tdarr flow that transcodes video to AV1 using Intel GPU hardware encoding via VAAPI. Handles HDR10, HDR10+, and Dolby Vision (Profiles 4, 5, 7, and 8) correctly, skipping files that should not be processed.

> **Tested on:** Intel Arc Pro B70 (Battlemage BMG-G31, 32GB VRAM) running on Unraid with the `xe` driver.

---

![Flow Diagram](FLOW.png)

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

Install them before importing this flow — the HDR and Dolby Vision detection nodes will fail without them. Follow the installation instructions in that repo. The plugins go into your local FlowPlugins directory, **not** the community folder.

### 3. `av1_vaapi` encoder support

Your ffmpeg build must support `av1_vaapi`. Verify with:

```bash
ffmpeg -encoders | grep av1_vaapi
```

---

## Configuration After Import

### Step 1 — Set your render device

After importing the flow, find the **"Set The Device ID"** comment node. Immediately below it is the **"Custom Arguments DeviceID"** node. Edit its `inputArguments` field and replace `/dev/dri/renderD128` with your actual render device path if it differs.

To find your render device:

```bash
ls /dev/dri/
# Look for renderD128, renderD129, etc.
```

### Step 2 — Set library variables

Go to **Libraries → [your library] → Options → Library Variables** and add the following:

| Variable | Description | Example Value |
|---|---|---|
| `av1Quality` | AV1 encoding quality. Lower = better quality, larger file. | `24` |
| `isTest` | Set to `true` to move output to a temp directory instead of replacing the original. Set to `false` for normal operation. | `false` |
| `checkFileSize` | Set to `true` to skip the post-encode file size comparison. Set to `false` to discard the encode if the output is larger than the input. | `false` |

> **Note on `isTest`:** When `isTest` is `true`, the flow moves the output to `/mnt/media/temp`. This path is currently hardcoded in the **Move To Directory** node — edit it to match your own temp directory after importing.

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
5. **Hardware check** — verifies `av1_vaapi` is available on the node; fails the flow if not
6. **Device ID** — applies your configured render device to the ffmpeg command
7. **HDR type check** — routes HDR and SDR files to separate argument sets
   - HDR: preserves BT.2020 color metadata (`p010le`, `smpte2084`, `bt2020nc`)
   - SDR: encodes to `p010le`
8. **Stream reorder** — prioritizes English and French audio/subtitle tracks
9. **Execute** — runs the ffmpeg command
10. **Test mode check** — if `isTest` is `true`, moves output to temp directory; otherwise proceeds to size comparison
11. **Post-encode size check** — if `checkFileSize` is `false`, keeps the original if the output is larger
12. **Output** — replaces the original file

---

## Encoding Settings

| Setting | Value | Notes |
|---|---|---|
| Encoder | `av1_vaapi` | Intel VAAPI hardware encoder |
| Quality | Configurable via `av1Quality` library variable | Passed as `-global_quality`. Lower = better quality. |
| Preset | Disabled | `veryslow` is not a valid preset for `av1_vaapi`. Quality is controlled solely via the quality value above. |
| HDR pixel format | `p010le` | Required for VAAPI 10-bit output |
| SDR pixel format | `p010le` | Encoded as 10-bit even for SDR sources |
| Audio | Copied | No re-encoding |
| Subtitles | Copied | No re-encoding |
| Stream language priority | `eng, fre` | Edit the **Reorder Streams** node directly to change language priority |

---

## Known Limitations

- **Intel-only.** This flow uses `av1_vaapi` and will explicitly fail on nodes without Intel GPU VAAPI support. There is no AMD or NVIDIA fallback path.
- **No QSV support.** Intel Battlemage (Arc B-series) does not support QSV for AV1. VAAPI is the correct path for this generation.
- **DV Profile 5 and 7 are skipped.** These profiles cannot be safely transcoded to AV1 without specialized tooling beyond this flow's scope. Use nichols89ben's flow to handle those files separately if needed.

---

## Credits

HDR and Dolby Vision detection plugins by [nichols89ben](https://github.com/nichols89ben/Tdarr_DoVi_Processing), originally based on work by [andrasmaroy](https://github.com/andrasmaroy/Tdarr_Plugins_DoVi).
