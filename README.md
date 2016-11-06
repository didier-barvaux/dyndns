# dyndns - simple dynamic DNS
Update the given dynamic DNS A record remotely.

A remote HTTP(S) URI is used to determine the public IP address of the host.
A crypto key is used to secure the communication with the DNS server that
hosts the DNS zone to update.

* author:   Didier Barvaux
* homepage: https://github.com/didier-barvaux/dyndns
* license:  Simplified BSD License (see COPYING)

## Features
* support Bind `nsupdate`
* support secure updates
* lightweight (no daemon, only shell script and a cron task)

## Installation
### Installation at DNS server
Create a new DNS zone within bind:
```
# cd /etc/named/dyn/
# touch dyn.example.com.zone
# chgrp named dyn.example.com.zone
# chmod g+w dyn.example.com.zone
# vim dyn.example.com.zone
```
Copy/paste the following content:
```
$ORIGIN .
$TTL 10 ; 10 seconds
dyn.example.com         IN SOA  ns1.example.com. root.example.com. (
                                1          ; serial
                                14400      ; refresh (4 hours)
                                3600       ; retry (1 hour)
                                604800     ; expire (1 week)
                                10         ; minimum (10 seconds)
                                )
$TTL 3600       ; 1 hour
                        NS      ns1.example.com.
                        NS      ns2.example.com.
                        MX      1 mail.example.com.

$ORIGIN dyn.example.com.
$TTL 600        ; 10 minutes
```

Create a crypto key:
```
# cd /etc/named/
# dnssec-keygen -b 512 -a HMAC-SHA512 -v 2 -n HOST dyn.example.com.
```
You should now have 2 new files in the current directory:
```
Kdyn.example.com.+xxx+yyyyy.key
Kdyn.example.com.+xxx+yyyyy.private
```

Configure bind with the new zone and the capacity to update the zone with the
crypto key:
```
# cd /etc/named/
# awk '$1 == "Key:" { print $2 }' Kdyn.example.com.+xxx+yyyyy.private
(copy/paste the printed line at next step)
# vim /etc/named/named.conf 
```
Add the following content:
```
key dyn.example.com. {
	algorithm "HMAC-SHA512";
	secret "<secret key here>";
};

zone "dyn.example.com" IN {
   type master;
   file "dyn/dyn.example.com.zone";
   allow-query { any; };
   update-policy {
      grant dyn.example.com. name myfirsthost.dyn.example.com. A;
      grant dyn.example.com. name my2ndhost.dyn.example.com. A;
   };
   notify yes;
};
```

Then reload bind configuration:
```
# /etc/init.d/named reload
```

### Installation on the dynamic host(s)
Build the man page as normal user: `make all`

Then install as root: `make install PREFIX=/usr`

It requires `help2man`, `bash`, `curl`, `logger`, `awk`, `grep`, `nsupdate`,
and `mktemp`.

Then copy the content of the 2 Kdyn.example.com.+xxx+yyyyy.{key,private} files
on the host:
```
# mkdir /etc/dyndns/
# touch /etc/dyndns/Kdyn.example.com.+xxx+yyyyy.{key,private}
# chgrp -R nobody /etc/dyndns/
# chmod -R o-rwx,g-w /etc/dyndns/
# vim /etc/dyndns/Kdyn.example.com.+xxx+yyyyy.key
(copy/paste content)
# vim /etc/dyndns/Kdyn.example.com.+xxx+yyyyy.private
(copy/paste content)
```

## Uninstall
### Uninstallation at DNS server
```
# rm -f /etc/named/dyn/dyn.example.com.zone
# rm -f /etc/named/Kdyn.example.com.+xxx+yyyyy.{key,private}
# vim /etc/named/named.conf
(remove the key dyn.example.com. and zone "dyn.example.com" blocks)
# /etc/init.d/named reload
```

### Uninstallation on the dynamic host(s)
Just run `make uninstall PREFIX=/usr`.

If you want, remove the crypto key too:
```
# rm -rf /etc/dyndns/
```

## Usage
Run manually:
```
# runuser -u nobody /usr/bin/dyndns https://example.com/checkip.php /etc/dyndns/Kdyn.example.com.+xxx+yyyyy.key ns1.example.com dyn.example.com myfirsthost.dyn.example.com.
```

Use cron to update the host's IP address periodically:
```
# vim /etc/crontab
```
then add the line below to update the DNS entry every 15 minutes:
```
*/15 * * * * nobody  /usr/bin/dyndns https://example.com/checkip.php /etc/dyndns/Kdyn.example.com.+xxx+yyyyy.key ns1.example.com dyn.example.com myfirsthost.dyn.example.com.
```

