# Настройка OpenWrt, PassWall2 и кастомных VPN-подписок

Данное руководство описывает полный процесс настройки роутера с прошивкой OpenWrt для работы с нестандартными форматами VPN-подписок (на примере сервисов типа GoVPN/easy-api.live, отдающих конфигурации в формате JSON вместо классических Base64-ссылок). Дополнительно рассматривается настройка роутера для работы в роли домашнего файлового сервера (SMB/FTP).

Руководство протестировано на роутере **Cudy TR3000 v1**, но общая логика применима к любым роутерам с поддержкой OpenWrt.

## Оглавление

1. [Сборка прошивки с нужными пакетами](#1-сборка-прошивки-с-нужными-пакетами)
2. [Установка PassWall2 и Xray](#2-установка-passwall2-и-xray)
3. [Скрипт-парсер для нестандартных подписок](#3-скрипт-парсер-для-нестандартных-подписок)
4. [Настройка маршрутизации PassWall2](#4-настройка-маршрутизации-passwall2)
5. [Обновление подписки с телефона (Termux)](#5-обновление-подписки-с-телефона-termux)
6. [Настройка файлового сервера (SMB и FTP)](#6-настройка-файлового-сервера-smb-и-ftp)

---

## 1. Сборка прошивки с нужными пакетами

Наиболее удобный способ получить готовую систему — собрать прошивку через официальный [Firmware Selector](https://firmware-selector.openwrt.org/), заранее интегрировав в нее необходимые драйверы и пакеты.

1. Откройте Firmware Selector и введите модель вашего роутера (например, Cudy TR3000).
2. Разверните меню **Customize** (Настроить).
3. В поле **Installed packages** (Установленные пакеты) скопируйте полный список ниже. В нем стандартный `dnsmasq` уже заменен на `dnsmasq-full` (это жесткое требование для работы PassWall2), а также добавлены все необходимые пакеты для USB, SMB и файловых систем:

```text
apk-mbedtls base-files ca-bundle dnsmasq-full dropbear firewall4 fitblk fstools kmod-crypto-hw-safexcel kmod-gpio-button-hotplug kmod-leds-gpio kmod-nft-offload libc libgcc libustream-mbedtls logd mtd netifd nftables odhcp6c odhcpd-ipv6only ppp ppp-mod-pppoe procd-ujail uboot-envtools uci uclient-fetch urandom-seed urngd wpad-basic-mbedtls kmod-usb3 kmod-mt7915e kmod-mt7981-firmware mt7981-wo-firmware luci luci-app-attendedsysupgrade luci-compat ksmbd-server luci-app-ksmbd vsftpd block-mount kmod-usb-storage kmod-fs-vfat kmod-fs-exfat kmod-fs-ext4 kmod-fs-ntfs3 usbutils
```

4. В поле **uci-defaults script** (Скрипт инициализации) можно добавить простой и надежный код для автоматического включения всех WiFi-сетей при первом запуске (актуально, если к роутеру нет доступа по кабелю):

```bash
#!/bin/sh

# Включение всех радиомодулей
uci set wireless.radio0.disabled=0
uci set wireless.radio1.disabled=0
uci set wireless.radio2.disabled=0
uci commit wireless

wifi reload
```

5. Нажмите **Request Build** и скачайте образ с пометкой **sysupgrade**.
6. Прошейте роутер через LuCI (`System` -> `Backup / Flash Firmware`), предварительно сняв галочку `Keep settings`.

---

## 2. Установка PassWall2 и Xray

Начиная с версии OpenWrt 25.12 используется пакетный менеджер `apk`. Пакет PassWall2 отсутствует в официальных репозиториях, поэтому его необходимо устанавливать из стороннего источника.

Подключитесь к роутеру по SSH и выполните следующие команды:

> Примечание: В примере ниже используется репозиторий для архитектуры `aarch64_cortex-a53`. Если ваш роутер использует другую архитектуру (узнать можно командой `uname -m`), замените архитектуру в ссылках.

```bash
# 1. Загрузка ключа репозитория
wget -O /etc/apk/keys/passwall.pem https://master.dl.sourceforge.net/project/openwrt-passwall-build/apk.pub

# 2. Добавление репозиториев PassWall2
echo "https://master.dl.sourceforge.net/project/openwrt-passwall-build/releases/packages-25.12/aarch64_cortex-a53/passwall_packages/packages.adb" >> /etc/apk/repositories.d/passwall.list
echo "https://master.dl.sourceforge.net/project/openwrt-passwall-build/releases/packages-25.12/aarch64_cortex-a53/passwall2/packages.adb" >> /etc/apk/repositories.d/passwall.list

# 3. Обновление списков пакетов
apk update

# 4. Установка PassWall2 и необходимых зависимостей (включая ядро Xray)
apk add luci-app-passwall2 xray-core v2ray-geoip v2ray-geosite geoview chinadns-ng tcping

# 5. Перезапуск веб-сервера интерфейса
/etc/init.d/uhttpd restart
```

---

## 3. Скрипт-парсер для нестандартных подписок

Некоторые провайдеры (например, приложения v2RayTun / Happ) защищают подписку специальными заголовками и отдают список серверов в формате JSON-массива с полными Xray-конфигами, а не стандартными Base64-ссылками. PassWall2 не умеет обрабатывать такие форматы "из коробки".

Для решения этой задачи создается локальный bash-скрипт с внедренным Lua-кодом, который скачивает JSON с нужными заголовками, преобразует его в формат VLESS-ссылок и кодирует в Base64.

1. В терминале SSH выполните команду для создания скрипта:

```bash
cat > /usr/bin/govpn-sub.sh << 'EOF'
#!/bin/sh
# Конвертер подписок GoVPN для PassWall2

# ВАША ССЫЛКА НА ПОДПИСКУ (замените на свою):
SUB_URL="https://auth.easy-api.live/****************"

# Загрузка JSON с использованием авторизационных заголовков
wget -qO /tmp/govpn.json --header="User-Agent: happ" --header="X-HWID: 1234567890" "$SUB_URL"

# Парсинг JSON и генерация VLESS ссылок
lua -e '
local jsonc = require("luci.jsonc")
local f = io.open("/tmp/govpn.json","r")
if not f then print("No JSON file"); os.exit(1) end
local t = f:read("*all")
f:close()
local cfgs = jsonc.parse(t)
if not cfgs then print("ERROR parsing JSON"); os.exit(1) end
local out = {}
for _,c in ipairs(cfgs) do
  local name = c.remarks or "unknown"
  if c.outbounds then
    for _,ob in ipairs(c.outbounds) do
      if ob.protocol=="vless" and ob.settings and ob.settings.vnext then
        local v = ob.settings.vnext[1]
        if v and v.address ~= "0.0.0.0" then
          local uid = v.users and v.users[1] and v.users[1].id or ""
          local flow = v.users and v.users[1] and v.users[1].flow or ""
          local ss = ob.streamSettings or {}
          local p = {}
          if ss.network then p[#p+1]="type="..ss.network end
          if ss.security then p[#p+1]="security="..ss.security end
          if ss.realitySettings then
            local r=ss.realitySettings
            if r.serverName then p[#p+1]="sni="..r.serverName end
            if r.fingerprint then p[#p+1]="fp="..r.fingerprint end
            if r.publicKey then p[#p+1]="pbk="..r.publicKey end
            if r.shortId then p[#p+1]="sid="..r.shortId end
          end
          if ss.tlsSettings then
            local ts=ss.tlsSettings
            if ts.serverName then p[#p+1]="sni="..ts.serverName end
            if ts.fingerprint then p[#p+1]="fp="..ts.fingerprint end
          end
          if flow~="" then p[#p+1]="flow="..flow end
          if ss.wsSettings then
            local ws=ss.wsSettings
            if ws.path then p[#p+1]="path="..ws.path end
            if ws.headers and ws.headers.Host then p[#p+1]="host="..ws.headers.Host end
          end
          out[#out+1]=string.format("vless://%s@%s:%d?%s#%s",uid,v.address,v.port,table.concat(p,"&"),name)
        end
      end
    end
  end
end
local result=table.concat(out,"\n")
io.write(result)
' > /tmp/govpn-links.txt

# Кодирование результата в Base64 для PassWall2
base64 /tmp/govpn-links.txt > /www/govpn-sub.txt
echo "Done! $(wc -l < /tmp/govpn-links.txt) servers found"
EOF

chmod +x /usr/bin/govpn-sub.sh
```

2. Выполните первичный запуск скрипта:
```bash
govpn-sub.sh
```

Для обновления подписки в будущем достаточно подключиться к роутеру по SSH, запустить команду `govpn-sub.sh`, а затем в интерфейсе PassWall2 нажать кнопку обновления подписки.

---

## 4. Настройка маршрутизации PassWall2

После того как список серверов сформирован локально, его нужно добавить в PassWall2.

1. В веб-интерфейсе перейдите в **Services** -> **PassWall2**.
2. Откройте вкладку **Node Subscribe** и создайте новую подписку.
3. В поле **Subscribe URL** укажите локальный адрес: `http://127.0.0.1/govpn-sub.txt`
4. Сохраните изменения (**Save & Apply**) и нажмите **Manual subscription**. Маршрутизатор загрузит все сконвертированные серверы.
5. На вкладке **Node List** найдите предпочтительный сервер (с наименьшим пингом) и нажмите кнопку **Use**.
6. На вкладке **Basic Settings**:
   - Включите **Main switch**.
   - Активируйте **Localhost Proxy** и **Client Proxy** (для маршрутизации трафика локальной сети через VPN).
   - Нажмите **Save & Apply**.
7. Если требуется направлять **весь трафик** через VPN без исключений, перейдите на вкладку **Access control** (или **Rule Manage**) и убедитесь, что действие по умолчанию для LAN настроено на **Proxy**, а не Direct.

> Важно: Интерфейс LuCI может отображать статус `Core NOT RUNNING` даже при корректно работающем ядре Xray. Это известная проблема отображения, которая не влияет на фактическую работу туннеля.

---

## 5. Обновление подписки с телефона (Termux)

Поскольку роутер может использоваться как портативное устройство, удобно иметь скрипт на мобильном телефоне для быстрого обновления серверов в один клик без необходимости заходить в веб-интерфейс.

На устройстве Android можно использовать терминал **Termux**.

1. Откройте Termux и установите пакеты для работы SSH:
```bash
pkg update
pkg install openssh sshpass -y
```

2. Создайте файл скрипта:
```bash
nano update_vpn.sh
```

3. Вставьте следующий код, заменив IP-адрес роутера и пароль на свои:
```bash
#!/bin/bash

ROUTER_IP="192.168.1.1" # Измените на актуальный адрес вашего роутера
ROUTER_PASS="ваш_пароль_от_роутера"

echo "Подключение к роутеру $ROUTER_IP..."

# Запуск локального скрипта-парсера на роутере
sshpass -p "$ROUTER_PASS" ssh -o StrictHostKeyChecking=no root@$ROUTER_IP "govpn-sub.sh"

echo "Подписка успешно обновлена на роутере!"
```

4. Сохраните файл (`Ctrl+O`, `Enter`, `Ctrl+X`) и сделайте его исполняемым:
```bash
chmod +x update_vpn.sh
```

5. Теперь для обновления подписки с телефона достаточно зайти в Termux и запустить:
```bash
./update_vpn.sh
```

> Рекомендация: Также можно настроить автоматическое ежедневное обновление подписки в самом роутере. Для этого перейдите в **System** -> **Scheduled Tasks** и добавьте строку `0 4 * * * /usr/bin/govpn-sub.sh`, чтобы подписка автоматически загружалась каждый день в 4 утра.

---

## 6. Настройка файлового сервера (SMB и FTP)

Все необходимые пакеты для работы с USB-накопителями (`kmod-usb-storage`, драйверы ФС, `ksmbd-server`, `vsftpd`) были интегрированы на первом шаге. Настройка сводится к нескольким кликам в веб-интерфейсе.

1. Подключите USB-накопитель к роутеру.
2. Перейдите в **System** -> **Mount Points**. Убедитесь, что диск был обнаружен и смонтирован (например, по пути `/mnt/sda1`). Активируйте чекбокс `Enabled` и сохраните настройки.
3. **Настройка SMB-сервера:**
   - Перейдите в **Services** -> **Network Shares** (Настройки ksmbd).
   - Добавьте новую директорию. Укажите произвольное имя (например, `USB`) и путь `/mnt/sda1`.
   - Для публичного доступа без пароля установите флажки `Browseable` и `Allow guests`.
   - Нажмите **Save & Apply**.
4. **Настройка FTP-сервера:**
   - Перейдите в **Services** -> **FTP Server**.
   - Укажите в качестве корневой директории `/mnt/sda1`.
   - Включите опцию анонимного доступа, если не планируете использовать строгую авторизацию.

Роутер готов к работе в качестве защищенного шлюза и домашнего файлового хранилища.
