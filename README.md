# kunzite_fixes

A collection of fix guides for the **Xiaomi Redmi Note 15 5G (codename `kunzite`)** running a non-Xiaomi ROM port (Motorola HelloUI, Android 17) on an Xiaomi `parrot` vendor (SM6450).

> **Applies to any non-Xiaomi ROM port that uses an Xiaomi vendor.**
> Both fixes are **not** device- or ROM-specific — they work on any AOSP-style port (AOSP, Pixel, Motorola, etc.) built on top of a Xiaomi HyperOS/MIUI vendor for the `parrot` platform. The root causes live entirely inside the Xiaomi vendor/ODM audio config, so as long as your port uses a Xiaomi `parrot` vendor, the symptoms and the fixes are the same.

## Contents

| File | Problem | Fix Summary |
|---|---|---|
| [`AUDIO_VOIP_FIX.md`](AUDIO_VOIP_FIX.md) | VoIP calls (WhatsApp/Telegram/Discord/Meet) disconnect after 3–6 s, mic sends nothing. `pal_stream_start failed ret:-131` on `VOIP_RECORD`. | 3-layer fix: disable hardware `voip_record` in `audio_module_config_primary.xml`, turn off Fluence voicecomm & dynamic ECNS in `build.prop`, normalize `resourcemanager_parrot_qrd.xml` (VOIP_TX + EC ref). |
| [`AUDIO_VOLUME_FIX.md`](AUDIO_VOLUME_FIX.md) | Volume stuck at max, 7 volume groups report `Max: 0` (no curves), repeated `Invalid device category 1` errors. | Delete the 4 Xiaomi HyperOS `audio_policy_engine_*.xml` files from `/odm/etc/`, disable `ro.vendor.audio.policy.engine.odm`, copy `audio_policy_volumes.xml` from the AOSP vendor. |

## Device Info

- **Device:** Xiaomi Redmi Note 15 5G (`kunzite`)
- **Platform:** Qualcomm SM6450 (`parrot`)
- **Vendor base:** Xiaomi HyperOS Android 15 (`OS2.0.203.0.VPQCNXM`)
- **ROM port:** Motorola HelloUI, Android 17

## Document Format

Each guide covers: symptoms, logcat signatures, root-cause analysis (including relevant binary offsets), step-by-step fixes, before/after verification, and `adb` diagnostic commands.
