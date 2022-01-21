# WS107
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
  
# Настройка пропуска DNS на FW для внешних клиентов
iptables -t nat -A PREROUTING -i ens192 -p udp -m udp --dport 53 -j DNAT --to-destination 172.20.30.10
iptables-save > /etc/iptables/rules.v4
//Далее на ISP
zone "msk" {
  type master;
  file "/etc/bind/db.local";
  forwarders { 200.100.200.100; };
};
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

# Для добавления линь машин в виндовый родительский домен (DC) 
apt install realmd adcli sssd
realm join company.msk --install=/ -U Administrator
И вводим пароль от Administrator машины DC

В конфиге /etc/pam.d/common-session дописываем строку
session optional  pam_mkhomedir.so skel=/etc/skel umask=0077

apt install isc-dhcp-client
Чтобы появилась запись линь системы на DC надо вписать или изменить строку в /etc/dhcp/dhclient.conf 
send host-name = "<Полное доменное имя машины>" //Пример - CLI-L.company.msk

!dhclient -r перезапрашивает IP!

# Назначение доступа определенным айпи до определнных доменных зон
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
