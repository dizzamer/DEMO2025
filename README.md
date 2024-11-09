# Модуль № 1:Настройка сетевой инфраструктуры  

## 1. Произведите базовую настройку устройств  
 ### ● Настройте имена устройств согласно топологии. Используйте полное доменное имя
    HQ-RTR | BR-RTR:  
    в режиме глобальной конфигурации hostname {hq-rtr, br-rtr}    
    HQ-SRV | HQ-CLI | BR-SRV:  
    hostnamectl set-hostname {hq-srv, hq-cli, br-srv}.au-team.irpo; exec bash  
 ### ● На всех устройствах необходимо сконфигурировать IPv4  
    Настройка адресов производиться через nmtui  
 ### ● IP-адрес должен быть из приватного диапазона, в случае, если сеть локальная, согласно RFC1918   
    RFC 1918 включает себя следующие адреса:  
    10.0.0.0/8  
    172.16.0.0/12  
    192.168.0.0/16  
 ### ● Локальная сеть в сторону HQ-SRV(VLAN100) должна вмещать не более 64 адресов  
    маска /26  255.255.255.192
 ### ● Локальная сеть в сторону HQ-CLI(VLAN200) должна вмещать не более 16 адресов  
    маска /28  255.255.255.240
 ### ● Локальная сеть в сторону BR-SRV должна вмещать не более 32 адресов  
    маска /27  255.255.255.224
 ### ● Локальная сеть для управления(VLAN999) должна вмещать не более 8 адресов  
    маска /29  255.255.255.248
### ● Сведения об адресах занесите в отчёт, в качестве примера используйте Таблицу 3
     
 | Имя Устройства   | IPv4                     |  Интерфейс  | NIC       | Шлюз         | 
 | ---------        | ---------                | ---------   | --------- | ---------    |
 | ISP              | NAT (inet)               | ens3        | Internet  |              |
 |                  | 172.16.4.14/28           | ens4        | ISP_HQ    |              |
 |                  | 172.16.5.14/28           | ens5        | ISP_BR    |              |
 | HQ-RTR           | 172.16.4.1/28            | te0         | ISP_HQ    | 172.16.4.14  |
 |                  | 192.168.0.81/29          | te1         | HQ_NET    |              |
 |                  | 192.168.0.62/26          | te1.100     |           |              |
 |                  | 192.168.1.78/28          | te1.200     |           |              |
 |                  | 172.16.0.1/30            | GRE         | TUN       |              |
 | HQ-SW            | 192.168.0.82/29          | ens3        | HQ_NET    |              |
 |                  | -                        | ens4        | SRV_NET   |              |
 |                  | -                        | ens5        | CLI_NET   |              |
 | HQ-SRV           | 192.168.0.2/26           | ens3        | SRV_NET   | 192.168.0.62 |
 | HQ-CLI           | 192.168.1.65/28(DHCP)    | ens3        | CLI_NET   | 192.168.1.78 |
 | BR-RTR           | 172.16.5.1/28            | te0         | ISP_BR    | 172.16.5.14  |
 |                  | 192.168.1.1/27           | te1         | BR_NET    |              |
 |                  | 172.16.0.2/30            | GRE         | TUN       |              |
 | BR-SRV           | 192.168.1.2/27           | ens3        | BR_NET    | 192.168.1.1  |

## 2. Настройка ISP
 ### ● Настройте адресацию на интерфейсах:  
    Интерфейс, подключенный к магистральному провайдеру, получает адрес по DHCP  
    o Настройте маршруты по умолчанию там, где это необходимо   
    Маршруты по умолчанию настраиваются на роутерах:  
    HQ-RTR - ip route 0.0.0.0/0 172.16.4.14  
    BR-RTR - ip route 0.0.0.0/0 172.16.5.14  
    o Интерфейс, к которому подключен HQ-RTR, подключен к сети 172.16.4.0/28  
    Настройка производится на EcoRouter:  
    en  
    conf t  
    int ISP  
    ip add 172.16.4.1/28  
    port te0  
    service-instance toISP  
    encapsulation untagged  
    connect ip interface ISP  
    wr  mem  
    o Интерфейс, к которому подключен BR-RTR, подключен к сети 172.16.5.0/28  
    Настройка производится на EcoRouter:  
    en  
    conf t  
    int ISP  
    ip add 172.16.5.1/28  
    port te0  
    service-instance toISP  
    encapsulation untagged  
    connect ip interface ISP  
    wr  mem  
    o На ISP настройте динамическую сетевую трансляцию в сторону HQ-RTR и BR-RTR  
    для доступа к сети Интернет:  
    echo net.ipv4.ip_forward=1 > /etc/sysctl.conf
    dnf install iptables-services –y   
    systemctl enable ––now iptables  
    iptables –t nat –A POSTROUTING –s 172.16.4.0/28 –o ens3 –j MASQUERADE  
    iptables –t nat –A POSTROUTING –s 172.16.5.0/28 –o ens3 –j MASQUERADE  
    iptables-save > /etc/sysconfig/iptables  
    systemctl restart iptables  
    iptables –L –t nat - должны высветится в Chain POSTROUTING две настроенные подсети.  
