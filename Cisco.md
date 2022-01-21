# Задаем IP адреса интерфейсам

# Указываем маршрут внутренних устройств до ISP (Маршрут последней надежды)
conf t
ip route 0.0.0.0 0.0.0.0 200.100.100.254
end
!do show ip route - показывает таблицу маршрутизации!

# Настройка листов доступа(ACL) и NAT
conf t
ip access-list standard 1
permit 192.168.0.0 0.0.255.255
exit
ip nat inside source list 1 interface g1 overload 	// Правило для НАТа из внутренней сети,list 1 - номер ACL, 
							// интерфейс g1 - интерфейс выхода на ISP, overlord - трансляция не в адрес в адрес, а порт в порт
int g1
ip nat outside
int g2
ip nat inside
int g3
ip nat inside
end

# Настройка GRE туннеля
conf t
int tun1
tunnel source 200.100.100.100 				// Адрес внешнего IP CSR
tunnel destination 200.100.200.100			// Адрес внешнего IP FW
ip address 10.5.5.2 255.255.255.252			// Адрес туннеля CSR
end

#
