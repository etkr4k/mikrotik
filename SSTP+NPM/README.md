# Настройка SSTP и Nginx Reverse Proxy
Для работы NPM требуется настроить переадресацию 80 и 433 порта с роутера на прокси-сервер.
В моей конфигурации используется следующая настройка в ip/firewall/nat

### Netmap из WAN на NPM
`/ip/firewall/nat add chain=dstnat action=netmap to-addresses=IP-NPM-SERVER protocol=tcp in-interface-list=WAN dst-port=443,80 log=no log-prefix=""`
### Доступ из LAN к NGINX
`/ip/firewall/nat add chain=srcnat action=src-nat to-addresses=IP-ROUTER protocol=tcp dst-address=IP-NPM-SERVER src-address-list=lan-network dst-port=443,80 log=no log-prefix=""`
### Доступ из WAN к NGINX
`/ip/firewall/nat add chain=dstnat action=netmap to-addresses=IP-NPM-SERVER protocol=tcp dst-address=PUBLIC-IP src-address-list=lan-network dst-port=443,80 log=no log-prefix=""`

Для корректной и стойкой работы SSTP мы будем настраивать внешний порт 443, а внутренний сменим на 8443

## Настройка SSTP сервера

### Настраиваем pool для SSTP
`/ip pool add name=sstp_pool ranges=10.20.0.2-10.20.0.6`
### Создаем профиль
`/ppp profile add local-address=10.20.0.1 name=SSTP-profile remote-address=sstp-pool`
### Создаем пользователя
`/ppp secret add name=nsername password=Pa$$word profile=SSTP-profile service=sstp remote-address=10.20.0.2`
### Создадим интерфейс
`/interface sstp-server server set authentication=mschap2 certificate=none default-profile=SSTP-profile port=8443 enabled=yes pfs=yes tls-version=only-1.2`
### На этом настройка сервера почти закончена, осталось добавить сертификат, это сделаем чуть позже

## Перейдем к донастройке NAT и правил
Так как мы хотим осуществлять соедениение по 443 порту, а на нем у нас изначально работает NPM, мы создадим фильтрацию трафика.

Смысл следующий, мы создаем новый домен, к примеру vpn.domain.com и прописываем A запись на наш внешний IP.
Далее создаем в Mangle правило, которое при запросе из WAN на домен vpn.domain.com будет добавлять IP в адрес лист SSTP-List на 1 минуту.
При повторном подключение с этого IP в течении минуты на порт 443 мы будем переадресованы на IP роутера с портом SSTP - 8443.
После соединения, через минуту на IP будет удален из списка и за обычные запросы https на порт 443 будет снова отвечать NPM.

### Для начала создадим правило Mangle
`/ip/firewall/mangle add chain=prerouting action=add-src-to-address-list protocol=tcp address-list=SSTP-List address-list-timeout=1m dst-port=443 content=vpn.domain.com log=no log-prefix=""`
### Добавим правило форварда на 8443 для адрес листа SSTP-List
`/ip/firewall/nat add chain=dstnat action=dst-nat to-ports=8443 protocol=tcp dst-address-type=local src-address-list=SSTP-List dst-port=443 log=no log-prefix=""`
### Изменим правило Netmap для доступа из WAN к NPM
`/ip/firewall/nat add chain=dstnat action=netmap to-addresses=IP-NPM-SERVER protocol=tcp src-address-list=!SSTP-List in-interface-list=WAN dst-port=80,443 log=no log-prefix="" 

## Донастраиваем SSTP
Осталось сгенерировать сертификат и указать его в SSTP. Для получения сертификата необходимо отключить правила NAT для NMP
### Создаем правило открывающий 80 порт для генерации
`/ip/firewall/filter add chain=input action=accept protocol=tcp dst-port=80 log=no log-prefix=""
### Включаем сервис WWW
`/ip service enable [find name="www"]`
### Генерируем сертификат
/certificate enable-ssl-certificate dns-name=vpn.domain.com`
### Отключаем сервис WWW
`/ip service disable [find name="www"]`
### Незабываем отключить 80 порт
`ip/firewall/filter/ disable numbers="номер правила"`
### Назначаем сертификат для SSTP
`/interface sstp-server server set certificate="lets"

## Обновление сертификата
Сертификат от Let`s Encrypt действует в течении 90 дней, так что стоит настроить автоматические обновление.

Скрипт находится в этом репозитории в директории /script, его надо настроить под себя

### Скрипт надо добавить в задание планировщика:
`/system scheduler add interval=9w name=letsencrypt-scheduled-renew on-event=letsencrypt-renew policy=read,write start-date=feb/1/2023 start-time=04:00:00`
