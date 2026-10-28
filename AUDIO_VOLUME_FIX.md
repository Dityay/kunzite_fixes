# Volume Stuck Max di ROM Port (vendor Xiaomi `parrot`)

**Device:** `Redmi Note 15 5G` (codename `kunzite`) — menjalankan ROM port Motorola HelloUI yang memakai vendor Xiaomi Redmi Note (`ro.board.platform=parrot`).
**Target Android:** 17 · **Vendor OS:** Android 15 (HyperOS `OS2.0.203.0.VPQCNXM`)
**Tanggal:** 2026-10-10

---

## 1. Gejala

Di ROM port non-Xiaomi yang memakai vendor Xiaomi `parrot`:

- Slider volume **mentok** di angka tertinggi, tetapi suara tetap maksimal — tidak ada gradasi.
- Sound effect / audio policy **tidak termuat** dengan benar.
- Setiap kali tombol volume ditekan, log走出 error policy.

Bandingkan dengan ROM port yang memakai **vendor AOSP** (`aosp/vendor`): volume normal, bertingkat.

---

## 2. Akar Masalah

Empat file volume engine Xiaomi disuntikkan ke partisi **ODM**:

```
/odm/etc/audio_policy_engine_configuration_mi.xml        993 B
/odm/etc/audio_policy_engine_default_stream_volumes.xml  5406 B
/odm/etc/audio_policy_engine_product_strategies.xml      5205 B
/odm/etc/audio_policy_engine_stream_volumes_mi.xml       22557 B
```

Suffix `_mi` = **MIUI/HyperOS**. File-file ini bukan generic AOSP —它们 punya device category dan strategy yang dirancang untuk device Xiaomi tertentu, dan **tidak ada** di vendor Xiaomi asli (`aosp/odm/etc/` tidak punya satu pun file `audio_policy_engine_*`).

IndexerCertificate ini dipaksa aktif oleh property di vendor:

```
# vendor_a/build.prop:181
ro.vendor.audio.policy.engine.odm=true
```

### Bagaimana policy manager memilih file

Terd逆转 dari `AudioPolicyManagerImpl::loadAudioPolicyEngineConfigCustom()`
(`system_ext/lib64/libaudiopolicymanagerimpl.so`, fungsi mulai di offset `0x386bc`):

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

Logikanya:

```
sku = getprop("ro.boot.product.hardware.sku")

if sku == "mars"  →  stat("/odm/etc/audio_policy_engine_configuration_mi.xml")
else              →  stat("/vendor/etc/audio_policy_engine_configuration.xml")
```

Di device ini `ro.boot.product.hardware.sku = rgl`, jadi **seharusnya** membaca dari `/vendor/` — tapi property `policy.engine.odm=true` memaksa jalur ODM dan membuat table Xiaomi yang tidak cocok device tersebut dipakai.

### Gejala yang dihasilkan

Dump sebelum perbaikan:

```
- VOLUME GROUP AUDIO_STREAM_MUSIC:  Max: 15
- VOLUME GROUP AUDIO_STREAM_ALARM: Max:  0     ← tidak ada curve
- VOLUME GROUP AUDIO_STREAM_SYSTEM: Max:  0    ← tidak ada curve
- VOLUME GROUP AUDIO_STREAM_DTMF:  Max:  0     ← tidak ada curve
```

`Max: 0` = **tidak ada volume curve sama sekali** untuk stream tersebut → tidak ada attenuation → output selalu di gain penuh.slider jadi tidak terasa.

Error di log:

```
E APM::VolumeCurve: Invalid device category 1 for Volume Curve    ← 47× per menit
```

`device category 1` = `DEVICE_CATEGORY_HEADSET`. Setiap kali volume diubah, lookup curve gagal → tidak ada attenuation.

Selain itu, MIUI volume boost juga mati:

```
D DeviceHalAidl: parameters: "volume_boost=0"                        ← 148×, selalu 0
E AHAL_Telephony_QTI: configureVolumeBoost: invalid pal handle      ← 37×
E DeviceHalAidl: parseAndSetVendorParameters Line: 412 Failed      ← 111×
```

---

## 3. Perbandingan dengan Vendor yang Volume-nya Normal

Perbandingan `hos/vendor_a` vs `aosp/vendor`:

| Aspek | `hos/vendor_a` | `aosp/vendor` |
|---|---|---|
| Fingerprint | `Redmi/kunzite/kunzite:15/OS2.0.203.0.VPQCNXM` | `Redmi/kunzite_global/kunzite:15/OS3.0.301.0.WPQMIXM` |
| `default_volume_tables.xml` | 5133 B | 5133 B — **identik byte-for-byte** |
| `audio_policy_volumes.xml` | 13553 B, 71 entries | 14576 B, 71 entries — **20 entries beda** |
| `ro.vendor.audio.policy.engine.odm` | `true` | `true` (sama) |
| `ro.config.vc_call_vol_steps` | `11` | — (tidak ada) |
| File `audio_policy_engine_*` di ODM | 4 file (MD5 mtime `Jan 1 2009`) | **0 file** |

