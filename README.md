🛠 Шпаргалка по настройке оборудования Eltex (MES / ESR)Краткий справочник конфигурации сети. Важно: на маршрутизаторах (ESR) команды применяются не сразу, поэтому в скриптах добавлены do commit и do confirm после основных логических блоков. Коммутаторы (MES) применяют команды на лету.🕒 1. Базовые настройки (Для всех устройств: S1, S2, R1, R2)Установка времени, имени хоста и создание администратора.clock set 13:00:00 21 June 2026
configure
hostname S1  ! Заменить на актуальное (S2, R1, R2)
username Eltex password P@ssw0rd privilege 1
enable password level 15 EltexEltex

🖧 2. Настройка коммутаторов (S1 и S2)Создание VLANvlan 21
name Managers
vlan active
exit
vlan 22
name Sale
vlan active
exit
vlan 66
name ShutDowned
vlan active
exit
vlan 100
name native
vlan active
exit
vlan 20
name Management
vlan active
exit
IP-адрес управления и шлюзПример для S1 (.239). Для S2 указать .240.interface vlan 20
 ip address 192.168.20.239 /24
 exit
ip default-gateway 192.168.20.1

Агрегированный канал (Port-Channel S1 ↔ S2)interface range gi0/23-24
 channel-group 1 mode auto 
 exit

interface port-channel 1
 switchport mode trunk
 switchport trunk native vlan 100
 switchport trunk allowed vlan add 20,21,22
 exit

Распределение портов (Пример для S1)Для S2 аналогично, только убираем транк на gi0/1.! Транк до маршрутизатора R1
interface gi0/1
 switchport mode trunk
 switchport trunk native vlan 100
 switchport trunk allowed vlan add 20,21,22
 exit

! Access-порты для ПК
interface gi0/5
 switchport mode access
 switchport access vlan 21
 exit
interface gi0/10
 switchport mode access
 switchport access vlan 22
 exit

! Выключение неиспользуемых портов
interface range gi0/2-4, gi0/6-9, gi0/11-22
 switchport mode access
 switchport access vlan 66
 shutdown
 exit

🌐 3. Настройка маршрутизатора R1 (Router-on-a-stick + GRE + OSPF)Базовые интерфейсы! Эмуляция внутренней сети
interface loopback 1
 ip address 172.16.1.1/24
 exit

! Внешний порт (к ISP)
interface gi1/0/3
 ip address 100.1.10.1/30
 no shut
 exit
 
do commit
do confirm

Сабинтерфейсы (Шлюзы для VLAN)interface gi1/0/2
 no shut
 exit

interface gi1/0/2.20
 encapsulation dot1q 20
 ip address 192.168.20.1/24
 exit

interface gi1/0/2.21
 encapsulation dot1q 21
 ip address 192.168.21.1/24
 ! Перенаправление DHCP-запросов на R2 (IP туннеля)
 ip helper-address 10.10.10.2
 exit

interface gi1/0/2.22
 encapsulation dot1q 22
 ip address 192.168.22.1/24
 exit
 
do commit
do confirm

GRE Туннель и Маршрутизация (OSPF)! Поднятие туннеля до R2
interface tunnel 1
 tunnel source 100.1.10.1
 tunnel destination 100.1.20.2
 ip address 10.10.10.1/30
 exit

! Настройка OSPF
router ospf 1
 network 192.168.20.0/24 area 0
 network 192.168.21.0/24 area 0
 network 192.168.22.0/24 area 0
 network 172.16.1.0/24 area 0
 network 10.10.10.0/30 area 0
 exit

! Маршрут по умолчанию в интернет (через провайдера)
ip route 0.0.0.0/0 100.1.10.2

do commit
do confirm

🌍 4. Настройка маршрутизатора R2 (DHCP + GRE + OSPF)Интерфейсы и DHCP-сервер! Петлевой интерфейс
interface loopback 2
 ip address 172.16.2.1/24
 exit

! Внешний порт (к ISP)
interface gi1/0/4
 ip address 100.1.20.2/30
 no shut
 exit

! Настройка пула DHCP для VLAN 21 (Менеджеры)
ip dhcp-server pool VLAN21
 network 192.168.21.0/24
 address-range 192.168.21.100-192.168.21.199
 default-router 192.168.21.1
 exit
ip dhcp-server enable

do commit
do confirm

GRE Туннель и Маршрутизация (OSPF)! Поднятие туннеля до R1
interface tunnel 1
 tunnel source 100.1.20.2
 tunnel destination 100.1.10.1
 ip address 10.10.10.2/30
 exit

! Настройка OSPF
router ospf 1
 network 172.16.2.0/24 area 0
 network 10.10.10.0/30 area 0
 exit

! Маршрут по умолчанию в интернет
ip route 0.0.0.0/0 100.1.20.1

do commit
do confirm

