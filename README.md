# DNS. Настройка и обслуживание #
1. На предложенном стенде https://github.com/erlong15/vagrant-bind:
   - добавить сервер client2;
   - завести в зоне   dns.lab имена:
     - web1 - смотрит на client1;
     - web2 - смотрит на client2;
   - завести зону newdns.lab;
   - завести в новой зоне запись:
     - www - смотрит на обоих клиентов;
2. Настроить split-dns:
   - client1 - видит обе зоны, но в зоне dns.lab только web1;
   - client2 - видит только dns.lab;
3. Дополнительное задание:
   - произвести настройки без выключения selinux.
### Исходные данные ###
&ensp;&ensp;ПК на Linux c 8 ГБ ОЗУ или виртуальная машина (ВМ) с включенной Nested Virtualization.<br/>
&ensp;&ensp;Предварительно установленное и настроенное ПО:<br/>
&ensp;&ensp;&ensp;Hashicorp Vagrant (https://www.vagrantup.com/downloads);<br/>
&ensp;&ensp;&ensp;Oracle VirtualBox (https://www.virtualbox.org/wiki/Linux_Downloads).<br/>
&ensp;&ensp;&ensp;Ansible (https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).<br/>
&ensp;&ensp;Все действия проводились с использованием Vagrant 2.4.0, VirtualBox 7.0.18, Ansible 9.4.0 и образа CentOS 7 версии 1804_2. <br/>
### Ход решения ###
1. Установка необходимого программного обеспечения (ПО):
```shell
yum install -y bind bind-utils ntp
```
2. Конфигурирование resolv.conf на серверах и клиентах:
```shell
[vagrant@ns01 ~]$ cat /etc/resolv.conf
domain dns.lab
search dns.lab
nameserver 192.168.56.10

[vagrant@ns02 ~]$ cat /etc/resolv.conf 
domain dns.lab
search dns.lab
nameserver 192.168.56.11

[vagrant@client1 ~]$ cat /etc/resolv.conf 
domain dns.lab
search dns.lab
nameserver 192.168.56.10
nameserver 192.168.56.11

[vagrant@client2 ~]$ cat /etc/resolv.conf 
domain dns.lab
search dns.lab
nameserver 192.168.56.10
nameserver 192.168.56.11
```
3. Конфигурирование зоны dns.lab - добавление в файл named.dns.lab информации об именах web1 и web2:
```shell
[root@ns01 ~]# cat /etc/named/named.dns.lab
$TTL 3600
$ORIGIN dns.lab.
@               IN      SOA     ns01.dns.lab. root.dns.lab. (
                            2711201407 ; serial
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

; Web
web1			IN		A		192.168.56.15
web2			IN		A		192.168.56.16
```
4. Проверка конфигурации мастер сервера:
```shell
[root@ns01 ~]# cat /etc/named.conf 
options {

    // network 
	listen-on port 53 { 192.168.56.10; };
	listen-on-v6 port 53 { ::1; };

    // data
	directory 	"/var/named";
	dump-file 	"/var/named/data/cache_dump.db";
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
        inet 192.168.56.16 allow { 192.168.56.15; } keys { "rndc-key"; }; 
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
    type master;
    allow-transfer { key "zonetransfer.key"; };
    file "/etc/named/named.dns.lab.rev";
};

// lab's ddns zone
zone "ddns.lab" {
    type master;
    allow-transfer { key "zonetransfer.key"; };
    allow-update { key "zonetransfer.key"; };
    file "/etc/named/named.ddns.lab";
};
```
5. Проверка конфигурации слейв сервера. Конфигурация схожа с мастер конфигурацией, за исключением описания зон. В частности, необходимо указать type slave, адрес мастер сервера и путь к файлу с настройками зоны на мастер сервере:
```shell
[root@ns02 ~]# cat /etc/named.conf
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
6. Создание новой зоны newdns.lab и добавление в неё записей:
```shell
[root@ns01 ~]# cat /etc/named/named.newdns.lab 
$TTL 3600
$ORIGIN newdns.lab.
@		IN	SOA	ns01.dns.lab. root.dns.lab. (
			0208202402 ; serial
			3600       ; refresh (1 hour)
			600        ; retry (10 minutes)
			86400      ; expire (1 day)
			600	   ; minimum (10 minutes) 
			)

		IN 	NS	ns01.dns.lab.
		IN 	NS	ns02.dns.lab.
; DNS Servers
ns01		IN	A	192.168.56.10
ns02		IN	A	192.168.56.11

; WWW
www		IN	A	192.168.56.15
www		IN	A	192.168.56.16 
```
7. Добавление информации о новой зоне в named.conf на серверах ns01 и ns02:
```shell
[root@ns01 ~]# cat /etc/named.conf 
...
// lab's newdns zone
zone "newdns.lab" {
    type master;
    allow-transfer { key "zonetransfer.key"; };
    allow-update { key "zonetransfer.key"; };
    file "/etc/named/named.newdns.lab";
};
[root@ns02 ~]# cat /etc/named.conf
...
// lab's newdns zone
zone "newdns.lab" {
    type slave;
    masters { 192.168.56.10; };
    file "/etc/named/named.newdns.lab";
};
```
8. Проверка работоспособности выполненных настроек DNS:
```shell
[vagrant@client1 ~]$ dig @192.168.56.10 ns01.dns.lab +short
192.168.56.10
[vagrant@client1 ~]$ dig @192.168.56.10 ns02.dns.lab +short
192.168.56.11
[vagrant@client1 ~]$ dig @192.168.56.10 web1.dns.lab +short
192.168.56.15
[vagrant@client1 ~]$ dig @192.168.56.10 web2.dns.lab +short
192.168.56.16
```

