# linotp-freeradius3-centos7

The current LinOTP/Freeradius documentation are written for Debian, and to get this working on CentOS we need do things a little differently. 


### assumptions
#### #1: working linotp

```
[root@yourserver:~]# curl -k "https://yourlinotpserver/validate/check?user=<USERNAME>&pass=<OTP>"
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

[root@yourserver:~]# curl -k "https://yourlinotpserver/validate/simplecheck?user=<USERNAME>&pass=<OTP>"
:-)
[root@yourserver:~]# 
```

#### #2: os version

```
[root@yourserver:~]# cat /etc/redhat-release
CentOS Linux release 7.4.1708 (Core)
[root@yourserver:~]# 
```

#### #3: installed packages

```
freeradius-3.0.13-8.el7_4.x86_64
freeradius-perl-3.0.13-8.el7_4.x86_64
perl-Config-IniFiles.noarch
```


### backup freeradius default config

```
[root@yourserver:~]# cp -a /etc/raddb/ /etc/raddb.old
[root@yourserver:~]# 
```


### update /etc/raddb/clients.conf

```
[root@yourserver:~]# cat clients.conf
client 10.0.0.10 {
        ipaddr = 10.0.0.10
        secret = itsasecret
}

client 10.0.0.20 {
        ipaddr = 10.0.0.20
        secret = itsasecret
}

client 10.0.0.30 {
        ipaddr = 10.0.0.30
        secret = itsasecret
}

[root@yourserver:~]#
```


### update users file
* on centos, this is a symlink, so just remove the symlink, and create a new file

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


### get perl module from github