## 3. Создание локальных учетных записей
 ### ● Создайте пользователя sshuser на серверах HQ-SRV и BR-SRV  
    useradd -m -u 1010 sshuser  
    o Пароль пользователя sshuser с паролем P@ssw0rd  
    echo "sshuser:P@ssw0rd" | sudo chpasswd  
    o Идентификатор пользователя 1010  
    o Пользователь sshuser должен иметь возможность запускать sudo
    без дополнительной аутентификации.  
    usermod -aG wheel sshuser  
    nano /etc/sudoers  
    sshuser ALL=(ALL) NOPASSWD:ALL  
   ### ● Создайте пользователя net_admin на маршрутизаторах HQ-RTR и BR-RTR  
    Настройка производится на EcoRouter:  
    username net_admin  
    o Пароль пользователя net_admin с паролем P@$$word  
    password P@$$word  
    o При настройке на EcoRouter пользователь net_admin должен обладать максимальными привилегиями  
    role admin  
    o При настройке ОС на базе Linux, запускать sudo без дополнительной аутентификации  
## 4. Настройте на интерфейсе HQ-RTR в сторону офиса HQ виртуальный коммутатор:  
 ### ● Создайте подсеть управления с ID VLAN 999  
    Настройка на HQ-RTR:  
    int vl999  
    ip add 192.168.0.81/29  
    description toSW  
    port te1  
    Service-instance toSW  
    Encapsulation untagged  
    ex  
    Int vl999  
    connect port te1 service-instance toSW  
    Настройка на HQ-SW:  
    Перед настройкой линк ens3 в nmtui должен быть в состоянии - отключено
    Адресации так же не должно быть
    ovs-vsctl add-br ovs0  
    ovs-vsctl add-port ovs0 ens3  
    ovs-vsctl set port ens3 vlan_mode=native-untagged tag=999 trunks=999,100,200  
    ovs-vsctl add-port ovs0 ovs0-vlan999 tag=999 -- set Interface ovs0-vlan999 type=internal  
    ifconfig ovs0-vlan999 inet 192.168.0.82/29 up  
 ### ● Сервер HQ-SRV должен находиться в ID VLAN 100  
    Настройка на HQ-RTR:  
    int te1.100  
    ip add 192.168.0.62/26  
    port te1  
    service-instance te1.100  
    encapsulation dot1q 100  
    rewrite pop 1  
    connect ip interface te1.100  
    Настройка на HQ-SW:  
    Перед настройкой линк ens4 в nmtui должен быть в состоянии - отключено
    Адресации так же не должно быть
    Так как при настройке на HQ-SW бридж ovs0 уже создан, его создавать не нужно
    ovs-vsctl add-port ovs0 ens4  
    ovs-vsctl set port ens4 tag=100 trunks=100  
    ovs-vsctl add-port ovs0 ovs0-vlan100 tag=100 -- set Interface ovs0-vlan100 type=internal  
    ifconfig ovs0-vlan100 up  
 ### ● Клиент HQ-CLI в ID VLAN 200  
    Настройка на HQ-RTR:  
    int te1.200  
    ip add 192.168.1.78/28  
    port te1  
    service-instance te1.200  
    encapsulation dot1q 200  
    rewrite pop 1  
    connect ip interface te1.200  
    Настройка на HQ-SW: 
    Перед настройкой линк ens5 в nmtui должен быть в состоянии - отключено
    Адресации так же не должно быть
    Так как при настройке на HQ-SW бридж ovs0 уже создан, его создавать не нужно
    ovs-vsctl add-port ovs0 ens5  
    ovs-vsctl set port ens5 tag=200 trunks=200  
    ovs-vsctl add-port ovs0 ovs0-vlan200 tag=200 -- set Interface ovs0-vlan200 type=internal  
    ifconfig ovs0-vlan200 up  
