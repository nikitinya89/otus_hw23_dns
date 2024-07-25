# Otus Homework 23. DNS.
### Цель домашнего задания
Создать домашнюю сетевую лабораторию. Изучить основы DNS. Научиться работать с технологией Split-DNS в Linux-based системах.
### Описание домашнего задания
1. Взять стенд https://github.com/erlong15/vagrant-bind
    - Добавить еще один сервер **client2**
    - Завести в зоне **dns.lab** имена:
        - **web1** - смотрит на клиент1
        - **web2** смотрит на клиент2
    - Завести еще одну зону **newdns.lab**
    - Завести в ней запись
      - **www** - смотрит на обоих клиентов
2. Настроить split-dns
    - **client1** - видит обе зоны, но в зоне **dns.lab** только **web1**
    - **client2** видит только **dns.lab**
## Выполнение
С помощью **Vagrant** создадим тестовый стенд:  

|Имя|IP-адрес|Описание|ОС|
|-|-|-|-|
|**ns01**|192.168.56.10|DNS Master|CentOS 7|
|**ns02**|192.168.56.11|DNS Slave|CentOS 7|
|**client1**|192.168.56.15|Клиент 1|CentOS 7|
|**client2**|192.168.56.16|Клиент 2|CentOS 7|

Установим на каждый сервер пакеты _bind_ и _bind-utils_
```bash
yum install bind bind-utils -y
```
Изменим файлы _/etc/resolv.conf_. На DNS серверах настроим их собственный адрес, на клиентах адреса обоих серверов:
```bash
/etc/resolv.conf

search dns.lab
nameserver 192.168.56.10
nameserver 192.168.56.11
```

Конфигурация мастер сервера:

```bash
/etc/named.conf

options {

    // network
        listen-on port 53 { 192.168.56.10; };
        listen-on-v6 port 53 { ::1; };

    // data
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";

    // server
        recursion yes;
        allow-query     { any; };
    allow-transfer { any; };

    // dnssec
        dnssec-enable yes;
        dnssec-validation yes;

    // others
        bindkeys-file "/etc/named.iscdlv.key";
        managed-keys-directory "/var/named/dynamic";
        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

// RNDC Control for client
key "rndc-key" {
    algorithm hmac-md5;
    secret "GrtiE9kz16GK+OKKU/qJvQ==";
};
controls {
        inet 192.168.56.10 allow { 192.168.56.15; } keys { "rndc-key"; };
};

// ZONE TRANSFER WITH TSIG
include "/etc/named.zonetransfer.key";
server 192.168.56.11 {
    keys { "zonetransfer.key"; };
};

// root zone
zone "." IN {
        type hint;
        file "named.ca";
};

// zones like localhost
include "/etc/named.rfc1912.zones";
// root's DNSKEY
include "/etc/named.root.key";

// lab's zone
zone "dns.lab" {
    type master;
    allow-transfer { key "zonetransfer.key"; };
    file "/etc/named/named.dns.lab";
};

// lab's zone reverse
zone "56.168.192.in-addr.arpa" {
    type slave;
    allow-transfer { key "zonetransfer.key"; };
    file "/etc/named/named.dns.lab.rev";
};
```

В слэйве настройка аналогичная, но необходимо указать type slave и адрес мастера 
```bash
...
// lab's zone
zone "dns.lab" {
    type slave;
    masters { 192.168.56.10; };
    file "/etc/named/named.dns.lab";
};

// lab's zone reverse
zone "56.168.192.in-addr.arpa" {
    type slave;
    masters { 192.168.56.10; };
    file "/etc/named/named.dns.lab.rev";
};
```
Настроим зону **dns.lab** на мастер сервере. Пропишем записи _SOA_ с основными настройками зоны, _NS_ для обоих серверов, а также _A_ записи для всех серверов:
```bash
named.dns.lab
$TTL 3600
$ORIGIN dns.lab.
@               IN      SOA     ns01.dns.lab. root.dns.lab. (
                            2507202401 ; serial
                            3600       ; refresh (1 hour)
                            600        ; retry (10 minutes)
                            86400      ; expire (1 day)
                            600        ; minimum (10 minutes)
                        )

                IN      NS      ns01.dns.lab.
                IN      NS      ns02.dns.lab.

; DNS Servers
ns01            IN      A       192.168.56.10
ns02            IN      A       192.168.56.11
web1            IN      A       192.168.56.15
web2            IN      A       192.168.56.16
```
Аналогично для обратной зоны:
```
named.dns.lab.rev
$TTL 3600
$ORIGIN 56.168.192.in-addr.arpa.
56.168.192.in-addr.arpa.  IN      SOA     ns01.dns.lab. root.dns.lab. (
                            2507202401 ; serial
                            3600       ; refresh (1 hour)
                            600        ; retry (10 minutes)
                            86400      ; expire (1 day)
                            600        ; minimum (10 minutes)
                        )

                IN      NS      ns01.dns.lab.
                IN      NS      ns02.dns.lab.

; DNS Servers
10              IN      PTR     ns01.dns.lab.
11              IN      PTR     ns02.dns.lab.
15              IN      PTR     web1.dns.lab.
16              IN      PTR     web2.dns.lab.
```
Сразу добавим в конфиг файл зону **newdns.lab**, а также файл **named.newdns.lab** с ее описанием. Добавим лве записи **www** для обоих клиентов:
```bash
/etc/named.conf

...
// lab's newdns zone
zone "newdns.lab" {
    type master;
    allow-transfer { key "zonetransfer.key"; };
    file "/etc/named/named.newdns.lab"
};
```
```bash
/etc/named/named.newdns.lab

$TTL 3600
$ORIGIN newdns.lab.
@               IN      SOA     ns01.dns.lab. root.dns.lab. (
                            2507202401 ; serial
                            3600       ; refresh (1 hour)
                            600        ; retry (10 minutes)
                            86400      ; expire (1 day)
                            600        ; minimum (10 minutes)
                        )

                IN      NS      ns01.dns.lab.
                IN      NS      ns02.dns.lab.

; DNS Servers
ns01            IN      A       192.168.56.10
ns02            IN      A       192.168.56.11
www            IN      A       192.168.56.15
www            IN      A       192.168.56.16
```
