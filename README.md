# Telegram Android MTProxy DPI-Fix

Патч для open-source клиента **Telegram Android** из репозитория **DrKLO/Telegram**.

Патч делает fake-TLS handshake у MTProxy с `ee-secret` менее статичным, чтобы простые DPI-сигнатуры хуже узнавали трафик Telegram MTProxy.

> Важно: патч рассчитан именно на исходники **DrKLO/Telegram**.  
> Не рекомендуется применять его напрямую к Telegram-FOSS, Nekogram, Forkgram, MDGram и другим форкам без ручной проверки diff.

---

## Что делает патч

Патч изменяет native-сетевой слой Telegram Android:

```text
TMessagesProj/jni/tgnet/ConnectionSocket.cpp
TMessagesProj/jni/tgnet/ConnectionSocket.h
```

Основная идея:

```text
Стандартный fake-TLS ClientHello у Telegram MTProxy имеет узнаваемый статичный паттерн.
Патч делает этот handshake более динамическим.
```

Поведение после патча:

```text
Обычная авторизация без прокси      → оригинальное поведение Telegram
Direct Telegram DC fakeTLS          → оригинальное поведение Telegram
Внешний MTProxy с ee-secret         → DPI-fix ClientHello
```

То есть патч **не должен ломать обычную авторизацию**, потому что DPI-fix включается только для внешнего MTProxy с `ee-secret`.

---

## Требования

Нужны:

```text
- исходники Telegram Android из DrKLO/Telegram
- Android Studio
- Android SDK / NDK
- JDK из Android Studio
- свои Telegram api_id / api_hash
```

Патч тестировался на структуре проекта Telegram Android, где есть:

```text
TMessagesProj/
TMessagesProj_App/
gradle/
```

---

## Важно: используйте исходники DrKLO

Сначала скачайте официальный Android-код Telegram:

```text
https://github.com/DrKLO/Telegram
```

Пример:

```bash
git clone https://github.com/DrKLO/Telegram.git
cd Telegram
```

Не применяйте патч вслепую к другим форкам. У них может отличаться:

```text
- ConnectionSocket.cpp
- ConnectionSocket.h
- MtProto/TGNet логика
- структура Gradle-проекта
- сборочные flavor'ы
- native CMake/NDK конфигурация
```

Если используете не DrKLO-код, сначала сравните файлы вручную.

---

## Обязательно: свой api_id и api_hash

В исходниках Telegram Android по умолчанию могут быть публичные тестовые API-данные:

```java
APP_ID = 4
APP_HASH = "014b35b6184100b085b0d0572f9b5103"
```

Их нельзя использовать для нормальной сборки. С ними авторизация может зависать, не присылать код или падать на серверной стороне.

Получите свои данные здесь:

```text
my.telegram.org → API development tools
```

Затем откройте файл:

```text
TMessagesProj/src/main/java/org/telegram/messenger/BuildVars.java
```

И замените:

```java
public static int APP_ID = YOUR_API_ID;
public static String APP_HASH = "YOUR_API_HASH";
```

Пример:

```java
public static int APP_ID = 12345678;
public static String APP_HASH = "abcdef1234567890abcdef1234567890";
```

Не выкладывайте свой `APP_HASH` в публичный GitHub.

---

## Что именно меняется

Патч добавляет:

```text
- отдельный DPI-fix ClientHello для внешнего MTProxy
- динамическую генерацию handshake
- GREASE/random-поля
- динамический final padding
- корректный размер fake-TLS ClientHello
- chunked sending первого ClientHello
- обработку EAGAIN / EWOULDBLOCK / EINTR
```

Патч не добавляет:

```text
- VPN
- свой прокси-сервер
- изменение Telegram-протокола
- сбор логинов/паролей
- скрытые фоновые функции
- автозапуск
- вредоносную логику
```

---

## Поддерживаемый тип прокси

Патч включается только для внешнего MTProxy с secret, начинающимся на:

```text
ee
```

Пример ссылки:

```text
tg://proxy?server=144.31.169.255&port=853&secret=ee2676ac90d85a6e00bcb106d9d635243169642e766b2e7275
```

Где:

```text
server=144.31.169.255
port=853
secret=ee...
```

`ee-secret` содержит fake-TLS домен внутри самого secret.

