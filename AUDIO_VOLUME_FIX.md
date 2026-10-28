# Volume Stuck at Max on ROM Port (Xiaomi `parrot` Vendor)

**Device:** `Redmi Note 15 5G` (codename `kunzite`) — running a Motorola HelloUI ROM port that uses an Xiaomi Redmi vendor (`ro.board.platform=parrot`).
**Target Android:** 17 · **Vendor OS:** Android 15 (HyperOS `OS2.0.203.0.VPQCNXM`)
**Date:** 2026-10-10

---

## 1. Symptoms

On non-Xiaomi ROM ports that use an Xiaomi `parrot` vendor:

- The volume slider is **stuck** at the top step — the sound is always at full volume, with no gradation.
- Sound effects / audio policy do not load correctly.
- Every time a volume key is pressed, policy errors appear in the log.

Compare this with ROM ports that use an **AOSP vendor** (`aosp/vendor`): volume behaves normally, with smooth steps.

---

## 2. Root Cause

Four Xiaomi volume engine files are injected into the **ODM** partition:

```
/odm/etc/audio_policy_engine_configuration_mi.xml        993 B
/odm/etc/audio_policy_engine_default_stream_volumes.xml  5406 B
/odm/etc/audio_policy_engine_product_strategies.xml      5205 B
/odm/etc/audio_policy_engine_stream_volumes_mi.xml       22557 B
```

The `_mi` suffix = **MIUI/HyperOS**. These files are not generic AOSP — they carry device categories and strategies designed for specific Xiaomi devices, and they do **not** exist in the original Xiaomi vendor (`aosp/odm/etc/` has no `audio_policy_engine_*` files at all).

They are force-activated by a property in the vendor:

```
# vendor_a/build.prop:181
ro.vendor.audio.policy.engine.odm=true
```

### How the policy manager picks the file

The reverse-engineered flow of `AudioPolicyManagerImpl::loadAudioPolicyEngineConfigCustom()`
(`system_ext/lib64/libaudiopolicymanagerimpl.so`, function starting at offset `0x386bc`):

```asm
; x0 = "ro.boot.product.hardware.sku"
38760: bl   property_get
38768: mov  w9, #0x616d
38770: movk w9, #0x7372, lsl #16      ; w9 = 0x7372616d = "mars"
3877c: cmp  w8, w9
38780: cset w8, ne                     ; sku == "mars" ?
38784: cbz  w10, ...
387a0: bl   stat(...)                  ; "/odm/etc/audio_policy_engine_configuration_mi.xml"
387e4: bl   stat(...)                  ; "/vendor/etc/audio_policy_engine_configuration.xml"
```

The logic:

```
sku = getprop("ro.boot.product.hardware.sku")

if sku == "mars"  →  stat("/odm/etc/audio_policy_engine_configuration_mi.xml")
else              →  stat("/vendor/etc/audio_policy_engine_configuration.xml")
```

On this device `ro.boot.product.hardware.sku = rgl`, so it **should** read from `/vendor/` — but the `policy.engine.odm=true` property forces the ODM path and makes the mismatched Xiaomi tables get used on this device.

### Resulting symptoms

Dump before the fix:

```
- VOLUME GROUP AUDIO_STREAM_MUSIC:  Max: 15
- VOLUME GROUP AUDIO_STREAM_ALARM: Max:  0     ← no curve
- VOLUME GROUP AUDIO_STREAM_SYSTEM: Max:  0    ← no curve
- VOLUME GROUP AUDIO_STREAM_DTMF:  Max:  0     ← no curve
```

`Max: 0` = **no volume curve at all** for that stream → no attenuation → output is always at full gain, so the slider feels meaningless.

Error in the log:

```
E APM::VolumeCurve: Invalid device category 1 for Volume Curve    ← 47× per minute
```

`device category 1` = `DEVICE_CATEGORY_HEADSET`. Every volume change fails the curve lookup → no attenuation.

On top of that, the MIUI volume boost never activates:

