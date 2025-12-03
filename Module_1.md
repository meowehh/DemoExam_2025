# 🛠️ Руководство по выполнению демонстрационного экзамена

## Общие требования
- **Время выполнения:** 2 часа 30 минут (1 модуль = 1 час, 2 модуль = 1.5 часа)
- **Оборудование:** 10 серверов (Alt JeOS, Alt Server), 2 клиентские машины (Alt Workstation), в моем случае все выполнялось на развернутом Proxmox внутри VMware Workstation.

---

# Модуль 1: Сетевое администрирование (1 час)

## 📋 Задание 1: Базовая настройка устройств + Задание 4: Настройте на интерфейсе HQ-RTR в сторону офиса HQ виртуальный коммутатор.

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
- Основные сведения о настройке коммутатора и выбора реализации разделения на VLAN занесите в отчёт

## Выполнение:
### 1.1 Настройка hostname
```bash
hostnamectl set-hostname <полное_доменное_имя>; exec bash

# ISP
hostnamectl set-hostname isp.au-team.irpo; exec bash
# HQ-RTR
hostnamectl set-hostname hq-rtr.au-team.irpo; exec bash
# HQ-SRV
hostnamectl set-hostname hq-srv.au-team.irpo; exec bash
# HQ-CLI
hostnamectl set-hostname hq-cli.au-team.irpo; exec bash
# BR-RTR
hostnamectl set-hostname br-rtr.au-team.irpo; exec bash
# BR-SRV
hostnamectl set-hostname br-srv.au-team.irpo; exec bash
```

> ⚠️ 💡 **Примечание**: Хоть в задании не указано дать название ISP, но для корректного функционирования DNS и других сервисов требуется выдать полное доменное имя всем устройствам.

>⚠️ **Важно**: Команда hostnamectl set-hostname применяет изменения немедленно без перезагрузки. Флаг ; exec bash обновляет текущую сессию shell для отображения нового hostname в приглашении командной строки.

### 1.2 Конфигурация IPv4 адресов.

```bash
# ISP
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


```bash
# HQ-RTR
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
vim /etc/net/ifaces/ens19.99/options
BOOTPROTO=static
TYPE=vlan
VID=99
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
```bash


# BR-RTR
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

> ⚠️ 💡 **Примечание!**: HQ-CLI будет настроен позднее так как там будет использоваться DHCP настройка, на данном этапе теперь требуется настроить проброс портов чтобы пинг начал ходить между устройствами и появился доступ в интернет со всех машин, так же все отчеты будут приведны в отдельном файле, сейчас заполнять ничего не требуется, несмотря на задание.

## 📋 Задание 2: Настройка ISP

- Настройте адресацию на интерфейсах.
- Интерфейс, подключенный к магистральному провайдеру, получает адрес по DHCP
- Настройте маршруты по умолчанию там, где это необходимо
- Интерфейс, к которому подключен HQ-RTR, подключен к сети 172.16.40.0/28
- Интерфейс, к которому подключен BR-RTR, подключен к сети 172.16.50.0/28
- На ISP настройте динамическую сетевую трансляцию в сторону HQ-RTR и BR-RTR для доступа к сети Интернет

```bash
# ISP
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

```bash
# HQ-RTR
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

```bash
# BR-RTR
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

```bash
# HQ-SRV и BR-SRV
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

```bash
# HQ-RTR и BR-RTR
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

```bash
# BR-SRV
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

```bash
# HQ-SRV
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

- Сведения о туннеле занесите в отчёт. (Отчет будет приложен отдельным файлом.)
- На выбор технологии GRE или IP in IP.

```bash
# HQ-RTR
mkdir /etc/net/ifaces/gre1
vim /etc/net/ifaces/gre1/options
TYPE=iptun
TUNTYPE=gre
TUNLOCAL=172.16.40.2
TUNREMOTE=172.16.50.2
TUNOPTIONS='ttl 64'
vim /etc/net/ifaces/gre1/ipv4address
10.10.0.1/30
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

```bash
# BR-RTR
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
