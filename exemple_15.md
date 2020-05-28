# Pràctica Tunnel-SSH

## INDEX:

- [Exemple 15: Ldap-remot i phpldapadmin-local](#Exemple-15-Ldap-remot-i-phpldapadmin-local)



# Exemple 15: Ldap-remot i phpldapadmin-local

- Objectiu: Desplegem dins d’un container Docker (host-remot) en una AMI (host-destí) el servei ldap amb el firewall de la AMI només obrint el port 22. Localment al host de l’aula (host-local) desplegem un container amb phpldapadmin. Aquest container ha de poder accedir a les dades ldap. des del host de l’aula volem poder visualitzar el phpldapadmin.

## Desplegar el servei Ldap:

- Primer de tot, despleguem en el host-remot (AMI) un container Ldap sense fer map dels ports. En aquest cas l'executem de manera interactiva per poder modificar el valor "ulimit" abans d'executar-lo:

```
[fedora@june-ami ~]$ docker run --rm -h ldap.edt.org --name ldap.edt.org --net mynet -it adriaquintero61/ldapserver19:grup /bin/bash
[root@ldap docker]# ulimit -n 4096
[root@ldap docker]# bash startup.sh 
5ed00760 mdb_db_open: database "dc=edt,dc=org" cannot be opened: No such file or directory (2). Restore from backup!
5ed00760 backend_startup_one (type=mdb, suffix="dc=edt,dc=org"): bi_db_open failed! (2)
slap_startup failed (test would succeed using the -u switch)
_#################### 100.00% eta   none elapsed            none fast!         
Closing DB...

```

- Un cop hem arrancat el servei Ldap, revisem que la AMI només té obert el port 22:

```
[fedora@june-ami ~]$ nmap localhost
Starting Nmap 7.70 ( https://nmap.org ) at 2020-05-28 18:50 UTC
Nmap scan report for localhost (127.0.0.1)
Host is up (0.00042s latency).
Other addresses for localhost (not scanned): ::1
Not shown: 999 closed ports
PORT   STATE SERVICE
22/tcp open  ssh

Nmap done: 1 IP address (1 host up) scanned in 0.09 seconds
```

- Una vegada hem comprobat que només el port 22 està obert, configurem el fitxer "/etc/hosts" de la AMI per poder accedir al container ldap per nom de host:

```
[fedora@june-ami ~]$ cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.18.0.2 ldap.edt.org
```

- Un cop configurat, comprovem que desde el host de l'aula (host-local) podem fer consultes ldap:

```
[adria@pc ~]$ ssh -i .ssh/sshawskey.pem -L 9001:ldap.edt.org:389 fedora@35.178.42.0
Last login: Thu May 28 18:49:47 2020 from 85.219.36.234
```
- Un cop establert el túnel, comprovem a fer una consulta Ldap:

```
[adria@pc ~]$ ldapsearch -x -LLL -h localhost -p 9001 -b 'dc=edt,dc=org' | tail -n16
dn: uid=user10,ou=usuaris,dc=edt,dc=org
objectClass: posixAccount
objectClass: inetOrgPerson
cn: user10
cn: alumne10 de 2asix de todos los santos
sn: alumne10
homePhone: 555-222-0016
mail: user10@edt.org
description: alumne de 2asix
ou: 2asix
uid: user10
uidNumber: 7010
gidNumber: 1104
homeDirectory: /tmp/home/2asix/user10
userPassword:: e1NIQX1vdmY4dGEvcmVZUC91MnpqMGFmcEh0OHlFMUE9
```

## Desplegar el servei phpldapadmin:

- Per començar, hem d'engegar en le host de l'aula (host-local) un container docker amb el servei phpldapadmin fent map del seu port 8080 al host-local:

```
[adria@pc ~]$ docker run --rm -h phpldapadmin --name phpldapadmin -p 8080:80 --net mynet -it docker.io/edtasixm06/phpldapadmin /bin/bash
[root@phpldapadmin /]#
```

- Una vegada arrencat el phpldapadmin, creem el túnel directe ssh des del host de l'aula (host-local) al servei ldap (host-remot) connectant via SSH al host AMI (host-destí):

```
[adria@pc ~]$ ssh -i .ssh/sshawskey.pem -L 172.18.0.1:9001:ldap.edt.org:389 fedora@35.178.42.0
Last login: Thu May 28 20:03:36 2020 from 85.219.36.234
[fedora@june-ami ~]$
```

- Un cop tenim el túnel SSH establert, hem de configurar el phpldapadmin per que trobi la base de dades Ldap accedint al host de l'aula al port acabat de crear amb el túnel directe ssh:

```
[root@phpldapadmin /]# vi /etc/phpldapadmin/config.php

$servers->setValue('server','host','172.18.0.1');
$servers->setValue('server','port',9001);
$servers->setValue('server','base',array('dc=edt,dc=org'));
```

- Quan ja tenim el servei configurat, executem el servei httpd per poder accedir al phpldapadmin:

```
[root@phpldapadmin /]# /usr/sbin/httpd
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 172.19.0.2. Set the 'ServerName' directive globally to suppress this message
[root@phpldapadmin /]# /usr/sbin/httpd -S
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 172.19.0.2. Set the 'ServerName' directive globally to suppress this message
VirtualHost configuration:
ServerRoot: "/etc/httpd"
...
User: name="apache" id=48
Group: name="apache" id=48
```

- Un cop configurat i executat, ja podem visualitzar des del host de l’aula el servei phpldapadmin, accedint al port 8080 del container phpldapadmin o al port que hem fet map del host de l’aula:

![](./aux/phpldapadmin_15.1.PNG)

- Iniciem sessió amb l'usuari "anonymous" i comprovem que podem veure la Base de dades Ldap:

![](./aux/phpldapadmin_15.2.PNG)