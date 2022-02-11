# WS107
# Если нет НАТа на ISP
iptables -t nat -A POSTROUTING -s 0.0.0.0/0 -о ens192 -j MASQUERADE

# Настройка пропуска DNS на FW для внешних клиентов
iptables -t nat -A PREROUTING -i ens192 -p udp -m udp --dport 53 -j DNAT --to-destination 172.20.30.10
iptables-save > /etc/iptables/rules.v4
//Далее на ISP
zone "msk" {
  type master;
  file "/etc/bind/db.local";
  forwarders { 200.100.200.100; };
};

# Донастройка IPTABLES на FW 
iptables -t nat -A POSTROUTING -s 172.20.0.0/16 -o ens192 -j MASQUERADE

# Настройка дочернего DNS на SRV-1 и SRV-2
apt install bind9
Конфигурация в /etc/bind/named.conf.default-zones
zone "company.msk" {
  type slave;
  masters { 172.20.10.100; }; //Указываем IP машины DC
  file "/opt/dns/company.msk;
};

zone "_msdcs.company.msk" {
  type slave;
  masters { 172.20.10.100; };
  file "/opt/dns/msdcs.msk;
};

Добавляем папки указанные в путях выше и выдаем права
mkdir /opt/dns/
chown bind:bind /opt/dns -R или chmod 777 /opt/dns -R

Конфигурация в /etc/bind/named.conf.options
forwarders {
  200.100.100.254 //IP родительского DNS (Машина DC)
};
dnssec-validation no;
recursion yes;
allow-query { any; };
listen-on { any; };

Перезапускаем DNS
systemctl restart bind9

Отключаем apparmor
systemctl disable --now apparmor

С SRV-2 тоже самое, но меняем IP в masters в конфиге names.conf.default-zones

!Для проверки конфигураций есть команда - named-checkconf!

# Настройка подключения к ISCSI
apt install open-iscsi
vim /etc/iscsi/iscsid.conf
Раскомментируем строку node.startup = automatic		//Автозапуск соединения с ISCSI
Комментируем строку node.startup = manual
Раскомментируем строку node.session.auth.authmethon = CHAP
Раскомментируем строку и редактируем node.session.auth.username = <Логин ISCSI>
node.session.auth.password = <Пароль ISCSI>
:wq!
systemctl restart iscsi
Команда для нахождения iqn iscsi:
iscsiadm -m discovery -t sendtargets -p <ip>
Команда для коннекта:
iscsiadm -m node --targetname "<iqn>" --portal "<ip>" --login
mkfs.ext4 /dev/sdb
vim /etc/fstab
Пишем:
/dev/sdb	/mnt	ext4	defaults	0 0
:wq!
mount -a

!На винде не забудь настроить права доступа!
	
# Настройка отказоустойчивости с помощью пакета keepalived на SRV-1 и SRV-2
apt install keepalived
В конфигурации /etc/keepalived/keepalived.conf
global_defs {
  enables_script_security
  script_user root
}

vrrp_script dns_check {
  script "/usr/keepalived/dns_check" //Место скрипта
  interval 2 //Интервалы отправки пакетов
  timeout 2 //Таймаут на получение пакета
  rise 1 //Требуемое количество принятых пакетов
  fall 2 //Максимальное количество потерь пакетов
}

vrrp_script ntp_check {
  script "/usr/keepalived/ntp_check"
  interval 2
  timeout 2
  rise 1
  fall 2
}

vrrp_instance DNS {
  state MASTER //Указываем, что эта машина является основной(иначе BACKUP, если является резервной)
  interface ens192
  virtual_router_id 53
  advert_int 1 //Интервал проверки серверов
  priority 10 //Выше значение - выше приоритет
  virtual_ipaddress {
    172.20.30.10/24
  }
  track_interface { //Проверка состояния интерфейса
    ens192
  }
  track_script { //Какой скрипт будет использован для проверки
    dns_check
  }
}

vrrp_instance NTP {
  state MASTER //Указываем, что эта машина является основной(иначе BACKUP, если является резервной)
  interface ens192
  virtual_router_id 123
  advert_int 1 //Интервал проверки серверов
  priority 10 //Выше значение - выше приоритет
  virtual_ipaddress {
    172.20.30.15/24
  }
  track_interface { //Проверка состояния интерфейса
    ens192
  }
  track_script { //Какой скрипт будет использован для проверки
    ntp_check
  }
}

Создаем далее скрипт dns_check и ntp_check в заданной нами директории
mkdir /usr/keepalived
chmod 777 /usr/keepalived -R
vim /usr/keepalived/dns_check
Пишем и сохраняем - systemctl status bind9 > /dev/null 2>&1
vim /usr/keepalived/ntp_check
Пишем и сохраняем - systemctl status ntp > /dev/null 2>&1
systemctl restart keepalived

Проделываем тоже самое и на SRV-2, но значения state меняем с MASTER на BACKUP, значения priority меняем на 9

# Настройка GRE туннеля на машише FW
vim /etc/gre.up
Вводим - ip tunnel add tun1 mode gre local 200.100.200.100 remote 200.100.100.100 ttl 64 // Создаем туннель
         ip link set tun1 up // Поднимаем туннель
         ip addr add 10.5.5.1/30 dev tun1 // Назначаем IP
chmod +x /etc/gre.up 
/etc/gre.up

