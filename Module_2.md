# Модуль 2: Системное администрирование (1.5 часа)

## 📋 Задание 1: Настройте доменный контроллер Samba на машине BR-SRV.

**Задание 1**:
- Создайте 5 пользователей для офиса HQ: имена пользователей формата user№.hq. Создайте группу hq, введите в эту группу созданных пользователей
- Введите в домен машину HQ-CLI
- Пользователи группы hq имеют право аутентифицироваться на клиентском ПК
- Пользователи группы hq должны иметь возможность повышать привилегии для выполнения ограниченного набора команд: cat, grep, id. Запускать другие команды с повышенными привилегиями пользователи группы не имеют права
- Выполните импорт пользователей из файла users.csv. Файл будет располагаться на виртуальной машине BR-SRV в папке /opt

> **Важно**: Для импорта пользователей из users.csv нужно использовать скрипт, в интернете я не нашел скриптов которые бы переносили все дополнительные флаги для пользователей которые указаны в /opt/users.csv, поэтому я написал свой собственный скрипт, его можно найти [тут](https://github.com/meowehh/samba-import).

### BR-SRV
```bash
apt-get update && apt-get install -y task-samba-dc alterator-fbi alterator-net-domain admx-* admc gpui
```
```bash
vim /etc/sysconfig/network
HOSTNAME=br-srv.au-team.irpo
```
```bash
reboot
```
```bash
systemctl enable --now ahttpd alteratord
rm -rf /etc/samba/smb.conf /var/{lib.cache}/samba
mkdir -p /var/lib/samba/sysvol
```
```bash
rm -rf /etc/net/ifaces/ens18/resolv.conf
systemctl restart network
vim /etc/resolvconf.conf
name_servers=127.0.0.1
name_servers=192.168.1.10
resolvconf -u
systemctl restart network
```
```bash
samba-tool domain provision --realm=au-team.irpo --domain=au-team --adminpass='P@ssw0rd' --dns-backend=BIND9_DLZ --server-role=dc --use-rfc2307 
```

### HQ-CLI
```bash
apt-get update && apt-get install -y admx-* admc gpui sudo gpupdate
```
**Заходим через вкладку Console внутри Proxmox VE, чтобы получить доступ к графической части.**
```
login: user
password: resu
```
**Открываем Firefox:** 
- Переходим по адресу 192.168.3.10:8080
- Данные для авторизации:
```bash
login: root
password: toor
```
- Configuration > Expert mode > Apply
- Web Interface
- Меняем порт 8080 на 8081 > Apply > Restart http server
- Переходим по адресу 192.168.3.10:8081
- Вкладка Domain
- Выбираем Active Directory Domain Controller
- DNS Forwarders - 192.168.1.10
- Domain - au-team.irpo
- Password - P@ssw0rd
- Apply
- Запускаем и дожидаемся состояния - OK.

### BR-SRV:
**Перепускаем систему и проверяем**:
```bash
ping ya.ru
ping br-srv.au-team.irpo
ping hq-rtr.au-team.irpo
```
> Если все работает - то ОК!

### HQ-CLI:
**Перепроверяем 192.168.3.10:8081**
- Заходим в Domain
> Если сервер показывает статус OK, то идем дальше.

**От рута выполняем:**
```bash
nmcli con modify CLI-NET \
	ipv4.method auto \
	ipv4.ignore-auto-dns yes \
	ipv4.dns 192.168.3.10
```
```bash
nmcli con down CLI-NET
nmcli con up CLI-NET
```
**Открываем снова GUI и запускаем терминал, там прописываем**:
```bash
acc
```
- Пароль toor
- Выбрать Auth в Networking
- Прописать в Domain - au-team.irpo
- Прописать в Workgroup - au-team
- Apply
- Login: Administrator, Password: P@ssw0rd
> Если вход в домен произошел, то - ОК!

### BR-SRV
```bash
samba-tool group add hq
for i in $(seq 1 5); do samba-tool user add user$i.hq 'P@ssw0rd'; done
for i in $(seq 1 5); do samba-tool group addmembers hq user$i.hq; done
```
Проверим наличие группы hq в Samba, и созданных пользователей:
```bash
samba-tool group list
samba-tool group listmembers hq
```
```bash
admx-msi-setup
```
### HQ-CLI

**Перезапускаем и входим как Administrator**:
- Пароль: P@ssw0rd

**Открываем терминал**:
```bash
su -
toor
```
```bash
admx-msi-setup
```
```bash
roleadd hq wheel
```
```bash
rolelst
```
> **Проверяем наличие hq:wheel**
**Добавляем в sudoers данные строки:**
```bash
mcedit /etc/sudoers
, %AU-TEAM\\hq
Cmnd_Alias	SHELLCMD = /usr/bin/id, /bin/cat, /bin/grep
SHELLCMD
```
Для понимания где находятся эти строки, и куда их нужно добавить - пример того как это реализовано у меня:
```bash
## User alias specification
##
## Groups of users.  These may consist of user names, uids, Unix groups,
## or netgroups.
# User_Alias ADMINS = millert, dowdy, mikef
User_Alias WHEEL_USERS = %wheel, AU-TEAM\\hq # Первая строка
User_Alias XGRP_USERS = %xgrp
# User_Alias SUDO_USERS = %sudo

##
## Runas alias specification
##
Cmnd_Alias SHELLCMD = /usr/bin/id, /bin/cat, /bin/grep # Вторая строка
##
## User privilege specification
##
# root ALL=(ALL:ALL) ALL

## Uncomment to allow members of group wheel to execute any command
WHEEL_USERS ALL=(ALL:ALL) SHELLCMD # Третья строка
```
```
exit
```
**После того как вышли из под рута, выполняем kinit:**
```bash
kinit
P@ssw0rd
```
```bash
admc
```
**Настроим групповую политку:**
- Group Policy Objects
- au-team.irpo > правой кнопкой мыши
- Create a GPO and link to ths GPU
- Название: sudoers
- Ставим галочку в поле enforced
- Правой кнопкой мыши по sudoers > edit
- Machine
- Administative Templates
- Samba
- Unix Settings
- Sudo rights
- Enabled
- /usr/bin/id
- /bin/cat
- /bin/grep
- Применяем и выходим из admc.
```bash
gpupdate -f # Прописываем 2 раза подряд чтобы команда точно применилась, иногда не срабатывает с 1 раза.
```
**Заходим из под user5.hq:**
- Пароль - P@ssw0rd

```bash
sudo id
P@ssw0rd
sudo cat /root/.bashrc
sudo cat /root/.bashrc | grep root
```
> Если все команды выполняются, значит все выполнено верно.

### BR-SRV
```bash
apt-get update && apt-get install git -y
git clone https://github.com/meowehh/samba-import
cd samba-import/

cat before_script
sed -i -e 's/\r$//' import.sh

chmod +x import.sh 
./import.sh /opt/users.csv
```
> Ждем завершения выполнения скрпта, после этого заходим на HQ-CLI и пытаемся войти в одну из учеток, флаги учетных записей проверять не нужно.

### HQ-CLI
```
login: wesley.cain 
password: P@ssw0rd1
```
> За пример взята эта учетка из /opt/users.csv, можно взять любую другую, если войти получилось, значит все выполенено верно.


## 📋 Задание 2: Сконфигурируйте файловое хранилище.

**Задание 2**:
- При помощи трёх дополнительных дисков, размером 1Гб каждый, на HQ-SRV сконфигурируйте дисковый массив уровня 0
- Имя устройства – md0, конфигурация массива размещается в файле /etc/mdadm.conf
- Обеспечьте автоматическое монтирование в папку /raid0
- Создайте раздел, отформатируйте раздел, в качестве файловой системы используйте ext4
- Настройте сервер сетевой файловой системы(nfs), в качестве папки общего доступа выберите /raid0/nfs, доступ для чтения и записи для всей сети в сторону HQ-CLI 
- На HQ-CLI настройте автомонтирование в папку /mnt/nfs
- Основные параметры сервера отметьте в отчёте.

> Готовый отчет можно взять - [тут.](./report_2025.docx)

### HQ-SRV
```bash
lsblk
mdadm -C /dev/md0 -l 0 -n 3 /dev/sd{b,c,d}
lsblk
mkfs.ext4 /dev/md0
echo DEVICE partitions >> /etc/mdadm.conf
mdadm --detail --scan >> /etc/mdadm.conf
mkdir /raid0
```
```bash
mcedit /etc/fstab
/dev/md0	/raid0 ext4 defaults	0	0
```
```bash
mount -a
df -h
lsblk
```
```bash
apt-get update && apt-get install -y nfs-{server,utils}
mkdir /raid0/nfs
chmod 766 /raid0/nfs
```
Комментируем первую строку в файле /etc/exports, в самом низу прописываем, то что идет ниже.
```
mcedit /etc/exports
/raid0/nfs 192.168.2.0/28(rw,no_subtree_check,no_root_squash)
```
**Применяем изменения**:
```bash
exportfs -arv
systemctl enable --now nfs-server.service
systemctl restart nfs-server.service
```
### HQ-CLI
```bash
apt-get update && apt-get install -y nfs-{server,utils}
mkdir /mnt/nfs
chmod 777 /mnt/nfs
```
```bash
mcedit /etc/fstab
192.168.1.10:/raid0/nfs	/mnt/nfs	nfs	defaults	0	0
systemctl enable --now nfs-server.service
systemctl restart nfs-server.service
```
**Монтируем файловую систему и делаем финальную проверку RAID 0**:
```bash
mount -a
df -h
```
**Вывод команды:**
```bash
Filesystem               Size  Used Avail Use% Mounted on
udevfs                   5.0M   64K  5.0M   2% /dev
runfs                    3.9G  1.1M  3.9G   1% /run
/dev/sda                  12G  9.1G  2.1G  82% /
tmpfs                    3.9G     0  3.9G   0% /dev/shm
tmpfs                    3.9G  4.0K  3.9G   1% /tmp
tmpfs                    796M   84K  796M   1% /run/user/60601109
tmpfs                    796M   56K  796M   1% /run/user/0
192.168.1.10:/raid0/nfs  2.9G     0  2.8G   0% /mnt/nfs
```
> ⚠️ 💡 **Примечание**: Пробуем перезагрузку обоих устройств и проверяем вывод через df -h снова, расшаренная файловая система с RAID 0 - должна автоматически быть доступной.

## 📋 Задание 9: Удобным способом установите приложение Яндекс Браузер для организаций на HQ-CLI.

**Задание 2**:
- Установку браузера отметьте в отчёте.

> Готовый отчет можно взять - [тут.](./report_2025.docx)

**Самый лучший способ: установить напрямую через пакетный менеджер.**
```bash
apt-get update && apt-get install yandex-browser -y
```
> ⚠️ 💡 **Примечание**: После установки он не появится на рабочем столе, а только в панели быстрого доступа, после ребута появится и на рабочем столе, но запускать его не нужно до задания с разверткой Docker, так как там он пригодится чтобы обойти блокировку hub.docker через встроенный прокси в браузере, лишний раз его лучше не запускать так как на HQ-CLI по умолчанию всего 1.5ГБ оперативной памяти, из которых доступно пользователю 1 ГБ, это может положить HQ-CLI.

## 📋 Задание 3: Настройте службу сетевого времени на базе сервиса chrony

**Задание 3**:
- В качестве сервера выступает HQ-RTR
- На HQ-RTR настройте сервер chrony, выберите стратум 5
- В качестве клиентов настройте HQ-SRV, HQ-CLI, BR-RTR, BR-SRV

### HQ-RTR

```bash
apt-get update && apt-get install -y chrony
```
```bash
vim /etc/chrony.conf
pool ntp0.ntp-servers.net iburst prefer
pool 127.0.0.1 iburst 
hwtimestamp *
local stratum 5
allow 0/0
```
**Запустим службу времени:**
```bash
systemctl restart chronyd
systemctl enable --now chronyd
timedatectl set-timezone Asia/Novosibirsk
```
В качестве клиентов настроим: HQ-SRV, HQ-CLI, BR-RTR, BR-SRV, выполнить настройку нужно идентично нижней на всех 4-ех клиентах.
```bash
apt-get update && apt-get install -y chrony
vim /etc/chrony.conf
pool 192.168.1.1 iburst prefer
systemctl restart chronyd
systemctl enable --now chronyd
timedatectl set-timezone Asia/Novosibirsk
```
> ⚠️ 💡 **Примечание**: На HQ-CLI уже будет сервер времени, нужно лишь добавить новый pool.

**Проведем проверку настроек, в ответе должен быть новый сервер времени с stratum 5, на HQ-CLI будет 2 сервера времени, с stratum 0 (неактивный) и с stratum 5 (активный).**
```bash
chronyc sources
```
### HQ-RTR
```bash
chronyc clients
```
```bash
Hostname                      NTP   Drop Int IntL Last     Cmd   Drop Int  Last
===============================================================================
localhost.localdomain          13      0   7   -    35       0      0   -     -
hq-srv.au-team.irpo             8      0   6   -    17       0      0   -     -
hq-cli.au-team.irpo             6      0   5   -    39       0      0   -     -
192.168.5.2                     5      0   4   -    10       0      0   -     -
192.168.3.10                    4      0   1   -    16       0      0   -     -
```
> ⚠️ 💡 **Примечание**: В ответе должны быть все клиенты, у них может быть не тот IP что на самом деле, это нормально, главное чтобы количество совпадало с настроенными, а так же на самих клиентах через команду chronyc sources, - был указан stratum 5.

## 📋 Задание 4:  Сконфигурируйте ansible на сервере BR-SRV

**Задание 4**:
- Сформируйте файл инвентаря, в инвентарь должны входить HQ-SRV, HQ-CLI, HQ-RTR и BR-RTR
- Рабочий каталог ansible должен располагаться в /etc/ansible
- Все указанные машины должны без предупреждений и ошибок отвечать pong на команду ping в ansible посланную с BR-SRV

### BR-SRV
```bash
apt-get update && apt-get install openssh-server ansible sshpass nano -y
```
```bash
nano /etc/ansible/hosts
[Alt]
192.168.2.1 ansible_ssh_user=net_admin ansible_ssh_pass=P@ssw0rd
192.168.1.10 ansible_ssh_user=sshuser ansible_ssh_pass=P@ssw0rd
192.168.2.10 ansible_ssh_user=sysadmin ansible_ssh_pass=P@ssw0rd
192.168.3.1 ansible_ssh_user=net_admin ansible_ssh_pass=P@ssw0rd

[Alt:vars]
ansible_port=2024
```
```bash
nano /etc/ansible/ansible.cfg
[defaults]

interpreter_python = /usr/bin/python3

# some basic default values...
# uncomment this to disable SSH key host checking
host_key_checking = False
```

### HQ-SRV
```bash
apt-get update && apt-get install openssh-server -y
```
```bash
vim /etc/openssh/sshd_config
Port 2024
MaxAuthTries 2
AllowUsers sshuser
```
```bash
systemctl enable --now sshd
systemctl restart sshd
```

### HQ-RTR и BR-RTR
```bash
apt-get update && apt-get install openssh-server -y
```
```bash
vim /etc/openssh/sshd_config
Port 2024
MaxAuthTries 2
AllowUsers net_admin
```
```bash
systemctl enable --now sshd
systemctl restart sshd
```

### HQ-CLI 
```bash
apt-get update && apt-get install openssh-server -y
```
```bash
useradd sysadmin
passwd sysadmin
P@ssw0rd
usermod -a -G remote sysadmin
```
```bash
vim /etc/openssh/sshd_config
Port 2024
MaxAuthTries 2
AllowGroups wheel remote
```
```bash
systemctl enable --now sshd
systemctl restart sshd
```

### BR-SRV
```bash
ansible -m ping all
```
**Вывод команды**:
```bash
192.168.3.1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
192.168.2.1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
192.168.1.10 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
192.168.2.10 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```
> ⚠️ 💡 **Важно**: Проверяем чтобы вывод команды совпадал, если все совпдает, значит задание выполнено верно.


## 📋 Задание 5:  Развертывание приложений в Docker на сервере BR-SRV.

**Задание 5**: 

- Создайте в домашней директории пользователя файл wiki.yml для приложения MediaWiki.
- Средствами docker compose должен создаваться стек контейнеров с приложением MediaWiki и базой данных.
- Используйте два сервиса
- Основной контейнер MediaWiki должен называться wiki и использовать образ mediawiki
- Файл LocalSettings.php с корректными настройками должен находиться в домашней папке пользователя и автоматически монтироваться в образ.
- Контейнер с базой данных должен называться mariadb и использовать образ mariadb.
- Разверните 
- Он должен создавать базу с названием mediawiki, доступную по стандартному порту, пользователя wiki с паролем WikiP@ssw0rd должен иметь права доступа к этой базе данных 
- MediaWiki должна быть доступна извне через порт 8086.

### BR-SRV
```bash
apt-get update && apt-get install openssh-server docker-ce docker-compose -y
```
**Настроим и запустим SSH, - чтобы в дальнейшем перебросить файл конфига mediawiki с HQ-CLI на BR-SRV**:
```bash
vim /etc/openssh/sshd_config
Port 2024
MaxAuthTries 2
AllowUsers sshuser
```
```bash
systemctl enable --now sshd
systemctl restart sshd
```
### HQ-CLI

- Запускаем Яндекс браузер, который установили [ранее](https://github.com/meowehh/DemoExam_2025/blob/main/Module_2.md#-%D0%B7%D0%B0%D0%B4%D0%B0%D0%BD%D0%B8%D0%B5-9-%D1%83%D0%B4%D0%BE%D0%B1%D0%BD%D1%8B%D0%BC-%D1%81%D0%BF%D0%BE%D1%81%D0%BE%D0%B1%D0%BE%D0%BC-%D1%83%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%B8%D1%82%D0%B5-%D0%BF%D1%80%D0%B8%D0%BB%D0%BE%D0%B6%D0%B5%D0%BD%D0%B8%D0%B5-%D1%8F%D0%BD%D0%B4%D0%B5%D0%BA%D1%81-%D0%B1%D1%80%D0%B0%D1%83%D0%B7%D0%B5%D1%80-%D0%B4%D0%BB%D1%8F-%D0%BE%D1%80%D0%B3%D0%B0%D0%BD%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D0%B9-%D0%BD%D0%B0-hq-cli). Через Firefox не будет работать.
- Переходим на сайт hub.docker.com
- В поиске вбиваем mediawiki
- На странице с mediawiki листаем вниз и находим конфиг compose.yaml
- Копируем конфиг
- Запускаем терминал
```bash
su -
toor
ssh sshuser@192.168.3.10 -p 2024 # Заходим на BR-SRV
P@ssw0rd
su -
toor
mcedit wiki.yml
```
**Вставляем скопированный конфиг, теперь можно закрыть Яндекс браузер.**

Далее приводим файл wiki.yml к следующему виду:
```bash
# MediaWiki with MariaDB
#
# Access via "http://localhost:8080"
services:
  wiki:
    image: mediawiki
    restart: always
    ports:
      - 8086:80
    links:
      - mariadb
    volumes:
      - images:/var/www/html/images
      # After initial setup, download LocalSettings.php to the same directory as
      # this yaml and uncomment the following line and use compose to restart
      # the mediawiki service
      # - ./LocalSettings.php:/var/www/html/LocalSettings.php
  mariadb: # <- This key defines the name of the database during setup
    image: mariadb
    restart: always
    environment:
      # @see https://phabricator.wikimedia.org/source/mediawiki/browse/master/includes
      MYSQL_DATABASE: mediawiki
      MYSQL_USER: wiki
      MYSQL_PASSWORD: WikiP@ssw0rd
      MYSQL_RANDOM_ROOT_PASSWORD: 'yes'
    volumes:
      - db:/var/lib/mysql

volumes:
  images:
  db:
```
**Запускаем docker и развертываем mediawiki**:
```bash
systemctl enable --now docker.service docker.socket
docker compose -f wiki.yml up -d
```
> Дожидаемся завершения.

- Открываем Firefox, переходим по 192.168.3.10:8086
- set up the wiki
- 2 раза продолжить без изменений.
- Database type - mariadb
- Database host - mariadb
- Database name - mediawiki
- Databse username - wiki
- Databse password - WikiP@ssw0rd
- Use the same account as for installation
- URL host name - wiki
- Project namespace - as the wiki name: wiki
- Administation account:
- User: wiki
- Password: WikiP@ssw0rd
- Do NOT share data about this installation
- I'm bored already, just install the wiki
- Жмем продолжить до того момента как не скачается LocalSettings.php

**Запускаем ещё один терминал:**
```bash
cd Downloads/
scp -P 2024 LocalSettings.php sshuser@192.168.3.10:/home/sshuser
yes
P@ssw0rd
```
**Открываем прошлый терминал где уже установлено соединение SSH с BR-SRV:**
```bash
cd /home/sshuser
cp LocalSettings.php ~
cd ~
```
```bash
docker compose -f wiki.yml down
```
**Убираем комментирование этой строки:**
```bash
mcedit wiki.yml
- ./LocalSettings.php:/var/www/html/LocalSettings.php
```
**Ищем указанную строку и меняем ее на данную:**
```bash
mcedit LocalSettings.php
$wgServer = "http://192.168.3.10:8086";
```
```bash
docker compose -f wiki.yml up -d
```
> Сайт должен открываться по 192.168.3.10:8086, если все работает значит настроено верно, пробуем авторизоваться как пользователь: wiki, с паролем WikiP@ssw0rd. Если получилось, задание выполнено верно.

## 📋 Задание 6: На маршрутизаторах сконфигурируйте статическую трансляцию портов.

**Задание 6**: 
- Пробросьте порт 80 в порт 8086 на BR-SRV на маршрутизаторе BR-RTR, для обеспечения работы сервиса wiki
- Пробросьте порт 80 в порт 80 на HQ-SRV на маршрутизаторе HQ-RTR, для обеспечения работы сервиса moodle
- Пробросьте порт 3015 в порт 3015 на HQ-SRV на маршрутизаторе HQ-RTR
- Пробросьте порт 3015 в порт 3015 на BR-SRV на маршрутизаторе BR-RTR

### BR-RTR
```bash
apt-get update && apt-get install iptables -y
```
```bash
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.3.10:8086
iptables -A FORWARD -p tcp -d 192.168.3.10 --dport 8086 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT

iptables -t nat -A PREROUTING -p tcp --dport 3015 -j DNAT --to-destination 192.168.3.10:3015
iptables -A FORWARD -p tcp -d 192.168.3.10 --dport 3015 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
```
```bash
iptables-save > /etc/sysconfig/iptables
systemctl restart iptables
systemctl enable --now iptables
```
**Выполним проверку**:
```bash
iptables -t nat -L -n -v
```
**Получаем вывод**:
```bash
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:192.168.3.10:8086
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:3015 to:192.168.3.10:3015

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination 
```
**Повторим проверку**:
```bash
iptables -L -n -v
```
**Сверяем вывод с моим**:
```bash
Chain INPUT (policy ACCEPT 4 packets, 320 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            192.168.3.10         tcp dpt:8086 state NEW,RELATED,ESTABLISHED
    0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            192.168.3.10         tcp dpt:3015 state NEW,RELATED,ESTABLISHED

Chain OUTPUT (policy ACCEPT 6 packets, 480 bytes)
 pkts bytes target     prot opt in     out     source               destination
```

### HQ-RTR
```bash
apt-get update && apt-get install iptables -y
```
```bash
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.1.10:80
iptables -A FORWARD -p tcp -d 192.168.1.10 --dport 80 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT

iptables -t nat -A PREROUTING -p tcp --dport 3015 -j DNAT --to-destination 192.168.1.10:3015
iptables -A FORWARD -p tcp -d 192.168.1.10 --dport 3015 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
```
```bash
iptables-save > /etc/sysconfig/iptables
systemctl restart iptables
systemctl enable --now iptables
```
**Выполним проверку**:
```bash
iptables -t nat -L -n -v
```
**Получаем вывод**:
```bash
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:192.168.1.10:80
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:3015 to:192.168.1.10:3015

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination 
```
**Повторим проверку**:
```bash
iptables -L -n -v
```
**Сверяем вывод с моим**:
```bash
Chain INPUT (policy ACCEPT 8 packets, 624 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain FORWARD (policy ACCEPT 6 packets, 456 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            192.168.1.10         tcp dpt:80 state NEW,RELATED,ESTABLISHED
    0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            192.168.1.10         tcp dpt:3015 state NEW,RELATED,ESTABLISHED

Chain OUTPUT (policy ACCEPT 6 packets, 464 bytes)
 pkts bytes target     prot opt in     out     source               destination 
```
> Если вывод совпадает, задание выполнено верно.

## 📋 Задание 7: Запустите сервис moodle на сервере HQ-SRV:

- Используйте веб-сервер apache
- В качестве системы управления базами данных используйте mariadb
- Создайте базу данных moodledb
- Создайте пользователя moodle с паролем P@ssw0rd и предоставьте ему права доступа к этой базе данных
- У пользователя admin в системе обучения задайте пароль P@ssw0rd
- На главной странице должен отражаться номер рабочего места в виде арабской цифры, других подписей делать не надо
- Основные параметры отметьте в [отчёте](./report_2025.docx)

### HQ-SRV
```bash
apt-get update && apt-get install deploy -y
```
```bash
mcedit /usr/share/deploy/moodle/tasks/main.yml
```
**Жмем F4, входим в режим Replace.**
- Заменяем все moodle1 на moodledb, - 4 замены.
- И так же заменяем все moodleuser на moodle, - 2 замены.
  
**Ищем строку:**
```bash
shell: pwgen 16 1
```
**Заменяем на:**
```bash
shell: echo "P@ssw0rd"
```
> Сохраняем изменения, запускаем развертку moodle.
```bash
deploy moodle
```
> Дожидаемся полной развертки, требуется хорошая и стабильная скорость интернета для завершения установки, занимает много времени.

**Корректный вывод, в процессе установки выглядит так:**
```bash
Deploying moodle...
Executing playbook moodle.yml

- deploy Moodle on hosts: local -
install Apache packages...
  localhost done
  localhost done
  localhost done
  localhost done
check certificate file...
  localhost ok
generate certificate file...
  localhost done
enable Apache2 module filter...
  localhost done
enable Apache2 module ssl...
  localhost done
enable Apache2 module rewrite...
  localhost done
enable Apache2 module headers...
  localhost done
enable Apache2 module env...
  localhost done
enable Apache2 module dir...
  localhost ok
enable Apache2 module mime...
  localhost ok
enable Apache2 module mod_php8.2...
  localhost ok
disable Apache2 module mod_php7...
  localhost ok
enable HTTPS (default_https)...
  localhost done
enable HTTPS (https)...
  localhost done
configure port 80...
  localhost done | msg: line added
configure port 443...
  localhost done | msg: line added
change example server name...
  localhost done | msg: 1 replacements made
change _default_ placeholder for https...
  localhost done | msg: 1 replacements made
set port 80 for default server...
  localhost done | msg: line replaced
add RewriteEngine On...
  localhost done | msg: line added
add RewriteCond...
  localhost done | msg: line added
add RewriteRule...
  localhost done | msg: line added
detect PHP settings...
  localhost ok
configure PHP memory_limit setting...
  localhost done
configure PHP upload_max_filesize setting...
  localhost done
configure PHP max_input_vars setting...
  localhost done
reload Apache2 configuration...
[WARNING]: Consider using the service module rather than running 'service'.  If you need to use command because service is insufficient you can add 'warn: false' to this command task or set
'command_warnings=False' in ansible.cfg to get rid of this message.
  localhost ok
start Apache service...
  localhost done
detect HTTP DocumentRoot...
  localhost ok | stdout: DocumentRoot for http: "/var/www/html"
detect HTTPS DocumentRoot...
  localhost ok | stdout: DocumentRoot for https: "/var/www/html"
install MariaDB server packages...
  localhost done | item: mariadb-server | msg: mariadb-server present(s)
  localhost done
start MariaDB service...
  localhost done
install Moodle packages...
  localhost done | item: moodle | msg: moodle present(s)
  localhost done | item: moodle-apache2 | msg: moodle-apache2 present(s)
  localhost ok | msg: Nothing to install
  localhost done | item: moodle-local-mysql | msg: moodle-local-mysql present(s)
  localhost done | item: python3-module-pymysql | msg: python3-module-pymysql present(s)
  localhost done | item: pwgen | msg: pwgen present(s)
  localhost ok | msg: Nothing to install
  localhost done
check if database moodledb exists...
  localhost done
generate password for Moodle...
  localhost ok
create database user...
  localhost done
check for config file...
  localhost ok
generate configuration by install script from moodle...
  localhost done
reload Apache2 configuration...
  localhost ok
one try to open web page...
  localhost ok
Change password to Moodle for user admin...

- Play recap -
  localhost                  : ok=40   changed=27   unreachable=0    failed=0    rescued=0    ignored=0   
Deploy complete successful.
```
> **⚠️ 💡 Важно!** Теперь необходимо проверить действительно ли создались все нужные в дальнейшем config файлы, иногда возникает баг при котором они должны были создаться, но не создались.

**Проверка конфигов:**

```bash
vim /var/www/webapps/moodle/config.php 
```
> Открываем данный файл, если он есть и в нем есть содержимое, то значит все хорошо, если этого файла нет или он пустой, то запускаем команду deploy moodle ещё раз, после 2 раза config.php появляется всегда.

Приводим файл config.php к следующем виду:
```bash
vim /var/www/webapps/moodle/config.php
$CFG->wwwroot   = 'http://hq-srv.au-team.irpo/moodle';# Обратить внимание на эту строку, убираем капс и оставляем только http
```
```bash
mcedit /etc/httpd2/conf/sites-enabled/000-default.conf
# Комментируем 3 последних строки, которые начинаются с Rewrite
```
```bash
systemctl restart httpd2
```

### HQ-CLI

**⚠️ 💡 Внимание:** После настроек ниже не будет работать авторизация через домен, но позже я покажу как настроить ее обратно, пока что делаем так, вернем после того как настроим nginx.

```bash
nmcli connection modify CLI-NET \
> ipv4.ignore-auto-dns no
```

**Настройки сети:**
- Нажимаем правой кнопкой мыши в панеле быстрого доступа справа снизу на сеть.
- Параметры
- Выбираем CLI-NET
- DNS Settings
- Additional DNS servers # Удаляем значение, оставляем пустое поле.
- Сохраняем изменения.
- Нажимаем правой кнопкой мыши в панеле быстрого доступа справа снизу на сеть.
- Enable Networking # Выключить и включить


**Открываем Firefox:**
- Заходим на hq-srv.au-team.irpo/moodle
- Логин: Admin
- Пароль: P@ssw0rd
- Авторизуемся, указываем почту: moodle@au-team.irpo
- В начало
- Настройки
- Полное название сайта - 1 (Или любое другое, это место рабочего стола из задания)

> Задание выполнено.

## 📋 Задание 8: Настройте веб-сервер nginx как обратный прокси-сервер на ISP.

**Задание 8**:
- При обращении по доменному имени moodle.au-team.irpo у клиента должен открываться сервис moodle
- При обращении по доменному имени wiki.au-team.irpo клиента должен открываться сервис mediwiki

### ISP
```bash
apt-get update && apt-get install nginx nano -y
```
```bash
cp /etc/nginx/sites-available.d/default.conf /etc/nginx/sites-available.d/moodle.conf
```
```bash
nano /etc/nginx/sites-available.d/moodle.conf
upstream moodle.au-team.irpo {
        server 172.16.4.4;
}
server {
        listen  80;
        server_name _;

        location / {
                # root /var/www/html;

                # autoindex off;
                # autoindex_exact_size on;
                # autoindex_localtime off;

                # expires off;

                # cooperate with mod_realip in apache-1.3 or mod_rpaf in apache-2.x
                #       proxy_redirect off;
                #       proxy_set_header Host $host;
                #       proxy_set_header X-Real-IP $remote_addr;
                #       proxy_set_header X-Forwarded-For $remote_addr;
                        proxy_pass http://moodle.au-team.irpo/moodle;
                #
                # NB: it's better for URI canonicalization that apache sits on :80
                # (even if that's only 127.0.0.1:80)
                #
                # see also set_real_ip_from, real_ip_header if this nginx
                # would need to cooperate with another one acting as a frontend
        }

#               charset         on;
#               source_charset  koi8-r;

#               access_log  /var/log/nginx/access.log;
}
```
```bash
cp /etc/nginx/sites-available.d/moodle.conf /etc/nginx/sites-available.d/wiki.conf
```
```bash
nano /etc/nginx/sites-available.d/wiki.conf
upstream wiki.au-team.irpo {
        server 172.16.5.5:80;
}
server {
        listen  8086;
        server_name _;

        location / {
                # root /var/www/html;

                # autoindex off;
                # autoindex_exact_size on;
                # autoindex_localtime off;

                # expires off;

                # cooperate with mod_realip in apache-1.3 or mod_rpaf in apache-2.x
                #       proxy_redirect off;
                #       proxy_set_header Host $host;
                #       proxy_set_header X-Real-IP $remote_addr;
                #       proxy_set_header X-Forwarded-For $remote_addr;
                        proxy_pass http://wiki.au-team.irpo;
                #
                # NB: it's better for URI canonicalization that apache sits on :80
                # (even if that's only 127.0.0.1:80)
                #
                # see also set_real_ip_from, real_ip_header if this nginx
                # would need to cooperate with another one acting as a frontend
        }

#               charset         on;
#               source_charset  koi8-r;

#               access_log  /var/log/nginx/access.log;
}
```
```bash
ln -s /etc/nginx/sites-available.d/moodle.conf /etc/nginx/sites-enabled.d/moodle.conf
ln -s /etc/nginx/sites-available.d/wiki.conf /etc/nginx/sites-enabled.d/wiki.conf
```
```bash
systemctl restart nginx
systemctl reload nginx
systemctl enable --now nginx
```
```bash
systemctl status nginx
```
```bash
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: disabled)
     Active: active (running) since Mon 2025-12-08 00:35:22 UTC; 1min 14s ago
   Main PID: 5749 (nginx)
      Tasks: 11 (limit: 1142)
     Memory: 9.1M (peak: 18.8M)
        CPU: 50ms
     CGroup: /system.slice/nginx.service
             ├─5749 "nginx: master process /usr/sbin/nginx -g daemon off;"
             ├─5768 "nginx: worker process"
             ├─5770 "nginx: worker process"
             ├─5771 "nginx: worker process"
             ├─5772 "nginx: worker process"
             ├─5773 "nginx: worker process"
             ├─5774 "nginx: worker process"
             ├─5775 "nginx: worker process"
             ├─5776 "nginx: worker process"
             ├─5777 "nginx: worker process"
             └─5779 "nginx: worker process"
```
### HQ-SRV
```bash
vim /var/www/webapps/moodle/config.php
$CFG->wwwroot   = 'http://moodle.au-team.irpo/moodle';
```
```bash
systemctl restart httpd2
```
### BR-SRV
```bash
docker compose -f wiki.yml down
```
```bash
mcedit LocalSettings.php
$wgServer = "http://wiki.au-team.irpo:8086";
```
```bash
docker compose -f wiki.yml up -d
```
### HQ-CLI
```bash
su -
toor
mcedit /etc/hosts # Добавляем строки
172.16.4.1      moodle.au-team.irpo moodle
172.16.5.1      wiki.au-team.irpo wiki
```
```bash
nmcli connection modify CLI-NET \
ipv4.ignore-auto-dns yes \
ipv4.dns 192.168.3.10
```
```bash
nmcli connection down CLI-NET
nmcli connection up CLI-NET
```
**Открываем Firefox:**
- Переходим на http://moodle.au-team.irpo/moodle
- Переходим на http://wiki.au-team.irpo:8086

> ⚠️ 💡 **Важно**: Если оба сайта открылись без редиректа на http://hq-srv.au-team.irpo/moodle и 192.168.3.10:8086 (адрес в строке поиска остался тот же), значит все выполенено верно.

**Демонстрационный экзамен полностью выполнен, готовый отчет можно взять [тут.](./report_2025.docx)**
