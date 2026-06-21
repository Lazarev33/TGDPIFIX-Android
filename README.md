# Telegram Android MTProxy DPI-Fix

Patch for the open-source Telegram Android client that makes MTProxy `ee-secret` fake-TLS traffic less static and harder to match by simple DPI signatures.

The patch is focused on **external MTProxy connections only**. Normal Telegram authorization and direct Telegram DC connections are intentionally left untouched.

## What this patch does

This patch modifies the native Telegram Android network layer:

```text
TMessagesProj/jni/tgnet/ConnectionSocket.cpp
TMessagesProj/jni/tgnet/ConnectionSocket.h
```

Main idea:

```text
Official/default Telegram MTProxy fake-TLS ClientHello has a recognizable static pattern.
This patch makes the MTProxy fake-TLS handshake more dynamic.
```

Implemented behavior:

```text
Normal authorization without proxy       → original Telegram behavior
Direct Telegram DC fakeTLS               → original Telegram behavior
External MTProxy with ee-secret          → DPI-fix ClientHello
```

The patch adds:

```text
- dynamic fake-TLS ClientHello generation for MTProxy
- randomized GREASE / random fields
- dynamic final padding
- fixed valid ClientHello size
- chunked ClientHello sending
- EAGAIN / EWOULDBLOCK / EINTR handling
```

The patch does **not** add:

```text
- VPN functionality
- proxy server functionality
- malware/self-propagation code
- credential collection
- Telegram protocol changes
```

## Current status

Tested behavior:

```text
Clean self-built Telegram Android auth works with custom api_id/api_hash.
Patched build keeps normal authorization working.
Patch works with at least some external MTProxy ee-secret links.
```

Known limitation:

```text
Not every MTProxy is guaranteed to work.
Some providers may block by IP, DNS, SNI, hosting ASN, or other traffic properties.
```

In testing, links where `server` is an IP address were more reliable than links where `server` is a hostname.

Recommended MTProxy format:

```text
tg://proxy?server=<IP>&port=<PORT>&secret=ee...
```

Less reliable format:

```text
tg://proxy?server=<hostname>&port=<PORT>&secret=ee...
```

## Important: use your own Telegram API credentials

The official Telegram Android source contains public sample credentials:

```java
APP_ID = 4
APP_HASH = "014b35b6184100b085b0d0572f9b5103"
```

Do **not** use them for real builds. Authorization may fail or hang.

You need to get your own Telegram API credentials from:

```text
my.telegram.org → API development tools
```

Then edit:

```text
TMessagesProj/src/main/java/org/telegram/messenger/BuildVars.java
```

Example:

```java
public static int APP_ID = YOUR_API_ID;
public static String APP_HASH = "YOUR_API_HASH";
```

Do not commit your private `APP_HASH` into a public repository.

## Applying the patch

From the root of Telegram Android source:

```bash
git apply telegram_android_proxy_only_dynamic_padding_v3.patch
```

If the source tree changed and the patch does not apply cleanly:

```bash
git apply --reject telegram_android_proxy_only_dynamic_padding_v3.patch
```

Then resolve `.rej` files manually.

## Verifying that the patch is applied

Run:

```bash
grep -n "DpiFinalPadding\|dpi_final_padding\|getDpiFixDefault\|useDpiFixTlsHello\|eagainRetries" \
  TMessagesProj/jni/tgnet/ConnectionSocket.cpp
```

You should see patch markers.

Make sure old experimental/broken code is not present:

```bash
grep -n "NID_ED25519\|EVP_PKEY\|TCP_WINDOW_CLAMP\|random_device\|mt19937\|send_dpi_fix\|TcpSplit\|tlsHelloSplit" \
  TMessagesProj/jni/tgnet/ConnectionSocket.cpp
```

Expected result: no output.

## Building on Windows with Android Studio JBR

Open Android Studio Terminal in the project root.

Set Java path:

```powershell
$java = "$env:ProgramFiles\Android\Android Studio\jbr\bin\java.exe"
```

Build:

```powershell
& $java -classpath ".\gradle\wrapper\gradle-wrapper.jar" org.gradle.wrapper.GradleWrapperMain :TMessagesProj_App:assembleAfatRelease --stacktrace
```

Full clean rebuild:

```powershell
& $java -classpath ".\gradle\wrapper\gradle-wrapper.jar" org.gradle.wrapper.GradleWrapperMain --stop

Remove-Item .\TMessagesProj\.cxx -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item .\TMessagesProj\build -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item .\TMessagesProj_App\build -Recurse -Force -ErrorAction SilentlyContinue

& $java -classpath ".\gradle\wrapper\gradle-wrapper.jar" org.gradle.wrapper.GradleWrapperMain :TMessagesProj_App:assembleAfatRelease --rerun-tasks --stacktrace
```

