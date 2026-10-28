# Fix VoIP Call Mic Crash / Call Disconnected on ROM Port (Xiaomi `parrot` Vendor)

**Device:** `Redmi Note 15 5G` (codename `kunzite`) — running a Motorola HelloUI (Android 17) ROM port that uses an Xiaomi `parrot` vendor (`ro.board.platform=parrot`, SM6450).
**Target Android:** 17 · **Vendor Base:** Android 15/16/17 QTI Audio HAL AIDL
**Date:** 2026-10-18

---

## 1. Symptoms

On non-Xiaomi ROM ports (such as Moto HelloUI / AOSP / Pixel) that use an Xiaomi `parrot` vendor:

- VoIP calls in **WhatsApp, WhatsApp Business, Telegram, Discord, or Google Meet** disconnect immediately (*call ended*) within 3–6 seconds after connecting.
- The microphone sends no audio at all to the other party before the call drops.
- Regular calls (cellular / GSM telephony) and the voice recorder work fine — only app-based VoIP calls (`AUDIO_SOURCE_VOICE_COMMUNICATION`) fail.

### Logcat Signature

```text
AHAL_StreamIn_QTI: configure : usecase: VOIP_RECORD IoHandle:70 pal_stream_start failed!! ret:-131
AHAL_StreamIn_QTI: transfer : usecase: VOIP_RECORD IoHandle:70: failed to configure
AHAL_StreamIn_QTI: shutdown_I : usecase: VOIP_RECORD IoHandle:70
AHAL_StreamIn_QTI: configure : usecase: VOIP_RECORD IoHandle:70 set pal_stream_set_buffer_size to 640 with count 4
AHAL_StreamIn_QTI: configure : usecase: VOIP_RECORD IoHandle:70 pal_stream_start failed!! ret:-131
... (looping ~25 times per second for 6 seconds) ...
Telecom : CallsManager: markCallAsDisconnected: call=[...], voip=true; disconnectCause=DisconnectCause [ Code: (LOCAL) ... TelephonyCause: 36/65535 ]
wm_finish_activity: [...,com.whatsapp.w4b/com.whatsapp.calling.ui.VoipActivityV2,app-request,0]
```

---

## 2. Root Cause Analysis

### 2.1 Meaning of Return Code `ret: -131`

- The value `-131` is `-ENOTRECOVERABLE` (Linux `errno.h`: `131 ENOTRECOVERABLE /* State not recoverable */`).
- In Qualcomm Audio Graph Manager (`vendor/lib64/libagm.so`), the AR_OSAL error conversion table at offset `0x2c000` maps the SPF/ADSP statuses `AR_EUNEXPECTED` (`0x5`) and `AR_EVERSION` (`0x4`) directly to the POSIX value `-131`.
- This means the Qualcomm ADSP rejects the audio capture graph initialization when `pal_stream_start()` is executed.

### 2.2 Three Failure Points in the Xiaomi `parrot` Vendor

1. **Channel Mask & Device Profile Mismatch:**
   - The Android framework opens `voip_record` through `audio_module_config_primary.xml` as **MONO**:
     `channelLayouts="LAYOUT_MONO"` → 640-byte buffer (320 frames @ 16 kHz 16-bit mono).
   - In `vendor/etc/audio/sku_parrot/resourcemanager_parrot_qrd.xml`, the `PAL_STREAM_VOIP_TX` usecase under `PAL_DEVICE_IN_HANDSET_MIC` and `PAL_DEVICE_IN_SPEAKER_MIC` is forced to 2 channels and Xiaomi's dual-mic hardware:
     ```xml
     <usecase>
         <name>PAL_STREAM_VOIP_TX</name>
         <priority>2</priority>
         <bit_width>16</bit_width>
         <channels>2</channels>
         <snd_device_name>handset-dmic-endfire</snd_device_name>
     </usecase>
     ```
     This configuration mismatch causes the DSP driver to reject the frame mapping.

