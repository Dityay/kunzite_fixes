# kunzite_fixes

Kumpulan catatan perbaikan (fix) untuk **Redmi Note 15 5G (codename `kunzite`)** yang menjalankan ROM port Motorola HelloUI (Android 17) memakai vendor Xiaomi `parrot` (SM6450).

## Isi

| File | Masalah | Ringkasan Fix |
|---|---|---|
| [`AUDIO_VOIP_FIX.md`](AUDIO_VOIP_FIX.md) | Panggilan VoIP (WhatsApp/Telegram/Discord/Meet) langsung terputus 3–6 detik, mic tidak mengirim suara. `pal_stream_start failed ret:-131` di `VOIP_RECORD`. | 3-layer fix: disable hardware `voip_record` di `audio_module_config_primary.xml`, matikan Fluence voicecomm & dynamic ECNS di `build.prop`, normalisasi `resourcemanager_parrot_qrd.xml` (VOIP_TX + EC ref). |
| [`AUDIO_VOLUME_FIX.md`](AUDIO_VOLUME_FIX.md) | Volume stuck maksimal, 7 volume group `Max: 0` (tanpa curve), error `Invalid device category 1` berulang. | Hapus 4 file `audio_policy_engine_*.xml` Xiaomi HyperOS dari `/odm/etc/`, matikan `ro.vendor.audio.policy.engine.odm`, opsional hapus `ro.config.vc_call_vol_steps`. |

## Device Info

- **Device:** Redmi Note 15 5G (`kunzite`)
- **Platform:** Qualcomm SM6450 (`parrot`)
- **Vendor base:** Xiaomi HyperOS Android 15 (`OS2.0.203.0.VPQCNXM`)
- **ROM port:** Motorola HelloUI, Android 17

## Format Dokumen

Setiap dokumen memuat: gejala, logcat signature, analisis akar masalah (termasuk offset binary relevan), langkah perbaikan, verifikasi sebelum/sesudah, dan perintah diagnostik `adb`.
