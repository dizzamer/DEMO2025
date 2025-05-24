# Модуль № 1:Настройка сетевой инфраструктуры  

## 1. Произведите базовую настройку устройств  
 ### ● Настройте имена устройств согласно топологии. Используйте полное доменное имя
    HQ-RTR | BR-RTR:  
    в режиме глобальной конфигурации hostname {hq-rtr, br-rtr}    
    HQ-SRV | HQ-CLI | BR-SRV:  
    hostnamectl set-hostname {hq-srv, hq-cli, br-srv}.au-team.irpo; exec bash  
 ### ● На всех устройствах необходимо сконфигурировать IPv4  
    Настройка адресов производится через nmtui  
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
 |                  | 192.168.2.1/27           | te1         | BR_NET    |              |
 |                  | 172.16.0.2/30            | GRE         | TUN       |              |
 | BR-SRV           | 192.168.2.2/27           | ens3        | BR_NET    | 192.168.2.1  |

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
    port te0  
    service-instance toISP  
    encapsulation untagged  
    end  
    wr mem  
    en  
    conf t  
    int ISP  
    ip add 172.16.4.1/28  
    connect port te0 service-instance toISP  
    end  
    wr  mem  
    o Интерфейс, к которому подключен BR-RTR, подключен к сети 172.16.5.0/28  
    Настройка производится на EcoRouter:  
    en  
    conf t  
    port te0  
    service-instance toISP  
    encapsulation untagged  
    end  
    wr mem  
    en  
    conf t  
    int ISP  
    ip add 172.16.5.1/28  
    connect port te0 service-instance toISP  
    end  
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
    nano /etc/sysconfig/iptables - не должно быть ничего лишнего.  
    в случае если там есть то, что вы не добавляли - удалить  
    iptables –L –t nat - должны высветится в Chain POSTROUTING две настроенные подсети.  
## 3. Создание локальных учетных записей
 ### ● Создайте пользователя sshuser на серверах HQ-RTR | BR-RTR | HQ-CLI  
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
    port te1  
    Service-instance toSW  
    Encapsulation untagged  
    end  
    wr mem  
    en  
    conf t  
    Int vl999
    ip add 192.168.0.81/29  
    description toSW  
    connect port te1 service-instance toSW  
    end  
    wr mem  
    Настройка на HQ-SW:  
    Перед настройкой линк ens3 в nmtui должен быть в состоянии - отключено
    Адресации так же не должно быть
    ovs-vsctl add-br ovs0  
    ovs-vsctl add-port ovs0 ens3  
    ovs-vsctl set port ens3 vlan_mode=native-untagged tag=999 trunks=999,100,200  
    ovs-vsctl add-port ovs0 ovs0-vlan999 tag=999 -- set interface ovs0-vlan999 type=internal  
    ifconfig ovs0-vlan999 inet 192.168.0.82/29 up  
 ### ● Сервер HQ-SRV должен находиться в ID VLAN 100  
    Настройка на HQ-RTR:  
    port te1  
    service-instance te1.100  
    encapsulation dot1q 100  
    rewrite pop 1  
    end  
    wr mem  
    int te1.100  
    ip add 192.168.0.62/26  
    connect port te1 service-instance te1.100  
    end  
    wr mem  
    Настройка на HQ-SW:  
    Перед настройкой линк ens4 в nmtui должен быть в состоянии - отключено
    Адресации так же не должно быть
    Так как при настройке на HQ-SW бридж ovs0 уже создан, его создавать не нужно
    ovs-vsctl add-port ovs0 ens4  
    ovs-vsctl set port ens4 tag=100 trunks=100  
    ovs-vsctl add-port ovs0 ovs0-vlan100 tag=100 -- set interface ovs0-vlan100 type=internal  
    ifconfig ovs0-vlan100 inet up  
 ### ● Клиент HQ-CLI в ID VLAN 200  
    Настройка на HQ-RTR:  
    port te1  
    service-instance te1.200  
    encapsulation dot1q 200  
    rewrite pop 1  
    end  
    wr mem  
    int te1.200  
    ip add 192.168.1.78/28  
    connect port te1 service-instance te1.200 
    end  
    wr mem  
    Настройка на HQ-SW: 
    Перед настройкой линк ens5 в nmtui должен быть в состоянии - отключено
    Адресации так же не должно быть
    Так как при настройке на HQ-SW бридж ovs0 уже создан, его создавать не нужно
    ovs-vsctl add-port ovs0 ens5  
    ovs-vsctl set port ens5 tag=200 trunks=200  
    ovs-vsctl add-port ovs0 ovs0-vlan200 tag=200 -- set interface ovs0-vlan200 type=internal  
    ifconfig ovs0-vlan200 inet up  
