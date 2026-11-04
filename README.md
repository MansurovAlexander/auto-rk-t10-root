# Root для ГУ `auto_rk_t10` (Rockchip RK3326, Android 11)

> **English:** root guide for the `auto_rk_t10` Rockchip RK3326 head unit
> (Android 11) — Magisk boot patch, with a no-USB method via the vendor
> `flash_img` service. Tested on `V1.04.07-20241101`.

Практическое руководство по установке **Magisk** на китайское головное устройство
`auto_rk_t10` на Rockchip RK3326 без USB-кабеля — через vendor-сервис `flash_img`,
который умеет делать `dd` от root. Также приведён классический способ через
`fastboot`.

> **Проверено на:** `auto_rk_t10`, RK3326, Android 11, сборка
> `V1.04.07-20241101`, активный слот `b`, загрузчик разблокирован
> (`verifiedbootstate=orange`, `ro.boot.flash.locked=0`).
>
> **Дисклеймер.** Всё делается на свой страх и риск. Root аннулирует гарантию,
> может привести к bootloop и потере данных. Обязательно сделайте бэкап boot-разделов
> и держите под рукой стоковую прошивку/возможность подключить USB.
> Автор не несёт ответственности за «кирпич».

---

## Содержание

- [Что получится](#что-получится)
- [Почему `adb root` не работает](#почему-adb-root-не-работает)
- [Подготовка и бэкапы](#подготовка-и-бэкапы)
- [Способ A: без USB через `flash_img`](#способ-a-без-usb-через-flash_img)
- [Способ B: через fastboot](#способ-b-через-fastboot)
- [Установка приложения Magisk](#установка-приложения-magisk)
- [Проверка root](#проверка-root)
- [Откат на стоковый boot](#откат-на-стоковый-boot)
- [Постоянный ADB по Wi-Fi](#постоянный-adb-по-wi-fi)
- [Boot-анимация (опционально)](#boot-анимация-опционально)
- [Траблшутинг](#траблшутинг)
- [Ссылки](#ссылки)

---

## Что получится

- Magisk **28.0** в `boot_b`, root для приложений и `adb shell`.
- Прошивка и слот не меняются.
- `su -c id` → `uid=0(root) ... u:r:magisk:s0`.
- Опционально: постоянный ADB на `5555` после каждого включения.
- Опционально: своя boot-анимация из `/bootlogo`.

## Почему `adb root` не работает

На этой прошивке:

```text
ro.debuggable=0
adb root -> adbd cannot run as root in production builds
```

Старый трюк MTCE/PX5/PX6:

```sh
setprop persist.adb.tcp.port 5555
setprop sys.rkadb.root 1
reboot
```

на `auto_rk_t10` с Android 11 **не работает**: в `adbd` из APEX нет поддержки
`rkadb`, в init-скриптах упоминаний нет. Поэтому нужен патч boot-образа.

## Подготовка и бэкапы

Нужно:

- ADB на ПК (`adb devices` видит устройство);
- root на самом ГУ **не нужен** (если используется способ A);
- `Magisk-v28.0.apk` (или новее, но 28.0 проверена);
- свободное место на ГУ ~300 МБ;
- понимание, что активный слот — `b`, а `boot_a` — запасной.

Проверка устройства:

```sh
adb shell getprop ro.product.model
adb shell getprop ro.build.display.id
adb shell getprop ro.boot.slot_suffix
adb shell getprop ro.boot.verifiedbootstate
adb shell getprop ro.boot.flash.locked
```

Ожидаемо:

```text
auto_rk_t10
RD2A.211001.002.V1.04.07-20241101 release-keys
_b
orange
0
```

> **Важно:** boot-образ должен быть **от вашей сборки**. Нельзя шить boot.img
> от другой версии прошивки — это самый частый способ получить bootloop.

---

## Способ A: без USB через `flash_img`

В прошивке есть vendor-сервис, который запускает `dd` от **root** по значению
property. Это одновременно удобный способ бэкапа и «дыра» в безопасности —
используйте только на своём устройстве.

### A.1. Найти сервис

```sh
adb shell "su -c 'grep -n -A6 flash_img /vendor/etc/init/hw/init.rk30board.rc'"
```

Ожидаемо:

```text
service flash_img /vendor/bin/flash_img.sh
    class main
    oneshot
    disabled

on property:vendor.flash.success=2
    start flash_img
```

Сам скрипт:

```sh
#!/vendor/bin/sh
if=$(getprop vendor.flash_if_path)
of=$(getprop vendor.flash_of_path)
dd if=$if of=$of
setprop flash.success 1
```

Если сервиса нет — переходите к [способу B](#способ-b-через-fastboot).

### A.2. Снять стоковый `boot_b`

Создаём файл от shell, чтобы потом можно было забрать его через `adb pull`
(иначе root создаст файл `0600 root:root`):

```sh
adb shell "su -c 'touch /data/local/tmp/boot_b.img'"

adb shell "su -c '
  setprop flash.success 0
  setprop vendor.flash_if_path /dev/block/by-name/boot_b
  setprop vendor.flash_of_path /data/local/tmp/boot_b.img
  setprop vendor.flash.success 2
'"
```

Ждём 3–10 секунд и проверяем:

```sh
adb shell "su -c 'getprop flash.success; ls -l /data/local/tmp/boot_b.img'"
```

Должно быть `1` и размер 96 МБ (для этой разметки). Забираем:

```sh
adb pull /data/local/tmp/boot_b.img
```

> `flash.success` нужно сбрасывать в `0` **перед каждым** запуском. Иначе
> `on property:flash.success=1` не изменит значение и `vendor.flash.success=2`
> не сработает повторно.

Для запаса можно так же снять `boot_a`:

```sh
adb shell "su -c 'touch /data/local/tmp/boot_a.img; setprop flash.success 0; setprop vendor.flash_if_path /dev/block/by-name/boot_a; setprop vendor.flash_of_path /data/local/tmp/boot_a.img; setprop vendor.flash.success 2'"
adb pull /data/local/tmp/boot_a.img
```

### A.3. Проверить образ

```sh
md5sum boot_b.img
```

В начале файла должно быть `ANDROID!`:

```sh
python -c "print(open('boot_b.img','rb').read(8))"
# b'ANDROID!'
```

Для тестовой сборки `V1.04.07-20241101` стоковый `boot_b` имел:

```text
MD5 dc4b8be3e060b9c0b63a5a4650116a21
```

Это **не универсальный** хэш — у другой сборки он будет другим.

### A.4. Пропатчить Magisk

Скачайте `Magisk-v28.0.apk` и распакуйте. Нужны файлы:

- `assets/boot_patch.sh`
- `assets/util_functions.sh`
- `assets/stub.apk`
- `lib/armeabi-v7a/libmagiskboot.so` → `magiskboot`
- `lib/armeabi-v7a/libmagiskinit.so` → `magiskinit`
- `lib/armeabi-v7a/libmagisk.so` → `magisk`
- `lib/armeabi-v7a/libinit-ld.so` → `init-ld`

> Для 32-битного userspace (`armeabi-v7a`) берём именно `armeabi-v7a`.
> Для arm64 — `arm64-v8a`.

Кладём на ГУ:

```sh
adb shell "su -c 'rm -rf /data/local/tmp/magisk_patch; mkdir -p /data/local/tmp/magisk_patch'"
adb push boot_patch.sh util_functions.sh stub.apk magiskboot magiskinit magisk init-ld /data/local/tmp/magisk_patch/
adb push boot_b.img /data/local/tmp/magisk_patch/boot.img
adb shell "su -c 'chmod 755 /data/local/tmp/magisk_patch/*'"
```

Патчим с сохранением verity и шифрования (как в рабочем варианте для этого ГУ):

```sh
adb shell "su -c '
  cd /data/local/tmp/magisk_patch &&
  KEEPVERITY=true KEEPFORCEENCRYPT=true PATCHVBMETAFLAG=false RECOVERYMODE=false \
  ./boot_patch.sh boot.img
'"
```

В конце должно появиться:

```text
Repack to boot image: [new-boot.img]
```

Забираем:

```sh
adb pull /data/local/tmp/magisk_patch/new-boot.img magisk_patched_boot_b.img
```

Проверка патча:

- размер 96 МБ (Magisk сохраняет размер исходного файла);
- ядро байт-в-байт совпадает со стоком;
- в ramdisk есть `overlay.d/sbin/magisk.xz`, `stub.xz`, `.backup/.magisk`;
- `ro.build.display.id` не изменился.

### A.5. Записать патченный boot_b

```sh
adb push magisk_patched_boot_b.img /data/local/tmp/magisk_patched_boot_b.img

adb shell "su -c '
  setprop flash.success 0
  setprop vendor.flash_if_path /data/local/tmp/magisk_patched_boot_b.img
  setprop vendor.flash_of_path /dev/block/by-name/boot_b
  setprop vendor.flash.success 2
'"
```

Ждём завершения (`flash.success=1`), затем **контрольное чтение** и сверка MD5:

```sh
adb shell "su -c 'touch /data/local/tmp/boot_b_verify.img; setprop flash.success 0; setprop vendor.flash_if_path /dev/block/by-name/boot_b; setprop vendor.flash_of_path /data/local/tmp/boot_b_verify.img; setprop vendor.flash.success 2'"
adb shell "su -c 'md5sum /data/local/tmp/boot_b_verify.img /data/local/tmp/magisk_patched_boot_b.img'"
```

Хэши должны совпасть. Тогда:

```sh
adb shell "su -c 'sync'"
adb reboot
```

---

## Способ B: через fastboot

Если vendor-сервиса нет, но загрузчик разблокирован и доступен fastboot:

```sh
adb reboot bootloader
# или в терминале на ГУ: reboot bootloader

fastboot devices
fastboot getvar current-slot
fastboot flash boot magisk_patched_boot_b.img
fastboot reboot
```

Если fastboot нет, для Rockchip используется loader mode:

```sh
adb reboot loader
```

и `rkdeveloptool`/RKDevTool по USB: слить `boot_b`, пропатчить, записать обратно.
Для A/B-устройства убедитесь, что пишете в **активный** слот.

---

## Установка приложения Magisk

После первого boot с патченным образом Magisk-stub может сам установиться.
Если нет — поставьте полный APK:

```sh
adb install -r Magisk-v28.0.apk
```

Откройте Magisk:

1. Если появилось **«Требуется расширенная установка»** — нажмите **ОК**,
   устройство перезагрузится.
2. После перезагрузки: **Superuser** → найдите `com.android.shell` →
   включите переключатель (для `adb shell su`).
3. Для обычных приложений Magisk будет показывать запрос при первом `su`.

> Не обновляйте Magisk «вслепую» до 30.x, пока не убедитесь, что версия
> совместима с вашей прошивкой. 28.0 проверена.

## Проверка root

```sh
adb shell "su -c id"
# uid=0(root) gid=0(root) ... context=u:r:magisk:s0

adb shell "su -c 'magisk -v'"
# 28.0:MAGISK:R
```

## Откат на стоковый boot

Если Android загружается — тем же `flash_img`:

```sh
adb shell "su -c '
  setprop flash.success 0
  setprop vendor.flash_if_path /data/local/tmp/boot_b.img
  setprop vendor.flash_of_path /dev/block/by-name/boot_b
  setprop vendor.flash.success 2
'"
adb shell "su -c 'sync'"
adb reboot
```

Если не загружается:

- слот `a` часто содержит рабочую старую прошивку (проверьте
  `bootctl is-slot-bootable 0` и `bootctl is-slot-marked-successful 0`);
- USB + `fastboot flash boot stock_boot.img`;
- либо полная стоковая прошивка через recovery/loader.

## Постоянный ADB по Wi-Fi

Чтобы ADB был доступен после каждого включения на фиксированном порту:

```sh
adb shell "su -c 'setprop persist.adb.tcp.port 5555'"
```

Добавьте публичный ключ вашего ПК в `/data/misc/adb/adb_keys`
(файл `root:shell 0640`), иначе подключение будет `unauthorized`:

```sh
adb push "$HOME/.android/adbkey.pub" /sdcard/adbkey.pub
adb shell "su -c 'cat /sdcard/adbkey.pub >> /data/misc/adb/adb_keys; chmod 640 /data/misc/adb/adb_keys; chown root:shell /data/misc/adb/adb_keys'"
```

После перезагрузки:

```sh
adb connect <IP>:5555
```

> **Безопасность:** классический ADB на `5555` идёт **без TLS**. Подключиться
> может только хост с ключом, но порт виден в локальной сети. Если это критично,
> не используйте `persist.adb.tcp.port`, а включайте штатную «Отладку по Wi-Fi»
> с TLS в настройках разработчика.

## Boot-анимация (опционально)

`/bootlogo` на этом ГУ — ext4, смонтирован **rw**, поэтому анимацию можно
положить без Magisk-модуля:

```text
/bootlogo/config/bootanimation.zip
```

Требования:

- zip **stored** (без сжатия), иначе bootanimation ругается;
- `desc.txt` вида:

```text
1280 720 30
p 0 0 part0
```

- кадры `part0/000.png`, `part0/001.png`, ... в разрешении экрана
  (здесь 1280×720).

Исходный логотип загрузки лежит в:

```text
/bootlogo/config/logo/logo_1280x720.ppm
```

Это P3-заголовок + сырой RGB (не обычный PPM), кадры можно собрать скриптом
на Pillow.

Откат:

```sh
adb shell "su -c 'rm /bootlogo/config/bootanimation.zip'"
adb reboot
```

## Траблшутинг

| Симптом | Причина / решение |
|---|---|
| `adb root` → `cannot run as root in production builds` | Это normal для `user`-сборки. Нужен Magisk, не `adb root`. |
| `sys.rkadb.root=1` не помогает | На этой прошивке adbd не поддерживает `rkadb`. |
| `su: not found` после прошивки | Установите полный Magisk APK, выполните «Расширенную установку». |
| `su: request rejected (2000)` | Magisk → Superuser → включить доступ для `com.android.shell`. |
| Bootloop после прошивки | Boot.img не от той сборки. Верните стоковый boot через `flash_img`/fastboot. |
| `flash_img` не срабатывает | Сбросьте `flash.success` в `0` перед `vendor.flash.success=2`. |
| `adb pull` не видит файл | Создайте целевой файл через `touch` от shell до `dd`. |
| Порт ADB меняется после ребута | Включите `persist.adb.tcp.port=5555` и добавьте ключ в `adb_keys`. |

## Ссылки

- 4PDA: [Rockchip auto rk_t10 — обсуждение](https://4pda.to/forum/index.php?showtopic=1093150)
- XDA: [Auto RK T10 new Firmware from 2025](https://xdaforums.com/t/auto-rk-t10-new-firmware-from-2025-tested.4735971/)
- Magisk: [github.com/topjohnwu/Magisk](https://github.com/topjohnwu/Magisk)
- rootAVD (для эмуляторов): [github.com/newbit1/rootAVD](https://github.com/newbit1/rootAVD)

## Благодарности

- участникам темы 4PDA `deaddog2`, `Em0758`, `zima2778` и другим;
- `topjohnwu` за Magisk;
- всем, кто тестировал на своих `auto_rk_t10`.

## Лицензия

MIT — см. [LICENSE](LICENSE).