2. **Echo Cancellation Reference (EC Ref) Binding Failure:**
   - The `PAL_STREAM_VOIP_TX` stream on the Qualcomm DSP requires an echo cancellation reference (EC Ref) from the speaker/earpiece downlink stream (`VOIP_RX`).
   - VoIP apps (like WhatsApp) often open the mic capture before the audio output stream is ready.
   - In the stock `resourcemanager_parrot_qrd.xml`, the low-latency protection line is intentionally commented out by Xiaomi:
     ```xml
     <!--disabled_stream>PAL_STREAM_LOW_LATENCY</disabled_stream-->
     ```
     As a result, when `VOIP_RX` is not yet active, PAL tries to borrow the system `LOW_LATENCY` stream (e.g. touch UI sounds) as the EC reference. The ADSP graph fails to link two streams with different formats, producing the status `AR_EUNEXPECTED` (`-131`).

3. **Fluence Hardware Offload Initialization:**
   - The properties `persist.vendor.audio.fluence.voicecomm=true` and `vendor.audio.feature.dynamic_ecns.enable=true` force the HAL to invoke the Fluence HW module on the ADSP, which depends on Xiaomi OEM ACDB calibration. On a ROM port, these dependencies are broken.

---

## 3. Integrated Solution (3-Layer Fix)

To guarantee 100% reliability (*failsafe & defense-in-depth*), all 3 layers of fixes are applied at once:

### Layer 1: Bypass Hardware `VOIP_TX` via Audio Policy

File: `vendor/etc/audio/audio_module_config_primary.xml`

Disable the `mixPort` and `route` for `voip_record`:

```xml
        <!-- Disable hardware voip_record so it falls back to primary_in automatically -->
        <!--
        <mixPort name="voip_record" role="sink" flags="VOIP_TX">
            <profile samplingRates="8000 16000 32000 48000" channelLayouts="LAYOUT_MONO"
                formatType="PCM" pcmType="INT_16_BIT" />
        </mixPort>
        -->
```

And in the `<routes>` section:

```xml
        <!--
        <route type="mix" sink="voip_record" sources="built_in_bottom_mic,built_in_top_mic,built_in_back_mic,wired_headset_mic,bt_sco_headset_mic,usb_mic,usb_headset_mic,bt_a2dp_mic,echo_reference_mic,bt_le_mic" />
        -->
```

**Effect:** Android's AudioPolicyManager routes WhatsApp/Meet calls to `primary_in`. AudioFlinger automatically activates the WebRTC software AEC/NS (`libaudiopreprocessing.so`), giving stable noise suppression and echo cancellation without touching the broken DSP graph.

---

### Layer 2: Disable Fluence VoiceComm & Dynamic ECNS

File: `vendor/build.prop`

Set the following properties:

```properties
persist.vendor.audio.fluence.speaker=false
persist.vendor.audio.fluence.voicecomm=false
vendor.audio.feature.dynamic_ecns.enable=false
```

**Effect:** Prevents the audio HAL and DSP from attempting to build the broken Fluence hardware graph when apps request communication parameters.

---

### Layer 3: Normalize the PAL Resource Manager (DSP Safety Net)

File: `vendor/etc/audio/sku_parrot/resourcemanager_parrot_qrd.xml`

1. Under `PAL_DEVICE_IN_HANDSET_MIC` and `PAL_DEVICE_IN_SPEAKER_MIC`, remove the forced 2-channel & dual-mic endfire configuration from `PAL_STREAM_VOIP_TX`:

```xml
            <usecase>
                <name>PAL_STREAM_VOIP_TX</name>
                <priority>2</priority>
            </usecase>
```

2. In the `<in_streams>` section for `PAL_STREAM_VOIP_TX`, uncomment the low-latency protection:

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

**Effect:** If any sub-service still tries to open the `PAL_STREAM_VOIP_TX` stream, PAL won't crash because the channel profile and EC reference policies are already clean.

---

## 4. Alternative Option: Non-Vendor (If the Vendor Must Stay Intact)

If your porting rules forbid touching the `/vendor` partition entirely:

1. Copy the already-patched `audio_module_config_primary.xml` to the **ODM** partition:
   `/odm/etc/audio/audio_module_config_primary.xml`
   *(Qualcomm's AIDL Audio HAL always reads `/odm/etc/audio/` first, before `/vendor/etc/audio/`)*.
2. Add the Layer 2 properties to `/odm/etc/build.prop` or `/system/build.prop`.