* [linotp-perl.pm](linotp-perl.pm)
* download it to: /etc/raddb/mods-config/perl/radius_linotp.pm
* ordinarily i would recommend getting this from the [official repo](https://github.com/LinOTP/linotp-auth-freeradius-perl), but i _KNOW_ that it doesnt work. This differs from the linotp-provided modules in that it uses the "Config::IniFiles" instead of "Config::File" which is not avialable in default CentOS/Redhat repo's



### enable the perl module

```
[root@yourserver:~]# ln -s /etc/raddb/mods-available/perl /etc/raddb/mods-enabled/perl
[root@yourserver:~]# readlink -f /etc/raddb/mods-enabled/perl
/etc/raddb/mods-available/perl
[root@yourserver:~]#
```

### configure the perl module

```
[root@yourserver:~]# cat /etc/raddb/mods-available/perl
perl {
    filename = ${modconfdir}/${.:instance}/radius_linotp.pm

    func_authenticate = authenticate
    func_authorize = authorize
}
[root@yourserver:~]#
```


### create the config file for the perl module

* again this is _VERY_ similar to the official config file, we've just create a default section

```
[root@yourserver:~]# cat /etc/raddb/mods-config/perl/radius_linotp.ini
[Default]
#IP of the linotp server
URL=https://yourlinotpserver/validate/simplecheck
#optional: limits search for user to this realm
#REALM=example.net
#optional: only use this UserIdResolver
#RESCONF=flat_file
#optional: comment out if everything seems to work fine
Debug=True
#optional: use this, if you have selfsigned certificates, otherwise comment out
SSL_CHECK=False
[root@yourserver:~]# 
```

### remove the existing sites

```
[root@yourserver:~]#  rm /etc/raddb/sites-enabled/default
rm: remove symbolic link ‘/etc/raddb/sites-enabled/default’? y
[root@yourserver:~]#  rm /etc/raddb/sites-enabled/inner-tunnel
rm: remove symbolic link ‘/etc/raddb/sites-enabled/inner-tunnel’? y
[root@yourserver:~]# 
```

### create a new default site

```
[root@yourserver:~]# /etc/raddb/sites-enabled/default
server default {

    listen {
        type = auth
        ipaddr = *
        port = 0
        limit {
            max_connections = 16
            lifetime = 0
            idle_timeout = 30
        }
    }

    listen {
        ipaddr = *
        port = 0
        type = acct
        limit {
        }
    }

    authorize {
        preprocess
        digest
        suffix
        ntdomain
        files
        expiration
        logintime
        pap
        update control {
            Auth-Type := Perl
        }
    }

    authenticate {
        Auth-Type Perl {
            perl
        }
        digest
    }

    preacct {
        suffix
        files
    }

    accounting {
        detail
    }

    session {
    }

    post-auth {
    }

    pre-proxy {
    }

    post-proxy {
    }

}
[root@yourserver:~]# 
```

### test

```
[root@yourserver:~]# radiusd -X
FreeRADIUS Version 3.0.13
Copyright (C) 1999-2017 The FreeRADIUS server project and contributors
There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE
You may redistribute copies of FreeRADIUS under the terms of the
GNU General Public License
For more information about these matters, see the file named COPYRIGHT
Starting - reading configuration files ...
...
...
Listening on auth address * port 1812 bound to server default
Listening on acct address * port 1813 bound to server default
Listening on proxy address * port 42847
Ready to process requests
(0) Received Access-Request Id 81 from 10.75.1.9:49450 to 10.52.10.10:1812 length 68
(0)   User-Name = "username@yourdomain"
(0)   User-Password = "339078"
(0) # Executing section authorize from file /etc/raddb/sites-enabled/default
(0)   authorize {
(0)     [preprocess] = ok
(0)     [digest] = noop
(0) suffix: Checking for suffix after "@"
(0) suffix: Looking up realm "yourdomain" for User-Name = "username@yourdomain"
(0) suffix: No such realm "yourdomain"
(0)     [suffix] = noop
(0) ntdomain: Checking for prefix before "\"
(0) ntdomain: No '\' in User-Name = "username@yourdomain", looking up realm NULL
(0) ntdomain: No such realm "NULL"
(0)     [ntdomain] = noop
(0)     [files] = noop
(0)     [expiration] = noop
(0)     [logintime] = noop
(0) pap: WARNING: No "known good" password found for the user.  Not setting Auth-Type
(0) pap: WARNING: Authentication will fail unless a "known good" password is available
(0)     [pap] = noop
(0)     update control {
(0)       Auth-Type := Perl
(0)     } # update control = noop
(0)   } # authorize = ok
(0) Found Auth-Type = Perl
(0) # Executing group from file /etc/raddb/sites-enabled/default
(0)   Auth-Type Perl {
(0) perl:   $RAD_REQUEST{'User-Name'} = &request:User-Name -> 'username@yourdomain'
(0) perl:   $RAD_REQUEST{'User-Password'} = &request:User-Password -> '339078'
(0) perl:   $RAD_REQUEST{'NAS-IP-Address'} = &request:NAS-IP-Address -> '10.75.1.9'
(0) perl:   $RAD_REQUEST{'Event-Timestamp'} = &request:Event-Timestamp -> 'Jan 25 2018 10:58:43 AEST'
(0) perl:   $RAD_CHECK{'Auth-Type'} = &control:Auth-Type -> 'Perl'
(0) perl:   $RAD_CONFIG{'Auth-Type'} = &control:Auth-Type -> 'Perl'
rlm_perl: Config File /etc/raddb/mods-config/perl/radius_linotp.ini found!
rlm_perl: Default URL https://yourlinotpserver/validate/simplecheck
rlm_perl: Auth-Type: Perl
rlm_perl: Url: https://yourlinotpserver/validate/simplecheck
rlm_perl: User: username@yourdomain
rlm_perl: urlparam client
rlm_perl: urlparam pass
rlm_perl: urlparam user
rlm_perl: LinOTP access granted
rlm_perl: return RLM_MODULE_OK
(0) perl: &request:User-Name = $RAD_REQUEST{'User-Name'} -> 'username@yourdomain'
(0) perl: &request:Event-Timestamp = $RAD_REQUEST{'Event-Timestamp'} -> 'Jan 25 2018 10:58:43 AEST'
(0) perl: &request:User-Password = $RAD_REQUEST{'User-Password'} -> '339078'
(0) perl: &request:NAS-IP-Address = $RAD_REQUEST{'NAS-IP-Address'} -> '10.75.1.9'
(0) perl: &reply:Reply-Message = $RAD_REPLY{'Reply-Message'} -> 'LinOTP access granted'
(0) perl: &control:Auth-Type = $RAD_CHECK{'Auth-Type'} -> 'Perl'
(0)     [perl] = ok
(0)   } # Auth-Type Perl = ok
(0) Sent Access-Accept Id 81 from 10.52.10.10:1812 to 10.75.1.9:49450 length 0
(0)   Reply-Message = "LinOTP access granted"
(0) Finished request
Waking up in 4.9 seconds.
(0) Cleaning up request packet ID 81 with timestamp +10
Ready to process requests
```



### other problems

* initially freeradius complained about the EAP module - removing it resolved the problem.

```
[root@yourserver:~]# radiusd -X
FreeRADIUS Version 3.0.13
Copyright (C) 1999-2017 The FreeRADIUS server project and contributors
There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE
You may redistribute copies of FreeRADIUS under the terms of the
GNU General Public License
For more information about these matters, see the file named COPYRIGHT
Starting - reading configuration files ...
including dictionary file /usr/share/freeradius/dictionary
including dictionary file /usr/share/freeradius/dictionary.dhcp
...
...
...
rlm_detail (auth_log): 'User-Password' suppressed, will not appear in detail output
# Instantiating module "reply_log" from file /etc/raddb/mods-enabled/detail.log
# Instantiating module "pre_proxy_log" from file /etc/raddb/mods-enabled/detail.log
# Instantiating module "post_proxy_log" from file /etc/raddb/mods-enabled/detail.log
# Instantiating module "eap" from file /etc/raddb/mods-enabled/eap
/etc/raddb/mods-enabled/eap[14]: Failed to find 'Auth-Type EAP' section.  Cannot authenticate users.
/etc/raddb/mods-enabled/eap[14]: Instantiation failed for module "eap"
[root@yourserver:~]#


[root@yourserver:~]# rm /etc/raddb/mods-enabled/eap
rm: remove symbolic link ‘/etc/raddb/mods-enabled/eap’? y
[root@yourserver:~]#
```
