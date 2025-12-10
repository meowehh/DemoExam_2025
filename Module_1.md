# 🛠️ Руководство по выполнению демонстрационного экзамена

## Общие требования
- **Время выполнения:** 2 часа 30 минут (1 модуль = 1 час, 2 модуль = 1.5 часа)
- **Оборудование:** 10 серверов (Alt JeOS, Alt Server), 2 клиентские машины (Alt Workstation), в моем случае все выполнялось на развернутом Proxmox внутри VMware Workstation.

---

# Модуль 1: Сетевое администрирование (1 час)

## 📋 Задание 1: Базовая настройка устройств + Задание 4: Настройте на интерфейсе HQ-RTR в сторону офиса HQ виртуальный коммутатор. + Задание 11. Настройте часовой пояс на всех устройствах, согласно месту проведения экзамена.

**Задание 1**:
- Настройте имена устройств согласно топологии. Используйте полное доменное имя.
- На всех устройствах необходимо сконфигурировать IPv4
- IP-адрес должен быть из приватного диапазона, в случае, если сеть локальная, согласно RFC1918
- Локальная сеть в сторону HQ-SRV(VLAN10) должна вмещать не более 32 адресов
- Локальная сеть в сторону HQ-CLI(VLAN20) должна вмещать не менее 16 адресов
- Локальная сеть в сторону BR-SRV должна вмещать не более 16 адресов
- Локальная сеть для управления(VLAN99) должна вмещать не более 8 адресов
- Сведения об адресах занесите в таблицу
  
**Задание 4**:
- Сервер HQ-SRV должен находиться в ID VLAN 10
- Клиент HQ-CLI в ID VLAN 20
- Создайте подсеть управления с ID VLAN 99
- Основные сведения о настройке коммутатора и выбора реализации разделения на VLAN занесите в [отчёт](./report_2025.docx)

**Задание 11**:
- Настройте часовой пояс на всех устройствах, согласно месту проведения экзамена.

## Выполнение:
### Настройка hostname и часового пояса.
```bash
hostnamectl set-hostname <полное_доменное_имя>; exec bash
```
### ISP
```bash
hostnamectl set-hostname isp.au-team.irpo; exec bash

apt-get update && apt-get install tzdata -y # Нужно скачать tzdata только на ISP так как его по умолчанию здесь нет (Исключение: Москва), из-за обочки Alt JeOS, на Alt Server И Alt Workstation можно сразу установить часовой пояс.

timedatectl set-timezone Asia/Novosibirsk
```
**Выполним проверку**:
```bash
timedatectl
```
```bash
               Local time: Thu 2025-12-04 03:27:44 +07
           Universal time: Wed 2025-12-03 20:27:44 UTC
                 RTC time: Wed 2025-12-03 20:27:44
                Time zone: Asia/Novosibirsk (+07, +0700)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```
### HQ-RTR
```bash
hostnamectl set-hostname hq-rtr.au-team.irpo; exec bash
timedatectl set-timezone Asia/Novosibirsk
```
### HQ-SRV
```bash
hostnamectl set-hostname hq-srv.au-team.irpo; exec bash
timedatectl set-timezone Asia/Novosibirsk
```
### HQ-CLI
```bash
hostnamectl set-hostname hq-cli.au-team.irpo; exec bash
timedatectl set-timezone Asia/Novosibirsk
```
### BR-RTR
```bash
hostnamectl set-hostname br-rtr.au-team.irpo; exec bash
timedatectl set-timezone Asia/Novosibirsk
```
### BR-SRV
```bash
hostnamectl set-hostname br-srv.au-team.irpo; exec bash
timedatectl set-timezone Asia/Novosibirsk
```

> ⚠️ 💡 **Важно**: Хоть в задании не указано дать название ISP, но для корректного функционирования DNS и других сервисов требуется выдать полное доменное имя всем устройствам.

>⚠️ **Примечание**: Команда hostnamectl set-hostname применяет изменения немедленно без перезагрузки. Флаг ; exec bash обновляет текущую сессию shell для отображения нового hostname в приглашении командной строки.

### Конфигурация IPv4 адресов.

### ISP
```bash
mkdir /etc/net/ifaces/ens19
mkdir /etc/net/ifaces/ens20
```
```bash
vim /etc/net/ifaces/ens19/ipv4address
172.16.40.1/28
vim /etc/net/ifaces/ens20/ipv4address
172.16.50.1/28
```
```bash
vim /etc/net/ifaces/ens19/options
BOOTPROTO=static
TYPE=eth
vim /etc/net/ifaces/ens20/options
BOOTPROTO=static
TYPE=eth
```
```bash
systemctl restart network
ip -c -br a
```
Должен быть такой вывод у команды:
```bash
lo               UNKNOWN        127.0.0.1/8 ::1/128 
ens18            UP             192.168.120.155/24 fe80::be24:11ff:fe1b:fd7a/64 
ens19            UP             172.16.40.1/28 fe80::be24:11ff:fe46:6db7/64 
ens20            UP             172.16.50.1/28 fe80::be24:11ff:fe03:76a0/64 
```

> ⚠️ 💡 **Примечание**: Для ens18 вывод может отличаться из-за того что у всех этот интерфейс зависит от их собственной локальной сети, так как это интерфейс через который идет выход в интернет с помощью Bridge из Proxmox в VMware, в VMware обязательно нужно было указать Bridge в типе сетевого подключения, тип NAT или создание отдельной Network внутри VMware может вызывать нестабильность в работе!