```
D DeviceHalAidl: parameters: "volume_boost=0"                        ← 148×, always 0
E AHAL_Telephony_QTI: configureVolumeBoost: invalid pal handle      ← 37×
E DeviceHalAidl: parseAndSetVendorParameters Line: 412 Failed      ← 111×
```

---

## 3. Comparison With the Vendor Whose Volume Works

Comparison of `hos/vendor_a` vs `aosp/vendor`:

| Aspect | `hos/vendor_a` | `aosp/vendor` |
|---|---|---|
| Fingerprint | `Redmi/kunzite/kunzite:15/OS2.0.203.0.VPQCNXM` | `Redmi/kunzite_global/kunzite:15/OS3.0.301.0.WPQMIXM` |
| `default_volume_tables.xml` | 5133 B | 5133 B — **identical byte-for-byte** |
| `audio_policy_volumes.xml` | 13553 B, 71 entries | 14576 B, 71 entries — **20 entries differ** |
| `ro.vendor.audio.policy.engine.odm` | `true` | `true` (same) |
| `ro.config.vc_call_vol_steps` | `11` | — (not present) |
| `audio_policy_engine_*` files in ODM | 4 files (MD5 mtime `Jan 1 2009`) | **0 files** |

### What differs in `audio_policy_volumes.xml`

The global vendor uses **explicit** curves for more device categories, and its limiter is tighter on the speaker:

| Stream / Category | `hos/vendor_a` | `aosp/vendor` (global) |
|---|---|---|
| MUSIC / HEADSET | `ref="DEFAULT_MEDIA_VOLUME_CURVE"` | explicit `1,-5750 → 100,0` |
| MUSIC / EARPIECE | `ref="DEFAULT_MEDIA_VOLUME_CURVE"` | explicit `1,-5550 … 100,-600` |
| MUSIC / EXT_MEDIA | `ref="DEFAULT_MEDIA_VOLUME_CURVE"` | explicit `1,-5800 … 100,0` |
| SYSTEM / SPEAKER | `ref="DEFAULT_SYSTEM_VOLUME_CURVE"` | explicit `1,-2600 … 100,-800` |
| SYSTEM / HEADSET | `1,-3000 … 100,-1800` | `1,-1800 … 100,-600` |
| NOTIFICATION / SPEAKER | `1,-2970 … 100,0` | `1,-4200 … 100,-800` |
| DTMF / SPEAKER | `ref="DEFAULT_SYSTEM_VOLUME_CURVE"` | explicit `1,-4200 … 100,-2800` |
| ACCESSIBILITY / SPEAKER | `ref="DEFAULT_NON_MUTABLE_SPEAKER_VOLUME_CURVE"` | explicit `0,-5250 … 100,0` |

Global pattern: **the speaker is limited** (SYSTEM/NOTIFICATION/DTMF no longer hit `0` dB at max), while **earpiece/headset are more open**. This is a more conservative "safe volume" tuning.

Key point: **`default_volume_tables.xml` is identical**. The broken part is not the reference tables — it's the engine files injected into ODM.

---

## 4. Fixes

### 4.1 Delete the Xiaomi files from ODM

```bash
# on device (root)
adb shell su -c "mkdir -p /data/local/tmp/bak_audio"
adb shell su -c "mv /odm/etc/audio_policy_engine_*.xml /data/local/tmp/bak_audio/"
adb reboot
```

In the MIO Kitchen repo:

```
delete:
  hos/odm_a/etc/audio_policy_engine_configuration_mi.xml
  hos/odm_a/etc/audio_policy_engine_default_stream_volumes.xml
  hos/odm_a/etc/audio_policy_engine_product_strategies.xml
  hos/odm_a/etc/audio_policy_engine_stream_volumes_mi.xml
```

The entries in `config/odm_a_fs_config` and `config/odm_a_file_contexts` **can be left as-is** — `mkfs.erofs` ignores entries for files that don't exist.

### 4.2 Disable the ODM redirect property

