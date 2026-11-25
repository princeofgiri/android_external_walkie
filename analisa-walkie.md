# Analisis `com.android.walkietalkie` — Privilege Khusus

## Status: System App dengan UID 1000 (android.uid.system)

```
userId=1000
sharedUser=SharedUserSetting{android.uid.system/1000}
codePath=/system/app/WalkieTalkie
flags=[ SYSTEM DEBUGGABLE ]
```

Aplikasi ini **bukan app biasa** — dia berjalan sebagai **system user (UID 1000)**, yang berarti dia memiliki level akses yang sama dengan `system_server`. App ini terinstall di `/system/app/`, bukan `/data/app/`.

## Akses Perangkat Radio via Native Library

APK mengandung native library **`libintercomjni.so`** yang membuka langsung device node hardware:

| Device Node | SELinux Context | Permissions |
|---|---|---|
| `/dev/intercom_device` | `intercom_device:s0` | `crwxrwxrwx` (777) |
| `/dev/ttyS0` | — | `crwxrwxrwx` milik `system:system` |

**JNI functions** yang tersedia:
- `JNI_intercomPowerOn` / `JNI_intercomPowerOff` — menghidupkan/mematikan radio interkom
- `JNI_intercomSpeakerMode` / `JNI_intercomHeadsetMode` — switching audio mode
- `JNI_getIntercomVersion` — membaca versi firmware radio
- `JNI_resumeIntercomSetting` — restore settingan interkom

## Permissions yang Sangat Luas

### Install permissions (otomatis granted, tidak bisa dicabut)

- `MASTER_CLEAR` — reset pabrik
- `INSTALL_PACKAGES` — install APK tanpa user interaction
- `WRITE_SECURE_SETTINGS` — mengubah system settings
- `MANAGE_EXTERNAL_STORAGE` — akses semua file
- `SYSTEM_ALERT_WINDOW` — overlay di atas app lain
- `RECOVERY` — reboot ke recovery
- `CONFIGURE_WIFI_DISPLAY`
- `DEVICE_POWER`
- `MOUNT_UNMOUNT_FILESYSTEMS`
- `BIND_INCALL_SERVICE`
- `WRITE_SETTINGS`
- `CONFIGURE_DISPLAY_COLOR_MODE`
- `RECOVERY`
- `LOCATION_BYPASS`
- `BIND_ATTENTION_SERVICE`
- `STORAGE_INTERNAL`
- `USE_CREDENTIALS`
- `MODIFY_AUDIO_SETTINGS`
- `ACCESS_CHECKIN_PROPERTIES`
- `MODIFY_AUDIO_ROUTING`
- `QUERY_AUDIO_STATE`
- `INSTALL_LOCATION_PROVIDER`
- `USE_RESERVED_DISK`
- `BROADCAST_PHONE_ACCOUNT_REGISTRATION`
- `CLEAR_APP_USER_DATA`
- `BROADCAST_CALLLOG_INFO`

### Runtime permissions (semua granted via SYSTEM_FIXED)

- `RECORD_AUDIO`, `CAMERA`, `CALL_PHONE`, `SEND_SMS`
- `READ/WRITE_CONTACTS`, `READ/WRITE_CALL_LOG`
- `ACCESS_FINE/COARSE_LOCATION`, `ACCESS_BACKGROUND_LOCATION`
- `BLUETOOTH_CONNECT/SCAN/ADVERTISE`
- `POST_NOTIFICATIONS`
- `READ_CALENDAR`
- `READ_PHONE_STATE`
- `PROCESS_OUTGOING_CALLS`
- `GET_ACCOUNTS`
- `READ/WRITE_EXTERNAL_STORAGE`
- `READ_MEDIA_IMAGES/AUDIO/VIDEO`
- `RECEIVE_WAP_PUSH`
- `com.mediatek.permission.CTA_CONFERENCE_CALL`
- `com.mediatek.permission.CTA_CALL_FORWARD`

### Requested permissions

- `android.permission.RECORD_AUDIO`
- `android.permission.MODIFY_AUDIO_SETTINGS`
- `android.permission.BROADCAST_STICKY`
- `android.permission.BLUETOOTH_ADMIN`
- `android.permission.BLUETOOTH`
- `android.permission.NFC`
- `android.permission.PROCESS_OUTGOING_CALLS`
- `android.permission.WAKE_LOCK`
- `android.permission.READ_EXTERNAL_STORAGE`
- `android.permission.DEVICE_POWER`
- `android.permission.ACCESS_NETWORK_STATE`
- `android.permission.RECEIVE_USER_PRESENT`
- `android.permission.INTERNET`
- `android.permission.READ_PHONE_STATE`
- `android.permission.WRITE_EXTERNAL_STORAGE`
- `android.permission.WRITE_SETTINGS`
- `android.permission.VIBRATE`
- `android.permission.MOUNT_UNMOUNT_FILESYSTEMS`
- `android.permission.SYSTEM_ALERT_WINDOW`
- `android.permission.INSTALL_PACKAGES`
- `android.permission.MASTER_CLEAR`
- `android.permission.RECEIVE_BOOT_COMPLETED`
- `android.permission.READ_CALENDAR`
- `android.permission.ACCESS_WIFI_STATE`
- `android.permission.WRITE_SOCIAL_STREAM`
- `android.permission.ACCESS_FINE_LOCATION`
- `android.permission.WRITE_SECURE_SETTINGS`
- `android.permission.FOREGROUND_SERVICE`
- `android.permission.ACCESS_COARSE_LOCATION`

## Process Running

```
system  4433  558  ...  S  com.android.walkietalkie
UID: 1000, GID: 1000
Groups: 1000 1004 1007 1065 1077 1079 2002 3001 3002 3003 3007 9997
```

## Content Providers

- `com.android.walkietalkie/androidx.startup.InitializationProvider`
- `com.android.walkietalkie/.dbhelper.ChatProvider`
- Authority: `com.android.walkietalkie.ContentProvider.firstContentProvider`

## Receivers

- `SOSKeyReceiver` — menangani aksi SOS (location send, message)
- `PhoneCallBroadcaster` — intercept outgoing/incoming calls
- `BluetoothDKPTTReceiver` — Bluetooth DKPTT headset events (vendor specific company ID 0x85)
- `IPOShutdownReceiver` — handle shutdown event
- `ProfileInstallReceiver` — AndroidX profile installer

## Decoded Permissions (declared by app)

- `com.android.walkietalkie.PROVIDER` — prot=normal
- `com.android.walkietalkie.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION` — prot=signature

---

## Kesimpulan

1. **Bukan app 3rd party biasa** — ini system app yang di-sign oleh vendor (Mediatek) dan ditanam di `/system/app/`
2. **Berjalan sebagai UID 1000 (system)** — level akses yang sama dengan Android system itu sendiri
3. **Akses hardware radio langsung** via JNI ke `/dev/intercom_device` dan `/dev/ttyS0` (UART serial) — ini bukan akses via Android RIL/Telephony framework, tapi **akses langsung ke device node** sehingga bisa berkomunikasi dengan firmware radio interkom
4. **DEBUGGABLE flag aktif** — app ini dalam mode debug, bisa di-attach debugger dari luar (potensi security risk)
5. **Tidak ada sandbox** — dengan UID 1000, app ini bisa membaca/menulis hampir semua file di device

Aplikasi ini kemungkinan besar adalah **firmware walkie-talkie/vendor intercom app** (kemungkinan dari vendor Tintele/Tintel berdasarkan namespace `com.tintele.ltd.sos`) yang memang didesain untuk device Android khusus dengan hardware interkom bawaan.
