# fail2ban

### quick start
```bash
# checkout enabled jails
sudo fail2ban-client status
```

example output:
```
Status
|- Number of jail:	2
 - Jail list:	npm, sshd
```

```
# checkout jail status
sudo fail2ban-client status npm
```

example output:
```
Status for the jail: npm
|- Filter
|  |- Currently failed:	0
|  |- Total failed:	32
|  `- File list:	/home/username/Docker/nginx-proxy-manager/data/logs/proxy-host-1_access.log
   |- Currently banned:	0
   |- Total banned:	4
   `- Banned IP list: xxx.xxx.xxx.xxx
```

### create conf file
```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

### mail template
```bash
# /etc/fail2ban/action.d/mail-whois.local
```

### filter rule for Nginx Proxy Manager
```bash
# /etc/fail2ban/filter.d/npm.conf
[INCLUDES]

[Definition]
failregex = ^.* (405|404|403|401|\-) (405|404|403|401) - .* \[Client <HOST>\] \[Length .*\] .* \[Sent-to <F-CONTAINER>.*</F-CONTAINER>\] <F-USERAGENT>".*"</F-USERAGENT> .*$

ignoreregex = ^.* (404|\-) (404) - .*".*(\.png|\.txt|\.jpg|\.ico|\.js|\.css|\.ttf|\.woff|\.woff2)(/)*?" \[Client <HOST>\] \[Length .*\] ".*" .*$
```