### HQ-RTR
```bash
mkdir /etc/net/ifaces/ens19
mkdir /etc/net/ifaces/ens19.10
mkdir /etc/net/ifaces/ens19.20
mkdir /etc/net/ifaces/ens19.99
```
```bash
vim /etc/net/ifaces/ens18/ipv4address
172.16.40.2/28
vim /etc/net/ifaces/ens19.10/ipv4address
192.168.10.1/27
vim /etc/net/ifaces/ens19.20/ipv4address
192.168.20.65/28
vim /etc/net/ifaces/ens19.99/ipv4address
192.168.99.91/29
```
⚠️ 💡 **Для ens18 (/etc/net/ifaces/ens18/options) в HQ-RTR, нужно заменить**:
```bash
BOOTPROTO=dhcp
TYPE=eth
CONFIG_WIRELESS=no
SYSTEMD_BOOTPROTO=dhcp4
CONFIG_IPV4=yes
DISABLED=no
NM_CONTROLLED=no
SYSTEMD_CONTROLLED=no
```
**На те параметры что указаны ниже**:
```bash
vim /etc/net/ifaces/ens18/options
BOOTPROTO=static
TYPE=eth
```
```bash
vim /etc/net/ifaces/ens19/options
BOOTPROTO=none
TYPE=eth
vim /etc/net/ifaces/ens19.10/options
BOOTPROTO=static
TYPE=vlan
VID=10
HOST=ens19
vim /etc/net/ifaces/ens19.20/options
BOOTPROTO=static
TYPE=vlan
VID=20
HOST=ens19
vim /etc/net/ifaces/ens19.99/options
BOOTPROTO=static
TYPE=vlan
VID=99
HOST=ens19
```
```bash
vim /etc/net/ifaces/ens18/ipv4route
default via 172.16.40.1
```
```bash
vim /etc/net/ifaces/ens18/resolv.conf
nameserver 77.88.8.8
```
```bash
systemctl restart network
ip -c -br a
```
Должен быть такой вывод у команды:
```bash
lo               UNKNOWN        127.0.0.1/8 ::1/128 
ens18            UP             172.16.40.2/28 fe80::be24:11ff:fe5e:1371/64 
ens19            UP             fe80::be24:11ff:fea9:5f29/64 
ens19.10@ens19   UP             192.168.10.1/27 fe80::be24:11ff:fea9:5f29/64 
ens19.20@ens19   UP             192.168.20.65/28 fe80::be24:11ff:fea9:5f29/64 
ens19.99@ens19   UP             192.168.99.91/29 fe80::be24:11ff:fea9:5f29/64 
```
> ⚠️ 💡 **Важно!**: Так как VLAN созданы через network внутри Proxmox, обязательно идем в веб панель Proxmox VE, заходим в раздел Server View > Datacenter > pve. В этом разделе в открытом списке выбираем 10103,10104 машины (HQ-SRV,HQ-CLI), заходим в настройки во вкладку Hardware, меняем в графе Network Device (net0) VLAN tag, с того который там указан на 10 для HQ-CLI, и на 20 для HQ-SRV. Перезапускать машины не нужно.


### HQ-SRV
⚠️ 💡 **Для ens18 (/etc/net/ifaces/ens18/options) в HQ-SRV, нужно заменить**:
```bash
BOOTPROTO=dhcp
TYPE=eth
CONFIG_WIRELESS=no
SYSTEMD_BOOTPROTO=dhcp4
CONFIG_IPV4=yes
DISABLED=no
NM_CONTROLLED=no
SYSTEMD_CONTROLLED=no
```
**На те параметры что указаны ниже**:
```bash
vim /etc/net/ifaces/ens18/options
BOOTPROTO=static
TYPE=eth
```
```bash
vim /etc/net/ifaces/ens18/ipv4address
192.168.10.2/27
vim /etc/net/ifaces/ens18/ipv4route
default via 192.168.10.1
vim /etc/net/ifaces/ens18/resolv.conf
nameserver 77.88.8.8
```
```bash
systemctl restart network
ip -c -br a
```
Должен быть такой вывод у команды:
```bash
lo               UNKNOWN        127.0.0.1/8 ::1/128 
ens18            UP             192.168.10.2/27 fe80::be24:11ff:feff:6538/64 
```
### BR-RTR
```bash
mkdir /etc/net/ifaces/ens19
```
```bash
vim /etc/net/ifaces/ens18/ipv4address
172.16.50.2/28
vim /etc/net/ifaces/ens19/ipv4address
192.168.30.1/28
```
⚠️ 💡 **Для ens18 (/etc/net/ifaces/ens18/options) в BR-RTR, нужно заменить**:
```bash
BOOTPROTO=dhcp
TYPE=eth
CONFIG_WIRELESS=no
SYSTEMD_BOOTPROTO=dhcp4
CONFIG_IPV4=yes
DISABLED=no
NM_CONTROLLED=no
SYSTEMD_CONTROLLED=no
```
**На те параметры что указаны ниже**:
```bash
vim /etc/net/ifaces/ens18/options
BOOTPROTO=static
TYPE=eth
```
```bash
vim /etc/net/ifaces/ens19/options
BOOTPROTO=static
TYPE=eth
```
```bash
vim /etc/net/ifaces/ens18/ipv4route
default via 172.16.50.1
vim /etc/net/ifaces/ens18/resolv.conf
nameserver 77.88.8.8
```
```bash
systemctl restart network
ip -c -br a
```
Должен быть такой вывод у команды:
```bash
lo               UNKNOWN        127.0.0.1/8 ::1/128 
ens18            UP             172.16.50.2/28 fe80::be24:11ff:feab:8a59/64 
ens19            UP             192.168.30.1/28 fe80::be24:11ff:fe58:e15d/64 
```

### BR-SRV
⚠️ 💡 **Для ens18 (/etc/net/ifaces/ens18/options) в BR-SRV, нужно заменить**:
```bash
BOOTPROTO=dhcp
TYPE=eth
CONFIG_WIRELESS=no
SYSTEMD_BOOTPROTO=dhcp4
CONFIG_IPV4=yes
DISABLED=no
NM_CONTROLLED=no
SYSTEMD_CONTROLLED=no
```
**На те параметры что указаны ниже**:
```bash
vim /etc/net/ifaces/ens18/options
BOOTPROTO=static
TYPE=eth
```
```bash
vim /etc/net/ifaces/ens18/ipv4address
192.168.30.2/28
vim /etc/net/ifaces/ens18/ipv4route
default via 192.168.30.1
vim /etc/net/ifaces/ens18/resolv.conf
nameserver 77.88.8.8
```
```bash
systemctl restart network
ip -c -br a
```
Должен быть такой вывод у команды:
```bash
lo               UNKNOWN        127.0.0.1/8 ::1/128 
ens18            UP             192.168.30.2/28 fe80::be24:11ff:fe3c:a3dd/64 
```

