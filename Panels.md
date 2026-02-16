# Инструкция по созданию ВПН с маскировкой под собственный сайт на панелях

> Маскировка запросов под собственный сайт - один из самых надежных способов обезопасить свой сервер от блокировок, так как, в отличие от Vless Reality мы маскируем запросы под обращение к сайту на том же самом сервере, что и впн, а не к стороннему сайту, который расположен на сервере с другим ip. Также при прямом запросе по домену сервер отдает обычный сайт и понять что на сервере помимо вебсервера установлен еще и сервер ВПН будет очень сложно.

Здесь представлена инструкция по созданию ВПН с маскировкой под собственный сайт на панелях Marzban и 3x-ui. Инструкция для голого ядра находится [здесь](https://github.com/ServerTechnologies/xray-with-selfsni).

## VPS для создания ВПН
Для установки панели нам понадобится VPS-сервер. Приобрести его можно в [Fornex](https://fornex.com/c/ftsg5x/).

## Минимальные системные требования

- 1 CPU  
- 1 GB RAM  
- 10 GB диска  
- ОС Ubuntu 24 x64

## Подготовка.
Так как на сервере будет функционировать обычный сайт, нам обязательно нужен домен. Можно купить домен в любой зоне. Можно выбрать самый дешевый, но не забудьте про продление домена. Оно может быть существенно дороже покупки. Направьте домен на сервер, создав А запись в DNS.
- Направьте домен на сервер, создав А запись в DNS. проверить обновление DNS записей можно командой ping <имя домена>. <имя домена> надо заменить на ваш домен. В ответ вы должны молучить ip сервера.
```sh
ping <имя домена>
```

### Перед началом работы с сервером
Обовление списка репозиториев и установленных пакетов
```sh
apt update && apt upgrade -y
```

Очистить экран терминала
```sh
clear
```

## Установка панели
Установите панель с помощью этой команды:
```sh
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```
## Настройка для 3x-ui.
В первую очередь стоит подключить домен к панели и получить TLS сертификат. Для этого через меню x-ui выбираем соответствующие пункты (SSL Certificate Management или Cloudflare SSL Certificate). Чтобы избежать проблем с обновлением сертификатов, рекомендую подключать через Cloudflare.

Далее создаем инбаунд со следующими настройками в вебе:\
Протокол vless\
Порт 443\
Транспорт TCP (RAW)\
Безопасность TLS\
ALPN http/1.1\
SSL сертификат Путь к сертификату\
Установить сертификат панели\
\
В настройках клиента указываем flow: xtls-rprx-vision\
\
В Fallbacks, в поле Dest, указываем 8080\
\
Жмем создать.

Так же можно попробовать в терминале вместо веба:

импорт вашего домена
```
# Задаем переменную с доменом
export domain=vash-domen
```
создаем инбаунт
``` 
#cat > vless-tls.sh << 'SCRIPT_END'
#!/bin/bash

GREEN='\033[92m'
YELLOW='\033[93m'
RED='\033[91m'
BLUE='\033[94m'
CYAN='\033[96m'
RESET='\033[0m'
BOLD='\033[1m'

print_status() {
  local message="$1"
  local status="${2:-info}"
  local prefix=""
  case "$status" in
    success) prefix="${GREEN}[OK]" ;;
    error)   prefix="${RED}[ERR]" ;;
    warning) prefix="${YELLOW}[WARN]" ;;
    info)    prefix="${BLUE}[INFO]" ;;
    debug)   prefix="${CYAN}[DBG]" ;;
  esac
  printf "${prefix} ${message}${RESET}\n"
}

check_dependencies() {
  local missing=""
  for cmd in jq sqlite3; do
    if ! command -v "$cmd" &>/dev/null; then
      missing="$missing $cmd"
    fi
  done
  if [ -n "$missing" ]; then
    print_status "Missing dependencies:$missing" "error"
    print_status "Install via: apt install jq sqlite3 -y" "info"
    exit 1
  fi
}

setup_certificates() {
  local domain="$1"
  local cert_path="/root/cert/$domain"
  
  if [ -f "$cert_path/fullchain.pem" ] && [ -f "$cert_path/privkey.pem" ]; then
    print_status "Certificates already exist at $cert_path" "success"
    
    read -p "Reissue certificates? (y/N): " -n 1 -r
    echo
    if [[ ! $REPLY =~ ^[Yy]$ ]]; then
      print_status "Using existing certificates" "info"
      return 0
    fi
  fi
  
  print_status "Setting up SSL certificates..." "info"
  
  if [ ! -f ~/.acme.sh/acme.sh ]; then
    print_status "Installing acme.sh..." "info"
    curl -s https://get.acme.sh | sh
    source ~/.bashrc
  fi
  
  print_status "Stopping x-ui to free port 80..." "info"
  systemctl stop x-ui 2>/dev/null
  systemctl stop nginx 2>/dev/null
  
  print_status "Issuing certificate for $domain..." "info"
  
  if ~/.acme.sh/acme.sh --issue -d "$domain" --standalone --force; then
    print_status "Certificate issued successfully" "success"
  else
    print_status "Failed to issue certificate" "error"
    systemctl start x-ui 2>/dev/null
    return 1
  fi
  
  mkdir -p "$cert_path"
  
  print_status "Installing certificate..." "info"
  ~/.acme.sh/acme.sh --install-cert -d "$domain" \
    --key-file "$cert_path/privkey.pem" \
    --fullchain-file "$cert_path/fullchain.pem"
  
  if [ -f "$cert_path/fullchain.pem" ] && [ -f "$cert_path/privkey.pem" ]; then
    print_status "Certificates installed to $cert_path" "success"
    systemctl start x-ui 2>/dev/null
    sleep 2
    return 0
  else
    print_status "Failed to install certificates" "error"
    systemctl start x-ui 2>/dev/null
    return 1
  fi
}

find_database() {
  local paths=(
    "/etc/x-ui/x-ui.db"
    "/opt/etc/x-ui/x-ui.db"
    "/opt/x-ui/x-ui.db"
    "/usr/local/x-ui/bin/x-ui.db"
    "./x-ui.db"
  )
  
  for path in "${paths[@]}"; do
    if [ -f "$path" ]; then
      print_status "Database found: $path" "success" >&2
      echo "$path"
      return 0
    fi
  done
  return 1
}

configure_panel_certificates() {
  local db_path="$1"
  local domain="$2"
  
  print_status "Configuring panel and subscription SSL certificates..." "info"
  
  local cert_path="/root/cert/$domain/fullchain.pem"
  local key_path="/root/cert/$domain/privkey.pem"
  
  # Настройки веб-панели
  sqlite3 "$db_path" "INSERT OR REPLACE INTO settings (key, value) VALUES ('webCertFile', '$cert_path');" 2>/dev/null
  sqlite3 "$db_path" "INSERT OR REPLACE INTO settings (key, value) VALUES ('webKeyFile', '$key_path');" 2>/dev/null
  
  # Настройки подписки (subscription)
  sqlite3 "$db_path" "INSERT OR REPLACE INTO settings (key, value) VALUES ('subCertFile', '$cert_path');" 2>/dev/null
  sqlite3 "$db_path" "INSERT OR REPLACE INTO settings (key, value) VALUES ('subKeyFile', '$key_path');" 2>/dev/null
  
  if [ $? -eq 0 ]; then
    print_status "Panel and subscription certificates configured" "success"
    return 0
  else
    print_status "Failed to configure certificates" "warning"
    return 1
  fi
}

insert_inbound_to_db() {
  local db_path="$1"
  local domain="$2"
  
  local settings
  settings=$(jq -nc --arg domain "$domain" '{
    "clients": [],
    "decryption": "none",
    "encryption": "none",
    "fallbacks": [{
      "alpn": "http/1.1",
      "dest": "8080",
      "name": $domain,
      "path": "/",
      "xver": 0
    }]
  }')
  
  local stream_settings
  stream_settings=$(jq -nc --arg domain "$domain" '{
    "network": "tcp",
    "security": "tls",
    "tcpSettings": {
      "acceptProxyProtocol": false,
      "header": {"type": "none"}
    },
    "tlsSettings": {
      "alpn": ["http/1.1"],
      "certificates": [{
        "buildChain": false,
        "certificateFile": ("/root/cert/" + $domain + "/fullchain.pem"),
        "keyFile": ("/root/cert/" + $domain + "/privkey.pem"),
        "oneTimeLoading": false,
        "usage": "encipherment"
      }],
      "cipherSuites": "",
      "disableSystemRoot": false,
      "echForceQuery": "none",
      "echServerKeys": "",
      "enableSessionResumption": false,
      "maxVersion": "1.3",
      "minVersion": "1.2",
      "rejectUnknownSni": false,
      "serverName": $domain
    }
  }')
  
  local sniffing
  sniffing=$(jq -nc '{
    "enabled": true,
    "destOverride": ["http", "tls", "quic", "fakedns"],
    "metadataOnly": false,
    "routeOnly": true
  }')
  
  local existing
  existing=$(sqlite3 "$db_path" "SELECT id, remark FROM inbounds WHERE port=443;" 2>/dev/null)
  
  if [ -n "$existing" ]; then
    print_status "Inbound on port 443 already exists:" "warning"
    print_status "  $existing" "warning"
    read -p "Replace it? (y/N): " -n 1 -r
    echo
    if [[ ! $REPLY =~ ^[Yy]$ ]]; then
      print_status "Cancelled by user" "info"
      return 1
    fi
    sqlite3 "$db_path" "DELETE FROM inbounds WHERE port=443;" 2>/dev/null
    print_status "Old inbound deleted" "success"
  fi
  
  local settings_escaped="${settings//\'/\'\'}"
  local stream_settings_escaped="${stream_settings//\'/\'\'}"
  local sniffing_escaped="${sniffing//\'/\'\'}"
  
  local sql="INSERT INTO inbounds (
    user_id, up, down, total, all_time, remark, enable, expiry_time,
    traffic_reset, last_traffic_reset_time, listen, port, protocol,
    settings, stream_settings, tag, sniffing
  ) VALUES (
    1, 0, 0, 0, 0, 'VLESS TLS Self-SNI', 1, 0,
    'never', 0, '0.0.0.0', 443, 'vless',
    '$settings_escaped',
    '$stream_settings_escaped',
    'inbound-443',
    '$sniffing_escaped'
  );"
  
  if sqlite3 "$db_path" "$sql" 2>/dev/null; then
    print_status "VLESS TLS inbound added to database" "success"
    
    local new_id
    new_id=$(sqlite3 "$db_path" "SELECT id FROM inbounds WHERE port=443 ORDER BY id DESC LIMIT 1;" 2>/dev/null)
    print_status "Inbound ID: $new_id" "debug"
    
    return 0
  else
    print_status "Failed to insert inbound (SQL error)" "error"
    return 1
  fi
}

setup_nginx() {
  local domain="$1"
  
  print_status "Setting up Nginx fallback..." "info"
  
  if ! command -v nginx &>/dev/null; then
    print_status "Nginx not found, installing..." "info"
    apt update -qq
    apt install nginx -y
  fi
  
  cat > /etc/nginx/sites-available/default << NGINX_EOF
server {
        listen 80;
        server_name $domain;
        return 301 https://\$http_host\$request_uri;
}

server {
        listen 127.0.0.1:8080;
        server_name $domain;
        root /var/www/html/;
        index index.html;
        add_header Strict-Transport-Security "max-age=63072000" always;
}
NGINX_EOF
  
  print_status "Downloading HTML template..." "info"
  if command -v wget &>/dev/null; then
    wget -q -O /var/www/html/index.html https://raw.githubusercontent.com/pegakmop/xray-with-selfsni/refs/heads/main/var/www/html/index.html
  elif command -v curl &>/dev/null; then
    curl -s -o /var/www/html/index.html https://raw.githubusercontent.com/pegakmop/xray-with-selfsni/refs/heads/main/var/www/html/index.html
  else
    print_status "Neither wget nor curl found, skipping HTML download" "warning"
  fi
  
  if [ -f /var/www/html/index.html ]; then
    print_status "HTML template installed" "success"
  fi
  
  if systemctl restart nginx 2>/dev/null; then
    print_status "Nginx configured and restarted" "success"
    return 0
  else
    print_status "Failed to restart Nginx" "error"
    return 1
  fi
}

restart_xui_service() {
  print_status "Restarting x-ui service..." "info"
  
  if command -v x-ui &>/dev/null; then
    if x-ui restart &>/dev/null; then
      sleep 2
      print_status "x-ui service restarted" "success"
      return 0
    fi
  fi
  
  if systemctl restart x-ui 2>/dev/null; then
    sleep 2
    print_status "x-ui service restarted" "success"
    return 0
  fi
  
  print_status "Failed to restart service automatically" "error"
  print_status "Restart manually: x-ui restart" "info"
  return 1
}

main() {
  printf "\n${BOLD}=== 3x-ui VLESS TLS Inbound Auto-Setup Tool ===${RESET}\n\n"
  
  check_dependencies
  
  local domain="$1"
  
  if [ -z "$domain" ]; then
    print_status "Usage: $0 <domain>" "error"
    print_status "Example: $0 example.com" "info"
    exit 1
  fi
  
  print_status "Domain: $domain" "info"
  
  if ! setup_certificates "$domain"; then
    print_status "Certificate setup failed" "error"
    exit 1
  fi
  
  local db_path
  db_path=$(find_database)
  
  if [ -z "$db_path" ]; then
    print_status "Database not found" "error"
    exit 1
  fi
  
  local backup_path="${db_path}.backup.$(date +%Y%m%d_%H%M%S)"
  cp "$db_path" "$backup_path"
  print_status "Backup created: $backup_path" "success"
  
  print_status "Adding VLESS TLS inbound to database..." "info"
  
  if ! insert_inbound_to_db "$db_path" "$domain"; then
    print_status "Failed to add inbound" "error"
    exit 1
  fi
  
  configure_panel_certificates "$db_path" "$domain"
  
  printf "\n${BOLD}Verifying inbound:${RESET}\n"
  sqlite3 "$db_path" "SELECT id, remark, port, protocol, tag FROM inbounds WHERE port=443;" | while IFS='|' read -r id remark port protocol tag; do
    print_status "ID: $id | Name: $remark | Port: $port | Protocol: $protocol | Tag: $tag" "success"
  done
  
  printf "\n"
  restart_xui_service
  
  printf "\n"
  setup_nginx "$domain"
  
  printf "\n"
  print_status "Setup completed successfully!" "success"
  printf "\n"
  print_status "${BOLD}Summary:${RESET}" "info"
  print_status "  ✓ SSL certificates configured" "success"
  print_status "  ✓ VLESS TLS inbound added on port 443" "success"
  print_status "  ✓ Panel SSL certificates configured" "success"
  print_status "  ✓ Subscription SSL certificates configured" "success"
  print_status "  ✓ Domain: $domain" "success"
  print_status "  ✓ Nginx fallback configured on port 8080" "success"
  print_status "  ✓ HTML template installed" "success"
  print_status "  ✓ Website: https://$domain" "success"
  printf "\n"
  print_status "${BOLD}Next steps:${RESET}" "info"
  print_status "  1. Open 3x-ui web panel" "info"
  print_status "  2. Add clients to 'VLESS TLS Self-SNI' inbound" "info"
  print_status "  3. Configure clients with flow: xtls-rprx-vision" "info"
  printf "\n"
}

main "$@"
SCRIPT_END

chmod +x vless-tls.sh
```
потом запустить ./vless-tls.sh vash-domen где последнее ваш домен(если скриптом то нгинх автоматом установит и настроит)

### Установка Nginx, создание сайта

Устанавливаем Nginx
```sh
sudo apt install nginx -y
```

Задаем переменную с именем вашего домена, замените "vash-domen" на ваше доменное имя. Указывайте только имя домена, без http:// или https://
```sh
export domain=vash-domen
```

Заменяем дефолтный файл конфигурации.
```sh
cat << EOF > /etc/nginx/sites-available/default
server {
        listen 80;
        server_name $domain;
        return 301 https://\$http_host\$request_uri;
       }


server {
        listen 127.0.0.1:8080;
        server_name $domain;
        root /var/www/html/;
        index index.html;
        add_header Strict-Transport-Security "max-age=63072000" always;
}
EOF
mv /var/www/html/index.nginx-debian.html /var/www/html/index.html
systemctl restart nginx
```

Можете поставить главной страницей шаблон предложенный мною установка одной командой
```
sudo bash -c 'if command -v wget &> /dev/null; then echo "Используется: wget" && wget -O /var/www/html/index.html https://raw.githubusercontent.com/pegakmop/xray-with-selfsni/refs/heads/main/var/www/html/index.html && echo "Файл успешно скачан в /var/www/html/index.html"; elif command -v curl &> /dev/null; then echo "Используется: curl" && curl -o /var/www/html/index.html https://raw.githubusercontent.com/pegakmop/xray-with-selfsni/refs/heads/main/var/www/html/index.html && echo "Файл успешно скачан в /var/www/html/index.html"; else echo "Ошибка: ни wget, ни curl не установлены"; exit 1; fi'

```

Готово.



## Настройка для Marzban.

### Установка Nginx
В первую очередь устанавливаем Nginx
```sh
apt install nginx -y
```

### Установка Marzban

Установка Marzban описана [здесь](https://github.com/ServerTechnologies/youtube-instructions/blob/main/marzban-settings.md)

Разница заключается только в команде получения сертификата. В пункте "Создание SSL сертификатов", в разделе "И запускаем утилиту, которая создаст сертификат:" вместо указанной там команды для получения сертификатов стоит использовать эти команды:

Задаем переменную с именем нащего домена, замените <имя домена> на ваш домен:
```sh
export DOMAIN=<имя домена>
```

И запускаем команду создания сертификата:

```sh
mkdir -p /var/lib/marzban/certs

~/.acme.sh/acme.sh \
  --issue --force -w /var/www/html -d "$DOMAIN" \
  --fullchain-file "/var/lib/marzban/certs/$DOMAIN.cer" \
  --key-file "/var/lib/marzban/certs/$DOMAIN.cer.key"
```

Остальные пункты инструкции выполняются как есть, заменяем только одну команду для получение сертификатов.

### Создание сайта

Задаем переменную с вашим доменом. Замените "vash-domen" на ваше доменнное имя.
```sh
export domain=vash-domen
```

Заменяем дефолтный файл конфигурации Nginx.
```sh
cat << EOF > /etc/nginx/sites-available/default
server {
        listen 80;
        server_name $domain;
        return 301 https://\$http_host\$request_uri;
       }


server {
        listen 127.0.0.1:8080;
        server_name $domain;
        root /var/www/html/;
        index index.html;
        add_header Strict-Transport-Security "max-age=63072000" always;
}
EOF
mv /var/www/html/index.nginx-debian.html /var/www/html/index.html
systemctl restart nginx
```

### Настройка инбаундов
Настройка инбаундов производится через иконку настроек в правом верхнем углу экрана (иконка в виде шестеренки). Туда, в inbounds, необходимо добавить слудующий блок:

```sh
{
    "tag": "VLESS TCP TLS",
    "listen": "0.0.0.0",
    "port": 443,
    "protocol": "vless",
    "settings": {
    "clients": [],
    "decryption": "none",
    "fallbacks": [
        {
        "dest": 8080
        }
    ]
    },
    "streamSettings": {
    "network": "tcp",
    "security": "tls",
    "tlsSettings": {
        "alpn": "http/1.1",
        "certificates": [
        {
            "certificateFile": "/var/lib/marzban/certs/vash-domen.cer",
            "keyFile": "/var/lib/marzban/certs/vash-domen.cer.key"
        }
        ],
        "minVersion": "1.2",
        "cipherSuites": "TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256:TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256:TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384:TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384:TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256:TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256"
    }
}
}
```

В **certificateFile** и **keyFile** необходимо указать путь до папок в которые установлены сертификаты к вашему домену.\
После чего нужно нажать **"Сохранить"**, **"Перезагрузить ядро"** и обновить страницу в браузере.\
Далее в правом верхнем углу иконка в виде трех тире (бургер меню) откройте настройки хоста, выбирите соответствующий инбаунд (VLESS TCP TLS), замените **{SERVER_IP}** на ваш домен, далее в дополнительных опциях надо выставить **ALPN** равный **http/1.1** и выбрать **fingerprint**. Для сохранения настроек нажмите **"Применить"**. Обновите страницу.\
\
Далее можно создавать подключения чрез интерфейс. При создании подключения выберите "VLESS TCP TLS"

## Работа с файлами сайта
Xтобы удалить дефолтную страницу
```sh
rm /var/www/html/index.html
```

Или можно открыть дефолтную страницу, используя консольный текстовый редактор, чтобы вставить другой текст
```sh
nano /var/www/html/index.html
```
Чтобы сохранить документ и выйти из текстового редактора, нажмите CTRL + X Y Enter\

Адрес папки, куда надо закинуть файлы сайта. Главный файл сайта должен называться index.html
```sh
/var/www/html/
```

Чтобы открыть папку с файлами сайта
```sh
cd /var/www/html/index.html
```

Установить unzip
```sh
apt install unzip
```

Разархивировать zip архив. Замените <имя архива.zip> на имя архива
```sh
unzip <имя архива.zip>
```

Скопировать файлы из папки с шаблоном в папку с сайтом. Тут я заменил "." на абсолютный путь. Замените <имя папки> на имя папки с шаблоном
```sh
cp -r <имя папки>/* /var/www/html/index.html
```

Можете поставить главной страницей шаблон предложенный мною установка одной командой
```
sudo bash -c 'if command -v wget &> /dev/null; then echo "Используется: wget" && wget -O /var/www/html/index.html https://raw.githubusercontent.com/pegakmop/xray-with-selfsni/refs/heads/main/var/www/html/index.html && echo "Файл успешно скачан в /var/www/html/index.html"; elif command -v curl &> /dev/null; then echo "Используется: curl" && curl -o /var/www/html/index.html https://raw.githubusercontent.com/pegakmop/xray-with-selfsni/refs/heads/main/var/www/html/index.html && echo "Файл успешно скачан в /var/www/html/index.html"; else echo "Ошибка: ни wget, ни curl не установлены"; exit 1; fi'

```



