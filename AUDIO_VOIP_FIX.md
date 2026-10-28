# Fix VoIP Call Mic Crash / Call Disconnected di ROM Port (Vendor Xiaomi `parrot`)

**Device:** `moto g57` (codename `kunzite`) — Motorola HelloUI (Android 17) port yang memakai vendor Xiaomi Redmi Note (`ro.board.platform=parrot`, SM6450).  
**Target Android:** 17 · **Vendor Base:** Android 15/16/17 QTI Audio HAL AIDL  
**Tanggal:** 2026-10-18  

---

## 1. Gejala

Pada ROM port non-Xiaomi (seperti Moto HelloUI / AOSP / Pixel) yang memakai vendor Xiaomi platform `parrot`:

- Panggilan VoIP di **WhatsApp, WhatsApp Business, Telegram, Discord, atau Google Meet** langsung terputus (*call ended*) dalam 3–6 detik setelah tersambung.
- Mic tidak mengirim suara sama sekali ke lawan bicara sebelum panggilan putus.
- Panggilan biasa (seluler / telephony GSM) dan voice recorder normal, hanya panggilan berbasis aplikasi VoIP (`AUDIO_SOURCE_VOICE_COMMUNICATION`) yang gagal.

### Logcat Signature
```text
AHAL_StreamIn_QTI: configure : usecase: VOIP_RECORD IoHandle:70 pal_stream_start failed!! ret:-131
AHAL_StreamIn_QTI: transfer : usecase: VOIP_RECORD IoHandle:70: failed to configure
AHAL_StreamIn_QTI: shutdown_I : usecase: VOIP_RECORD IoHandle:70
AHAL_StreamIn_QTI: configure : usecase: VOIP_RECORD IoHandle:70 set pal_stream_set_buffer_size to 640 with count 4
AHAL_StreamIn_QTI: configure : usecase: VOIP_RECORD IoHandle:70 pal_stream_start failed!! ret:-131
... (looping ~25 kali per detik selama 6 detik) ...
Telecom : CallsManager: markCallAsDisconnected: call=[...], voip=true; disconnectCause=DisconnectCause [ Code: (LOCAL) ... TelephonyCause: 36/65535 ]
wm_finish_activity: [...,com.whatsapp.w4b/com.whatsapp.calling.ui.VoipActivityV2,app-request,0]
```

---

## 2. Akar Masalah (Root Cause Analysis)

### 2.1 Makna Return Code `ret: -131`
- Nilai `-131` adalah `-ENOTRECOVERABLE` (Linux `errno.h`: `131 ENOTRECOVERABLE /* State not recoverable */`).
- Di Qualcomm Audio Graph Manager (`vendor/lib64/libagm.so`), tabel konversi error AR_OSAL pada offset `0x2c000` memetakan status SPF/ADSP `AR_EUNEXPECTED` (`0x5`) dan `AR_EVERSION` (`0x4`) secara langsung ke nilai POSIX `-131`.
- Ini menandakan DSP Audio Qualcomm (ADSP) menolak inisialisasi graph audio capture saat `pal_stream_start()` dieksekusi.

### 2.2 Tiga Titik Kegagalan di Vendor Xiaomi `parrot`

1. **Mismatch Channel Mask & Device Profile:**
   - Framework Android membuka `voip_record` lewat `audio_module_config_primary.xml` sebagai **MONO**:
     `channelLayouts="LAYOUT_MONO"` -> buffer 640 bytes (320 frame @ 16kHz 16-bit mono).
   - Di `vendor/etc/audio/sku_parrot/resourcemanager_parrot_qrd.xml`, usecase `PAL_STREAM_VOIP_TX` di bawah `PAL_DEVICE_IN_HANDSET_MIC` dan `PAL_DEVICE_IN_SPEAKER_MIC` dipaksa 2 channel dan dual-mic hardware Xiaomi:
     ```xml
     <usecase>
         <name>PAL_STREAM_VOIP_TX</name>
         <priority>2</priority>
         <bit_width>16</bit_width>
         <channels>2</channels>
         <snd_device_name>handset-dmic-endfire</snd_device_name>
     </usecase>
     ```
     Konfigurasi mismatch ini menyebabkan driver DSP menolak frame mapping.

2. **Gagal Binding Echo Cancellation Reference (EC Ref):**
   - Stream `PAL_STREAM_VOIP_TX` di DSP Qualcomm mewajibkan adanya referensi echo cancellation (EC Ref) dari stream speaker/earpiece downlink (`VOIP_RX`).
   - Aplikasi VoIP (seperti WhatsApp) sering kali membuka capture mic sebelum stream audio output siap.
   - Pada file stock `resourcemanager_parrot_qrd.xml`, baris proteksi low-latency sengaja di-comment oleh Xiaomi:
     ```xml
     <!--disabled_stream>PAL_STREAM_LOW_LATENCY</disabled_stream-->
     ```
     Akibatnya, saat `VOIP_RX` belum aktif, PAL mencoba meminjam stream sistem `LOW_LATENCY` (misal suara touch UI) sebagai referensi EC. Graph ADSP gagal mengaitkan dua stream yang berbeda format ini, menghasilkan status `AR_EUNEXPECTED` (`-131`).

