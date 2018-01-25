# linotp-freeradius3-centos7



### assumption #1: working linotp

```
[root@yourserver:~]# curl -k "https://yourserver/validate/check?user=<USERNAME>&pass=<OTP>"
{
"version": "LinOTP 2.9.3.3",
"jsonrpc": "2.0802",
"result": {
    "status": true,
    "value": true
},
"id": 0
}
[root@yourserver:~]# 

[root@yourserver:~]# curl -k "https://yourserver/validate/simplecheck?user=<USERNAME>&pass=<OTP>"
:-)
[root@yourserver:~]# 
```

### assumption #2: os version

```
[root@yourserver:~]# cat /etc/redhat-release
CentOS Linux release 7.4.1708 (Core)
[root@yourserver:~]# 
```

### assumption #3: installed packages

```
[root@yourserver:~]# rpm -qa | grep -iP 'LinOTP|freeradius'
freeradius-3.0.13-8.el7_4.x86_64
LinOTP_repos-1.1-1.el7.x86_64
LinOTP_mariadb-2.9.3-1.el7.x86_64
freeradius-perl-3.0.13-8.el7_4.x86_64
LinOTP_apache-2.9.3-1.el7.x86_64
LinOTP-2.9.3.3-1.el7.x86_64
[root@yourserver:~]#
```


### backup freeradius default config

```
[root@yourserver:~]# cp -a /etc/raddb/ /etc/raddb.old
[root@yourserver:~]# 
```


### update /etc/raddb/clients.conf

```
[root@yourserver:~]# cat clients.conf
client 172.16.251.210 {
        ipaddr = 172.16.251.210
        secret = secret123
}

client 172.16.251.211 {
        ipaddr = 172.16.251.211
        secret = secret123
}

client 10.75.1.9 {
        ipaddr = 10.75.1.9
        secret = secret123
}

[root@yourserver:~]#
```


### update users file
* on centos, this is symlink, so just remove the symlink, and create a new file

```
[root@yourserver:~]# readlink -f users
/etc/raddb/mods-config/files/authorize
[root@yourserver:~]# rm users
rm: remove symbolic link ‘users’? y
[root@yourserver:~]# vim users
...
[root@yourserver:~]# cat users
DEFAULT Auth-type := perl
[root@yourserver:~]#
```
