## Стенд для поднятие OpenVPN между между вертуалками.

![](topology.jpeg)

### Реализация.

### OpenVPN server

Установим пакеты
```
[root@vpn-server ~]# yum install -y epel-release
[root@vpn-server ~]# yum install -y openvpn
```

Включим forwarding, пересылка пакетов между интерфейсами
```
[root@vpn-server ~]# echo net.ipv4.ip_forward = 1 >> /etc/sysctl.conf | sysctl -p
```

Сгенерируем секретный ключ:
```
[root@vpn-server ~]# openvpn --genkey --secret /etc/openvpn/server/ta.key
```

Сгенерирующий секретный ключ скопируем на клиент сервер `vpn-client` в каталог `/etc/openvpn/client`.

Создадим каталог для логов:
```
[root@vpn-server ~]# mkdir -p /var/log/openvpn
```

Создадим конфигурационный файл (tap режим) на сервере `vpn-server`:
```
[root@server-ovpn ~]# vi /etc/openvpn/server/server.conf

#
# OpenVPN Server Config
#

dev tun

secret ta.key

ifconfig 10.10.1.1 255.255.255.0
route 192.168.2.0 255.255.255.0 10.10.1.2

topology subnet

compress lzo

status /var/log/openvpn-status.log
log /var/log/openvpn/openvpn.log
verb 3
```

Запускаем сервис и добавляем в автозагрузку:
```
[root@vpn-server ~]# systemctl enable --now openvpn-server@server
```

Проверим статус сервиса `openvpn-server@server`:
```
[root@vpn-server ~]# systemctl status openvpn-server@server
```

### OpenVPN client

Установим пакеты
```
[root@vpn-client ~]# yum install -y epel-release
[root@vpn-client ~]# yum install -y openvpn
```

Включим forwarding, пересылка пакетов между интерфейсами
```
[root@vpn-client ~]# echo net.ipv4.ip_forward = 1 >> /etc/sysctl.conf | sysctl -p
```

Создадим конфигурационный файл (tap режим) на клиент сервере `vpn-client`:
```
[root@vpn-client ~]# vi /etc/openvpn/client/server.conf

#
# OpenVPN Client Config
#

dev tun

secret ta.key

remote 172.20.1.10

ifconfig 10.10.1.2 255.255.255.0
route 192.168.1.0 255.255.255.0 10.10.1.1

topology subnet

compress lzo

status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3
```

Запускаем сервис и добавляем в автозагрузку:
```
[root@vpn-client ~]# systemctl enable --now openvpn-client@server
```

Проверим статус сервиса `openvpn-client@server`:
```
[root@vpn-client ~]# systemctl status openvpn-client@server
```

### Тестируем канал

Протестируем созданный канал с помощью инструмента `iperf3`:
На машине `PC1` запустим утилиту `iperf3` в режиме сервер, а на `PC2` в режиме клиент:
```
[root@pc1 ~]# iperf3 -s
[root@pc2 ~]# iperf3 -c 192.168.1.20 -t 10 -i 5 -b 1000M -u
```

Результат тестирования показал:
```
[ ID] Interval           Transfer     Bandwidth       Jitter    Lost/Total Datagrams
[  4]   0.00-10.00  sec   551 MBytes   462 Mbits/sec  0.208 ms  386026/427024 (90%)
```

Изменим в конфигах /etc/openvpn/server.conf на сервере и клиенте режим с tun на tap.
Сново протестируем.
```
[ ID] Interval           Transfer     Bandwidth       Jitter    Lost/Total Datagrams
[  4]   0.00-10.00  sec   496 MBytes   416 Mbits/sec  0.191 ms  345574/393431 (88%)
```

Тестирование показало, что режим `tun` лучше по сравнению с `tap`. Так же есть разниза между ними, `tap` ведет себя как полноценный сетевой адапптер.

- [Разнеца меду TUN & TAP](https://ru.wikipedia.org/wiki/TUN/TAP)


Проверка задания
----------------

1. Выполнить `vagrant up`, и автоматически поднимает Server и Client с установленным VPN соединением (через tun).

2. Для установления VPN соединения (через tap), выполнить `ansible-playbook vpn-tap.yml`

Ссылка на дополнительную информацию
- [Как настроить openvpn на CentOS](https://serveradmin.ru/nastroyka-openvpn-na-centos/)