Output APK:

```text
TMessagesProj_App/build/outputs/apk/afat/release/app.apk
```

## Installing with ADB

Install fresh:

```bash
adb install TMessagesProj_App/build/outputs/apk/afat/release/app.apk
```

Reinstall over existing app while keeping data:

```bash
adb install -r TMessagesProj_App/build/outputs/apk/afat/release/app.apk
```

For testing patches, `install -r` is faster because it keeps the Telegram session.

## Testing procedure

Recommended test order:

```text
1. Build clean Telegram Android with your own api_id/api_hash.
2. Install it.
3. Confirm that authorization works without proxy.
4. Apply the DPI-fix patch.
5. Rebuild.
6. Confirm that authorization still works without proxy.
7. Add external MTProxy with ee-secret.
8. Test proxy connection.
```

Do not debug MTProxy before confirming normal authorization works.

## MTProxy ee-secret notes

This patch only activates for external MTProxy secrets starting with:

```text
ee
```

Example structure:

```text
secret=ee<16-byte-secret><hex-encoded-fake-tls-domain>
```

You can decode the fake TLS domain from a secret with:

```python
secret = "ee..."
if secret.startswith("ee") and len(secret) > 34:
    domain_hex = secret[34:]
    print(bytes.fromhex(domain_hex).decode("utf-8", errors="replace"))
else:
    print("Not an ee-secret")
```

Example:

```text
ee2676ac90d85a6e00bcb106d9d635243169642e766b2e7275
```

Decoded fake TLS domain:

```text
id.vk.ru
```

## Proxy compatibility tips

More reliable:

```text
server=<proxy IP>
secret=ee...
fake TLS domain inside secret
```

Less reliable:

```text
server=<proxy hostname>
secret=ee...
```

If a hostname-based proxy does not work, resolve it to IP and try the same secret with the IP:

```bash
nslookup example.proxy.host
```

Then create a link manually:

```text
tg://proxy?server=<resolved_ip>&port=<port>&secret=<same_secret>
```

## Troubleshooting

### Authorization does not work

Most likely cause:

```text
You are still using APP_ID = 4 and the public APP_HASH.
```

Fix:

```text
Use your own api_id/api_hash from my.telegram.org.
```

### Authorization works, but proxy does not

Check:

```text
- Is the secret really starting with ee?
- Is this an external MTProxy, not direct Telegram DC fakeTLS?
- Does the server IP/port respond?
- Does the same proxy work on another client/network?
- Is the fake TLS SNI blocked by your provider?
- Does replacing server hostname with IP help?
```

### Patch does not seem to change anything

Verify the APK really changed:

```powershell
Get-FileHash .\TMessagesProj_App\build\outputs\apk\afat\release\app.apk -Algorithm SHA256
```

After installing to emulator/device, pull the installed APK and compare hashes:

```bash
adb shell pm path org.telegram.messenger
adb pull /path/from/pm/path installed.apk
```

Then compare SHA256.

### Logcat

Clear logs:

```bash
adb logcat -c
```

Run Telegram, try connecting to proxy, then dump logs:

```bash
adb logcat -d > tg_dpifix_log.txt
```

Useful filters:

```bash
adb logcat | grep -i "tgnet\|proxy\|socket\|mtproto\|connection"
```

## Design notes

The patch intentionally avoids changing normal Telegram authorization flow.

The key flag is reset by default:

```text
useDpiFixTlsHello = false
```

It becomes enabled only for external MTProxy with `ee-secret`:

```text
external MTProxy ee-secret → useDpiFixTlsHello = true
```

Direct Telegram DC fakeTLS remains disabled:

```text
direct fakeTLS DC → useDpiFixTlsHello = false
```

This separation is important. Without it, a DPI-fix experiment can accidentally break normal authorization.

## What this patch is not

This is not a universal censorship bypass.

DPI systems may also block by:

```text
- proxy IP reputation
- DNS query
- SNI/domain policy
- hosting provider ASN
- TLS timing
- packet size distribution
- active probing
```

This patch only targets one class of detection: static or recognizable MTProxy fake-TLS handshake patterns.

## Repository recommendation

Suggested repo layout:

```text
telegram-android-mtproxy-dpifix/
├── README.md
├── patches/
│   └── telegram_android_proxy_only_dynamic_padding_v3.patch
└── docs/
    └── troubleshooting.md
```

Do not upload your personal `BuildVars.java` with private `APP_HASH`.

## Disclaimer

This patch is provided for research, interoperability, and user-controlled connectivity through legitimate MTProxy servers.

Use responsibly and comply with local laws and upstream project licenses.