**● Основные сведения о настройке коммутатора и выбора реализации разделения на VLAN занесите в отчёт**  
## 5. Настройка безопасного удаленного доступа на серверах HQ-SRV и BR-SRV:  
 ### ● Для подключения используйте порт 2024  
     Перед настройкой выполните команду setenforce 0, далее переводим selinux в состояние  
     permissive в файле /etc/selinux/config
     dnf install openssh - если не установлен
     systemctl enable --now sshd
     nano /etc/ssh/sshd_config
     Меняем порт на 2024
 ### ● Разрешите подключения только пользователю sshuser  
      nano /etc/ssh/sshd_config
      AllowUsers sshuser
 ### ● Ограничьте количество попыток входа до двух  
      nano /etc/ssh/sshd_config
      MaxAuthTries 2
 ### ● Настройте баннер «Authorized access only»  
      echo «Authorized access only» > /etc/ssh/sshd_banner
      nano /etc/ssh/sshd_config 
      echo Banner /etc/ssh/sshd_banner >> /etc/ssh/sshd_config
      systemctl restart sshd
## 6. Между офисами HQ и BR необходимо сконфигурировать ip туннель  
  ### o Сведения о туннеле занесите в отчёт  
    Настройка на HQ-RTR:
    Interface tunnel.1  
    Ip add 172.16.0.1/30  
    Ip mtu 1476
    ip ospf network broadcast
    ip ospf mtu-ignore
    Ip tunnel 172.16.4.1 172.16.5.1 mode gre
    end
    Conf t
    Router ospf 1
    Ospf router-id  172.16.0.1
    network 172.16.0.0 0.0.0.3 area 0
    network 192.168.0.0 0.0.0.63 area 0
    network 192.168.1.0 0.0.0.15 area 0
    passive-interface default
    no passive-interface tunnel.1
    Настройка на BR-RTR:
    Interface tunnel.1
    Ip add 172.16.0.2/30
    Ip mtu 1476
    ip ospf mtu-ignore
    ip ospf network broadcast
    Ip tunnel 172.16.5.1 172.16.4.1 mode gre
    end
    Conf t
    Router ospf 1
    Ospf router-id 172.16.0.2
    Network 172.16.0.0 0.0.0.3 area 0
    Network 192.168.1.0 0.0.0.31 area 0
    Passive-interface default
    no passive-interface tunnel.1  
  o На выбор технологии GRE или IP in IP  
## 7. Обеспечьте динамическую маршрутизацию: ресурсы одного офиса должны быть доступны из другого офиса. Для обеспечения динамической  маршрутизации используйте link state протокол на ваше усмотрение.  
  ● Разрешите выбранный протокол только на интерфейсах в ip туннеле  
  ## ● Маршрутизаторы должны делиться маршрутами только друг с другом  
  ### ● Обеспечьте защиту выбранного протокола посредством парольной защиты 
     Настройка производится на EcoRouter HQ-RTR:
     router ospf 1
     area 0 authentication 
     ex
     interface tunnel.1  
     ip ospf authentication-key ecorouter  
     wr mem  
     Настройка производится на EcoRouter BR-RTR:
     router ospf 1  
     area 0 authentication 
     ex  
     interface tunnel.1  
     ip ospf authentication-key ecorouter  
     wr mem  
  ● Сведения о настройке и защите протокола занесите в отчёт  
## 8. Настройка динамической трансляции адресов.  
 ### ● Настройте динамическую трансляцию адресов для обоих офисов.  
    Настройка производится на EcoRouter HQ-RTR: 
    ip nat pool nat1 192.168.0.1-192.168.0.254  
    ip nat source dynamic inside-to-outside pool nat1 overload 172.16.4.1  
    ip nat pool nat2 192.168.1.65-192.168.1.79  
    ip nat source dynamic inside-to-outside pool nat2 overload 172.16.4.1  
    Настройка производится на EcoRouter BR-RTR: 
    ip nat pool nat3 192.168.2.2-192.168.2.31  
    ip nat source dynamic inside-to-outside pool nat3 overload 172.16.5.1 