### Apa yang beda di `audio_policy_volumes.xml`

Vendor global memakai curve **eksplisit** untuk lebih banyak device category, dan limiter-nya lebih ketat di speaker:

| Stream / Category | `hos/vendor_a` | `aosp/vendor` (global) |
|---|---|---|
| MUSIC / HEADSET | `ref="DEFAULT_MEDIA_VOLUME_CURVE"` | eksplisit `1,-5750 → 100,0` |
| MUSIC / EARPIECE | `ref="DEFAULT_MEDIA_VOLUME_CURVE"` | eksplisit `1,-5550 … 100,-600` |
| MUSIC / EXT_MEDIA | `ref="DEFAULT_MEDIA_VOLUME_CURVE"` | eksplisit `1,-5800 … 100,0` |
| SYSTEM / SPEAKER | `ref="DEFAULT_SYSTEM_VOLUME_CURVE"` | eksplisit `1,-2600 … 100,-800` |
| SYSTEM / HEADSET | `1,-3000 … 100,-1800` | `1,-1800 … 100,-600` |
| NOTIFICATION / SPEAKER | `1,-2970 … 100,0` | `1,-4200 … 100,-800` |
| DTMF / SPEAKER | `ref="DEFAULT_SYSTEM_VOLUME_CURVE"` | eksplisit `1,-4200 … 100,-2800` |
| ACCESSIBILITY / SPEAKER | `ref="DEFAULT_NON_MUTABLE_SPEAKER_VOLUME_CURVE"` | eksplisit `0,-5250 … 100,0` |

Pola global: **speaker dibatasi** (SYSTEM/NOTIFICATION/DTMF tidak lagi `0` dB di max), sementara **earpiece/headset lebih terbuka**. Ini tuning "safe volume" yang lebih konservatif.

Poin penting: **`default_volume_tables.xml` identik**. Yang rusak bukan tabel reference-nya, melainkan file engine yang disuntikkan ke ODM.

---

## 4. Perbaikan

### 4.1 Hapus file Xiaomi dari ODM

```bash
# di device (root)
adb shell su -c "mkdir -p /data/local/tmp/bak_audio"
adb shell su -c "mv /odm/etc/audio_policy_engine_*.xml /data/local/tmp/bak_audio/"
adb reboot
```

Di repo MIO Kitchen:

```
hapus:
  hos/odm_a/etc/audio_policy_engine_configuration_mi.xml
  hos/odm_a/etc/audio_policy_engine_default_stream_volumes.xml
  hos/odm_a/etc/audio_policy_engine_product_strategies.xml
  hos/odm_a/etc/audio_policy_engine_stream_volumes_mi.xml
```

Entri di `config/odm_a_fs_config` dan `config/odm_a_file_contexts` **boleh dibiarkan** — `mkfs.erofs` mengabaikan entri untuk file yang tidak ada.

### 4.2 Matikan property pengarah ODM

`hos/vendor_a/build.prop` baris **181**:

```
# hapus / set false
ro.vendor.audio.policy.engine.odm=true
```

### 4.3 (Opsional) Samakan voice call steps

`hos/vendor_a/build.prop` baris **189**:

```
ro.config.vc_call_vol_steps=11
```

Property ini hanya ada di vendor CN, tidak di vendor global. Setelah perbaikan device melaporkan `VOICE_CALL Max: 15`, sedangkan default AOSP adalah 5. Hapus baris ini kalau call volume terasa tidak sesuai.

---

## 5. Verifikasi (Sebelum vs Sesudah)

### Volume group Max

| Stream | Sebelum | Sesudah |
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

### Curve MUSIC setelah perbaikan

```
DEVICE_CATEGORY_HEADSET    : { (1,-5750), (34,-3950), (66,-2000), (100,0) }
DEVICE_CATEGORY_SPEAKER    : { (1,-5800), (20,-4000), (60,-1700), (100,0) }
DEVICE_CATEGORY_EARPIECE   : { (1,-5550), (33,-3950), (66,-2300), (100,-600) }
DEVICE_CATEGORY_EXT_MEDIA  : { (1,-5800), (26,-4200), (66,-1800), (100,0) }
DEVICE_CATEGORY_HEARING_AID: { (1,-12700), (20,-8000), (60,-4000), (100,0) }
```

`HEADSET -5750` dan `EXT_MEDIA -5800` persis sama dengan isi `aosp/vendor/etc/audio_policy_volumes.xml` — jadi fallback ke vendor + AOSP default sudah benar.

