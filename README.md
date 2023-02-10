# Antifilter BGP-mikrotik + DOH  
**Настройка BGP для обхода блокировок  
Ориентироваться можно на инструкцию**  
https://habr.com/ru/post/549282/  
https://xn----7sba7aachdbqfnhtigrl.xn--j1amh/mikrotik-doh-nastrojka-dns-over-https/  
https://antifilter.download/  
 
# Для работы необходим wireguard или другой VPN. Быстрый способ установки - использовать скрипт.  
curl -O https://raw.githubusercontent.com/angristan/wireguard-install/master/wireguard-install.sh  
chmod +x wireguard-install.sh  
./wireguard-install.sh  


# После успешной установки настраиваем интерфейс и Peer на mikrotik  

**#Добавляем интерфейс wireguard**  
/interface wireguard add listen-port=50954 mtu=1420 name=wireguard1  

**#Добавляем Peer**  
/interface wireguard peers add allowed-address=0.0.0.0/0 endpoint-address=******** endpoint-port=50954 interface=wireguard1 public-key="***********"  

**#Добавляем адрес для подключения к wireguard из конфига**  
/ip address add address=10.11.11.25/24 interface=wireguard1 network=10.11.11.0  

**#Добавляем wireguard1 в интерфейсы WAN**  
/interface list member add interface=wireguard1 list=WAN  

**#Добавляем таблицу маршрутизации для ручной маркировки трафика. Будет необходима позднее, если какие-то сайты будут попадать по ошибке не в те маршруты. Списки маршрутов идут с маской 20-24, могут быть редкие ошибки. В adress lists можно руками добавлять сайты (доменами, но домен через * не работает, только полные доменные имена или сети с нужной маской) и выбирать таблицу маршрутизации, RT или wg_mark. Через RT выход будет через интерфейс провайдера, через wg_mark - через wireguard1.**  
/routing table add disabled=no fib name=wg_mark  
/routing table add disabled=no fib name=RT  

**#Добавляем правило для NAT - masquerade трафика из Сети WAN и wireguard1**  
/ip firewall nat add action=masquerade chain=srcnat out-interface-list=WAN  

**#Добавляем маршрут для проброса BGP через wireguard1**  
/ip route add dst-address=45.154.73.71/32 gateway=wireguard1  

**#Добавляем маршрут для точечной маршрутизации адресов по списку. Первый маршрут будет отправлять адреса из адрес листа с маркировкой wg_mark через wireguard, а второй для маркировки RT через провайдера.**  
/ip route add comment=WG disabled=no distance=1 dst-address=0.0.0.0/0 gateway=wireguard1 pref-src="" routing-table=wg_mark scope=30 suppress-hw-offload=no target-scope=10  
/ip route add comment=RT disabled=no distance=1 dst-address=0.0.0.0/0 gateway=77.37.178.1 pref-src=0.0.0.0 routing-table=RT scope=30 suppress-hw-offload=no target-scope=10  

**#Добавляем mangle для маркировки пакетов ручной маршрутизации адресов**  
/ip firewall mangle add action=mark-routing chain=prerouting dst-address-list=wg_list new-routing-mark=wg_mark  
/ip firewall mangle add action=mark-routing chain=prerouting dst-address-list=RT new-routing-mark=RT  

**#Добавляем шаблон для BGP туннеля. 65000 берётся случайно как идентификатор сети.**
/routing bgp template add as=65000 disabled=no hold-time=4m input.filter=bgp_in_wg ignore-as-path-len=yes keepalive-time=1s multihop=yes name=antifilter_wg routing-table=main  

**#Добавляем фильтр для BGP туннеля. В правиле указываем "set gw-interface wireguard1;accept"**  
/routing filter rule add chain=bgp_in_wg disabled=no rule="set gw-interface wireguard1;accept"  

**#Добавляем BGP туннель.  
45.154.73.71/32 - Адрес сервера со списком маршрутов, берётся в гугле. 61432 - Идентификатор сети BGP, используется так же из инструкции на сайте из шапки. 65000 - Индентификатор нашего подключения, указывали выше в шаблоне. router-id= Внешний адрес маршрутизатора для регистрации в сети, можно использовать любой, но возможны петли.**  
/routing bgp connection  
add as=65000 disabled=no hold-time=4m input.filter=bgp_in_wg \  
    .ignore-as-path-len=yes keepalive-time=1s local.address=//АДРЕС НАШЕГО ВПН// \  
    .role=ebgp multihop=yes name=bgp_wg remote.address=45.154.73.71/32 .as=\  
    65432 router-id=//ВНЕШНИЙ АДРЕС МАРШРУТИЗАТОРА// routing-table=main templates=antifilter_wg  

**#Готовим редирект запорсов DNS из сети LAN для DOH**  
/ip firewall nat add action=redirect chain=dstnat dst-port=53 in-interface-list=LAN protocol=udp  

**Импорт сертификат DigiCert Global Root CA в хранилище сертификатов роутера:**  
/tool fetch url=https://cacerts.digicert.com/DigiCertGlobalRootCA.crt.pem  
/certificate import file-name=DigiCertGlobalRootCA.crt.pem passphrase=""  

**Указать использование DoH Server. В поле Cache Size лучше поставить значение 4096**  
/ip dns set allow-remote-requests=yes cache-size=4096KiB use-doh-server=https://1.1.1.1/dns-query verify-doh-cert=yes  

**#Указать MikroTik в качестве DNS сервера. Адреса сети использовать свои**
/ip dhcp-server network add address=192.168.1.0/24 **dns-server=192.168.1.1** gateway=192.168.1.1


**НАСТРОЙКА ЗАВЕРШЕНА**

**#Я всё настроил, а мой любимый ресурс все равно блокируется. При этом подходящий префикс для его адреса в списке есть, но nslookup выдает другой адрес из сети моего провайдера**  

**#Вероятно, ваш оператор связи использует многоуровневую систему блокировки контента, в том числе перехватывающую DNS-запросы с соответствующей коррекцией ответа. В этом случае вам может помочь перенаправление DNS в VPN или более интеллектуальные способы решения, описанные в частности в статье Переводим на DoH домашнюю сеть.**  

**Нужно создать руками маршрут и выбрать Distance 1.**  
**Отсделить нужные хосты можно через консоль GoogleChrome/network**  

/ip firewall address-list add address=pikabu.ru list=RT  
# Будет выходить через шлюз провайдера  
/ip firewall address-list add address=chat.openai.com list=wg_list  
# Будет выходить через WG  

