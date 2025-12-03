# 🛠️ Руководство по выполнению демонстрационного экзамена

## Общие требования
- **Время выполнения:** 2 часа 30 минут (1 модуль = 1 час, 2 модуль = 1.5 часа)
- **Оборудование:** 10 серверов (Alt JeOS, Alt Server), 2 клиентские машины (Alt Workstation), в моем случае все выполнялось на развернутом Proxmox внутри VMware Workstation.

---

# Модуль 1: Сетевое администрирование (1 час)

## 📋 Задание 1: Базовая настройка устройств

- Настройте имена устройств согласно топологии. Используйте полное доменное имя.
- На всех устройствах необходимо сконфигурировать IPv4
- IP-адрес должен быть из приватного диапазона, в случае, если сеть локальная, согласно RFC1918
- Локальная сеть в сторону HQ-SRV(VLAN10) должна вмещать не более 32 адресов
- Локальная сеть в сторону HQ-CLI(VLAN20) должна вмещать не менее 16 адресов
- Локальная сеть в сторону BR-SRV должна вмещать не более 16 адресов
- Локальная сеть для управления(VLAN99) должна вмещать не более 8 адресов
- Сведения об адресах занесите в таблицу

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
vim /etc/net/ifaces/ens18/resolvconf
77.88.8.8
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
vim /etc/net/ifaces/ens18/resolvconf
77.88.8.8
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
192.168.30.1
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
vim /etc/net/ifaces/ens18/resolvconf
77.88.8.8
```
```bash
systemctl restart network
ip -c -br a
```
Должен быть такой вывод у команды:
```bash
lo               UNKNOWN        127.0.0.1/8 ::1/128 
ens18            UP             172.16.50.2/28 fe80::be24:11ff:feab:8a59/64 
ens19            UP             192.168.30.1/32 fe80::be24:11ff:fe58:e15d/64 
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
default via 172.16.30.1
vim /etc/net/ifaces/ens18/resolvconf
77.88.8.8
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
```bash
# ISP
vim /etc/net/sysctl.conf
net.ipv4.ip_forward = 1

systemctl restart network
```
```shell
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
