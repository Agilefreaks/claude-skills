# AndroidManifest.xml — canonical shapes

Every Android module has `src/main/AndroidManifest.xml`, even library modules with no components. The Android Gradle Plugin merges them into the app manifest at build time.

Two non-negotiable rules:

1. **Always declare the `xmlns:android` namespace on `<manifest>`** — without it, any `android:*` attribute (e.g. `android:name` on `<uses-permission>` or `<application>`) fails to resolve and the AAPT merger errors out:

   ```
   AndroidManifest.xml:N: AAPT: error: attribute 'android:name' not found.
   ```

2. **Library manifests are minimal.** No `<application>` tag, no `package` attribute (AGP derives it from the module's `namespace` in `build.gradle.kts`).

## Library module — no contributions to the merged manifest

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android" />
```

## Library module — declares a permission (e.g. `core/data` declaring `INTERNET`)

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.INTERNET" />
</manifest>
```

## Application module (`app-mobile/src/main/AndroidManifest.xml`, similarly `app-tv/`)

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:name=".<Project>Application"
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.<Project>">

        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:theme="@style/Theme.<Project>">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

    </application>

</manifest>
```

## TV variant — `app-tv/src/main/AndroidManifest.xml`

Same shape as the mobile manifest, with two changes on the launcher activity:

- Add `<category android:name="android.intent.category.LEANBACK_LAUNCHER" />` inside the intent-filter so Android TV's launcher picks it up.
- Add `android:banner="@drawable/banner"` on the `<application>` element.
