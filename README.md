# BGP-mikrotik + DOH
Настройка BGP для обхода блокировок  
Ориентироваться можно на инструкцию https://habr.com/ru/post/549282/  
Нужен VPS с wireguard  
Настраиваем интерфейс и Peer  
#Добавляем интерфейс wireguard 
/interface wireguard add listen-port=50954 mtu=1420 name=wireguard1  
#Добавляем Peer  
/interface wireguard peers add allowed-address=0.0.0.0/0 endpoint-address=******** endpoint-port=50954 interface=wireguard1 public-key="***********"  
#Добавляем адрес для подключения к wireguard из конфига  
/ip address add address=10.66.66.25/24 interface=wireguard1 network=10.66.66.0  
#Добавляем wireguard1 в интерфейсы WAN  
/interface list member add interface=wireguard1 list=WAN  
#Добавляем таблицу маршрутизации для ручной маркировки трафика  
/routing table add disabled=no fib name=wg_mark  
#Добавляем правило для NAT - masquerade трафика из Сети WAN и wireguard1  
/ip firewall nat add action=masquerade chain=srcnat out-interface-list=WAN  
/ip firewall nat add action=masquerade chain=srcnat disabled=yes out-interface=wireguard1  
#Добавляем маршрут для проброса BGP через wireguard1  
/ip route add dst-address=45.154.73.71/32 gateway=wireguard1  
#Добавляем маршрут для точечной маршрутизации адресов по списку  
/ip route add dst-address=0.0.0.0/0 gateway=wireguard1  
#Добавляем шаблон для BGP туннеля. 64988 берётся случайно как идентификатор сети.
/routing bgp template add as=64988 disabled=no hold-time=4m input.filter=bgp_in_wg ignore-as-path-len=yes keepalive-time=1s multihop=yes name=antifilter_wg routing-table=main  

#Добавляем фильтр для BGP туннеля. В правиле указываем ""set gw-interface wireguard1;accept""
/routing filter rule  
add chain=bgp_in_wg disabled=no rule="set gw-interface *0x8;accept"  
#Добавляем BGP туннель.  
/routing bgp connection  
add as=64988 disabled=no hold-time=4m input.filter=bgp_in_wg \  
    .ignore-as-path-len=yes keepalive-time=1s local.address=10.66.66.25 \  
    .role=ebgp multihop=yes name=bgp_wg remote.address=45.154.73.71/32 .as=\  
    65432 router-id=77.37.178.210 routing-table=main templates=antifilter  


/ip firewall nat add action=redirect chain=dstnat dst-port=53 in-interface-list=LAN protocol=udp 
#Готовим редирект запорсов DNS из сети LAN для DOH