3. **Inisialisasi Fluence Hardware Offload:**
   - Property `persist.vendor.audio.fluence.voicecomm=true` dan `vendor.audio.feature.dynamic_ecns.enable=true` memaksa HAL memanggil modul Fluence HW di ADSP yang bergantung pada kalibrasi ACDB OEM Xiaomi. Di ROM port, dependensi ini broken.

---

## 3. Solusi Terintegrasi (3-Layer Fix)

Untuk menjamin keandalan 100% (*failsafe & defense-in-depth*), diterapkan 3 layer perbaikan sekaligus:

### Layer 1: Bypass Hardware `VOIP_TX` via Audio Policy
File: `vendor/etc/audio/audio_module_config_primary.xml`

Nonaktifkan `mixPort` dan `route` untuk `voip_record`:
```xml
        <!-- Matikan hardware voip_record agar fallback otomatis ke primary_in -->
        <!--
        <mixPort name="voip_record" role="sink" flags="VOIP_TX">
            <profile samplingRates="8000 16000 32000 48000" channelLayouts="LAYOUT_MONO"
                formatType="PCM" pcmType="INT_16_BIT" />
        </mixPort>
        -->
```
Dan di bagian `<routes>`:
```xml
        <!--
        <route type="mix" sink="voip_record" sources="built_in_bottom_mic,built_in_top_mic,built_in_back_mic,wired_headset_mic,bt_sco_headset_mic,usb_mic,usb_headset_mic,bt_a2dp_mic,echo_reference_mic,bt_le_mic" />
        -->
```
**Efek:** AudioPolicyManager Android akan merouting panggilan WhatsApp ke `primary_in`. AudioFlinger secara otomatis mengaktifkan software AEC/NS WebRTC (`libaudiopreprocessing.so`), memberikan noise suppression dan echo cancellation yang stabil tanpa menyentuh graph DSP bermasalah.

---

### Layer 2: Matikan Fluence VoiceComm & Dynamic ECNS
File: `vendor/build.prop`

Setel property berikut:
```properties
persist.vendor.audio.fluence.speaker=false
persist.vendor.audio.fluence.voicecomm=false
vendor.audio.feature.dynamic_ecns.enable=false
```
**Efek:** Mencegah HAL audio dan DSP mencoba membangun graph Fluence hardware yang rusak saat aplikasi meminta parameter komunikasi.

---

### Layer 3: Normalisasi Resource Manager PAL (Jaring Pengaman DSP)
File: `vendor/etc/audio/sku_parrot/resourcemanager_parrot_qrd.xml`

1. Pada `PAL_DEVICE_IN_HANDSET_MIC` dan `PAL_DEVICE_IN_SPEAKER_MIC`, hilangkan force 2-channel & dual-mic endfire pada `PAL_STREAM_VOIP_TX`:
```xml
            <usecase>
                <name>PAL_STREAM_VOIP_TX</name>
                <priority>2</priority>
            </usecase>
```

2. Pada bagian `<in_streams>` untuk `PAL_STREAM_VOIP_TX`, uncomment proteksi low latency:
```xml
        <in_stream>
            <name>PAL_STREAM_VOIP_TX</name>
                <policies>
                    <ec_ref>
                        <disabled_stream>PAL_STREAM_LOW_LATENCY</disabled_stream>
                        <disabled_stream>PAL_STREAM_GENERIC</disabled_stream>
                    </ec_ref>
                </policies>
        </in_stream>
```
**Efek:** Jika ada sub-layanan yang masih mencoba membuka stream `PAL_STREAM_VOIP_TX`, PAL tidak akan crash karena profil channel dan kebijakan EC reference sudah bersih.

---

## 4. Opsi Alternatif: Non-Vendor (Jika Vendor Dipertahankan Intact)

Jika dalam aturan porting partisi `/vendor` tidak boleh disentuh sama sekali:
1. Salin `audio_module_config_primary.xml` yang sudah di-patch ke partisi **ODM**:  
   `/odm/etc/audio/audio_module_config_primary.xml`  
   *(Audio HAL AIDL Qualcomm selalu membaca `/odm/etc/audio/` lebih dulu sebelum `/vendor/etc/audio/`)*.
2. Tambahkan property Layer 2 ke `/odm/etc/build.prop` atau `/system/build.prop`.