Например:

```text
ee2676ac90d85a6e00bcb106d9d635243169642e766b2e7275
```

Содержит домен:

```text
id.vk.ru
```

---

## Как расшифровать домен из ee-secret

Можно использовать простой Python-скрипт:

```python
secret = "ee2676ac90d85a6e00bcb106d9d635243169642e766b2e7275"

if secret.startswith("ee") and len(secret) > 34:
    domain_hex = secret[34:]
    domain = bytes.fromhex(domain_hex).decode("utf-8", errors="replace")
    print(domain)
else:
    print("Это не ee-secret")
```

Вывод:

```text
id.vk.ru
```

---

## Рекомендации по MTProxy

Лучше работает формат, где `server` — это IP:

```text
tg://proxy?server=<IP>&port=<PORT>&secret=ee...
```

Менее надёжный вариант:

```text
tg://proxy?server=<hostname>&port=<PORT>&secret=ee...
```

Если ссылка с hostname не работает, попробуйте узнать IP:

```bash
nslookup vpn.example.com
```

И собрать ссылку вручную:

```text
tg://proxy?server=<resolved_ip>&port=<port>&secret=<same_secret>
```

Пример:

```text
Было:
tg://proxy?server=vpn.example.com&port=8443&secret=ee...

Стало:
tg://proxy?server=1.2.3.4&port=8443&secret=ee...
```

---

## Применение патча

Из корня проекта Telegram Android:

```bash
git apply patches/telegram_android_proxy_only_dynamic_padding_v3.patch
```

Если патч не применился чисто:

```bash
git apply --reject patches/telegram_android_proxy_only_dynamic_padding_v3.patch
```

После этого вручную проверьте `.rej` файлы.

---

## Проверка, что патч применился

В Linux/macOS/Git Bash:

```bash
grep -n "DpiFinalPadding\|dpi_final_padding\|getDpiFixDefault\|useDpiFixTlsHello\|eagainRetries" \
  TMessagesProj/jni/tgnet/ConnectionSocket.cpp
```

Должны появиться строки с этими маркерами.

Проверьте, что в коде нет старых экспериментальных вариантов:

```bash
grep -n "NID_ED25519\|EVP_PKEY\|TCP_WINDOW_CLAMP\|random_device\|mt19937\|send_dpi_fix\|TcpSplit\|tlsHelloSplit" \
  TMessagesProj/jni/tgnet/ConnectionSocket.cpp
```

Ожидаемый результат:

```text
пусто
```

---

## Сборка на Windows через Android Studio Terminal

Откройте Terminal в Android Studio в корне проекта.

Задайте путь к Java из Android Studio:

```powershell
$java = "$env:ProgramFiles\Android\Android Studio\jbr\bin\java.exe"
```

Обычная сборка:

```powershell
& $java -classpath ".\gradle\wrapper\gradle-wrapper.jar" org.gradle.wrapper.GradleWrapperMain :TMessagesProj_App:assembleAfatRelease --stacktrace
```

Полная пересборка:

```powershell
& $java -classpath ".\gradle\wrapper\gradle-wrapper.jar" org.gradle.wrapper.GradleWrapperMain --stop

Remove-Item .\TMessagesProj\.cxx -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item .\TMessagesProj\build -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item .\TMessagesProj_App\build -Recurse -Force -ErrorAction SilentlyContinue

& $java -classpath ".\gradle\wrapper\gradle-wrapper.jar" org.gradle.wrapper.GradleWrapperMain :TMessagesProj_App:assembleAfatRelease --rerun-tasks --stacktrace
```

APK будет здесь:

```text
TMessagesProj_App/build/outputs/apk/afat/release/app.apk
```

---

## Установка через ADB

Свежая установка:

```bash
adb install TMessagesProj_App/build/outputs/apk/afat/release/app.apk
```

Установка поверх старой версии с сохранением данных:

```bash
adb install -r TMessagesProj_App/build/outputs/apk/afat/release/app.apk
```

Для тестирования патчей удобнее использовать:

```bash
adb install -r
```

Так Telegram-сессия сохраняется.

---

## Проверка установленного APK

Можно сравнить SHA256 локального APK:

