# DEMO2025

## 1. Произведите базовую настройку устройств
 **● Настройте имена устройств согласно топологии. Используйте полное доменное имя**
 HQ-RTR | BR-RTR  :  
 в режиме глобальной конфигурации hostname {hq-rtr, br-rtr}    
 HQ-SRV | HQ-CLI | BR-SRV :  
 hostnamectl set-hostname {hq-srv, hq-cli, br-srv}.au-team.irpo; exec bash  
 **● На всех устройствах необходимо сконфигурировать IPv4**  
 Настройка адресов производиться через nmtui  
 **● IP-адрес должен быть из приватного диапазона, в случае, если сеть**
 локальная, согласно RFC1918  
 RFC 1918 включает себя следующие адреса:  
 10.0.0.0/8  
 172.16.0.0/12  
 192.168.0.0/16  
 **● Локальная сеть в сторону HQ-SRV(VLAN100) должна вмещать не более 64 адресов**  
 маска /26  
 **● Локальная сеть в сторону HQ-CLI(VLAN200) должна вмещать не более 16 адресов**  
 маска /28  
 **● Локальная сеть в сторону BR-SRV должна вмещать не более 32 адресов**  
 маска /27  
 **● Локальная сеть для управления(VLAN999) должна вмещать не более 8 адресов**  
 маска /29  
 **● Сведения об адресах занесите в отчёт, в качестве примера используйте Таблицу 3**

| Имя Устройства   | IPv4                |  Интерфейс  | NIC       | Шлюз         | 
| ---------        | -------             | -------     | -------   |-------       |
| ISP              | NAT (inet)          | ens3        | Internet  | -            |
|                  | 172.16.4.14/28      | ens4        | ISP_HQ    | -            |
|                  | 172.16.5.14/28      | ens5        | ISP_BR    | -            |
| HQ-RTR           | 172.16.4.1/28       | te0         | ISP_HQ    | 172.16.4.14  |
|                  | 192.168.0.81/29     | te1         | HQ_NET    | -            |
|                  | 192.168.0.3/26      | te1.100     | -         | -            |
|                  | 192.168.1.3/27      | te1.200     | -         |              |
|                  | 172.16.0.1/30       | GRE         | TUN       |              |
| HQ-SW            | 192.168.0.82/29     | ens3        | HQ_NET    |              |
|                  | 192.168.0.1/26      | ens4        | SRV_NET   |              |
|                  | 192.168.0.65/28     | ens5        | CLI_NET   |              |
| HQ-SRV           | 192.168.0.2/26      | ens3        | SRV_NET   | 192.168.0.3  |
| HQ-CLI           | 192.168.0.80/28     | ens3        | CLI_NET   | 192.168.0.65 |
| BR-RTR           | 172.16.5.1/28       | ens3        | ISP_BR    | 172.16.5.14  |
|                  | 192.168.1.1/27      | ens4        | BR_NET    |              |
|                  | 172.16.0.2/30       | GRE         | TUN       |              |
| BR-SRV           | 192.168.1.2/27      | BR_NET      | BR_NET    | 192.168.1.1  |

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
    wr  
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
    wr  
    o На ISP настройте динамическую сетевую трансляцию в сторону HQ-RTR и BR-RTR для доступа к сети Интернет  
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
 ### ● Сервер HQ-SRV должен находиться в ID VLAN 100  
    Настройка на HQ-RTR:  
    int te1.100  
    ip add 192.168.0.3/26  
    port te1  
    service-instance te1.100  
    encapsulation dot1q 100  
    rewrite pop 1  
    connect ip interface te1.100  
    Настройка на HQ-SW:  
    ovs-vsctl add-port ovs0 ens4  
    ovs-vsctl set port ens4 tag=100 trunks=100  
    ovs-vsctl add-port ovs0 ovs0-vlan100 tag=100 -- set Interface ovs0-vlan100 type=internal  
    ifconfig ovs0-vlan100 up  
 ### ● Клиент HQ-CLI в ID VLAN 200  
    Настройка на HQ-RTR:  
    int te1.200  
    ip add 192.168.1.3/27  
    port te1  
    service-instance te1.200  
    encapsulation dot1q 200  
    rewrite pop 1  
    connect ip interface te1.200  
    Настройка на HQ-SW:  
    ovs-vsctl add-port ovs0 ens5  
    ovs-vsctl set port ens5 tag=200 trunks=200  
    ovs-vsctl add-port ovs0 ovs0-vlan200 tag=200 -- set Interface ovs0-vlan200 type=internal  
    ifconfig ovs0-vlan200 up  
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
    ovs-vsctl add-br ovs0  
    ovs-vsctl add-port ovs0 ens3  
    ovs-vsctl set port ens3 vlan_mode=native-untagged tag=999 trunks=999,100,200  
    ovs-vsctl add-port ovs0 ovs0-vlan999 tag=999 -- set Interface ovs0-vlan999 type=internal  
    ifconfig ovs0-vlan999 inet 192.168.0.82/29 up  
