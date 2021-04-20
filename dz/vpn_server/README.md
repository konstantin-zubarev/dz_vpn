## Стенд для поднятие RAS на базе OpenVPN с клиенскими сертификатами.

![](topology.jpeg)

### Реализация.

### Server-ovpn

Установим пакеты
```
[root@server-ovpn ~]# yum install -y epel-release
[root@server-ovpn ~]# yum install -y openvpn easy-rsa
```

Включим forwarding, пересылка пакетов между интерфейсами
```
[root@server-ovpn ~]# echo net.ipv4.ip_forward = 1 >> /etc/sysctl.conf | sysctl -p
```

#### Сертификаты

Настроим центр сертификации на базе Easy-RSA на `server-ovpn`, скопируем скрипты easy-rsa в каталог `/etc/openvpn/easy-rsa/`:
```
[root@server-ovpn ~]# mkdir -p /etc/openvpn/easy-rsa
[root@server-ovpn ~]# cp -r /usr/share/easy-rsa/3.0.8/* /etc/openvpn/easy-rsa
```

Создадим шаблон для создпния сертификата.
```
[root@server-ovpn ~]# cd /etc/openvpn/easy-rsa
[root@server-ovpn easy-rsa]# vi vars

export KEY_COUNTRY="RU"
export KEY_PROVINCE="Tula"
export KEY_CITY="Tula"
export KEY_ORG="My organisation"
export KEY_EMAIL="root@example.org"
export KEY_CN="IT department"
export EASYRSA_KEY_SIZE=2048
export EASYRSA_ALGO=rsa
export EASYRSA_CA_EXPIRE=3650
export EASYRSA_CERT_EXPIRE=3650
```

Применим:
```
[root@server-ovpn easy-rsa]# . ./vars
```

Инициализируем PKI:
```
[root@server-ovpn easy-rsa]# ./easyrsa init-pki
```

Генерируем корневой сертификат, создаем удостоверяющий центр CA:
```
[root@server-ovpn easy-rsa]# ./easyrsa build-ca nopass
```

Создаем ключ Диффи-Хеллмана:
```
[root@server-ovpn easy-rsa]# ./easyrsa gen-dh
```

Сгенерируем секретный ключ:
```
[root@server-ovpn easy-rsa]# openvpn --genkey --secret pki/ta.key
```

Генирируем сертификат (vpnserver) для сервера:
```
[root@server-ovpn easy-rsa]# ./easyrsa gen-req vpnserver nopass
```

Подпишим сертификат (vpnserver) для сервира:
```
[root@server-ovpn easy-rsa]# ./easyrsa sign-req server vpnserver

Type the word 'yes' to continue, or any other input to abort.
  Confirm request details: yes
```
Подтверим правильность `yes`

Сгенерируем список отозвонных сертивикатов:
```
[root@server-ovpn easy-rsa]# ./easyrsa gen-crl
```

Заблокировать выданный сертификат (для примера user):
```
[root@server-ovpn easy-rsa]# ./easyrsa revoke user
```

#### Настройки OpenVPN

Создадим каталог для хранение сертификатов полученных от CA:
```
[root@server-ovpn easy-rsa]# mkdir -p /etc/openvpn/server/keys
```

Создадим каталог для индивидуальных настроек, клиентов OpenVPN сервера:
```
[root@server-ovpn easy-rsa]# mkdir -p /etc/openvpn/server/ccd
```

Создадим каталог для логов:
```
[root@server-ovpn easy-rsa]# mkdir -p /var/log/openvpn
```

Копируем нужные файлы в каталог `keys`
```
[root@server-ovpn easy-rsa]# cp pki/issued/vpnserver.crt /etc/openvpn/server/keys/
[root@server-ovpn easy-rsa]# cp pki/private/vpnserver.key /etc/openvpn/server/keys/
[root@server-ovpn easy-rsa]# cp pki/dh.pem /etc/openvpn/server/keys/
[root@server-ovpn easy-rsa]# cp pki/ca.crt /etc/openvpn/server/keys/
[root@server-ovpn easy-rsa]# cp pki/ta.key /etc/openvpn/server/keys/
[root@server-ovpn easy-rsa]# cp pki/crl.pem /etc/openvpn/server/keys/
```

