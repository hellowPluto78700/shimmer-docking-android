# Android clone backend migration

## Scope

Baseline: `a32f6324dc08c22edd9342ecdd77686cb58fd4de`.
Deployment branch: `deploy/aws-clone`.
API: `https://wwr1eh4vg9.execute-api.us-east-2.amazonaws.com`.

Only the six backend URL literals were changed: three device-mapping calls in
`MainActivity.java` and three file-transfer calls in `ShimmerFileTransferClient.java`.
Endpoint paths, request bodies, response parsing, S3 upload handling, local database
sync state, retries, and timeouts remain unchanged. Module build output is ignored.
No backend, frontend, AWS infrastructure, Firebase configuration, or Bluetooth
behavior was changed.

## Build and artifact validation

Validated in WSL on 2026-10-06. Java and Android SDK were absent, so isolated tools
were installed under `/tmp/shimmer-android-tools`, with SDK licenses accepted.
Official Java and command-line-tools archive checksums were verified before use.
Tools: Temurin Java 17.0.20.1, existing Gradle wrapper 8.11.1, Android platform 35,
Build Tools 35.0.0. Existing build scripts and dependencies needed no changes.

```bash
JAVA_HOME=/tmp/shimmer-android-tools/jdk \
ANDROID_HOME=/tmp/shimmer-android-tools/sdk \
ANDROID_USER_HOME=/tmp/shimmer-android-tools/user \
GRADLE_USER_HOME=/tmp/shimmer-android-tools/gradle \
./gradlew assembleDebug --no-daemon --console=plain
```

Result: `BUILD SUCCESSFUL`, 35 actionable tasks executed.
APK: `app/build/outputs/apk/debug/app-debug.apk` (approximately 28 MB).
SHA-256: `0749f43657faf81de1f1f24375b24b49b6042e00f73577d3667d21c4b2fd4280`.
APK contents confirm package `com.example.myapplication`, version 1.0/code 1,
minimum API 31 (Android 12), target API 35, Internet permission, and existing
Bluetooth permissions. APK signature verification passed (v2, Android Debug signer).
DEX string inspection found the new API's four endpoint URL literals and no retired
backend hostname. The three mapping calls share one deduplicated DEX string.
Generated APK, caches, and signing keys are not committed.

Non-blocking build warnings: SDK XML version mismatch, ignored legacy manifest
package attribute, deprecated Java APIs, and an unstripped native datastore library.
These do not require changing existing application behavior.

## Android-side contract review

Compared the Android client with the validated backend's existing route handlers.
This is source/artifact validation, not a phone runtime or Bluetooth test.

| Call | Preserved request and response assumptions |
| --- | --- |
| `POST /missing-files/` | JSON array of filenames; reads `missing_files` array. |
| `GET /generate-upload-url/?filename=...` | Uses local basename; reads `upload_url`. |
| Presigned S3 `PUT` | Uses returned URL directly; uploads file bytes as `text/plain`. |
| `POST /decode-and-store/` | JSON object with `full_file_name`; checks HTTP success. |
| `GET /ddb/device-patient-map/{device}` | URL-encoded device; reads existing mapping fields. |
| `PUT /ddb/device-patient-map/{device}` | Required `patient`, optional string `shimmer1`/`shimmer2`; backend accepts strings and normalizes them to lists. |

The existing UI's string rendering of returned shimmer lists and existing decode
failure/sync-state handling were preserved, not redesigned. Mapping connect/read
timeouts remain 10/15 seconds, and OkHttp defaults remain unchanged.

## Manual handoff

Install the APK on the phone and run the final
Shimmer → Android → new AWS → new frontend E2E test.
No phone, emulator, Bluetooth connection, or live cloud write was attempted here.
Existing Firebase analytics/Crashlytics configuration remains unchanged and is not
part of the AWS backend migration.

This is a debug-signed APK. If the installed app has a different signing key, do
not uninstall it without first preserving its local data; use the original signing
key or an appropriate test device. The isolated build tools and debug key are in
temporary storage, so retain the key securely if future in-place test updates are
needed. On another development machine, use its existing Java 17/Android SDK setup.
