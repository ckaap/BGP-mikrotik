# BGP-mikrotik + DOH
Настройка BGP для обхода блокировок  
Ориентироваться можно на инструкцию https://habr.com/ru/post/549282/  
Нужен VPS с wireguard  
Настраиваем интерфейс и Peer  

/ip firewall nat add action=redirect chain=dstnat dst-port=53 in-interface-list=LAN protocol=udp  
/ip firewall nat add action=masquerade chain=srcnat out-interface-list=WAN  
/ip firewall nat add action=masquerade chain=srcnat disabled=yes out-interface=wireguard1  
/ip route add dst-address=45.154.73.71/32 gateway=wireguard1
/ip route add dst-address=0.0.0.0/0 gateway=wireguard1 #Этот маршрут нужно добавить для возможности таргетного перенаправления сайтов через вкладку Firewall/Address List  