**● Основные сведения о настройке коммутатора и выбора реализации разделения на VLAN занесите в отчёт**  
## 5. Настройка безопасного удаленного доступа на серверах HQ-SRV и BR-SRV:  
  ● Для подключения используйте порт 2024  
  ● Разрешите подключения только пользователю sshuser  
  ● Ограничьте количество попыток входа до двух  
  ● Настройте баннер «Authorized access only»  
## 6. Между офисами HQ и BR необходимо сконфигурировать ip туннель  
  ### o Сведения о туннеле занесите в отчёт  
    Настройка на HQ-RTR:
    Interface tunnel.1
    Ip add 172.16.0.1/30
    Ip mtu 1476
    ip ospf network broadcast
    ip ospf mtu-ignore
    Ip tunnel 172.16.4.1/28 172.16.5.1/28 mode gre
    end
    Conf t
    Router ospf 1
    Ospf router-id  192.168.10.1
    Network 172.16.0.1/24 area 0
    Network 172.16.4.0/28 area 0
    passive-interface default
    no passive-interface default tunnel.0
    Настройка на BR-RTR:
    Interface tunnel.1
    Ip add 172.16.0.2/30
    Ip mtu 1476
    ip ospf mtu-ignore
    ip opsf network broadcast
    Ip tunnel 172.16.5.1 172.16.4.1 mode gre
    end
    Conf t
    Router ospf 1
    Ospf router-id  192.168.4.1
    Network 172.16.0.1/24 area 0
    Network 172.16.4.0/28 area 0
    Passive-interface default
  o На выбор технологии GRE или IP in IP  
## 7. Обеспечьте динамическую маршрутизацию: ресурсы одного офиса должны быть доступны из другого офиса. Для обеспечения динамической  маршрутизации используйте link state протокол на ваше усмотрение.  
  ● Разрешите выбранный протокол только на интерфейсах в ip туннеле  
  ### ● Маршрутизаторы должны делиться маршрутами только друг с другом  
     Настройка производится на EcoRouter HQ-RTR:
     conf t
     ip route 172.16.5.1/28 172.16.4.1
     Настройка производится на EcoRouter BR-RTR:
     conf t
     ip route 172.16.4.1/28 172.16.5.1
  ### ● Обеспечьте защиту выбранного протокола посредством парольной защиты 
     Настройка производится на EcoRouter HQ-RTR:
     router ospf 1
     area 0 authentication 
     ex
     interface tunnel.1
     ip ospf authentication-key ecorouter
     Настройка производится на EcoRouter BR-RTR:
     router ospf 1
     area 0 authentication 
     ex
     interface tunnel.1
     ip ospf authentication-key ecorouter
  ● Сведения о настройке и защите протокола занесите в отчёт  
## 8. Настройка динамической трансляции адресов.  
 ### ● Настройте динамическую трансляцию адресов для обоих офисов.  
    Настройка производится на EcoRouter HQ-RTR: 
    ip nat pool nat1 192.168.0.1-192.168.0.254  
    ip nat source dynamic inside-to-outside pool nat1 overload 172.16.4.1  
    Настройка производится на EcoRouter BR-RTR: 
    ip nat pool nat2 192.168.1.1-192.168.1.254  
    ip nat source dynamic inside-to-outside pool nat2 overload 172.16.5.1 
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
## 9. Настройка протокола динамической конфигурации хостов.  
  ● Настройте нужную подсеть  
  ● Для офиса HQ в качестве сервера DHCP выступает маршрутизатор
  HQ-RTR.  
  ● Клиентом является машина HQ-CLI.  
  ● Исключите из выдачи адрес маршрутизатора  
  ● Адрес шлюза по умолчанию – адрес маршрутизатора HQ-RTR.  
  ● Адрес DNS-сервера для машины HQ-CLI – адрес сервера HQ-SRV.  
  ● DNS-суффикс для офисов HQ – au-team.irpo  
  ● Сведения о настройке протокола занесите в отчёт  
## 10. Настройка DNS для офисов HQ и BR.  
  ● Основной DNS-сервер реализован на HQ-SRV.  
  ● Сервер должен обеспечивать разрешение имён в сетевые адреса устройств и обратно в соответствии с таблицей 2  
  ● В качестве DNS сервера пересылки используйте любой общедоступный DNS сервер  
## 11. Настройте часовой пояс на всех устройствах, согласно месту проведения
экзамена.  
Таблица 2. Таблица имен  
Устройство Запись Тип  
HQ-RTR hq-rtr.au-team.irpo A,PTR  
BR-RTR br-rtr.au-team.irpo A  
HQ-SRV hq-srv.au-team.irpo A,PTR  
HQ-CLI hq-cli.au-team.irpo A,PTR  
BR-SRV br-srv.au-team.irpo A  
HQ-RTR moodle.au-team.irpo CNAME  
HQ-RTR wiki.au-team.irpo CNAME  