`hos/vendor_a/build.prop` line **181**:

```
# delete the line / set to false
ro.vendor.audio.policy.engine.odm=true
```

### 4.3 Copy `audio_policy_volumes.xml` from the AOSP vendor (recommended)

The volume curves on the Xiaomi CN vendor usually use `ref="DEFAULT_MEDIA_VOLUME_CURVE"`, whose reference curves are thin, while the global/AOSP vendor has **explicit**, more conservative curves for the speaker (see the comparison in section 3). Copy the file from the AOSP vendor to the port's vendor:

```bash
# in the MIO Kitchen repo
cp aosp/vendor/etc/audio_policy_volumes.xml hos/vendor_a/etc/audio_policy_volumes.xml
```

Or directly on the device (root) — the active vendor path is `/vendor/etc/audio_policy_volumes.xml`:

```bash
adb push audio_policy_volumes.xml /data/local/tmp/
adb shell su -c "mount -o remount,rw /vendor"
adb shell su -c "cp /data/local/tmp/audio_policy_volumes.xml /vendor/etc/audio_policy_volumes.xml"
adb reboot
```

Notes:
- `default_volume_tables.xml` does **not** need to be touched — that file is identical byte-for-byte between both vendors.
- After this step, the MUSIC/HEADSET curve will exactly match AOSP global (`1,-5750 → 100,0`), and speaker curves for SYSTEM/NOTIFICATION/DTMF will no longer be `0` dB at max volume.
- Alternative without touching the vendor: copy the same file to **`/odm/etc/`** — ODM engine config takes higher priority and overrides the vendor file.

### 4.4 (Optional) Match the voice call steps

`hos/vendor_a/build.prop` line **189**:

```
ro.config.vc_call_vol_steps=11
```

This property only exists on the CN vendor, not the global one. After the fix the device reports `VOICE_CALL Max: 15`, while the AOSP default is 5. Delete this line if the call volume feels off.

---

## 5. Verification (Before vs After)

### Volume group Max

| Stream | Before | After |
|---|---|---|
| MUSIC | 15 | 15 |
| VOICE_CALL | 15 | 15 |
| RING | 7 | 7 |
| NOTIFICATION | 7 | 7 |
| **ALARM** | **0** | **7** ✓ |
| **SYSTEM** | **0** | **7** ✓ |
| **DTMF** | **0** | **15** ✓ |
| **ACCESSIBILITY** | **0** | **15** ✓ |
| **ASSISTANT** | **0** | **15** ✓ |
| **TTS** | **0** | **15** ✓ |
| **ENFORCED_AUDIBLE** | **0** | **7** ✓ |

### MUSIC curve after the fix

```
DEVICE_CATEGORY_HEADSET    : { (1,-5750), (34,-3950), (66,-2000), (100,0) }
DEVICE_CATEGORY_SPEAKER    : { (1,-5800), (20,-4000), (60,-1700), (100,0) }
DEVICE_CATEGORY_EARPIECE   : { (1,-5550), (33,-3950), (66,-2300), (100,-600) }
DEVICE_CATEGORY_EXT_MEDIA  : { (1,-5800), (26,-4200), (66,-1800), (100,0) }
DEVICE_CATEGORY_HEARING_AID: { (1,-12700), (20,-8000), (60,-4000), (100,0) }
```

`HEADSET -5750` and `EXT_MEDIA -5800` are exactly the same as in `aosp/vendor/etc/audio_policy_volumes.xml` — so the vendor + AOSP default fallback is correct.

### Error

```
E APM::VolumeCurve: Invalid device category 1 for Volume Curve
```

47× per minute → **0**.

---

## 6. Additional Notes

### Xiaomi traces still remaining

```
ro.vendor.audio.volume.boost.support=15     ← still present
D VolumeDialogBoostMode: getBoostLevelText error, boostLevel = 0
```

The ported SystemUI still carries MIUI code waiting for a volume boost, while the boost never activates. This only affects the **labels in the volume dialog**, not the actual volume level. Unrelated to the problem.