Добавляем в планировщик задач наш скриптик для автозапуска
vim /etc/crontab
Добавляем строку @reboot root /etc/gre.up

# Настройка OSPF с помощью пакета frr на FW
apt install frr
vim /etc/frr/daemons
Меняем значение ospfd на yes
systemctl restart frr
vtysh
conf t
router ospf
ospf router-id 3.3.3.3
network 172.20.0.0/16 area 0
network 10.5.5.0/30 area
do wr

!Проверить, что нет ip ospf network broadcast в конфиге!

# Настройка IPSEC на FW
apt install strongswan
vim /etc/ipsec.conf
Пишем:
conn gre			// Соединение под название gre
	left=200.100.200.100 	// Локальный внешний адрес
	leftprotoport=gre	// Используемый порт протокола GRE у локальной части
	right=200.100.100.100	// Удаленный внешний адрес
	rightprotoport=gre	// Используемый порт протокола GRE у удаленной части
	type=tunnel		// Тип туннеля
	ike=3des-sha1-modp2048	// Шифровка ключа, modp - группа Диффи - Хеллмана
	esp=aes128-sha2_256	// Шифровка передаваемых данных
	authby=secret		// Аутентификация по паролю
	auto=start		// Соединение в автозагрузке
:wq!
vim /etc/ipsec.secrets
Пишем:
200.100.200.100 200.100.100.100 : PSK "WSR-2022" // Первый IP - локальный, Второй - удаленный, PSK - Preshared Key, "WSR-2022" - Ключ(пароль)
:wq!
ipsec start

# Для добавления линь машин в виндовый родительский домен (DC) 
apt install realmd adcli sssd
realm join company.msk --install=/ -U Administrator
И вводим пароль от Administrator машины DC

В конфиге /etc/pam.d/common-session дописываем строку
session optional  pam_mkhomedir.so skel=/etc/skel umask=0077

#DHCP RELAY FW
apt install isc-dhcp-relay

далее указываем в псевдографике айпи dhcp сервера;
интерфесы, на которые перенаправляются dhcp запросы;
на опциях жмем окей

dpkg-reconfigure isc-dhcp-relay - переконфигурация dhcp relay

#Настройка dhcp на клиентах
apt install isc-dhcp-client
Чтобы появилась запись линь системы на DC надо вписать или изменить строку в /etc/dhcp/dhclient.conf 
send host-name = "<Полное доменное имя машины>" //Пример - CLI-L.company.msk

!dhclient -r перезапрашивает IP!

# Настройка ВЕБа на SRV-1 и SRV-2
apt install nginx
mkdir /opt/web
chmod 777 /opt/web -R
rm -rf /etc/nging/sites-enabled/default		// Удаление конфига по-умолчанию
vim /etc/nginx/conf.d/site.conf
Пишем:
server {
	listen 0.0.0.0:80;
	location / {
		root /opt/web;
		index index.html;
	}
}

nginx -t			 		// Проверяем правильность конфигурации
vim /opt/web
Пишем: "Код HTML"

systemctl restart nginx

# Перевод сертификатов под линукс систему
apt install openssl
openssl pkcs12 -in <Файл формата pfx> -out <APP.pem> -nodes		// Серверный серт
cp APP.pem /etc/ssl/certs/
update-ca-certificated
openssl x509 -inform DER -in <Файл формата cer> -out <Root.crt>		// Клиентский серт
cp Root.crt /etc/ssl/certs/
cp Root.crt /usr/share/ca-certificates
dpkg-reconfigure ca-certificates 
И выбираем наш серт

# Настройки haproxy для отказоустойчивости веб-сайта на app.company.msk на машине FW (Frontend и Backend)
global
defaults
  mode https
  timeout client  5s
  timeout server  5s
  timeout connect 5s

frontend MyFronted
  bind 200.100.200.100:443 ssl crt /etc/ssl/certs/APP.pem //Привязка порта к внешнему IP и указание сертификата безопасности
  default-backend TransparentBack_http
  http-request redirect scheme https unless { ssl_fc }
  
backend TransparentBack_http
  balance roundrobin
  option tcp-check
  server s1 172.20.30.100:80 check weight 3
  server s2 172.20.30.20:80 check weight 1

# Назначение доступа определенным айпи до определенных доменных зон
Конфигурация в /etc/bind/named.conf.default-zones переписываем
acl "internal" { 172.20.0.0/16; };
acl "external" { any; !172.20.0.0/16; };
view "internal" {
  match-clients {"internal";};
  forwarders { 200.100.200.254; };
  zone "." {
    type hint;
    file "/usr/share/dns/root.hints";
  };
  zone "company.msk" {
    type slave;
    masters { 172.20.10.100; }; //Указываем IP машины DC
    file "/opt/dns/company.msk;
  };
  zone "_msdcs.company.msk" {
    type slave;
    masters { 172.20.10.100; };
    file "/opt/dns/msdcs.msk;
  };
};
view "external" {
  match-clients { "external"; };
  forwarders {};
  zone "company.msk" {
    type slave;
    masters { 172.20.10.100; }; //Указываем IP машины DC
    file "/opt/dns/company1.msk;
  };
  zone "_msdcs.company.msk" {
    type slave;
    masters { 172.20.10.100; };
    file "/opt/dns/msdcs1.msk;
  };
};

В конфигурации /etc/bind/named.conf.options убираем forwarders