**● Основные сведения о настройке коммутатора и выбора реализации разделения на VLAN занесите в отчёт**  
## 5. Настройка безопасного удаленного доступа на серверах HQ-SRV и BR-SRV:  
 ### ● Для подключения используйте порт 2024  
     Перед настройкой выполните команду setenforce 0, далее переводим selinux в состояние  
     disabled в файле /etc/selinux/config
     dnf install openssh - если не установлен
     systemctl enable --now sshd
     echo Port 2024 >> /etc/ssh/sshd_config
  ### ● Разрешите подключения только пользователю sshuser  
      eccho AllowUsers sshuser >> nano /etc/ssh/sshd_config
 ### ● Ограничьте количество попыток входа до двух  
      echo MaxAuthTries 2 >> /etc/ssh/sshd_config
 ### ● Настройте баннер «Authorized access only»  
      echo «Authorized access only» > /etc/ssh/sshd_banner
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
    wr mem  
    Conf t
    Router ospf 1
    Ospf router-id  172.16.0.1
    network 172.16.0.0 0.0.0.3 area 0
    network 192.168.0.0 0.0.0.63 area 0
    network 192.168.1.78 0.0.0.15 area 0
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
    Network 192.168.2.0 0.0.0.31 area 0
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
    ip nat source dynamic inside-to-outside pool nat1 overload interface ISP 
    ip nat pool nat2 192.168.1.65-192.168.1.79  
    ip nat source dynamic inside-to-outside pool nat2 overload interface ISP   
    Настройка производится на EcoRouter BR-RTR: 
    ip nat pool nat3 192.168.2.2-192.168.2.31  
    ip nat source dynamic inside-to-outside pool nat3 overload interface ISP 
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
    dns 192.168.0.2    
    gateway 192.168.1.78    
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
    cp /etc/named.conf /etc/named.conf.backup  
    nano /etc/named.conf  
    ![named первая часть](https://github.com/dizzamer/DEMO2025/blob/main/dns.png)  
    ![named вторая часть](https://github.com/dizzamer/DEMO2025/blob/main/dns2.png)  
    mkdir /var/named/master  
    nano /var/named/master/au-team  
    ![au team irpo зона](https://github.com/dizzamer/DEMO2025/blob/main/au-teamn.png)  
    nano /var/named/master/168.192.zone    
    ![au team irpo зона](https://github.com/dizzamer/DEMO2025/blob/main/0.168.192zone.png)  
    chown -R root:named /var/named/master/
    chown -R named:named /var/named
    chown -R root:named /etc/named.conf  
    chmod 750 /var/named/  
    chmod 750 /var/named/master/  
    systemctl restart named  
    Проверить зоны можно командой named-checkconf -z  
     ![au team irpo зона](https://github.com/dizzamer/DEMO2025/blob/main/checkconf.png)  
     Для полной работоспособности на HQ-CLI нужно установить в качестве dns севрера HQ-SRV:  
     nano /etc/resolv.conf на всех устройствах должен иметь следюущий вид:  
     ![resolvconf](https://github.com/dizzamer/DEMO2025/blob/main/resolv.conf.png)  
     resolvectl dns ens3 192.168.0.2  
     Для полной работоспособности на HQ-RTR нужно установить в качестве dns севрера HQ-SRV:  
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
   ### Настройка проивзодится на всех устройствах:  
    timedatectl set-timezone Europe/Moscow  

# Модуль № 2:Организация сетевого администрирования операционных систем  

## 1.	Настройте доменный контроллер Samba на машине BR-SRV.   
  ### Настройка проивзодится на BR-SRV:  
   ## Предварительная настройка сервера:  
     выставляем 192.168.0.2 в качестве нащего днс сервера на линке в nmtui и домен поиска au-team.irpo  
     setenforce 0  
     nano /etc/selinux  
     Замените в файле конфигурации /etc/selinux/config режим enforcing на permissive   
     dnf install samba* krb5* -y  
     Проверьте доступные серверы имен, просмотрев файл resolv.conf:  
     cat /etc/resolv.conf  
     В выводе должно отобразиться наш dns сервер и домен для поиска.  
   ## Создание домена под управлением контроллера домена Samba DC:  
     Создание резервных копий файлов  
     Переименуйте файл /etc/smb.conf, он будет создан позднее в процессе выполнения команды samba-tool.  
     cp /etc/samba/smb.conf /etc/samba/smb.conf.back  
     Создайте резервную копию используемого по умолчанию конфигурационного файла kerberos:  
     cp /etc/krb5.conf /etc/krb5.conf.back  
   ## Конфигурирование сервера с помощью утилиты samba-tool  
     Файла /etc/samba/smb.conf быть не должно, он сам создаст.  
     rm -rf /etc/samba/smb.conf  
     samba-tool domain provision --use-rfc2307 --interactive  
   ![sambatool](https://github.com/dizzamer/DEMO2025/blob/main/samba-toolprovision.png) 
   ## Удаление использования службы dns  
     systemctl stop samba  
     Подчищаем, все где могут храниться наши записи  
     sudo rm -rf /var/lib/samba/private/dns_update_cache  
     sudo rm -rf /var/lib/samba/private/dns_update_list  
     sudo rm -rf /var/lib/samba/private/dns  
     sudo rm -rf /var/lib/samba/private/dns.keytab  
     Добавляем в файл /etc/samba/smb.conf следующее  
   ![smbconf](https://github.com/dizzamer/DEMO2025/blob/main/smbconf.png)  
   
      Запустите и добавьте в автозагрузку службы samba и named:  
      systemctl enable named samba --now  
      systemctl status named samba  
      Проверка созданного домена с помощью команды samba-tool domain info au-team.irpo:  
   ![sambatool](https://github.com/dizzamer/DEMO2025/blob/main/samba-tool.png)  
   ## •	Создайте 5 пользователей для офиса HQ: имена пользователей формата user№.hq. Создайте группу hq, введите в эту группу созданных пользователей  
     sudo samba-tool group add hq  
    for i in {1..5}; do  
    sudo samba-tool user create user$i.hq P@ssw0rd$i  
    sudo samba-tool group addmembers hq user$i.hq  
    done    
   ## •	Введите в домен машину HQ-CLI  
     Перед вводом необходимо ввести на HQ-SRV:  
     smbclient -L localhost -U administrator  
     Проверка kerberos: 
     kinit administrator@localhost
### Настройка проивзодится на HQ-CLI:  
  https://redos.red-soft.ru/base/redos-7_3/7_3-administation/7_3-domain-redos/7_3-domain-config/7_3-redos-in-samba/?nocache=1730793368537  
•	Пользователи группы hq имеют право аутентифицироваться на клиентском ПК  
•	Пользователи группы hq должны иметь возможность повышать привилегии для выполнения ограниченного набора команд: cat, grep, id. Запускать другие команды с повышенными привилегиями пользователи группы не имеют права  
•	Выполните импорт пользователей из файла users.csv. Файл будет располагаться на виртуальной машине BR-SRV в папке /opt  
## 2.	Сконфигурируйте файловое хранилище:  
 ### Настройка проивзодится на HQ-SRV:
  Перед тем как начать проверяем, что установлены следюущие пакеты
  dnf isntall mdadm nfs-utils -y
## •	При помощи трёх дополнительных дисков, размером 1Гб каждый, на HQ-SRV сконфигурируйте дисковый массив уровня 5  
    mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd     
 ![mdadmcreate](https://github.com/dizzamer/DEMO2025/blob/main/mdadm_create.png)    
 ![mdaddetail](https://github.com/dizzamer/DEMO2025/blob/main/mdadm_detail.png)  
## •	Имя устройства – md0, конфигурация массива размещается в файле /etc/mdadm.conf  
    mdadm --detail --scan >> /etc/mdadm.conf      
 ## •	Обеспечьте автоматическое монтирование в папку /raid5  
    Добавляем в /etc/fstab:    
    nano /etc/fstab  
    /dev/md0 /raid5 ext4 defaults 0 0  
![fstab](https://github.com/dizzamer/DEMO2025/blob/main/fstab.png)  
 ## •	Создайте раздел, отформатируйте раздел, в качестве файловой системы используйте ext4  
     mkfs.ext4 /dev/md0  
   ![mkfs](https://github.com/dizzamer/DEMO2025/blob/main/mkfs.png)   
 ## •	Создаем точку монтирования и примонтируемся     
    mkdir -p /raid5   
    mount -a   
  ![mount](https://github.com/dizzamer/DEMO2025/blob/main/mount.png)   
 ### •	Настройте сервер сетевой файловой системы(nfs), в качестве папки общего доступа выберите /raid5/nfs, доступ для чтения и записи для всей сети в сторону HQ-CLI   
  ## •	Создаем папку для NFS  
    mkdir -p /raid5/nfs  
    chmod 777 /raid5/nfs  
 ![mkdir_nfs](https://github.com/dizzamer/DEMO2025/blob/main/mkdir_nfs.png)  
 ## Настройка экспорта  
    Добавляем в /etc/exports:  
    nano /etc/exports  
    /raid5/nfs 192.168.1.64/28(rw,sync,insecure,nohide,all_squash,no_subtree_check)
![exports](https://github.com/dizzamer/DEMO2025/blob/main/etcexports.png)  
  ## Применяем изменения и перезагружаем службу
    exportfs -rav  
    systemctl restart nfs-server  
 ### Настройка проивзодится на HQ-CLI:
  ## •	На HQ-CLI настройте автомонтирование в папку /mnt/nfs  
      Добавляем в /etc/fstab:    
      nano /etc/fstab  
      hq-srv:/raid5/nfs /mnt/nfs nfs defaults 0 0
  ![fstab_hqcli](https://github.com/dizzamer/DEMO2025/blob/main/fstab_hqcli.png)  
  ## Создаем точку монтирования и примонтируемся  
    mkdir -p /mnt/nfs  
    mount -a 
  ![mountdir_hqcli](https://github.com/dizzamer/DEMO2025/blob/main/nount_cli.png)  
  ## Проверка монтирования
      После этого при создании файла на клиенте, он должен появляться и на сервере
   •	Основные параметры сервера отметьте в отчёте  
  ## 3.	Настройте службу сетевого времени на базе сервиса chrony  
  •	В качестве сервера выступает HQ-RTR  
  ### Настройка проивзодится на HQ-RTR:  
      en  
      conf t  
      ntp server 172.16.4.1   
      ntp timezone UTC+3  
      end  
      wr mem  
•	На HQ-RTR настройте сервер chrony, выберите стратум 5  
•	В качестве клиентов настройте HQ-SRV, HQ-CLI, BR-RTR, BR-SRV  
## 4.	Сконфигурируйте ansible на сервере BR-SRV  
  ### Настройка подключения по ssh BR-RTR | HQ-RTR
     Настройка производится на HQ-RTR:  
     en  
     conf  
     security-profile 1  
     rule 1 permit tcp any eq 22 any  
     end  
     wr mem  
     configure
     ip vrf vrf0  
     transport input ssh    
     security 1 vrf vrf0  
     end  
     wr mem  
     conf
     no security default  
     Настройка производится на BR-RTR: 
     conf  
     security-profile 1  
     rule 1 permit tcp any eq 22 any  
     end  
     wr mem  
     configure
     ip vrf vrf0  
     transport input ssh    
     security 1 vrf vrf0  
     end  
     wr mem  
     conf
     no security default
  ## •	Сформируйте файл инвентаря, в инвентарь должны входить HQ-SRV, HQ-CLI, HQ-RTR и BR-RTR  
   ### Настройка производится на BR-SRV:  
    • Рабочий каталог ansible должен располагаться в /etc/ansible  
      dnf install ansible -y  
      1) В файле можно прописывать как ip адреса так и имена хостов, сделаем следующим образом.  
      Так как у нас порт для покдлючения к серверам и клиентам 2024, укажим необходимые переменные для подключения  
      Для роутера так же указываем переменные, для подключения к роутерам будем использовать пользователя net_admin
      nano /etc/ansible/inventory.ini  
      [clients]
      hq-cli ansible_host=192.168.1.65
        
      [servers]
      hq-srv ansible_host=192.168.0.2
         
      [routers]
      hq-rtr ansible_host=192.168.0.62
      br-rtr ansible_host=172.16.5.1
         
      [clients:vars]
      ansible_port=2024
      ansible_user=sshuser
         
      [servers:vars]
      ansible_port=2024
      ansible_user=sshuser
 
      [routers:vars]
      ansible_user=net_admin
      ansible_password=P@$$word
   ![inventory](https://github.com/dizzamer/DEMO2025/blob/main/inventoryini.png) 
   ###  Настройка подключения по ключам на BR-SRV
    2) Подключение к хостам осуществляется по протоколу ssh с помощью rsa ключей.  
    Для начала переходим в ранее созданно пользователя sshuser командой:
    su sshuser
    Сгенерировать серверный ключ можно командой ниже. При её выполнении везде нажмите Enter.
    ssh-keygen
    3) Далее нужно распространить ключ на все подключенные хосты.  
    Распространить ключи на хосты можно командой:  
    На роутеры ключи пробрасывать не нужно, для них подключение по паролю!
    ssh-copy-id sshuser@{hq-srv, hq-cli} 
    где:  
    sshuser - это пользователь, от имени которого будут выполняться плейбуки;  
    server - IP-адрес хоста.  
  ### •	Все указанные машины должны без предупреждений и ошибок отвечать pong на команду ping в ansible посланную с BR-SRV  
    Пингуем удаленные хосты с помощью Ansible находясь в пользователе sshuser:  
    ansible -i /etc/ansible/inventory.ini all -m ping 
    В результате под каждым хостом должно быть написано "ping": "pong".  
![inventory](https://github.com/dizzamer/DEMO2025/blob/main/ansubleping.png) 
## 5.	Развертывание приложений в Docker на сервере BR-SRV. 
    Установка необходимых пакетов:  
    dnf install docker-ce docker-ce-cli docker-compose -y  
    systemctl enable docker --now
    Добавляем текущего пользователя в группу докер, текущий пользователь - student   
    usermod -aG docker $USER  
### •	Создайте в домашней директории пользователя файл wiki.yml для приложения MediaWiki.  
     •	Средствами docker compose должен создаваться стек контейнеров с приложением MediaWiki и базой данных.  
     •	Используйте два сервиса  
     •	Основной контейнер MediaWiki должен называться wiki и использовать образ mediawiki  
     •	Файл LocalSettings.php с корректными настройками должен находиться в домашней папке пользователя и автоматически 
      монтироваться в образ.  
     •	Контейнер с базой данных должен называться mariadb и использовать образ mariadb.  
     •	Он должен создавать базу с названием mediawiki, доступную по стандартному порту, пользователя wiki с паролем   
     WikiP@ssw0rd должен иметь права доступа к этой базе данных  
     •	MediaWiki должна быть доступна извне через порт 8080.  
     Для того, чтобы MediaWiki была доступна извен через порт 8080, нужно в ports сначала указывать 8080  
     Развертывание производится на сервере BR-SRV:   
     touch /home/student/wiki.yml  
     nano /home/student/wiki.yml  
      services:
      MediaWiki:
        container_name: wiki
        image: mediawiki
        restart: always
        ports: 
          - 8080:80
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
      Перед поднятием контейнеров необходимо прописать:
      docker volume create dbvolume
      Поднимаем стек контейнеров с помощью команды: 
      docker compose -f wiki.yml up -d  
 ![wikiyml](https://github.com/dizzamer/DEMO2025/blob/main/wikiyml.png)  
### Настройка mediawiki после успешного поднятия контейнеров  
  Переходим по доменному имени или адреса нашего сервера, должны увидеть такую картину:  
 ![wikistart](https://github.com/dizzamer/DEMO2025/blob/main/mediawiki.png)  
  Далее настройка выглядит следуюшим образом:  
  На скриншоте ниже здесь этап проверки всего необходимого для работы MediaWiki, проверка должна пройти успешно  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/mediawiki2.png)  
  Далее необходимо указать в хосте базы данных то, как у вас называется контнейнер на сервере, у меня mariadb  
  Проверить можно зайдя на сервер и выполнить команду docker ps:  
  ![hostwiki](https://github.com/dizzamer/DEMO2025/blob/main/hostwiki.png)  
  Указываем, все как по заданию и жмем далее:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/mediawiki3.png)  
  Далее будет такое у вас как снизу, жмем далее:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/mediawiki4.png)  
  Настройка дальше на скриншоте снизу:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/mediawiki5.png)  
  На след скриншоте жмем далее:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/mediawiki6.png)  
  Настройка базы данных должна быть выполнена успешно, как на скриншоте ниже:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/mediawiki77.png)  
  Далее у вас должен скачаться файл LocalSettings.php автомтаически:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/mediawiki8.png)  
  Потом переходим в директории загрузки:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/localsettings.png)  
  Открывем директорию через терминал:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/localsettingsterm.png)  
  Перекидываем его через scp следующим образом:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/localsettingsterm2.png)  
  Переходим на сервер и убеждаем, что файл находится рядом с нашим wiki.yml:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/localsettingsterm3.png)  
  Далее раскоменчиваем строку в файле wiki.yml:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/localsettingswikiyml.png)   
  Запускаем стек контейнеров командой docker-compose -f wiki.yml up -d    
  Далее переходим по доменному имени или адреса нашего сервера и наблюдаем вот это:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/wikidemo.png)  
  Для проверки того, что все получилось входим под админской учеткой Wiki:WikiP@ssw0rd:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/wikidemologin.png)  
  Должно получиться вот так:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/wikiuser.png)  
## 6.	На маршрутизаторах сконфигурируйте статическую трансляцию портов  
### •	Пробросьте порт 80 в порт 8080 на BR-SRV на маршрутизаторе BR-RTR, для обеспечения работы сервиса wiki  
     Настройка производится на EcoRouter BR-RTR:  
     ip nat destination static tcp 192.168.1.2 80 192.168.1.65 8080
     ip nat destination static tcp 172.16.5.1 80 192.168.1.2 8080
### •	Пробросьте порт 2024 в порт 2024 на HQ-SRV на маршрутизаторе HQ-RTR  
     Настройка производится на EcoRouter HQ-RTR:  
     ip nat destination static tcp 192.168.0.2 2024 192.168.1.65 2024  
### •	Пробросьте порт 2024 в порт 2024 на BR-SRV на маршрутизаторе BR-RTR  
     Настройка производится на EcoRouter BR-RTR:  
     ip nat destination static tcp 192.168.1.2 2024 192.168.1.65 2024  
## 7.	Запустите сервис moodle на сервере HQ-SRV:  
### Подготовка  
 Выключаем selinux:  
 setenforce 0  
 dnf install -y git httpd mariadb-server php php-cli php-common php-fpm php-gd php-intl php-json php-mbstring php-mysqlnd php-opcache php-pdo php-xml php-xmlrpc php-pecl-zip php-soap   
## •	Используйте веб-сервер apache  
    systemctl enable --now httpd  
    Создаем конфигурационный файл /etc/httpd/conf.d/moodle.conf:  
    nano /etc/httpd/conf.d/moodle.conf  
    <VirtualHost *:80>  
        DocumentRoot "/var/www/html/moodle"  
        ServerName HQ-SRV  
        <Directory "/var/www/html/moodle">  
            AllowOverride All  
            Require all granted  
        </Directory>  
        ErrorLog "/var/log/httpd/moodle_error.log"  
        CustomLog "/var/log/httpd/moodle_access.log" combined  
    </VirtualHost>  
     Перезапускаем Apache:  
     systemctl restart httpd   
## •	В качестве системы управления базами данных используйте mariadb  
     systemctl enable --now mariadb  
     mysql_secure_installation  
## •	Создайте базу данных moodledb  
     mysql -u root -p   
     CREATE DATABASE moodledb DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;  
## •	Создайте пользователя moodle с паролем P@ssw0rd и предоставьте ему права доступа к этой базе данных  
     CREATE USER 'moodle'@'localhost' IDENTIFIED BY 'P@ssw0rd';   
     GRANT ALL PRIVILEGES ON moodledb.* TO 'moodle'@'localhost';   
     FLUSH PRIVILEGES;   
     EXIT;   
## •	У пользователя admin в системе обучения задайте пароль P@ssw0rd  
     Создаем директории для нашего moodle  
     mkdir /opt/moodle  
     mkdir /usr/moodle_data  
     Далее переходим в директорию и клонируем  
     cd /opt/moodle  
     git clone git://git.moodle.org/moodle.git  
     cd /opt/moodle/moodle  
     cp config-dist.php config.php  
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
•	При обращении к HQ-RTR по доменному имени wiki.au-team.irpo клиента должно перенаправлять на BR-SRV на порт, на сервис mediwiki  
## 9.	Удобным способом установите приложение Яндекс Браузер для организаций на HQ-CLI  
•	Установку браузера отметьте в отчёте  

# Модуль № 3:Эксплуатация объектов сетевой инфраструктуры    


Необходимые приложения:  
Приложение. Файл users.csv (в отдельном файле).  
 