### ● Все устройства в офисах должны иметь доступ к сети Интернет  
    Настройка производится на EcoRouter HQ-RTR:
    en
    conf t
    int ISP
    ip nat outside
    ex
    int vl999
    ip nat inside
    ex
    int te1.100
    ip nat inside
    ex
    int te1.200
    ip nat inside
    Настройка производится на EcoRouter BR-RTR: 
    en
    conf t
    int ISP
    ip nat outside
    ex
    int SRV
    ip nat inside
    ex  
    Настройка производится на HQ-SRV:
    В nmtui прописывеем шлюз - 192.168.0.62/26  
    Настройка производится на BR-SRV:  
    В nmtui прописывет шлюз - 192.168.2.1/27
## 9. Настройка протокола динамической конфигурации хостов.  
  ● Настройте нужную подсеть  
  ### ● Для офиса HQ в качестве сервера DHCP выступает маршрутизатор HQ-RTR.  
    Настройка производится на EcoRouter HQ-RTR:
    ip pool dhcpHQ 192.168.1.65-192.168.1.79
    en
    conf t
    dhcp-server 1
    pool dhcpHQ 1
    domain-name au-team.irpo
    mask 255.255.255.240  
    gateway 192.168.1.78  
    dns 192.168.0.1  
    end  
    wr mem  
  ###  ● Клиентом является машина HQ-CLI.  
      interface te1.200
      dhcp-server 1
  ● Исключите из выдачи адрес маршрутизатора  
  ● Адрес шлюза по умолчанию – адрес маршрутизатора HQ-RTR.  
  ● Адрес DNS-сервера для машины HQ-CLI – адрес сервера HQ-SRV.  
  ● DNS-суффикс для офисов HQ – au-team.irpo  
  ● Сведения о настройке протокола занесите в отчёт  
