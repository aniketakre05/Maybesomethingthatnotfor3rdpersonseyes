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
int g1							// Определяем nat на всех интерфейсах
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

# Настройка OSPF
conf t
router ospf 1
router-id 2.2.2.2
network 192.168.0.0 255.255.0.0 area 0
network 10.5.5.0 255.255.255.252 area 0
exit
ip ospf mtu-ignore					// Игнорирование размера пакета, дабы не упал OSPF из-за IPSEC
do wr
end

!Проверить, что нет ip ospf network broadcast в конфиге!

# Настройка IPSEC
conf t
crypto isakmp policy 1					// isakmp = ike
hash sha						// ХЭШ sha
encryption 3des						// Шифрование 3des, шифрование ключа 
authentication pre-share				// Используем ключ
group 14						// Группа Деффи-Хеллмана (2048)
exit
crypto isakmp key WSR-2022 address 200.100.200.100
crypto ipsec transform-set gre esp-aes 128 esp-sha256-hmac 	// esp, шифрование данных
mode tunnel
exit
crypto ipsec profile gre 				// Создаем IPSEC профиль GRE
set transform-set gre 					// Привязываем к IPSEC
exit
int tun1
tunnel protection ipsec profile gre
do wr
end
