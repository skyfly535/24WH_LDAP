# LDAP. Централизованная авторизация и аутентификация.

Задание:

1) Установить FreeIPA;

2) Написать Ansible-playbook для конфигурации клиента;

Дополнительное задание:

3)* Настроить аутентификацию по SSH-ключам;

4)** Firewall должен быть включен на сервере и на клиенте.


Цель:

Научиться настраивать LDAP-сервер и подключать к нему LDAP-клиентов.

## Развертывание стенда для демострации настройки FreeIPA.

Стэнд состоит из хостовой машины под управлением ОС Ubuntu 20.04 на которой развернут Ansible и трех виртуальных машин `bento/centos-8.4`.

В исходном Vagranfile я изменил Vagrant boxes `centos/stream8` на `bento/centos-8.4` так как `centos/stream8` не доступен хранилище Vagrant.

Машина с именами: `ipa.otus.lan` выполняет роль FreeIPA сервера.

Машины с именами: `client1.otus.lan`, `client2.otus.lan` выполняют роль клиентов.

Схема коммутации:


Разворачиваем инфраструктуру на сервре в ручную (ввиду особенностей настройки), а на клиентах через Ansible.

Все коментарии по каждому блоку указаны в тексте Playbook - `ldap.yml`.

Файлs `hosts` и `hosts.j2` необходимо поместить в каталог с Playbook `ldap.yml` и Vagranfile.

Выполняем установку стенда:

```
vagrant up
```
## Установка и настройка FreeIPA сервера

Подключимся к нему по SSH с помощью команды: 
```
vagrant ssh ipa.otus.lan
```
и перейдём в root-пользователя: sudo -i.

Установим часовой пояс:

```
timedatectl set-timezone Asia/Vladivostok
```
в соям случае это Владивосток.

Установим утилиту chrony:

```
yum install -y chrony
```
Запустим chrony и добавим его в автозагрузку: 

```
systemctl enable chronyd —now
```

Если требуется, поменяем имя нашего сервера:

```
hostnamectl set-hostname <имя сервера>
```
Настроим Firewall:

```
firewall-cmd --permanent --add-port=53/{tcp,udp} --add-port={80,443}/tcp --add-port={88,464}/{tcp,udp} --add-port=123/udp --add-port={389,636}/tcp
```

```
firewall-cmd --reload
```
где:

53 — запросы DNS. Не обязателен, если мы не планируем использовать наш сервер в качестве сервера DNS.

80 и 443 — http и https для доступа к веб-интерфейсу управления.

88 и 464 — kerberos и kpasswd.

123 — синхронизация времени.

389 и 636 — ldap и ldaps соответственно.

Остановим Selinux: 

```
setenforce 0
```

```
 sed -i 's/^SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config
```
Для дальнейшей настройки FreeIPA нам потребуется, чтобы DNS-сервер хранил запись о нашем LDAP-сервере. В рамках данной лабораторной работы мы не будем настраивать отдельный DNS-сервер и просто добавим запись в файл `/etc/hosts`

```
nano /etc/hosts

127.0.0.1   localhost localhost.localdomain 
127.0.1.1 ipa.otus.lan ipa
192.168.57.10 ipa.otus.lan ipa
```

Установим модуль DL1: 

```
yum install -y @idm:DL1
```

Установим FreeIPA-сервер:

```
yum install -y ipa-server
```

Запустим скрипт установки:

```
ipa-server-install
```

Далее, нам потребуется указать параметры нашего LDAP-сервера, после ввода каждого параметра нажимаем Enter, если нас устраивает параметр, указанный в квадратных скобках, то можно сразу нажимать Enter:

Do you want to configure integrated DNS (BIND)? [no]: `no`

Server host name [ipa.otus.lan]: `<Нажимем Enter>`

Please confirm the domain name [otus.lan]: `<Нажимем Enter>`

Please provide a realm name [OTUS.LAN]: `<Нажимем Enter>`

Directory Manager password: `<Указываем пароль минимум 8 символов>`

Password (confirm): `<Дублируем указанный пароль>`

IPA admin password: `<Указываем пароль минимум 8 символов>`

Password (confirm): `<Дублируем указанный пароль>`

NetBIOS domain name [OTUS]: `<Нажимем Enter>`