### Error

```
E APM::VolumeCurve: Invalid device category 1 for Volume Curve
```

47× per menit → **0**.

---

## 6. Catatan Tambahan

### Jejak Xiaomi yang masih tertinggal

```
ro.vendor.audio.volume.boost.support=15     ← masih ada
D VolumeDialogBoostMode: getBoostLevelText error, boostLevel = 0
```

SystemUI port masih membawa kode MIUI yang menunggu volume boost, sementara boost-nya tidak pernah aktif. Ini hanya memengaruhi **label di dialog volume**, bukan level volume itu sendiri.和问题 tidak terkait.

### Service audio yang tidak ada di log boot

```
E init: service vendor.audio-hal not found
E init: service vendor.audio-effect-hal-aidl not found
E init: service vendor.audio-hal-4-0-msd not found
E init: service audio_proxy_service not found
```

Keempat service audio proprietary Xiaomi tidak ada di ROM port. Audio tetap berjalan karena AIDL HAL替代 mengambil alih. Tidak terkait dengan masalah volume.

### SKU audio

```
ro.boot.product.vendor.sku   = parrot
ro.boot.product.hardware.sku = rgl
ro.vendor.qti.va_aosp.support = 1     → suffix "_qssi"
```

SKU audio **termuat dengan benar** — `/vendor/etc/audio/sku_parrot/` berisi lengkap 13 file (mixer_paths, resourcemanager, audio_effects). Ini bukan penyebab masalah volume; sempat jadi suspect karena di port non-Xiaomi sering kosong.

---

## 7. Perintah Diagnostik

```bash
# Status volume group
adb shell dumpsys audio | grep -A 5 "VOLUME GROUP" | grep -E "VOLUME GROUP|Max:" | paste - - | sed 's/- VOLUME GROUP //'

# Curve per stream
adb shell dumpsys media.audio_policy | grep -A 20 "AUDIO_STREAM_MUSIC (id:"

# Error policy
adb logcat -c; adb shell input keyevent 24; sleep 3
adb logcat -d | grep -E "VolumeCurve|volume_boost|AHAL_Telephony_QTI"

# Konfirmasi file ODM sudah bersih
adb shell ls /odm/etc/ | grep -i audio
adb shell getprop ro.vendor.audio.policy.engine.odm

# Dump lengkap
adb shell dumpsys audio
adb shell dumpsys media.audio_policy
```

---

## 8. Referensi Offset Binary

### `system_ext/lib64/libaudiopolicymanagerimpl.so`

Fungsi `AudioPolicyManagerImpl::loadAudioPolicyEngineConfigCustom()` di offset `0x386bc`:

```
0x386bc  awal fungsi
0x38700  x9 → "/odm/etc/audio_policy_engine_configuration_mi.xml"  (rodata 0x1cbf4)
0x38718  x0 → "ro.boot.product.hardware.sku"                        (rodata 0x190d2)
0x38760  bl property_get
0x38768  w9 = 0x7372616d ("mars")
0x3877c  cmp w8, w9        ; sku == "mars" ?
0x387a0  bl stat           ; jalur ODM
0x387e4  bl stat           ; jalur vendor
```

### `vendor/lib64/hw/libaudiocorehal.default.so`

Fungsi build path config audio, offset `0xad90`:

```
0xadb0  x0 → "ro.boot.product.vendor.sku"
0xaddc  bl property_get
0xade0  cmp w0, #0
0xade4  b.le ...           ; nilai kosong → skip seluruh blok
0xae00  "/odm/etc"
0xae14  "/vendor/etc/audio/sku_"
0xae48  "_qssi"
0xaec0  "/vendor/etc/audio"
0xaee0  "/vendor/etc"
0xaef0  "/system/etc"
```

Branch A (SKU ada): `/odm/etc<sku>[_qssi]`, `/vendor/etc/audio/sku_<sku>`, `/vendor/etc/audio`, `/system/etc`
Branch B (SKU kosong): `/odm/etc`, `/vendor/etc/audio`, `/vendor/etc`, `/system/etc`

---

## 9. Kesimpulan

Penyebabnya **bukan** vendor, **bukan** SKU, dan **bukan** mixer path. Penyebabnya adalah empat file
`audio_policy_engine_*.xml` Xiaomi HyperOS yang disuntikkan ke ODM, dikunci aktif oleh
`ro.vendor.audio.policy.engine.odm=true`.

File Xiaomi itu tidak cocok dengan device port → 7 volume group dapat `Max: 0` (tanpa curve) → tidak ada
attenuation → volume mentok maksimal, plus 47 error `Invalid device category` per menit.

Menghapus file dari ODM dan mematikan property-nya mengembalikan volume ke curve AOSP default yang
lengkap dan konsisten.