> ⚠️ 💡 **Примечание!**: HQ-CLI будет настроен позднее так как там будет использоваться DHCP настройка, на данном этапе теперь требуется настроить проброс портов чтобы пинг начал ходить между устройствами и появился доступ в интернет со всех машин, так же все отчеты будут приведны в отдельном [файле](./report_2025.docx), сейчас заполнять ничего не требуется, несмотря на задание.

## 📋 Задание 2: Настройка ISP + Задание 8: Настройка динамической трансляции адресов.

### Задание 2:
- Настройте адресацию на интерфейсах. **(Уже выполнено [здесь](https://github.com/meowehh/DemoExam_2025/edit/main/Module_1.md#isp-1))**
- Интерфейс, подключенный к магистральному провайдеру, получает адрес по DHCP **(Изначально так и есть, ничего делать не нужно)**
- Настройте маршруты по умолчанию там, где это необходимо. **(Уже выполнено [здесь](https://github.com/meowehh/DemoExam_2025/edit/main/Module_1.md#isp-1))**
- Интерфейс, к которому подключен HQ-RTR, подключен к сети 172.16.40.0/28 **(Уже выполнено [здесь](https://github.com/meowehh/DemoExam_2025/edit/main/Module_1.md#isp-1))**
- Интерфейс, к которому подключен BR-RTR, подключен к сети 172.16.50.0/28 **(Уже выполнено [здесь](https://github.com/meowehh/DemoExam_2025/edit/main/Module_1.md#isp-1))**
- На ISP настройте динамическую сетевую трансляцию в сторону HQ-RTR и BR-RTR для доступа к сети Интернет.
### Задание 8:
- Настройте динамическую трансляцию адресов для обоих офисов.
- Все устройства в офисах должны иметь доступ к сети Интернет

### ISP
```bash
vim /etc/net/sysctl.conf
net.ipv4.ip_forward = 1

sysctl -p
systemctl restart network
```
```bash
apt-get update && apt-get install iptables -y

iptables -t nat -A POSTROUTING -o ens18 -s 172.16.40.0/28 -j MASQUERADE
iptables -t nat -A POSTROUTING -o ens18 -s 172.16.50.0/28 -j MASQUERADE

iptables -A FORWARD -i ens19 -o ens18 -s 172.16.40.0/28 -j ACCEPT
iptables -A FORWARD -i ens20 -o ens18 -s 172.16.50.0/28 -j ACCEPT

iptables-save > /etc/sysconfig/iptables
systemctl enable iptables --now
systemctl restart iptables
```
```bash
systemctl status iptables
iptables -t nat -L -n -v
```
Должны быть такие выводы у команд:
```bash
● iptables.service - IPv4 firewall with iptables
     Loaded: loaded (/usr/lib/systemd/system/iptables.service; enabled; preset: disabled)
     Active: active (exited) since Wed 2025-12-03 02:43:32 UTC; 7s ago
    Process: 6973 ExecStart=/etc/init.d/iptables start (code=exited, status=0/SUCCESS)
   Main PID: 6973 (code=exited, status=0/SUCCESS)
        CPU: 17ms
Chain PREROUTING (policy ACCEPT 1 packets, 68 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MASQUERADE  0    --  *      ens18   172.16.40.0/28       0.0.0.0/0           
    0     0 MASQUERADE  0    --  *      ens18   172.16.50.0/28       0.0.0.0/0  
```
> ⚠️ 💡 **Примечание!**: Сразу же настроим интернет на всех устройствах, для этого потребуется повтороить настройку на всех устройствах, детали приведены ниже.

### HQ-RTR
```bash
vim /etc/net/sysctl.conf
net.ipv4.ip_forward = 1

sysctl -p
systemctl restart network
```
```bash
apt-get update && apt-get install iptables -y

iptables -t nat -A POSTROUTING -o ens18 -s 192.168.10.0/27 -j MASQUERADE
iptables -t nat -A POSTROUTING -o ens18 -s 192.168.20.64/28 -j MASQUERADE
iptables -t nat -A POSTROUTING -o ens18 -s 192.168.99.88/29 -j MASQUERADE

iptables -A FORWARD -i ens19.10 -o ens18 -s 192.168.10.0/27 -j ACCEPT
iptables -A FORWARD -i ens19.20 -o ens18 -s 192.168.20.64/28 -j ACCEPT
iptables -A FORWARD -i ens19.99 -o ens18 -s 192.168.99.88/29 -j ACCEPT

iptables-save > /etc/sysconfig/iptables
systemctl enable iptables --now
systemctl restart iptables
```
```bash
systemctl status iptables
iptables -t nat -L -n -v
```
Должны быть такие выводы у команд:
```bash
● iptables.service - IPv4 firewall with iptables
     Loaded: loaded (/lib/systemd/system/iptables.service; enabled; vendor preset: disabled)
     Active: active (exited) since Wed 2025-12-03 06:07:07 MSK; 7s ago
    Process: 3143 ExecStart=/etc/init.d/iptables start (code=exited, status=0/SUCCESS)
   Main PID: 3143 (code=exited, status=0/SUCCESS)
        CPU: 9ms

Dec 03 06:07:07 hq-rtr.au-team.irpo systemd[1]: iptables.service: Deactivated successfully.
Dec 03 06:07:07 hq-rtr.au-team.irpo systemd[1]: Stopped IPv4 firewall with iptables.
Dec 03 06:07:07 hq-rtr.au-team.irpo systemd[1]: Starting IPv4 firewall with iptables...
Dec 03 06:07:07 hq-rtr.au-team.irpo iptables[3157]: Applying iptables firewall rules: succeeded
Dec 03 06:07:07 hq-rtr.au-team.irpo iptables[3143]: Applying iptables firewall rules: [ DONE ]
Dec 03 06:07:07 hq-rtr.au-team.irpo systemd[1]: Finished IPv4 firewall with iptables.

Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 1 packets, 76 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain POSTROUTING (policy ACCEPT 1 packets, 76 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MASQUERADE  all  --  *      ens18   192.168.10.0/27      0.0.0.0/0           
    0     0 MASQUERADE  all  --  *      ens18   192.168.20.64/28     0.0.0.0/0           
    0     0 MASQUERADE  all  --  *      ens18   192.168.99.88/29     0.0.0.0/0 
```
### BR-RTR
```bash
vim /etc/net/sysctl.conf
net.ipv4.ip_forward = 1

sysctl -p
systemctl restart network
```
```bash
apt-get update && apt-get install iptables -y

iptables -t nat -A POSTROUTING -o ens18 -s 192.168.30.0/28 -j MASQUERADE
iptables -A FORWARD -i ens19 -o ens18 -s 192.168.30.0/28 -j ACCEPT

iptables-save > /etc/sysconfig/iptables

systemctl enable iptables --now
systemctl restart iptables
```
```bash
systemctl status iptables
iptables -t nat -L -n -v
```
Должны быть такие выводы у команд:
```bash
● iptables.service - IPv4 firewall with iptables
     Loaded: loaded (/lib/systemd/system/iptables.service; enabled; vendor preset: disabled)
     Active: active (exited) since Wed 2025-12-03 06:12:50 MSK; 5s ago
    Process: 5577 ExecStart=/etc/init.d/iptables start (code=exited, status=0/SUCCESS)
   Main PID: 5577 (code=exited, status=0/SUCCESS)
        CPU: 9ms

Dec 03 06:12:50 br-rtr.au-team.irpo systemd[1]: iptables.service: Deactivated successfully.
Dec 03 06:12:50 br-rtr.au-team.irpo systemd[1]: Stopped IPv4 firewall with iptables.
Dec 03 06:12:50 br-rtr.au-team.irpo systemd[1]: Starting IPv4 firewall with iptables...
Dec 03 06:12:50 br-rtr.au-team.irpo iptables[5591]: Applying iptables firewall rules: succeeded
Dec 03 06:12:50 br-rtr.au-team.irpo iptables[5577]: Applying iptables firewall rules: [ DONE ]
Dec 03 06:12:50 br-rtr.au-team.irpo systemd[1]: Finished IPv4 firewall with iptables.

Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MASQUERADE  all  --  *      ens18   192.168.30.0/28      0.0.0.0/0   
```

>⚠️ **Важно**: На данном этапе уже должен работать выход в Интернет на всех устройствах (кроме HQ-CLI, его настроим позже по DHCP), а также пинг между ними. Если что-то не работает, значит где-то ошибка.


## 📋 Задание 3: Создание локальных учетных записей

- Создать пользователя sshuser на серверах HQ-SRV и BR-SRV.
- Пароль пользователя sshuser с паролем P@ssw0rd
- Идентификатор пользователя 1015
- Пользователь sshuser должен иметь возможность запускать sudo без дополнительной аутентификации.
- Создать пользователя net_admin на маршрутизаторах (в нашем случае ALT Server) HQ-RTR и BR-RTR
- Пароль пользователя net_admin с паролем P@$$word
- При настройке на EcoRouter пользователь net_admin должен обладать максимальными привилегиями (⚠️ Не выполняется, так как в нашем случае вместо EcoRouter испольузется ALT Server с эмуляцией роутера через FRR позднее.)
- При настройке ОС на базе Linux, запускать sudo без дополнительной аутентификации

### HQ-SRV и BR-SRV
```bash
useradd sshuser -u 1015 -U
passwd sshuser
usermod -a -G wheel sshuser

vim /etc/sudoers
## Same thing without a password
# WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL
sshuser ALL=(ALL) NOPASSWD: ALL
```
Из под нового пользователя sshuser должен быть доступ без пароля:
```bash
sudo cat /root/.bashrc
```

### HQ-RTR и BR-RTR
```bash
useradd net_admin
passwd net_admin
usermod -a -G wheel net_admin

vim /etc/sudoers
## Same thing without a password
# WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL
net_admin ALL=(ALL) NOPASSWD: ALL
```
Из под нового пользователя net_admin должен быть доступ без пароля:
```bash
sudo cat /root/.bashrc
```

>⚠️ 💡 Примечание: Пароль для пользователя sshuser установлен как P@ssw0rd, а для пользователя net_admin установлен как P@$$word. Оба пользователя добавлены в группу wheel и могут выполнять команды через sudo без дополнительной аутентификации.

>⚠️ Важно: После редактирования файла /etc/sudoers рекомендуется выполнить проверку синтаксиса командой visudo -c. Для применения изменений перезагрузка системы не требуется. В случае ошибок в /etc/sudoers.d/99-sudopw - игнорируем, главное чтобы не было ошибок в /etc/sudoers, ответ парсинга - OK.

## 📋 Задание 5: Настройка безопасного удаленного доступа на серверах HQ-SRV и BR-SRV.

- Для подключения используйте порт 3015.
- Разрешите подключения только пользователю sshuser.
- Ограничьте количество попыток входа до двух.
- Настройте баннер «Authorized access only».

### BR-SRV
```bash
apt-get update && apt-get install openssh-server -y
```
```bash
vim /etc/openssh/sshd_config
Port 3015
MaxAuthTries 2
Banner /etc/openssh/sshd_banner
AllowUsers sshuser
```
```bash
vim /etc/openssh/sshd_banner
«Authorized access only»
```
```bash
systemctl enable sshd --now
systemctl restart sshd

systemctl status sshd
```
Должен быть такой вывод у команды:
```bash
● sshd.service - OpenSSH server daemon
     Loaded: loaded (/lib/systemd/system/sshd.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2025-12-03 09:12:33 MSK; 11s ago
    Process: 16017 ExecStartPre=/usr/bin/ssh-keygen -A (code=exited, status=0/SUCCESS)
    Process: 16018 ExecStartPre=/usr/sbin/sshd -t (code=exited, status=0/SUCCESS)
   Main PID: 16019 (sshd)
      Tasks: 1 (limit: 1149)
     Memory: 744.0K
        CPU: 4ms
     CGroup: /system.slice/sshd.service
             └─ 16019 /usr/sbin/sshd -D

Dec 03 09:12:33 br-srv.au-team.irpo systemd[1]: Starting OpenSSH server daemon...
Dec 03 09:12:33 br-srv.au-team.irpo systemd[1]: Started OpenSSH server daemon.
Dec 03 09:12:33 br-srv.au-team.irpo sshd[16019]: Server listening on 0.0.0.0 port 3015.
Dec 03 09:12:33 br-srv.au-team.irpo sshd[16019]: Server listening on :: port 3015.
```
```bash
ssh sshuser@localhost -p 3015
```
Должен быть такой вывод у команды:
```bash
The authenticity of host '[localhost]:3015 ([127.0.0.1]:3015)' can't be established.
ED25519 key fingerprint is SHA256:P0ziiti85F5uQtqHghYPfz/ycFlMD9EElLUGd1txyxQ.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[localhost]:3015' (ED25519) to the list of known hosts.
«Authorized access only»
```

### HQ-SRV
```bash
apt-get update && apt-get install openssh-server -y
```
```bash
vim /etc/openssh/sshd_config
Port 3015
MaxAuthTries 2
Banner /etc/openssh/sshd_banner
AllowUsers sshuser
```
```bash
vim /etc/openssh/sshd_banner
«Authorized access only»
```
```bash
systemctl enable sshd --now
systemctl restart sshd

systemctl status sshd
```
Должен быть такой вывод у команды:
```bash
● sshd.service - OpenSSH server daemon
     Loaded: loaded (/lib/systemd/system/sshd.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2025-12-03 09:15:28 MSK; 12s ago
    Process: 15936 ExecStartPre=/usr/bin/ssh-keygen -A (code=exited, status=0/SUCCESS)
    Process: 15937 ExecStartPre=/usr/sbin/sshd -t (code=exited, status=0/SUCCESS)
   Main PID: 15938 (sshd)
      Tasks: 1 (limit: 1149)
     Memory: 740.0K
        CPU: 5ms
     CGroup: /system.slice/sshd.service
             └─ 15938 /usr/sbin/sshd -D

Dec 03 09:15:28 hq-srv.au-team.irpo systemd[1]: Starting OpenSSH server daemon...
Dec 03 09:15:28 hq-srv.au-team.irpo systemd[1]: Started OpenSSH server daemon.
Dec 03 09:15:28 hq-srv.au-team.irpo sshd[15938]: Server listening on 0.0.0.0 port 3015.
Dec 03 09:15:28 hq-srv.au-team.irpo sshd[15938]: Server listening on :: port 3015.
```
```bash
ssh sshuser@localhost -p 3015
```
Должен быть такой вывод у команды:
```bash
The authenticity of host '[localhost]:3015 ([127.0.0.1]:3015)' can't be established.
ED25519 key fingerprint is SHA256:EnOVdAN2p/vLibaqlXEHECQ9ORSWeIR8Hkckk4KbU0Y.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[localhost]:3015' (ED25519) to the list of known hosts.
«Authorized access only»
```
>⚠️ 💡 Примечание: В файле баннера, нужно поставить 1-2 отступа вниз чтобы баннер корректно отображался, в завимости от редактора vim/nano. Иначе баннер будет наезжать на поле авторизации или на строку приглашения.


## 📋 Задание 6: Между офисами HQ и BR необходимо сконфигурировать ip туннель.

- Сведения о туннеле занесите в отчёт. (Отчет будет приложен отдельным [файлом](./report_2025.docx))
- На выбор технологии GRE или IP in IP.

### HQ-RTR
```bash
mkdir /etc/net/ifaces/gre1
vim /etc/net/ifaces/gre1/options
TYPE=iptun
TUNTYPE=gre
TUNLOCAL=172.16.40.2
TUNREMOTE=172.16.50.2
TUNOPTIONS='ttl 64'
vim /etc/net/ifaces/gre1/ipv4address
10.10.0.1/30

systemctl restart network
ip -c -br a
```
Если все сделано верно получаем следующий вывод:
```bash
lo               UNKNOWN        127.0.0.1/8 ::1/128 
ens18            UP             172.16.40.2/28 fe80::be24:11ff:fe5e:1371/64 
ens19            UP             fe80::be24:11ff:fea9:5f29/64 
ens19.10@ens19   UP             192.168.10.1/27 fe80::be24:11ff:fea9:5f29/64 
ens19.20@ens19   UP             192.168.20.65/28 fe80::be24:11ff:fea9:5f29/64 
ens19.99@ens19   UP             192.168.99.91/29 fe80::be24:11ff:fea9:5f29/64 
gre0@NONE        DOWN           
gretap0@NONE     DOWN           
erspan0@NONE     DOWN           
gre1@NONE        UNKNOWN        10.10.0.1/30 fe80::5efe:ac10:2802/64
```

### BR-RTR
```bash
mkdir /etc/net/ifaces/gre1
vim /etc/net/ifaces/gre1/options
TYPE=iptun
TUNTYPE=gre
TUNLOCAL=172.16.50.2
TUNREMOTE=172.16.40.2
TUNOPTIONS='ttl 64'
vim /etc/net/ifaces/gre1/ipv4address
10.10.0.2/30

systemctl restart network
ip -c -br a
```
Если все сделано верно получаем следующий вывод:
```bash
lo               UNKNOWN        127.0.0.1/8 ::1/128 
ens18            UP             172.16.50.2/28 fe80::be24:11ff:feab:8a59/64 
ens19            UP             192.168.30.1/28 fe80::be24:11ff:fe58:e15d/64 
gre0@NONE        DOWN           
gretap0@NONE     DOWN           
erspan0@NONE     DOWN           
gre1@NONE        UNKNOWN        10.10.0.2/30 fe80::5efe:ac10:3202/64 
```

> **Проверка**: Проверяем работоспобность пингуя по туннелю с 10.10.0.1 на 10.10.0.2 и обратно.

## 📋 Задание 7: Обеспечьте динамическую маршрутизацию: ресурсы одного офиса должны быть доступны из другого офиса. Для обеспечения динамической маршрутизации используйте link state протокол на ваше усмотрение.

- Разрешите выбранный протокол только на интерфейсах в ip туннеле.
- Маршрутизаторы должны делиться маршрутами только друг с другом.
- Обеспечьте защиту выбранного протокола посредством парольной защиты.
- Сведения о настройке и защите протокола занесите в отчёт. ([Отдельный файл](./report_2025.docx))

### HQ-RTR
```bash
apt-get update && apt-get install frr -y
```
```bash
vim /etc/frr/daemons
ospfd=yes
```
```bash
systemctl enable --now frr
systemctl restart frr
reboot
```
```bash
vtysh
show run
```
> Если все было настроено верно (интерфейс gre,ospfd и нигде не было ошибки) получаем такой вывод, самое главное у вас сама по себе должна появиться строка с интерфейсом, не нужно создавать его самим на FRR, нужно выполнить все в точности как у меня, если интерфейс gre1 внутри FRR создается сам, отсекается большая часть проблем.

Вывод:
```bash
Building configuration...

Current configuration:
!
frr version 9.0.2
frr defaults traditional
hostname hq-rtr.au-team.irpo
log file /var/log/frr/frr.log
no ipv6 forwarding
!
interface gre1
 ip ospf network broadcast
exit
!
end
```

### BR-RTR
```bash
apt-get update && apt-get install frr -y
```
```bash
vim /etc/frr/daemons
ospfd=yes
```
```bash
systemctl enable --now frr
systemctl restart frr
reboot
```
```bash
vtysh
show run
```
```bash
Building configuration...

Current configuration:
!
frr version 9.0.2
frr defaults traditional
hostname br-rtr.au-team.irpo
log file /var/log/frr/frr.log
no ipv6 forwarding
!
interface gre1
 ip ospf network broadcast
exit
!
end
```
### HQ-RTR
```bash
hq-rtr.au-team.irpo# conf t
hq-rtr.au-team.irpo(config)# router ospf
hq-rtr.au-team.irpo(config-router)# ospf router-id 172.16.40.1
hq-rtr.au-team.irpo(config-router)# network 10.10.0.0/30 area 0
hq-rtr.au-team.irpo(config-router)# network 192.168.10.0/27 area 0
hq-rtr.au-team.irpo(config-router)# network 192.168.20.64/28 area 0
hq-rtr.au-team.irpo(config-router)# network 192.168.99.88/29 area 0
hq-rtr.au-team.irpo(config-router)# area 0 authentication
hq-rtr.au-team.irpo(config-router)# exit
hq-rtr.au-team.irpo(config)# interface gre1 
hq-rtr.au-team.irpo(config-if)# ip ospf authentication-key P@ssw0rd
hq-rtr.au-team.irpo(config-if)# ip ospf authentication             
hq-rtr.au-team.irpo(config-if)# no ip ospf passive
hq-rtr.au-team.irpo(config-if)# exit
hq-rtr.au-team.irpo(config)# exit
hq-rtr.au-team.irpo# wr
```
### BR-RTR
```bash
br-rtr.au-team.irpo# conf t
br-rtr.au-team.irpo(config)# router ospf
br-rtr.au-team.irpo(config-router)# ospf router-id 172.16.50.1
br-rtr.au-team.irpo(config-router)# network 10.10.0.0/30 area 0
br-rtr.au-team.irpo(config-router)# network 192.168.30.0/28 area 0
br-rtr.au-team.irpo(config-router)# area 0 authentication 
br-rtr.au-team.irpo(config-router)# exit
br-rtr.au-team.irpo(config)# interface gre1
br-rtr.au-team.irpo(config-if)# ip ospf authentication-key P@ssw0rd
br-rtr.au-team.irpo(config-if)# ip ospf authentication             
br-rtr.au-team.irpo(config-if)# no ip ospf passive
br-rtr.au-team.irpo(config-if)# exit
br-rtr.au-team.irpo(config)# exit
br-rtr.au-team.irpo# wr
```
Проверим работоспобность OSPF, для этого воспользуемся информацией о соседях полученных через OSPF, состояние должно быть Full/DR,Full/Backup.
```bash
hq-rtr.au-team.irpo# show ip ospf neighbor 

Neighbor ID     Pri State           Up Time         Dead Time Address         Interface                        RXmtL RqstL DBsmL
172.16.50.1       1 Full/DR         3m00s             34.229s 10.10.0.2       gre1:10.10.0.1                       0     0     0
```
```bash
br-rtr.au-team.irpo# show ip ospf neighbor

Neighbor ID     Pri State           Up Time         Dead Time Address         Interface                        RXmtL RqstL DBsmL
172.16.40.1       1 Full/Backup     3m33s             36.822s 10.10.0.1       gre1:10.10.0.2                       0     0     0
```

>⚠️ 💡 Примечание: После того как OSPF успешно работает, нужно проверить пинг, напимер с HQ-SRV попробовать пинговать BR-SRV и обратно, пинг должен успешно проходить между любыми устройствами, кроме ISP и пока не настроенного HQ-CLI.

## 📋 Задание 9: Настройка протокола динамической конфигурации хостов. 

**Задание 9**:
- Настройть нужную подсеть.
- Для офиса HQ в качестве сервера DHCP выступает маршрутизатор HQ-RTR.
- Клиентом является машина HQ-CLI.
- Исключить из выдачи адрес маршрутизатора.
- Адрес шлюза по умолчанию – адрес маршрутизатора HQ-RTR.
- Адрес DNS-сервера для машины HQ-CLI – адрес сервера HQ-SRV.
- DNS-суффикс для офисов HQ – au-team.irpo
- Сведения о настройке протокола занесите в [отчёт](./report_2025.docx)
  
### HQ-RTR
```bash
apt-get update && apt-get install dhcp-server nano -y #Рекомендуется настраивать через nano для корретной табуляци внутри dhcpd.conf.
nano /etc/dhcp/dhcpd.conf.sample #Взять шаблон конфига настроек можно отсюда или готовый ниже.
```
**Готовый конфиг**:
```bash
nano /etc/dhcp/dhcpd.conf
subnet 192.168.20.64 netmask 255.255.255.240 {
        option routers                  192.168.20.65;
        option subnet-mask              255.255.255.240;

        option domain-name              "au-team.irpo";
        option domain-name-servers      192.168.10.2;

        range dynamic-bootp 192.168.20.66 192.168.20.78;
        default-lease-time 600;
        max-lease-time 7200;
}
```
```bash
systemctl enable --now dhcpd
systemctl restart dhcpd
```
> С настройкой DHCP-сервера закончено, теперь получим IP если HQ-CLI ещё этого не сделал сам.

### HQ-CLI
```bash
dhcpcd
```
Вывод у команды должен быть таким:
```bash
dhcpcd-9.4.0 starting
DUID 00:04:a4:4f:22:43:ad:81:49:e1:b2:c7:06:fb:19:ec:1c:6a
ens18: soliciting a DHCP lease
ens18: offered 192.168.20.66 from 192.168.20.65
ens18: leased 192.168.20.66 for 600 seconds
ens18: adding route to 192.168.20.64/28
ens18: adding default route via 192.168.20.65
forked to background, child pid 2593
```

### HQ-RTR

**Проверка службы на возможные ошибки**:
```bash
systemctl status dhcpd
```
Вывод должен быть таким:
```bash
● dhcpd.service - DHCPv4 Server Daemon
     Loaded: loaded (/lib/systemd/system/dhcpd.service; enabled; vendor preset: disabled)
     Active: active (running) since Wed 2025-12-03 23:00:06 MSK; 1min 50s ago
       Docs: man:dhcpd(8)
             man:dhcpd.conf(5)
    Process: 3728 ExecStartPre=/etc/chroot.d/dhcpd.all (code=exited, status=0/SUCCESS)
   Main PID: 3808 (dhcpd)
      Tasks: 1 (limit: 1149)
     Memory: 4.3M
        CPU: 36ms
     CGroup: /system.slice/dhcpd.service
             └─ 3808 /usr/sbin/dhcpd -4 -f --no-pid

Dec 03 23:00:06 hq-rtr.au-team.irpo dhcpd[3808]:    you want, please write a subnet declaration
Dec 03 23:00:06 hq-rtr.au-team.irpo dhcpd[3808]:    in your dhcpd.conf file for the network segment
Dec 03 23:00:06 hq-rtr.au-team.irpo dhcpd[3808]:    to which interface ens18 is attached. **
Dec 03 23:00:06 hq-rtr.au-team.irpo dhcpd[3808]: Sending on   Socket/fallback/fallback-net
Dec 03 23:00:06 hq-rtr.au-team.irpo dhcpd[3808]: Wrote 0 leases to leases file.
Dec 03 23:00:06 hq-rtr.au-team.irpo dhcpd[3808]: Server starting service.
Dec 03 23:00:39 hq-rtr.au-team.irpo dhcpd[3808]: DHCPDISCOVER from bc:24:11:c6:90:5d via ens19.20
Dec 03 23:00:40 hq-rtr.au-team.irpo dhcpd[3808]: DHCPOFFER on 192.168.20.66 to bc:24:11:c6:90:5d (hq-cli) via ens19.20
Dec 03 23:00:40 hq-rtr.au-team.irpo dhcpd[3808]: DHCPREQUEST for 192.168.20.66 (192.168.20.65) from bc:24:11:c6:90:5d (hq-cli) via ens19.20
Dec 03 23:00:40 hq-rtr.au-team.irpo dhcpd[3808]: DHCPACK on 192.168.20.66 to bc:24:11:c6:90:5d (hq-cli) via ens19.20
```
>⚠️ 💡 **Примечание**: После этого пинг до интернета должен заработать, можно проверрить до 1.1.1.1, пинг по доменным именам пока что не работает, так как локальный DNS на HQ-SRV будет настроен ниже.

>⚠️ **Важно**: В случае перезапуска сетевой службы network на HQ-RTR, необходимо в ручную перезапускать каждый раз службу dhcpd (systemctl restart dhcpd), так как она будет сыпать ошибками и выключаться.

## 📋 Задание 10: Настройка DNS для офисов HQ и BR.

**Задание 9**:
- Основной DNS-сервер реализован на HQ-SRV.
- Сервер должен обеспечивать разрешение имён в сетевые адреса устройств и обратно в соответствии с таблицей 2.
- В качестве DNS сервера пересылки используйте любой общедоступный DNS сервер.
```bash
Таблица 2
Устройство	Запись	Тип
HQ-RTR	hq-rtr.au-team.irpo	A,PTR
BR-RTR	br-rtr.au-team.irpo	A
HQ-SRV	hq-srv.au-team.irpo	A,PTR
HQ-CLI	hq-cli.au-team.irpo	A,PTR
BR-SRV	br-srv.au-team.irpo	A
ISP (интерфейс смотрящий в сторону HQ-RTR)	moodle.au-team.irpo	A
ISP  (интерфейс смотрящий в сторону BR-RTR)	wiki.au-team.irpo	A
```

### HQ-SRV
```bash
apt-get update && apt-get install bind nano -y
```
```bash
nano /etc/bind/options.conf
listen-on { any; };
forward only;
forwarders {77.88.8.8; };
allow-query { any; };
```
```bash
nano /etc/bind/local.conf
// Add other zones here
// Зона прямого просмотра (A-записи)
zone "au-team.irpo" {
    type master;
    file "/etc/bind/db.au-team.irpo";
};

// Зона обратного просмотра для сети 192.168.10.0
zone "10.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.10";
};

// Зона обратного просмотра для сети 192.168.20.64
zone "64.20.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.20.64";
};

# Обязательно 2 пробела через Enter вниз при редактировании через nano, и 1 пробел вниз через Enter при редактирование через vim, иначе не будет работать.
```
```bash
nano /etc/bind/db.au-team.irpo
$TTL 86400
@   IN  SOA hq-srv.au-team.irpo. root.au-team.irpo. (
        2025120401 ; serial
        3600       ; refresh
        1800       ; retry
        604800     ; expire
        86400 )    ; minimum

    IN  NS  hq-srv.au-team.irpo.

hq-rtr   IN  A   192.168.10.1
br-rtr   IN  A   192.168.30.1
hq-srv   IN  A   192.168.10.2
hq-cli   IN  A   192.168.20.66
br-srv   IN  A   192.168.30.2

moodle   IN   A   172.16.40.1
wiki     IN   A   172.16.50.1

# Обязательно 2 пробела через Enter вниз при редактировании через nano, и 1 пробел вниз через Enter при редактирование через vim, иначе не будет работать.
```
```bash
nano /etc/bind/db.192.168.10
$TTL 86400
@   IN  SOA hq-srv.au-team.irpo. root.au-team.irpo. (
        2025120401
        3600
        1800
        604800
        86400 )

    IN  NS  hq-srv.au-team.irpo.

1   IN  PTR  hq-rtr.au-team.irpo.
2   IN  PTR  hq-srv.au-team.irpo.

# Обязательно 2 пробела через Enter вниз при редактировании через nano, и 1 пробел вниз через Enter при редактирование через vim, иначе не будет работать.
```
```bash
nano /etc/bind/db.192.168.20.64
$TTL 86400
@   IN  SOA hq-srv.au-team.irpo. root.au-team.irpo. (
        2025120401
        3600
        1800
        604800
        86400 )

    IN  NS  hq-srv.au-team.irpo.

1  IN  PTR  hq-cli.au-team.irpo.

# Обязательно 2 пробела через Enter вниз при редактировании через nano, и 1 пробел вниз через Enter при редактирование через vim, иначе не будет работать.
```

```bash
rm -rf /etc/net/ifaces/ens18/resolv.conf 
systemctl restart network
```
```bash
nano /etc/resolvconf.conf
name_servers=127.0.0.1
```
```bash
resolvconf -u
systemctl restart network
```
**Выполним проверку**:
```bash
cat /etc/resolv.conf | grep nameserver
```
**Если все настроено верно получаем такой ответ**:
```bash
nameserver 127.0.0.1
```
**Запускаем службу DNS**: 
```bash
systemctl enable --now bind
systemctl restart bind
```
```bash
systemctl status bind
```
```bash
● bind.service - Berkeley Internet Name Domain (DNS)
     Loaded: loaded (/lib/systemd/system/bind.service; enabled; vendor preset: disabled)
     Active: active (running) since Thu 2025-12-04 07:49:41 +07; 2s ago
    Process: 3082 ExecStartPre=/etc/init.d/bind rndc_keygen (code=exited, status=0/SUCCESS)
    Process: 3086 ExecStartPre=/usr/sbin/named-checkconf $CHROOT -z /etc/named.conf (code=exited, status=0/SUCCESS)
    Process: 3087 ExecStart=/usr/sbin/named -u named $CHROOT $RETAIN_CAPS $EXTRAOPTIONS (code=exited, status=0/SUCCESS)
      Tasks: 5 (limit: 1149)
     Memory: 10.2M
        CPU: 15ms
     CGroup: /system.slice/bind.service
             └─ 3088 /usr/sbin/named -u named

Dec 04 07:49:41 hq-srv.au-team.irpo named[3088]: zone 10.168.192.in-addr.arpa/IN: loaded serial 2025120401
Dec 04 07:49:41 hq-srv.au-team.irpo named[3088]: zone 64.20.168.192.in-addr.arpa/IN: loaded serial 2025120401
Dec 04 07:49:41 hq-srv.au-team.irpo named[3088]: zone au-team.irpo/IN: loaded serial 2025120401
Dec 04 07:49:41 hq-srv.au-team.irpo named[3088]: zone localdomain/IN: loaded serial 2025110500
Dec 04 07:49:41 hq-srv.au-team.irpo named[3088]: zone localhost/IN: loaded serial 2025110500
Dec 04 07:49:41 hq-srv.au-team.irpo named[3088]: all zones loaded
Dec 04 07:49:41 hq-srv.au-team.irpo systemd[1]: Started Berkeley Internet Name Domain (DNS).
Dec 04 07:49:41 hq-srv.au-team.irpo named[3088]: running
Dec 04 07:49:42 hq-srv.au-team.irpo named[3088]: managed-keys-zone: Key 20326 for zone . is now trusted (acceptance timer complete)
Dec 04 07:49:42 hq-srv.au-team.irpo named[3088]: managed-keys-zone: Key 38696 for zone . is now trusted (acceptance timer complete)
```
>⚠️ **Важно**: Проверяем с помощью пинга соседей по их доменным именам, пробуем пинговать br-srv.au-team.irpo, hq-cli.au-team.irpo, moodle.au-team.irpo, wiki.au-team.irpo и так далее. Проверяем выход в Интернет. Далее небходимо настроить этот локальный DNS сервер для всех машин, так как на hq-cli настроен DHCP где уже прописан этот сервер, то это будет нужно сделать только на HQ-RTR,BR-RTR,BR-SRV.

### HQ-RTR
```bash
vim /etc/net/ifaces/ens18/resolv.conf
nameserver 192.168.10.2 # Старую запись удаляем, оставляем только новую.
```
```bash
systemctl restart network
```
### BR-RTR
```bash
vim /etc/net/ifaces/ens18/resolv.conf
nameserver 192.168.10.2 # Старую запись удаляем, оставляем только новую.
```
```bash
systemctl restart network
```
### BR-SRV
```bash
vim /etc/net/ifaces/ens18/resolv.conf
nameserver 192.168.10.2 # Старую запись удаляем, оставляем только новую.
```
```bash
systemctl restart network
```
> Проверямем пинг до Интернета, локальных доменных имен, все должно работать, со всех машин на все машины.

**Дополнительная информация**: После этих манипуляций, - Модуль 1: полностью выполнен, необходимо заполнить отчет как указано [здесь.](./report_2025.docx)