Создадим конфигурационный файл сервера:
```
[root@server-ovpn ~]# vi /etc/openvpn/server/server.conf

# OpenVPN Server Config
port 1194
proto udp
dev tun

ca /etc/openvpn/server/keys/ca.crt

cert keys/vpnserver.crt
key keys/vpnserver.key

dh keys/dh.pem

crl-verify keys/crl.pem

tls-auth keys/ta.key 0

cipher AES-256-CBC

server 10.10.1.0 255.255.255.0
ifconfig-pool-persist ipp.txt
push "route 192.168.11.0 255.255.255.0"
#push "dhcp-option DNS 172.20.1.10"
#push "dhcp-option DOMAIN example.org"

client-to-client

client-config-dir ccd

keepalive 10 120
comp-lzo
max-clients 10

persist-key
persist-tun

status /var/log/openvpn/openvpn-status.log
log /var/log/openvpn.log
verb 3

mode server
daemon
```

Запускаем сервис и добавляем в автозагрузку:
```
[root@server-ovpn ~]# systemctl enable --now openvpn-server@server
```

Проверим статус сервиса `openvpn-server@server`:
```
[root@server-ovpn ~]# systemctl status openvpn-server@server
```

#### Создадим пользователя для OpenVPN

Генирируем сертификат (vpnuser) для пользователя:
```
[root@server-ovpn ~]# cd /etc/openvpn/easy-rsa/
[root@server-ovpn easy-rsa]# ./easyrsa gen-req vpnuser nopass
```

Подпишим сертификат (vpnuser) для пользователя:
```
[root@server-ovpn easy-rsa]# ./easyrsa sign-req client vpnuser
```

Скопируем сертификат пользователя в папку `client`.

```
[root@server-ovpn ~]# cp /etc/openvpn/easy-rsa/pki/issued/vpnuser.crt /etc/openvpn/client/
[root@server-ovpn ~]# cp /etc/openvpn/easy-rsa/pki/private/vpnuser.key /etc/openvpn/client/
```

Создадим конфигурационный файл клиента:

<details>
  <summary>vpnuser.ovpn</summary>

