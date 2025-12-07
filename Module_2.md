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
- DNS Forwards - 192.168.1.10
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
```
```bash
apt-get install -y nfs-{server,utils}
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
```
### HQ-CLI
```bash
apt-get install -y nfs-{server,utils}
mkdir /mnt/nfs
chmod 777 /mnt/nfs
```
```bash
mcedit /etc/fstab
192.168.1.10:/raid0/nfs	/mnt/nfs	nfs	defaults	0	0
```
**Монтируем файловую систему и делаем финальную проверку RAID 0**:
```bash
mount -a
df -h
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
pool 127.0.0.1 iburst prefer
hwtimestamp *
local stratum 5
allow 0/0
```
**Запустим службу времени:**
```bash
systemctl restart chronyd
systemctl enable --now chronyd
timedatectl set-timezone Asia/Novosibirsk
timedatectl set-local rtc yes
```
В качестве клиентов настроим: HQ-SRV, HQ-CLI, BR-RTR, BR-SRV, выполнить настройку нужно идентично нижней на всех 4-ех клиентах.
```bash
apt-get update && apt-get install -y chrony
vim /etc/chrony.conf
pool 192.168.10.1 iburst prefer
systemctl restart chronyd
systemctl enable --now chronyd
timedatectl set-timezone Asia/Novosibirsk
timedatectl set-local-rtc yes
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
**Вставляем скопированный конфиг, теперь можно закрыть Яндекс браузер. Далее приводим файл wiki.yml к следующему виду**
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
- Database type - Mariadb
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