## 10. Настройка DNS для офисов HQ и BR.  
  ● Основной DNS-сервер реализован на HQ-SRV.  
    dnf install bind -y  
    systemctl enable --now named  
    chattr +i  
    mv /etc/named/named.conf /etc/named/named.conf.backup  
    nano /etc/named.conf  
    ![named первая часть](https://github.com/dizzamer/DEMO2025/blob/main/dns.png)  
    ![named вторая часть](https://github.com/dizzamer/DEMO2025/blob/main/dns2.png)  
    mkdir /var/named/master  
    chown -R named:named /var/named/master  
    touch /var/named/master/au.team  
    chmod 750 /var/named/*  
    chmod 750 /var/named/master/*  
    nano /var/named/master/au.team  
    ![au team irpo зона](https://github.com/dizzamer/DEMO2025/blob/main/auteamzone.png)  
    nano /var/named/master/0.168.192.zone    
    ![au team irpo зона](https://github.com/dizzamer/DEMO2025/blob/main/0.168.192.zone.jpg) 
    systemctl restart named  
    Проверить зоны можно командой named-checkconf -z  
    ![au team irpo зона](https://github.com/dizzamer/DEMO2025/blob/main/checkconf.png)  
    nano /etc/nsswitch.conf  
    /etc/nsswitch.conf – это файл конфигурации Linux, который определяет, как система должна переключаться между различными поставщиками услуг имен.  
    Меняем hosts: files myhostname resolve [!UNAVAIL=return] dns на:    
    ![nsswitch](https://github.com/dizzamer/DEMO2025/blob/main/nsswitch.jpg)   
    Приведенная выше запись определяет порядок разрешения любого доменного имени.  
    Сначала система проверит отображение домена в файлах (/etc/hosts), если будет найдена соответствующая запись, она будет использовать ее.  
     Для полной работоспособности на HQ-CLI нужно установить в качестве dns севрера HQ-SRV:  
     resolvctl dns ens3 192.168.0.2  
     Для полной работоспособности на HQ-RTR нужно установить в качестве dns севрера HQ-SRV:  
     Удаляем ns-ы, если есть командой no ip name-server 8.8.8.8  
     ip name-server 192.168.0.2  
     На остальных устройствах делаем подобным образом.  
  ● Сервер должен обеспечивать разрешение имён в сетевые адреса устройств и обратно в соответствии с таблицей 2  
### Таблица 2. Таблица имен  
   | Устройство | Запись              | Тип    | 
   | ---------  |  ------             | ----   |
   | HQ-RTR     | hq-rtr.au-team.irpo | A,PTR  | 
   | BR-RTR     | br-rtr.au-team.irpo | A      |
   | HQ-SRV     | hq-srv.au-team.irpo | A,PTR  |
   | HQ-CLI     | hq-cli.au-team.irpo | A,PTR  |
   | BR-SRV     | br-srv.au-team.irpo | A      |
   | HQ-RTR     | moodle.au-team.irpo | CNAME  | 
   | HQ-RTR     | wiki.au-team.irpo   | CNAME  |  
      
  ● В качестве DNS сервера пересылки используйте любой общедоступный DNS сервер  
## 11. Настройте часовой пояс на всех устройствах, согласно месту проведения экзамена.  
   ### Настройка проивзодится на hq-srv:  
    timedatectl set-timezone  

# Модуль № 2:Организация сетевого администрирования операционных систем  

## 1.	Настройте доменный контроллер Samba на машине BR-SRV.   
  ### Настройка проивзодится на BR-SRV:  
   ## Предварительная настройка сервера:  
     setenforce 0  
     nano /etc/selinux  
     Замените в файле конфигурации /etc/selinux/config режим enforcing на permissive  
     Через nmtui добавить второй dns сервер 192.168.0.1/26  
     Впишите домен поиска au-team.irpo 
     Перезапускаем линк в nmtui  
     dnf install samba* krb5* bind -y  
     mv /etc/samba/smb.conf /etc/samba/smb.conf.back  
     Проверьте права на доступ к файлу /etc/krb5.conf:  
     ls -l /etc/krb5.conf  
     Пользователем-владельцем файла должен быть root, а группой-владельцем – named.  
     При необходимости смените владельцев:  
     chown root:named /etc/krb5.conf  
     Откройте файл /etc/krb5.conf:  
     nano /etc/krb5.conf  
     Присваивание серверу доменного имени, если еще не сделали:  
     hostnamectl set-hostname hq-srv.au-team.irpo; exec bash  
     Отключение DNS-службы systemd-resolved:  
     sudo nano /etc/systemd/resolved.conf   
     Установите параметр DNSStubListener в значение no, как показано в примере.  
     Это  необходимо, чтобы отключить прослушивание systemd-resolved на порту 53:  
     ![systemd resolved](https://github.com/dizzamer/DEMO2025/blob/main/systemdresolved.png)  
     После внесения изменений в файл необходимо перезапустить systemd-resolved и NetworkManager командой:  
     systemctl restart systemd-resolved.service NetworkManager  
     Проверьте изменения в настройках, выполните:  
     cat /etc/resolv.conf  
     В выводе должен быть указан адрес отличающийся от 127.0.0.53  
     Настройка сетевого интерфейса производится через nmtui  
     Укажите в качестве DNS-сервера IP-адрес создаваемого контроллера домена:  
     dns=192.168.1.2;  
     dns-search=hq-srv.au-team.irpo;  
     Перезагружаем линк через nmtui или systemctl restart NetworkManager.  
     Проверьте доступные серверы имен, просмотрев файл resolv.conf:  
     cat /etc/resolv.conf  
     В выводе должно отобразиться наши dns сервера и домен для поиска.  
   ## Создание домена под управлением контроллера домена Samba DC:  
     Создание резервных копий файлов  
     Переименуйте файл /etc/smb.conf, он будет создан позднее в процессе выполнения команды samba-tool.  
     cp /etc/samba/smb.conf /etc/samba/smb.conf.back  
     Создайте резервную копию используемого по умолчанию конфигурационного файла kerberos:  
     cp /etc/krb5.conf /etc/krb5.conf.back  
     Настройка конфигурации Kerberos:  
     ls -l /etc/krb5.conf  
     Проверьте права на доступ к файлу /etc/krb5.conf:  
     Пользователем-владельцем файла должен быть root, а группой-владельцем – named. Пример корректного вывода:  
     -rw-r--r-- 1 root named 104 фев 7 11:05 /etc/krb5.conf  
     При необходимости смените владельцев:  
     chown root:named /etc/krb5.conf    
     Откройте файл /etc/krb5.conf:  
     nano /etc/krb5.conf  
     В секции [libdefaults] установите имя домена, используемое по умолчанию:  
     default_realm = au-team.irpo  
     Добавьте в секции [realms] и [domain_realm] информацию об именах домена и сервера:  
     [realms]  
     AU-TEAM.IRPO = {  
       kdc = br-srv.au-team.irpo  
       admin_server = br-srv.au-team.irpo  
     }  
    [domain_realm]  
       .au-team.irpo = AU-TEAM.IRPO  
       au-team.irpo = AU-TEAM.IRPO  
    Откройте файл /etc/krb5.conf.d/crypto-policies:  
    nano /etc/krb5.conf.d/crypto-policies  
    и приведите его содержание к следующему виду:  
    [libdefaults]  
     default_tgs_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 RC4-HMAC DES-CBC-CRC DES3-CBC-SHA1 DES-CBC-MD5  
     default_tkt_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 RC4-HMAC DES-CBC-CRC DES3-CBC-SHA1 DES-CBC-MD5  
     preferred_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 RC4-HMAC DES-CBC-CRC DES3-CBC-SHA1 DES-CBC-MD5  

   ## Настройка DNS-сервера BIND
     Откройте файл /etc/named.conf:    
     nano /etc/named.conf    
     и внесите в блок options { следующие значения параметров (при необходимости, добавив отсутствующие параметры):    
     listen-on port 53 { 192.168.1.2; };  
     allow-query { any; };  
     dnssec-validation no;  
     tkey-gssapi-keytab "/var/lib/samba/bind-dns/dns.keytab";  
     minimal-responses yes;  
     forwarders { 8.8.8.8; };  
   ## Первоначальное полуавтоматическое конфигурирование сервера с помощью утилиты samba-tool  
     Файла /etc/samba/smb.conf быть не должно, он сам создаст.  
     rm /etc/samba/smb.conf  
     samba-tool domain provision --use-rfc2307 --interactive  
     Описание некоторых опций (параметров) этой команды:
     use-rfc2307 – параметр добавляет POSIX атрибуты (UID / GID) на схеме AD. 
     Он понадобится при аутентификации клиентов Linux, BSD, или OS X (в том числе, на локальной машине), в дополнение к Microsoft Windows;
     
     interactive – запуск в интерактивном режиме;  
     realm – указывает на полное DNS-имя домена, которое настроено в /etc/hosts, в верхнем регистре (в нашем случае это AU-TEAM.IRPO);  
     Domain – краткое имя домена NetBIOS (в примере – IRPO);  
     Server Rules – роль сервера (DC – domain controller);  
     DNS backend – DNS-сервер. Возможные значения – SAMBA_INTERNAL (внутренний DNS сервера), BIND9_FLATFILE, BIND9_DLZ, NONE(в нашем случае ничего не ставим);  
     DNS forwarder IP address – данный параметр позволяет указать IP-адрес DNS-сервера, на который будут перенаправлены DNS-запросы, в том случае, когда сервер не сможет их разрешить.  
   ## Запуск и проверка работоспособности службы samba  
     Запустите и добавьте в автозагрузку службы samba и named:  
     systemctl enable samba named --now  
     systemctl status samba named  
   ## •	Создайте 5 пользователей для офиса HQ: имена пользователей формата user№.hq. Создайте группу hq, введите в эту группу созданных пользователей  
### Настройка производится на BR-SRV:    
 ## Управление пользователями и группами
   Список пользователей:  
   samba-tool user list  
   Создадим пользователя в Active Directory и делаем тесты:  
   samba-tool user create user№.hq или  
    smbpasswd -a user№.hq  
    smbpasswd -e user№.hq
    username - имя нового пользователя в AD;
    ключ -a - создает пользователя;
    ключ -e - активирует пользователя   
    samba-tool group list  
    Добавление группы:  
    samba-tool group add hq  
    Добавление пользователя в группу:  
    samba-tool group addmembers hq user№.hq  
   ## Проверка подключения к samba ресурсам
     Вывести список сетевых ресурсов samba:  
      smbclient -L localhost -U%  
      
•	Введите в домен машину HQ-CLI  
### Настройка проивзодится на HQ-CLI:  
  https://redos.red-soft.ru/base/redos-7_3/7_3-administation/7_3-domain-redos/7_3-domain-config/7_3-redos-in-samba/?nocache=1730793368537  
  
•	Пользователи группы hq имеют право аутентифицироваться на клиентском ПК  
•	Пользователи группы hq должны иметь возможность повышать привилегии для выполнения ограниченного набора команд: cat, grep, id. Запускать другие команды с повышенными привилегиями пользователи группы не имеют права  
•	Выполните импорт пользователей из файла users.csv. Файл будет располагаться на виртуальной машине BR-SRV в папке /opt  
## 2.	Сконфигурируйте файловое хранилище:  
•	При помощи трёх дополнительных дисков, размером 1Гб каждый, на HQ-SRV сконфигурируйте дисковый массив уровня 5  
•	Имя устройства – md0, конфигурация массива размещается в файле /etc/mdadm.conf  
•	Обеспечьте автоматическое монтирование в папку /raid5  
•	Создайте раздел, отформатируйте раздел, в качестве файловой системы используйте ext4  
•	Настройте сервер сетевой файловой системы(nfs), в качестве папки общего доступа выберите /raid5/nfs, доступ для чтения и записи для всей сети в сторону HQ-CLI  
•	На HQ-CLI настройте автомонтирование в папку /mnt/nfs  
•	Основные параметры сервера отметьте в отчёте  
## 3.	Настройте службу сетевого времени на базе сервиса chrony  
•	В качестве сервера выступает HQ-RTR  
  ### Настройка проивзодится на HQ-RTR:  
      en  
      conf t  
      ntp server 172.16.14.1 5  
      ntp timezone ?  
      ntp timezone UTC+3  
      end  
      wr mem  
•	На HQ-RTR настройте сервер chrony, выберите стратум 5  
•	В качестве клиентов настройте HQ-SRV, HQ-CLI, BR-RTR, BR-SRV  
## 4.	Сконфигурируйте ansible на сервере BR-SRV  
  ### •	Сформируйте файл инвентаря, в инвентарь должны входить HQ-SRV, HQ-CLI, HQ-RTR и BR-RTR  
      dnf install ansible -y  
      1) В файле /etc/ansible/hosts.yml прописать все хосты, на которые будет распространяться конфигурация.  
      Хосты можно разделить по группам, а так же, если у вас есть домен, то автоматически экспортировать список из домена.  
      Можно прописывать как ip адреса так и имена хостов, если они резолвятся DNS ом в сети.  
      nano /etc/ansible/inventory.yml  
       clients:  
         hosts:  
           hq-cli.au-team.irpo:  
       servers:  
         hosts:  
           hq-srv.au-team.irpo:  
       routers:  
         hosts:  
           hq-rtr.au-team.irpo:  
           br-rtr.au-team.irpo:  
           
        2) Подключение к хостам осуществляется по протоколу ssh с помощью rsa ключей.    
        Сгенерировать серверный ключ можно командой ниже. При её выполнении везде нажмите Enter.    
        ssh-keygen -C "$(whoami)@$(hostname)-$(date -I)"  
        
        3) Далее нужно распространить ключ на все подключенные хосты.  
        Распространить ключи на хосты можно командой:  
        ssh-copy-id root@server  
        где:  
        root - это пользователь, от имени которого будут выполняться плейбуки;  
        server - IP-адрес хоста.  
 •	Рабочий каталог ansible должен располагаться в /etc/ansible  
   ### •	Все указанные машины должны без предупреждений и ошибок отвечать pong на команду ping в ansible посланную с BR-SRV  
          Пингуем удаленные хосты с помощью Ansible:  
           ansible test -m ping  
           В результате под каждым хостом должно быть написано "ping": "pong".  
## 5.	Развертывание приложений в Docker на сервере BR-SRV.  
### •	Создайте в домашней директории пользователя файл wiki.yml для приложения MediaWiki.  
     Развертывание производится на сервере BR-SRV:   
     touch /home/student/wiki.yml  
     nano /home/student/wiki.yml  
      services:
      MediaWiki:
        container_name: wiki
        image: mediawiki
        restart: always
        ports: 
          - 80:8080
        links:
          - database
        volumes:
          - images:/var/www/html/images
          # - ./LocalSettings.php:/var/www/html/LocalSettings.php
      database:
        container_name: mariadb
        image: mariadb
        environment:
          MYSQL_DATABASE: mediawiki
          MYSQL_USER: wiki
          MYSQL_PASSWORD: WikiP@ssw0rd
          MYSQL_RANDOM_ROOT_PASSWORD: 'yes'
        volumes:
          - dbvolume:/var/lib/mysql
    volumes:
      dbvolume:
          external: true
      images:
      Поднимаем стек контейнеров с помощью команды: 
      docker compose -f wiki.yml up -d  
      После установки необходимо раскоментировать строчку с решеткой в файле wiki.yml:  
      docker-compose -f wiki.yml stop  
      docker-compose -f wiki.yml up -d  
•	Средствами docker compose должен создаваться стек контейнеров с приложением MediaWiki и базой данных.  
•	Используйте два сервиса  
•	Основной контейнер MediaWiki должен называться wiki и использовать образ mediawiki  
•	Файл LocalSettings.php с корректными настройками должен находиться в домашней папке пользователя и автоматически монтироваться в образ.  
•	Контейнер с базой данных должен называться mariadb и использовать образ mariadb.  
•	Он должен создавать базу с названием mediawiki, доступную по стандартному порту, пользователя wiki с паролем WikiP@ssw0rd должен иметь права доступа к этой базе данных  
•	MediaWiki должна быть доступна извне через порт 8080.  
## 6.	На маршрутизаторах сконфигурируйте статическую трансляцию портов  
### •	Пробросьте порт 80 в порт 8080 на BR-SRV на маршрутизаторе BR-RTR, для обеспечения работы сервиса wiki  
     Настройка производится на EcoRouter BR-RTR:  
     ip nat source static tcp 192.168.1.2 80 192.168.1.65 8080
     ip nat source static tcp 172.16.5.1 80 192.168.1.2 8080
### •	Пробросьте порт 2024 в порт 2024 на HQ-SRV на маршрутизаторе HQ-RTR  
     Настройка производится на EcoRouter HQ-RTR:  
     ip nat source static tcp 192.168.0.2 2024 192.168.1.65 2024  
### •	Пробросьте порт 2024 в порт 2024 на BR-SRV на маршрутизаторе BR-RTR  
     Настройка производится на EcoRouter BR-RTR:  
     ip nat source static tcp 192.168.1.2 2024 192.168.1.65 2024  
## 7.	Запустите сервис moodle на сервере HQ-SRV:  
•	Используйте веб-сервер apache  
•	В качестве системы управления базами данных используйте mariadb  
•	Создайте базу данных moodledb  
•	Создайте пользователя moodle с паролем P@ssw0rd и предоставьте ему права доступа к этой базе данных  
•	У пользователя admin в системе обучения задайте пароль P@ssw0rd  
•	На главной странице должен отражаться номер рабочего места в виде арабской цифры, других подписей делать не надо  
•	Основные параметры отметьте в отчёте  
## 8.	Настройте веб-сервер nginx как обратный прокси-сервер на HQ-RTR  
  ### •	При обращении к HQ-RTR по доменному имени moodle.au-team.irpo клиента должно перенаправлять на HQ-SRV на стандартный порт, на сервис moodle  
     Настройка производится на EcoRouter HQ-RTR:  
     en  
     conf t  
     filter-map policy ipv4 moodle 1  
     match 80 172.16.4.1/28 192.168.0.2/26 dscp 0
     set redirect hq-rtr.moodle.au-team.irpo  
     end  
     wr mem  
     en  
     conf t  
     redirect-url SITEREDIRECT  
     url hq-rtr.moodle.au-team.irpo  
     end  
     wr mem  
•	При обращении к HQ-RTR по доменному имени wiki. au-team.irpo клиента должно перенаправлять на BR-SRV на порт, на сервис mediwiki  
## 9.	Удобным способом установите приложение Яндекс Браузер для организаций на HQ-CLI  
•	Установку браузера отметьте в отчёте  

# Модуль № 3:Эксплуатация объектов сетевой инфраструктуры    





Необходимые приложения:  
Приложение. Файл users.csv (в отдельном файле).  
 