```powershell
Get-FileHash .\TMessagesProj_App\build\outputs\apk\afat\release\app.apk -Algorithm SHA256
```

И установленного APK:

```bash
adb shell pm path org.telegram.messenger
adb pull /path/from/pm/path installed.apk
```

Затем сравнить хеши.

---

## Рекомендуемый порядок тестирования

Правильный порядок:

```text
1. Скачать чистый DrKLO/Telegram.
2. Вписать свой api_id/api_hash.
3. Собрать чистый APK без патча.
4. Проверить обычную авторизацию без прокси.
5. Применить DPI-fix patch.
6. Собрать APK снова.
7. Проверить, что обычная авторизация всё ещё работает.
8. Добавить внешний MTProxy с ee-secret.
9. Проверить подключение через прокси.
```

Не имеет смысла тестировать DPI-fix, пока обычная авторизация не работает.

---

## Troubleshooting

### Авторизация не работает

Самая частая причина:

```text
Вы используете APP_ID = 4 и публичный APP_HASH из исходников.
```

Решение:

```text
Получите свои api_id/api_hash на my.telegram.org.
```

---

### Авторизация работает, но прокси нет

Проверьте:

```text
- secret точно начинается с ee?
- это внешний MTProxy, а не direct Telegram DC fakeTLS?
- server указан IP или hostname?
- порт открыт?
- этот же прокси работает в другом клиенте?
- fake-TLS домен внутри secret не заблокирован?
- помогает ли замена hostname на IP?
```

---

### Один прокси работает, другой нет

Это нормально.

DPI может блокировать не только handshake, но и:

```text
- IP адрес прокси
- DNS-запрос
- домен в server=
- fake-TLS SNI
- ASN/хостинг
- порт
- активное probing-поведение
```

Например:

```text
server=IP + SNI=id.vk.ru → может работать
server=hostname + SNI=www.cloudflare.com → может не работать
```

---

### Патч не включается

Патч включается только в ветке:

```text
external MTProxy + ee-secret
```

Direct fakeTLS DC специально оставлен без DPI-fix.

Логика такая:

```text
useDpiFixTlsHello = false              // по умолчанию
external MTProxy ee-secret             // включаем
direct Telegram DC fakeTLS             // не включаем
```

Это нужно, чтобы не сломать обычную авторизацию Telegram.

---

## Logcat

Очистить логи:

```bash
adb logcat -c
```

Запустить Telegram, попробовать подключиться к MTProxy, затем сохранить лог:

```bash
adb logcat -d > tg_dpifix_log.txt
```

Полезные фильтры:

```bash
adb logcat | grep -i "tgnet\|proxy\|socket\|mtproto\|connection"
```

---

## Архитектура патча

Главный принцип:

```text
Не трогать обычный Telegram network flow.
```

Патч не заменяет весь `ConnectionSocket`.

Он добавляет отдельную ветку для внешнего MTProxy:

```text
if external MTProxy has ee-secret:
    use DPI-fix ClientHello
else:
    use original Telegram ClientHello / original behavior
```

Это снижает риск поломать:

```text
- логин
- отправку кода
- direct DC соединения
- внутренний fakeTLS Telegram
- обычную работу без прокси
```

---

## Ограничения

Это не универсальный способ обхода любых блокировок.

Патч нацелен на один класс DPI-детекта:

```text
статичный или узнаваемый fake-TLS handshake MTProxy
```

Он не гарантирует работу, если блокировка идёт по:

```text
- IP
- DNS
- SNI policy
- ASN провайдера
- активному probing
- спискам известных прокси
- блокировке конкретного порта
```

---

## Рекомендуемая структура репозитория

```text
telegram-android-mtproxy-dpifix/
├── README.md
├── patches/
│   └── telegram_android_proxy_only_dynamic_padding_v3.patch
└── docs/
    └── troubleshooting.md
```

Не выкладывайте в репозиторий:

```text
- свой APP_HASH
- приватные MTProxy secret'ы
- личные сборочные ключи
- google-services.json с приватными данными
```

---

## Disclaimer

Патч предназначен для исследования совместимости, пользовательского подключения к легитимным MTProxy-серверам и экспериментов с open-source клиентом Telegram Android.

Используйте ответственно и соблюдайте законы вашей страны, лицензии Telegram Android и правила GitHub.