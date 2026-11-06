# Прошивка CH32V003 через WCH-LinkE

Порядок для Linux (проверен на Linux Mint 22.3 x86_64, база — Ubuntu 24.04).

## Что нужно

- Программатор **WCH-LinkE** в режиме RISC-V (USB `1a86:8010`).
- Провода от программатора к плате: **SWIO → PD1**, **GND → GND**. Чип должен быть
  запитан от своей схемы (на стенде JXDIO — от DC OUT через LM2596S); программатор
  в этой схеме его не питает.
- Прошивальщик **minichlink** из [ch32fun](https://github.com/cnlohr/ch32fun) на
  коммите `b9cf4697d7150f0c216a453e80080dca960ca235` — на нём прошивки собраны и
  проверены.
- Файлы релиза: `<проект>-vX.Y-ch32v003.bin` и `SHA256SUMS`.

## 1. Собрать minichlink

```sh
sudo apt install build-essential git libusb-1.0-0-dev libudev-dev
git clone https://github.com/cnlohr/ch32fun.git
git -C ch32fun checkout b9cf4697d7150f0c216a453e80080dca960ca235
make -C ch32fun/minichlink minichlink
```

Доступ к программатору без root: правило udev из того же репозитория.

```sh
sudo cp ch32fun/minichlink/99-minichlink.rules /etc/udev/rules.d/
sudo udevadm control --reload && sudo udevadm trigger
```

Правило назначает программатору группу `plugdev` или `dialout` (в файле — обе, у udev
действует последнее присваивание). Какая досталась — `ls -l /dev/bus/usb/<шина>/<устройство>`
(номера — из `lsusb`); пользователь должен в ней состоять (`groups`), после добавления в
группу — перелогиниться.

## 2. Проверить файлы

```sh
sha256sum --ignore-missing -c SHA256SUMS
```

Строка `<файл>.bin: OK` (в русской локали — `ЦЕЛ`) — образ цел.

## 3. Проверить программатор и чип

```sh
ch32fun/minichlink/minichlink -i
```

Ожидается `WCH Programmer is LinkE …`, `Detected CH32V003`, `Flash Storage: 16 kB`,
`Part Type: 00-30-05-x0`. Part Type `0x003005x0` — корпус F4P6 (TSSOP20).
`Read protection: disabled` — флеш можно писать.

## 4. Прошить

```sh
ch32fun/minichlink/minichlink -w <проект>-vX.Y-ch32v003.bin flash -b
```

`Image written.` — записано; `-b` перезапускает чип с новой прошивкой.

## 5. Сверить записанное

```sh
size=$(stat -c %s <проект>-vX.Y-ch32v003.bin)
ch32fun/minichlink/minichlink -r readback.bin flash "$size"
cmp readback.bin <проект>-vX.Y-ch32v003.bin && echo OK
```

Чтение останавливает ядро на время сеанса; дольше секунды — и сторожевой таймер
прошивки перезапустит чип. На работу это не влияет.

## 6. Проверить версию

Версия прошивки `jxdio-expander` читается по I²C с мастера шины: регистры `0x48`
(major) и `0x49` (minor) по адресу `0x20` — см. [регистры](jxdio-expander.md).

## Если не получается

| Симптом | Что проверить |
| --- | --- |
| `Could not initialize any supported programmers` / нет `Found WCH Link` | USB-кабель, `lsusb` показывает `1a86:8010`; права (`plugdev`, правило udev) |
| `LinkE` найден, чип не отвечает | питание платы, провода SWIO → PD1 и GND |
| `Read protection: enabled` | флеш защищён от чтения; снятие защиты стирает чип — только осознанно (`minichlink -p`) |
