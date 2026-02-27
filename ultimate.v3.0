#!/bin/bash
set -e

# Цвета для вывода
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

echo -e "${GREEN}=== Xray VLESS Reality ULTIMATE 3.0 (исправленная) ===${NC}"

# Проверка root
if [ "$EUID" -ne 0 ]; then
  echo -e "${RED}Ошибка: запустите скрипт от root.${NC}"
  exit 1
fi

# Обновление пакетов и установка необходимых утилит
apt update -y
apt install -y curl openssl jq qrencode cron iputils-ping bc

# Установка Xray
echo -e "${YELLOW}Устанавливаем Xray...${NC}"
bash <(curl -Ls https://raw.githubusercontent.com/XTLS/Xray-install/main/install-release.sh) install
if ! command -v xray &> /dev/null; then
  echo -e "${RED}Xray не установлен. Прерывание.${NC}"
  exit 1
fi

# Включение BBR (если ещё не включён)
if ! sysctl net.ipv4.tcp_congestion_control | grep -q bbr; then
  echo -e "${YELLOW}Включаем BBR...${NC}"
  echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
  echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
  sysctl -p
fi
echo -e "${GREEN}TCP congestion control: $(sysctl net.ipv4.tcp_congestion_control | awk '{print $3}')${NC}"

# Генерация параметров
UUID=$(cat /proc/sys/kernel/random/uuid)
KEYS=$(xray x25519)
PRIVATE_KEY=$(echo "$KEYS" | awk '/Private key/ {print $3}')
PUBLIC_KEY=$(echo "$KEYS" | awk '/Public key/ {print $3}')
SHORTID=$(openssl rand -hex 8)

# Автоматический выбор лучшего SNI по пингу
SNI_LIST=("www.cloudflare.com" "www.microsoft.com" "www.amazon.com" "www.google.com" "www.discord.com" "www.github.com" "www.zoom.us")
BEST_SNI=""
BEST_RTT=999

echo -e "${GREEN}Определяем лучший SNI по времени отклика...${NC}"
for host in "${SNI_LIST[@]}"; do
  # Пингуем 2 раза с таймаутом 1 секунда, извлекаем средний RTT
  rtt=$(LANG=C ping -c 2 -W 1 "$host" 2>/dev/null | awk -F '/' '/^rtt/ {print $5}')
  if [[ -n "$rtt" && "$rtt" != "0" ]]; then
    echo "  $host: ${rtt} ms"
    if (( $(echo "$rtt < $BEST_RTT" | bc -l) )); then
      BEST_RTT=$rtt
      BEST_SNI=$host
    fi
  else
    echo "  $host: недоступен"
  fi
done

SNI=${BEST_SNI:-${SNI_LIST[0]}}
echo -e "${YELLOW}Выбран SNI: $SNI${NC}"

# Получение внешнего IP
SERVER_IP=$(curl -4 -s ifconfig.me || curl -s ipinfo.io/ip)

# Проверка, что порты 443 и 8443 не заняты
for port in 443 8443; do
  if ss -tulpn | grep -q ":$port"; then
    echo -e "${RED}Ошибка: порт $port уже используется. Освободите его и повторите.${NC}"
    exit 1
  fi
done

# Подготовка директорий и прав
mkdir -p /usr/local/etc/xray
mkdir -p /var/log/xray
chown xray:xray /var/log/xray 2>/dev/null || true

# Создание конфигурации Xray
cat > /usr/local/etc/xray/config.json <<EOF
{
  "log": {
    "loglevel": "warning",
    "access": "/var/log/xray/access.log",
    "error": "/var/log/xray/error.log"
  },
  "inbounds": [
    {
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [{"id": "$UUID"}],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "dest": "$SNI:443",
          "serverNames": ["$SNI"],
          "privateKey": "$PRIVATE_KEY",
          "shortIds": ["$SHORTID"]
        }
      }
    },
    {
      "port": 8443,
      "protocol": "vless",
      "settings": {
        "clients": [{"id": "$UUID"}],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "quic",
        "security": "reality",
        "realitySettings": {
          "dest": "$SNI:443",
          "serverNames": ["$SNI"],
          "privateKey": "$PRIVATE_KEY",
          "shortIds": ["$SHORTID"]
        }
      }
    }
  ],
  "outbounds": [{"protocol": "freedom"}]
}
EOF

# Валидация конфига
echo -e "${YELLOW}Проверка конфигурации...${NC}"
if ! xray validate -config /usr/local/etc/xray/config.json; then
  echo -e "${RED}Конфигурация невалидна. Проверьте вручную.${NC}"
  exit 1
fi

# Запуск и включение автозагрузки
systemctl restart xray
systemctl enable xray
sleep 2

# Проверка статуса
if systemctl is-active --quiet xray; then
  echo -e "${GREEN}Xray успешно запущен.${NC}"
else
  echo -e "${RED}Xray не запустился. Проверьте логи: journalctl -u xray${NC}"
  exit 1
fi

# Настройка автоматического обновления Xray раз в неделю (воскресенье в 4:00)
(crontab -l 2>/dev/null | grep -v "xray-install"; echo "0 4 * * 0 /usr/bin/bash <(curl -Ls https://raw.githubusercontent.com/XTLS/Xray-install/main/install-release.sh) install --force") | crontab -
echo -e "${GREEN}Автообновление Xray настроено (еженедельно).${NC}"

# Формирование ссылок для подключения
LINK_TCP="vless://$UUID@$SERVER_IP:443?encryption=none&security=reality&sni=$SNI&fp=chrome&pbk=$PUBLIC_KEY&sid=$SHORTID&type=tcp#Reality-TCP"
LINK_QUIC="vless://$UUID@$SERVER_IP:8443?encryption=none&security=reality&sni=$SNI&fp=chrome&pbk=$PUBLIC_KEY&sid=$SHORTID&type=quic#Reality-QUIC"

# Вывод результатов
echo ""
echo -e "${GREEN}============================================${NC}"
echo -e "${GREEN}      УСТАНОВКА ЗАВЕРШЕНА УСПЕШНО         ${NC}"
echo -e "${GREEN}============================================${NC}"
echo ""
echo -e "${YELLOW}UUID:${NC}      $UUID"
echo -e "${YELLOW}PublicKey:${NC} $PUBLIC_KEY"
echo -e "${YELLOW}ShortID:${NC}   $SHORTID"
echo -e "${YELLOW}SNI:${NC}       $SNI"
echo -e "${YELLOW}Server IP:${NC} $SERVER_IP"
echo ""
echo -e "${YELLOW}Ссылка TCP (443):${NC}"
echo "$LINK_TCP"
echo ""
echo -e "${YELLOW}Ссылка QUIC (8443):${NC}"
echo "$LINK_QUIC"
echo ""
echo -e "${GREEN}QR-код для TCP:${NC}"
qrencode -t ansiutf8 "$LINK_TCP"
echo -e "${GREEN}QR-код для QUIC:${NC}"
qrencode -t ansiutf8 "$LINK_QUIC"
echo ""
echo -e "${GREEN}============================================${NC}"