### Audio services missing from the boot log

```
E init: service vendor.audio-hal not found
E init: service vendor.audio-effect-hal-aidl not found
E init: service vendor.audio-hal-4-0-msd not found
E init: service audio_proxy_service not found
```

These four proprietary Xiaomi audio services don't exist in the ROM port. Audio still works because the AIDL HAL replacement takes over. Not related to the volume problem.

### Audio SKU

```
ro.boot.product.vendor.sku   = parrot
ro.boot.product.hardware.sku = rgl
ro.vendor.qti.va_aosp.support = 1     → suffix "_qssi"
```

The audio SKU **loads correctly** — `/vendor/etc/audio/sku_parrot/` contains the complete 13 files (mixer_paths, resourcemanager, audio_effects). This is not the cause of the volume problem; it was briefly a suspect because on non-Xiaomi ports this directory is often empty.

---

## 7. Diagnostic Commands

```bash
# Volume group status
adb shell dumpsys audio | grep -A 5 "VOLUME GROUP" | grep -E "VOLUME GROUP|Max:" | paste - - | sed 's/- VOLUME GROUP //'

# Per-stream curves
adb shell dumpsys media.audio_policy | grep -A 20 "AUDIO_STREAM_MUSIC (id:)"

# Policy errors
adb logcat -c; adb shell input keyevent 24; sleep 3
adb logcat -d | grep -E "VolumeCurve|volume_boost|AHAL_Telephony_QTI"

# Confirm the ODM files are gone
adb shell ls /odm/etc/ | grep -i audio
adb shell getprop ro.vendor.audio.policy.engine.odm

# Full dumps
adb shell dumpsys audio
adb shell dumpsys media.audio_policy
```

---

## 8. Binary Offset Reference

### `system_ext/lib64/libaudiopolicymanagerimpl.so`

The `AudioPolicyManagerImpl::loadAudioPolicyEngineConfigCustom()` function at offset `0x386bc`:

```
0x386bc  function start
0x38700  x9 → "/odm/etc/audio_policy_engine_configuration_mi.xml"  (rodata 0x1cbf4)
0x38718  x0 → "ro.boot.product.hardware.sku"                        (rodata 0x190d2)
0x38760  bl property_get
0x38768  w9 = 0x7372616d ("mars")
0x3877c  cmp w8, w9        ; sku == "mars" ?
0x387a0  bl stat           ; ODM path
0x387e4  bl stat           ; vendor path
```

### `vendor/lib64/hw/libaudiocorehal.default.so`

The audio config path-building function at offset `0xad90`:

```
0xadb0  x0 → "ro.boot.product.vendor.sku"
0xaddc  bl property_get
0xade0  cmp w0, #0
0xade4  b.le ...           ; empty value → skip the whole block
0xae00  "/odm/etc"
0xae14  "/vendor/etc/audio/sku_"
0xae48  "_qssi"
0xaec0  "/vendor/etc/audio"
0xaee0  "/vendor/etc"
0xaef0  "/system/etc"
```

Branch A (SKU present): `/odm/etc<sku>[_qssi]`, `/vendor/etc/audio/sku_<sku>`, `/vendor/etc/audio`, `/system/etc`
Branch B (SKU empty): `/odm/etc`, `/vendor/etc/audio`, `/vendor/etc`, `/system/etc`

---

## 9. Conclusion

The cause is **not** the vendor, **not** the SKU, and **not** the mixer paths. The cause is the four Xiaomi HyperOS `audio_policy_engine_*.xml` files injected into ODM, locked in by `ro.vendor.audio.policy.engine.odm=true`.

Those Xiaomi files don't match the ported device → 7 volume groups get `Max: 0` (no curves) → no attenuation → volume is stuck at maximum, plus 47 `Invalid device category` errors per minute.

Deleting the files from ODM and disabling the property restores the volume to the complete, consistent AOSP default curves.