Do you want to configure chrony with NTP server or pool address? [no]: `no`

The IPA Master Server will be configured with:
Hostname:       ipa.otus.lan
IP address(es): 192.168.57.10
Domain name:    otus.lan
Realm name:     OTUS.LAN

The CA will be configured with:
Subject DN:   CN=Certificate Authority,O=OTUS.LAN
Subject base: O=OTUS.LAN
Chaining:     self-signed

Проверяем параметры, если всё устраивает, то нажимаем yes

Continue to configure the system with these values? [no]: `yes`

Примечание: 

`Directory Manager password` — это пароль администратора сервера каталогов, У этого пользователя есть полный доступ к каталогу.

`IPA admin password` — пароль от пользователя FreeIPA admin.

После успешной установки FreeIPA, проверим, что сервер Kerberos может выдать нам билет:
```
[root@ipa vagrant]# kinit admin
Password for admin@OTUS.LAN: 
[root@ipa vagrant]# klist
Ticket cache: KCM:0
Default principal: admin@OTUS.LAN

Valid starting     Expires            Service principal
05/31/23 20:16:00  06/01/23 19:54:03  krbtgt/OTUS.LAN@OTUS.LAN
```

Зходим в Web-интерфейс нашего FreeIPA-сервера, для этого на нашей хостой машине нужно прописать следующую строку в файле `/etc/hosts`:

Картинка 1

```
192.168.57.10 ipa.otus.lan
```

## Настройка FreeIPA клиента

Настройка клиентов производится непосредственно по средствам Ansible playbook.

```
ansible-playbook ldap.yml -i hosts --extra-vars "host=clients"
```
После настройки клиентской части проверяем состояние и настройки `firewalld`:

```
[root@client1 ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since Sat 2023-06-03 23:44:34 +10; 11min ago
     Docs: man:firewalld(1)
 Main PID: 5433 (firewalld)
    Tasks: 3 (limit: 4955)
   Memory: 33.4M
   CGroup: /system.slice/firewalld.service
           └─5433 /usr/libexec/platform-python -s /usr/sbin/firewalld --nofork --nopid

Jun 03 23:44:34 client1.otus.lan systemd[1]: Starting firewalld - dynamic firewall daemon...
Jun 03 23:44:34 client1.otus.lan systemd[1]: Started firewalld - dynamic firewall daemon.
Jun 03 23:44:34 client1.otus.lan firewalld[5433]: WARNING: AllowZoneDrifting is enabled. This is considered an insecure configuration option. It will be removed in a future release. Please consider disa>
Jun 03 23:44:42 client1.otus.lan systemd[1]: Reloading firewalld - dynamic firewall daemon.
Jun 03 23:44:42 client1.otus.lan systemd[1]: Reloaded firewalld - dynamic firewall daemon.
Jun 03 23:44:42 client1.otus.lan firewalld[5433]: WARNING: AllowZoneDrifting is enabled. This is considered an insecure configuration option. It will be removed in a future release. Please consider disa>
Jun 03 23:47:49 client1.otus.lan systemd[1]: Reloading firewalld - dynamic firewall daemon.
Jun 03 23:47:49 client1.otus.lan systemd[1]: Reloaded firewalld - dynamic firewall daemon.
Jun 03 23:47:49 client1.otus.lan firewalld[5433]: WARNING: AllowZoneDrifting is enabled. This is considered an insecure configuration option. It will be removed in a future release. Please consider disa>

[root@client1 ~]# firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eth0 eth1
  sources: 
  services: cockpit dhcpv6-client ssh
  ports: 53/tcp 53/udp 80/tcp 443/tcp 88/tcp 464/tcp 88/udp 464/udp 123/udp 389/tcp 636/tcp
  protocols: 
  forward: no
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
```

Cоздадим поьзователя командой:

```
echo -e "yes\nyes" | ipa-client-install --mkhomedir --domain=OTUS.LAN --server=ipa.otus.lan --no-ntp -p admin -w Passw0rd!
```

При добавлении хоста к домену мы можем просто ввести команду `ipa-client-install` и следовать мастеру подключения к FreeIPA-серверу (как было в первом пункте).

Однако команда позволяет нам сразу задать требуемые нам параметры:

--domain — имя домена

--server — имя FreeIPA-сервера

--no-ntp — не настраивать дополнительно ntp (мы уже настроили chrony)

-p — имя админа домена

--w — пароль администратора домена (IPA password)

--mkhomedir — создать директории пользователей при их первом логине

Если мы сразу укажем все параметры, то можем добавить эту команду в Ansible и автоматизировать процесс добавления хостов в домен. 

## Настроить аутентификацию по SSH-ключам

Для демострации настройки аутентификации по SSH-ключам зайдем на `client1` под пользователем `otus-user` создадим пару ключей

```
root@root-ubuntu:/home/roman/LDAP# ssh otus-user@192.168.57.11
Password: 

This system is built by the Bento project by Chef Software
More information can be found at https://github.com/chef/bento
[otus-user@client1 ~]$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/otus-user/.ssh/id_rsa): 
Created directory '/home/otus-user/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/otus-user/.ssh/id_rsa.
Your public key has been saved in /home/otus-user/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:2gkyObI76sssq4nHxt1CSX3Bs+VwL/2LQBSCfRxPp7k otus-user@client1.otus.lan
The key's randomart image is:
+---[RSA 3072]----+
|       +..oo. .  |
|      . *.*o +   |
|     .   @ o+    |
|    ... o + o.   |
|  ..=...S. .E.   |
|   oo+ + ..   .  |
| o.o .. o  . . . |
|=.*.o .     . .  |
|XXo. .           |
+----[SHA256]-----+
```
и загрузим открытую часть ключа на наш сервер

```
[otus-user@client1 ~]$ ipa user-mod otus-user --sshpubkey="$(cat /home/otus-user/.ssh/id_rsa.pub)"
-------------------------
Modified user "otus-user"
-------------------------
  User login: otus-user
  First name: Otus
  Last name: User2023
  Home directory: /home/otus-user
  Login shell: /bin/sh
  Principal name: otus-user@OTUS.LAN
  Principal alias: otus-user@OTUS.LAN
  Email address: otus-user@otus.lan
  UID: 1411400003
  GID: 1411400003
  SSH public key: ssh-rsa
                  AAAAB3NzaC1yc2EAAAADAQABAAABgQDJ07Ruhxn2qHk0tgIXI8OXWzXllnEDdoTyG3R1eDAw0bMKY64O/Hxmg2y1zsXrAInmoa/7GI/bFd61KyNVbtNeG3rnLddVJKnD8u5R/PEL6E8OHJHp4xTYLmZmd7Gna8cQzxQ56E5cp5X3UWrucGPHEUWGsKMONCaF5/vMembSh8FyOkSBiVPmoi0ik2fHu4kRcLu91S+4QwdCyhuR+OCkKyEEWGtjt9X9lQIyYP15WmIvPPOIZb15CCsO1UYqZ7s0rmqv62SedyCLTdhsOS1icLGogX9j/4gaQgsySaACZqwGKpTERM+HQPRQEvflaWkBtZXKXqmNB1jPRSbClpg8CpCTigYlTzyb8KWjK86O1X5O4Wx8y7zK6Uj0FBVML+MVAPM5g7nCoRq+LsXUQodQd7DJsgaP762j4fI2BNtCFnt7A86TEIfGrARtjtwhqKzyZmiBo2GCrWkmQAhOD3JDFgV8V6yEEqQKB/yAyFQRhCIWWcafGOJ2AxHCuOewJGM=
                  otus-user@client1.otus.lan
  SSH public key fingerprint: SHA256:McqMH2gT68iN6ZkQvpjLlNIcl928oJP1KIALKEHG22U otus-user@client1.otus.lan (ssh-rsa)
  Account disabled: False
  Password: True
  Member of groups: ipausers
  Kerberos keys available: True
```
Проверяем на Web интерфейсе сервера, что ключ подцепился к польхователю

Картинка 2

Заходим на `client2` c `client1` по SSH-ключу без пароля

```
[otus-user@client1 ~]$ ssh otus-user@192.168.57.12

This system is built by the Bento project by Chef Software
More information can be found at https://github.com/chef/bento
Last login: Sat Jun  3 23:52:03 2023 from 192.168.57.11
[otus-user@client2 ~]$ 
```