```
# OpenVPN Client Config
client
proto udp
dev tun0

remote 172.20.1.10 1194

cipher AES-256-CBC

tls-client

remote-cert-tls server

auth-nocache

comp-lzo

persist-key
persist-tun

resolv-retry infinite

nobind

keepalive 10 120

verb 0

mute 20

<ca>
-----BEGIN CERTIFICATE-----
MIIDNTCCAh2gAwIBAgIJAPcVXFR4xK0gMA0GCSqGSIb3DQEBCwUAMBYxFDASBgNV
BAMMC0Vhc3ktUlNBIENBMB4XDTIxMDQxOTEyMjIxMloXDTMxMDQxNzEyMjIxMlow
FjEUMBIGA1UEAwwLRWFzeS1SU0EgQ0EwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAw
ggEKAoIBAQDeZI/n9JWUmYCQ1zYlGqt6/eIcogsXCzJ8VbYorRyltxnnmEsWbTbw
SuaXZP5UJFw6/B/78kfpmtkE3380D8eZosa8d3xwOkLQ8lCdwQp5DXe/de7Gr8S4
5IONqA0C0jRKtPHvAkkANyWtQc0e4AVd1MOsMb2n+SYsbp1iwLOl+eJzZE8fpoGY
rt+2uC1D2xggbf/MIJlvHxBu7MKQm5YMNzI4uTUNN0fGR2XjO3Rbn5QBMMpkJMJT
YKZsv70kvAbtPHZxvqvwV5+Dgd722BW+Pkt8gMmjSYssDV/DcQa7q3SE556xBTJE
p8MCMCNvmKDSNFSELjBa25zOP8HYeX3xAgMBAAGjgYUwgYIwHQYDVR0OBBYEFG1a
pykQlCnxaz3f705YW6AWhO6UMEYGA1UdIwQ/MD2AFG1apykQlCnxaz3f705YW6AW
hO6UoRqkGDAWMRQwEgYDVQQDDAtFYXN5LVJTQSBDQYIJAPcVXFR4xK0gMAwGA1Ud
EwQFMAMBAf8wCwYDVR0PBAQDAgEGMA0GCSqGSIb3DQEBCwUAA4IBAQCM3uI1bBlX
pA03Fl6uglIw2o58s114sdgJi79VpQJnXhPtiOqaO0LVtGIJyQrevPeY3dCBqFOJ
596G8gS2XLa/pxhIA6p6vV1OfQeaZ21odIuI/ucykYmjmng0GiQH6FePfb0RQAE3
lbZZbCFQ6bAZ8fVpAXz7JGfVc91y8DbnZVdPA6EardjLLsrBe07L/XcnX40+dVYK
KSjX0s/oJxSJl6HRnZ2qQlv/N22TaZFdm8gX4JAI4Nu4/1xFzRK65RxbSOX8fHMa
Ff+eMnxpRpDGkNrt4dXYA2DpKApZToos7PYwMcYoginQkubv32NPQJfU2hDsMLVg
u85/GB1FAQjO
-----END CERTIFICATE-----
</ca>

<cert>
-----BEGIN CERTIFICATE-----
MIIDSzCCAjOgAwIBAgIRAKa1CW10Bkn/D3l5rML0ZbswDQYJKoZIhvcNAQELBQAw
FjEUMBIGA1UEAwwLRWFzeS1SU0EgQ0EwHhcNMjEwNDE5MTI0NjU2WhcNMzEwNDE3
MTI0NjU2WjASMRAwDgYDVQQDDAd2cG51c2VyMIIBIjANBgkqhkiG9w0BAQEFAAOC
AQ8AMIIBCgKCAQEAy2Jy00N8VizhXlJbCChvQ2nc+TN5elYLPLzu8cupJVZw3Uoe
Nnk7UwIiaMrfmHAGa/o2JZrJ+7S7oDBMmBCKBXD+N4iy9Sc9OW4ewqPFmKFqXLU7
MCNtlk/+xKV8xpPWmlHYS5sl0cNiPX7vvZ1S2cXAKaP7+Wi15enPEzwuVczDBXD/
qoQRKQR/5BRBq8Wnyte/TH91NoCKChf2fu+hthiglsRRJHmj7wiAf5ALBLj6Ckg0
IwEzx/OpoCCpaKXD02NDwPALx4DYhOn27nZR4ZwMXcMYiRzsD8EAbe+x+wgWJQl/
Sm2fqO3VIMB1woZLfs2wINCqejPqxBmdJ9pNHwIDAQABo4GXMIGUMAkGA1UdEwQC
MAAwHQYDVR0OBBYEFNVXV3xt4qHPX9OisuvFT9kEQWzLMEYGA1UdIwQ/MD2AFG1a
pykQlCnxaz3f705YW6AWhO6UoRqkGDAWMRQwEgYDVQQDDAtFYXN5LVJTQSBDQYIJ
APcVXFR4xK0gMBMGA1UdJQQMMAoGCCsGAQUFBwMCMAsGA1UdDwQEAwIHgDANBgkq
hkiG9w0BAQsFAAOCAQEAAMlEswQLU0rHxeaz0xUmeKEuBJXZcYUGdrrcN/JNB5b5
uI3FJfrooIbkUIHIdkKop3TAr2jQLwu59Q+7/abqDOdQFaFxaaJiNe93j+UenAvg
BNpZW6ukOpBV3fRtVK6MCnqC8YnI2vVGcT0yteH6051Ydyy6xwAfSPqZj7l+ipQt
UUVnomKe1JD21WiKYEJTavzXz2s4FzXdFiFe6ogPGX0jT+zl7q4/PCneLFByaQtk
uglp8o+u1xfZYghwjnxj5emBw/6ruYd+EkxmF1HljsomZ5GMgAQf1FMbmbaHlJRe
LScsvtkR5HhNKFD2qACJmSKHoxkeaiP8JkLU3/vSiw==
-----END CERTIFICATE-----
</cert>

<key>
-----BEGIN PRIVATE KEY-----
MIIEvwIBADANBgkqhkiG9w0BAQEFAASCBKkwggSlAgEAAoIBAQDLYnLTQ3xWLOFe
UlsIKG9Dadz5M3l6Vgs8vO7xy6klVnDdSh42eTtTAiJoyt+YcAZr+jYlmsn7tLug
MEyYEIoFcP43iLL1Jz05bh7Co8WYoWpctTswI22WT/7EpXzGk9aaUdhLmyXRw2I9
fu+9nVLZxcApo/v5aLXl6c8TPC5VzMMFcP+qhBEpBH/kFEGrxafK179Mf3U2gIoK
F/Z+76G2GKCWxFEkeaPvCIB/kAsEuPoKSDQjATPH86mgIKlopcPTY0PA8AvHgNiE
6fbudlHhnAxdwxiJHOwPwQBt77H7CBYlCX9KbZ+o7dUgwHXChkt+zbAg0Kp6M+rE
GZ0n2k0fAgMBAAECggEBAJAMZUlORfT/Cov0hncLtGopw5FYrNRBtunI4u+skFta
ksuYoDdyu9q560cPUTg1N8coS0ttC1vUEQOO0eDt7eOWuOb6Uc2ighF39h9jQSMu
SBfrYEVjAk3w0H/UucAnm1pnV2uNMkcUPDNUAVbp43FMRyL2+xMV36c8Zya3fBXV
kQi6jf9aJAbaYff9/MSPCqnpNzmnGifG6mH+r2RKFWSRybZlTxrvTDtf5+sq8y7T
AwRBcrV/Yt3PyjtFQ8bsiCOZKZ38rIEmcliRYGfuw4tmRFk6NZsgvN6MEU+GSv9q
na7KfJjXL4xrtsG8xY/VvrRNP7Ci18l5AwHFkTTqjEkCgYEA++JHgWAdUKXULdVH
h6yDOxYNtx6RkAnZRO4rby8s0ycpUYexT2UkKXacDBJJV4zaW3/c9+ZXDyDfqduF
vp6Yi/0IcB/2ocy93jbVPykk+ZiRjnsZ9w65HEWQIhXpJIlTJTuBYQvH0MY50W75
5Ej5xCYCFXpzfBXPWDyfevAJVZMCgYEAzrVHcq/nGaIIesTj9qfwIuVx8//8VcpW
s+Y6NVP0lsjqUk5sBQy3OrBNmHOuGTVxdoJ/GIa2IZoFHP+SNr/V8yItJ2jiBAVH
iIl1llucslp9/5bx0+vTNrD0HoExgff13weWtvyoxe/AMOlHJgHjlsigshtShJlN
Lf+7ue7+ocUCgYEAqF/1jq1SS1PexxiYLUCLpNa0KmHbyh5bR7GyBSvS2f+GpkJ/
q2QfzTvFSmh68HRujtWlP22RasJ94Mym1eqsbH+jwVgMhNIZZc7fs83sMpDjVAxE
KhuEcDqsA60D4XGOzAfAD8BRPOUVR0PY9do2ZAFISVVGl3740LINpZ+JjdMCgYEA
mxq7efsz1bAX3MdQpwxszEKOtyAmH9msJWFj3BAbP+mSqh/ePgvEkW0pTHPA93l6
ogpZg/XRRHVl0NSUyjA4QjAq4Z95uLqdan56lVA4zDvKv7ZrzzDtU+SDyT/3/iYp
gCkZlkUP/zZfaFQi5woT1FjG+940lPrTRujV3DbRKmECgYAHQsCTyWw79d4zYJwi
eEPdKsUbtNXQmjyHgyyIZ6daOfhoqagKzzg8rZGw73bLkdkeur+6mRou8f+1aj7l
ZftDu+joTOvd/CaU9tEMgsj2vPDM0C4Tl9BiWK5dJgtYpayUfC3mCrftWy2NNwNa
9qFZxoSIH/l+xpT7P5oXxEePFA==
-----END PRIVATE KEY-----
</key>

<tls-auth>
-----BEGIN OpenVPN Static key V1-----
46acac840a3df00bc994d57de6f48859
c27d40917c3041d4eadc4d654d3b5407
83e75151a8ffa3a64c40864565ed2470
c574dbd6f8525eade651c8ca67b9cbfe
1250cda5be5b153578baf0938b371ab8
7beebefce654900a73789c9697db0275
9b6b56f31723b2b21bdd6f1ff0d7c782
6d6ee002cff9f6bcc4c8c8c9f8370cc8
24c52ff9697cf79a0aa8bf227a5f2fc4
fc524de86b145ccfc0bc238d19183729
258956d9945821fe96d0170750120df3
91988ff3be71a99805c13180f9af0021
156270d47310277060559ff6e60847e9
e475f7fffb93c79d5bb396a6c9714395
16c91e61aeef5908926ff1d40f945587
0ee8d63dd59365001ca6cd9d90850082
-----END OpenVPN Static key V1-----
</tls-auth>
```
</details>



Ссылка на дополнительную информацию
- [Как настроить openvpn на CentOS](https://serveradmin.ru/nastroyka-openvpn-na-centos/)
