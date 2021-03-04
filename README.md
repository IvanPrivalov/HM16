## Задание
Настраиваем центральный сервер для сбора логов
в вагранте поднимаем 2 машины web и log. 
На web поднимаем nginx. 
На log настраиваем центральный лог сервер на любой системе на выбор: 
- journald 
- rsyslog 
- elk 
1 - Настраиваем аудит следящий за изменением конфигов нжинкса.
2 - Логи аудита должны также уходить на удаленную систему.
3 - Все критичные логи с web должны собираться и локально и удаленно.
4 - Все логи с nginx должны уходить на удаленный сервер (локально только критичные).



## Выполнение задания

**1. Настроить аудит следящий на изменением конфигов нжинкса.**

Добавляем правила:

```
[root@web ~]# cat /etc/audit/rules.d/audit.rules 
## First rule - delete all
-D

## Increase the buffers to survive stress events.
## Make this bigger for busy systems
-b 8192

## Set failure mode to syslog
-f 1

## Audit of nginx configuration files changes
-w /etc/nginx/nginx.conf -p wa -k nginx_conf
-w /etc/nginx/default.d/ -p wa -k nginx_conf
```

Рестарт auditd:
```
[root@web ~]# service auditd restart
Stopping logging:                                          [  OK  ]
Redirecting start to /bin/systemctl start auditd.service
```

Проверяем правила:
```
[root@web ~]# auditctl -l
-w /etc/nginx/nginx.conf -p wa -k nginx_conf
-w /etc/nginx/default.d -p wa -k nginx_conf
```

**2. Логи аудита должны также уходить на удаленную систему.**

Установим пакет `audispd-plugins` на web и log. 
Изменим следующие опции в конфиг файлах:

- в файле `/etc/audisp/audisp-remote.conf`:
```
	remote_server = 192.168.10.20
	port = 514
```
- в файле `/etc/audisp/plugins.d/au-remote.conf`:
```
	active = yes
```
- в файле /etc/audit/auditd.conf:
```
	write_logs = no # чтобы логи не писались локально, а только на удаленный сервер
```
На самом сервере log разрешить прием логов в файле `/etc/rsyslog.conf`:
```
	# Provides UDP syslog reception
	$ModLoad imudp
	$UDPServerRun 514

	# Provides TCP syslog reception
	$ModLoad imtcp
	$InputTCPServerRun 514
```

**3. Все критичные логи с web должны собираться и локально и удаленно.**

В `/etc/rsyslog.conf` в раздел `#### RULES ####` добавляем следующие строки для отправки всех критических сообщений на сервер log:
```
*.crit action(type="omfwd" target="192.168.10.20" port="514" protocol="tcp"
              action.resumeRetryCount="100"
              queue.type="linkedList" queue.size="10000")
```

**4. Все логи с nginx должны уходить на удаленный сервер (локально только критичные).**

В конфиг nginx добавляем следующие строки:
```
error_log /var/log/nginx/error.log crit;
error_log syslog:server=192.168.10.20:514,tag=nginx_error;
...
access_log syslog:server=192.168.10.20:514,tag=nginx_access;

```

На сервере log для разделения логов `nginx_access` и `nginx_error` по отдельным директориям в `/etc/rsyslog.conf` доавлены следующие правила:
```
if ($hostname == 'web') and ($programname == 'nginx_access') then {
    action(type="omfile" file="/var/log/rsyslog/web/nginx_access.log")
    stop
}

if ($hostname == 'web') and ($programname == 'nginx_error') then {
    action(type="omfile" file="/var/log/rsyslog/web/nginx_error.log")
    stop
}
```

### Проверка задания
Необходимо скопировать файлы и запустить vagrant
```
vagrant up
```

Запуститься 2 сервреа web и log

На web логи хранятся по пути:

```
[root@web nginx]# pwd
/var/log/nginx
[root@web nginx]# ll
total 0
-rw-r--r--. 1 root root 0 Mar  4 13:02 access.log
-rw-r--r--. 1 root root 0 Mar  4 13:02 error.log
```

На log логи хранятся по пути:
```
[root@log web]# pwd
/var/log/rsyslog/web
[root@log web]# ll
total 8
-rw-------. 1 root root 2623 Mar  4 13:37 nginx_access.log
-rw-------. 1 root root  584 Mar  4 13:37 nginx_error.log
```

Изменим порт нжинкса в ./web/vars/main.yml перезапустим playbook:
```
nsible-playbook logging.yml
```

Проверим логи auditd:
```
[root@log ~]# ausearch -i -k nginx_conf
```

<details>
  <summary>Получим следующий вывод:</summary>
```
[root@log ~]# ausearch -i -k nginx_conf
----
node=web type=CONFIG_CHANGE msg=audit(03/04/21 13:03:13.465:1692) : auid=unset ses=unset subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key=nginx_conf list=exit res=yes 
----
node=web type=CONFIG_CHANGE msg=audit(03/04/21 13:03:13.465:1693) : auid=unset ses=unset subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key=nginx_conf list=exit res=yes 
[root@log ~]# ausearch -i -k nginx_conf
----
node=web type=CONFIG_CHANGE msg=audit(03/04/21 13:03:13.465:1692) : auid=unset ses=unset subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key=nginx_conf list=exit res=yes 
----
node=web type=CONFIG_CHANGE msg=audit(03/04/21 13:03:13.465:1693) : auid=unset ses=unset subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key=nginx_conf list=exit res=yes 
----
node=web type=CONFIG_CHANGE msg=audit(03/04/21 13:19:55.715:2051) : auid=vagrant ses=7 op=updated_rules path=/etc/nginx/nginx.conf key=nginx_conf list=exit res=yes 
----
node=web type=PROCTITLE msg=audit(03/04/21 13:19:55.715:2052) : proctitle=/usr/bin/python /home/vagrant/.ansible/tmp/ansible-tmp-1614853190.3035228-171281241278666/AnsiballZ_copy.py 
node=web type=PATH msg=audit(03/04/21 13:19:55.715:2052) : item=4 name=/etc/nginx/nginx.conf inode=12131 dev=08:01 mode=file,644 ouid=root ogid=root rdev=00:00 obj=unconfined_u:object_r:user_home_t:s0 objtype=CREATE cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 
node=web type=PATH msg=audit(03/04/21 13:19:55.715:2052) : item=3 name=/etc/nginx/nginx.conf inode=67559222 dev=08:01 mode=file,644 ouid=root ogid=root rdev=00:00 obj=system_u:object_r:httpd_config_t:s0 objtype=DELETE cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 
node=web type=PATH msg=audit(03/04/21 13:19:55.715:2052) : item=2 name=/home/vagrant/.ansible/tmp/ansible-tmp-1614853190.3035228-171281241278666/source inode=12131 dev=08:01 mode=file,644 ouid=root ogid=root rdev=00:00 obj=unconfined_u:object_r:user_home_t:s0 objtype=DELETE cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 
node=web type=PATH msg=audit(03/04/21 13:19:55.715:2052) : item=1 name=/etc/nginx/ inode=67551136 dev=08:01 mode=dir,755 ouid=root ogid=root rdev=00:00 obj=system_u:object_r:httpd_config_t:s0 objtype=PARENT cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 
node=web type=PATH msg=audit(03/04/21 13:19:55.715:2052) : item=0 name=/home/vagrant/.ansible/tmp/ansible-tmp-1614853190.3035228-171281241278666/ inode=12125 dev=08:01 mode=dir,700 ouid=vagrant ogid=vagrant rdev=00:00 obj=unconfined_u:object_r:user_home_t:s0 objtype=PARENT cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 
node=web type=CWD msg=audit(03/04/21 13:19:55.715:2052) :  cwd=/home/vagrant 
node=web type=SYSCALL msg=audit(03/04/21 13:19:55.715:2052) : arch=x86_64 syscall=rename success=yes exit=0 a0=0x278b960 a1=0x27c81c0 a2=0x7ff5e265e1c8 a3=0x0 items=5 ppid=5415 pid=5416 auid=vagrant uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root tty=pts1 ses=7 comm=python exe=/usr/bin/python2.7 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key=nginx_conf 
----
node=web type=PROCTITLE msg=audit(03/04/21 13:19:55.715:2053) : proctitle=/usr/bin/python /home/vagrant/.ansible/tmp/ansible-tmp-1614853190.3035228-171281241278666/AnsiballZ_copy.py 
node=web type=PATH msg=audit(03/04/21 13:19:55.715:2053) : item=0 name=/etc/nginx/nginx.conf inode=12131 dev=08:01 mode=file,644 ouid=root ogid=root rdev=00:00 obj=unconfined_u:object_r:user_home_t:s0 objtype=NORMAL cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 
node=web type=CWD msg=audit(03/04/21 13:19:55.715:2053) :  cwd=/home/vagrant 
node=web type=SYSCALL msg=audit(03/04/21 13:19:55.715:2053) : arch=x86_64 syscall=lsetxattr success=yes exit=0 a0=0x7ff5dd3656d4 a1=0x7ff5de352f6a a2=0x27158f0 a3=0x24 items=1 ppid=5415 pid=5416 auid=vagrant uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root tty=pts1 ses=7 comm=python exe=/usr/bin/python2.7 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key=nginx_conf 
[root@log ~]# ausearch -i -k nginx_conf
----
node=web type=CONFIG_CHANGE msg=audit(03/04/21 13:03:13.465:1692) : auid=unset ses=unset subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key=nginx_conf list=exit res=yes 
----
node=web type=CONFIG_CHANGE msg=audit(03/04/21 13:03:13.465:1693) : auid=unset ses=unset subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key=nginx_conf list=exit res=yes 
----
node=web type=CONFIG_CHANGE msg=audit(03/04/21 13:19:55.715:2051) : auid=vagrant ses=7 op=updated_rules path=/etc/nginx/nginx.conf key=nginx_conf list=exit res=yes 
----
node=web type=PROCTITLE msg=audit(03/04/21 13:19:55.715:2052) : proctitle=/usr/bin/python /home/vagrant/.ansible/tmp/ansible-tmp-1614853190.3035228-171281241278666/AnsiballZ_copy.py 
node=web type=PATH msg=audit(03/04/21 13:19:55.715:2052) : item=4 name=/etc/nginx/nginx.conf inode=12131 dev=08:01 mode=file,644 ouid=root ogid=root rdev=00:00 obj=unconfined_u:object_r:user_home_t:s0 objtype=CREATE cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 
node=web type=PATH msg=audit(03/04/21 13:19:55.715:2052) : item=3 name=/etc/nginx/nginx.conf inode=67559222 dev=08:01 mode=file,644 ouid=root ogid=root rdev=00:00 obj=system_u:object_r:httpd_config_t:s0 objtype=DELETE cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 
node=web type=PATH msg=audit(03/04/21 13:19:55.715:2052) : item=2 name=/home/vagrant/.ansible/tmp/ansible-tmp-1614853190.3035228-171281241278666/source inode=12131 dev=08:01 mode=file,644 ouid=root ogid=root rdev=00:00 obj=unconfined_u:object_r:user_home_t:s0 objtype=DELETE cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 
node=web type=PATH msg=audit(03/04/21 13:19:55.715:2052) : item=1 name=/etc/nginx/ inode=67551136 dev=08:01 mode=dir,755 ouid=root ogid=root rdev=00:00 obj=system_u:object_r:httpd_config_t:s0 objtype=PARENT cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 
node=web type=PATH msg=audit(03/04/21 13:19:55.715:2052) : item=0 name=/home/vagrant/.ansible/tmp/ansible-tmp-1614853190.3035228-171281241278666/ inode=12125 dev=08:01 mode=dir,700 ouid=vagrant ogid=vagrant rdev=00:00 obj=unconfined_u:object_r:user_home_t:s0 objtype=PARENT cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 
node=web type=CWD msg=audit(03/04/21 13:19:55.715:2052) :  cwd=/home/vagrant 
node=web type=SYSCALL msg=audit(03/04/21 13:19:55.715:2052) : arch=x86_64 syscall=rename success=yes exit=0 a0=0x278b960 a1=0x27c81c0 a2=0x7ff5e265e1c8 a3=0x0 items=5 ppid=5415 pid=5416 auid=vagrant uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root tty=pts1 ses=7 comm=python exe=/usr/bin/python2.7 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key=nginx_conf 
----
node=web type=PROCTITLE msg=audit(03/04/21 13:19:55.715:2053) : proctitle=/usr/bin/python /home/vagrant/.ansible/tmp/ansible-tmp-1614853190.3035228-171281241278666/AnsiballZ_copy.py 
node=web type=PATH msg=audit(03/04/21 13:19:55.715:2053) : item=0 name=/etc/nginx/nginx.conf inode=12131 dev=08:01 mode=file,644 ouid=root ogid=root rdev=00:00 obj=unconfined_u:object_r:user_home_t:s0 objtype=NORMAL cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 
node=web type=CWD msg=audit(03/04/21 13:19:55.715:2053) :  cwd=/home/vagrant 
node=web type=SYSCALL msg=audit(03/04/21 13:19:55.715:2053) : arch=x86_64 syscall=lsetxattr success=yes exit=0 a0=0x7ff5dd3656d4 a1=0x7ff5de352f6a a2=0x27158f0 a3=0x24 items=1 ppid=5415 pid=5416 auid=vagrant uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root tty=pts1 ses=7 comm=python exe=/usr/bin/python2.7 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key=nginx_conf 
----
node=web type=CONFIG_CHANGE msg=audit(03/04/21 13:24:48.152:2674) : auid=vagrant ses=8 op=updated_rules path=/etc/nginx/nginx.conf key=nginx_conf list=exit res=yes 
----
node=web type=PROCTITLE msg=audit(03/04/21 13:24:48.152:2675) : proctitle=/usr/bin/python /home/vagrant/.ansible/tmp/ansible-tmp-1614853483.113401-199003865889604/AnsiballZ_copy.py 
node=web type=PATH msg=audit(03/04/21 13:24:48.152:2675) : item=4 name=/etc/nginx/nginx.conf inode=101432955 dev=08:01 mode=file,644 ouid=root ogid=root rdev=00:00 obj=unconfined_u:object_r:user_home_t:s0 objtype=CREATE cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 
node=web type=PATH msg=audit(03/04/21 13:24:48.152:2675) : item=3 name=/etc/nginx/nginx.conf inode=12131 dev=08:01 mode=file,644 ouid=root ogid=root rdev=00:00 obj=system_u:object_r:httpd_config_t:s0 objtype=DELETE cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 
node=web type=PATH msg=audit(03/04/21 13:24:48.152:2675) : item=2 name=/home/vagrant/.ansible/tmp/ansible-tmp-1614853483.113401-199003865889604/source inode=101432955 dev=08:01 mode=file,644 ouid=root ogid=root rdev=00:00 obj=unconfined_u:object_r:user_home_t:s0 objtype=DELETE cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 
node=web type=PATH msg=audit(03/04/21 13:24:48.152:2675) : item=1 name=/etc/nginx/ inode=67551136 dev=08:01 mode=dir,755 ouid=root ogid=root rdev=00:00 obj=system_u:object_r:httpd_config_t:s0 objtype=PARENT cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 
node=web type=PATH msg=audit(03/04/21 13:24:48.152:2675) : item=0 name=/home/vagrant/.ansible/tmp/ansible-tmp-1614853483.113401-199003865889604/ inode=101432931 dev=08:01 mode=dir,700 ouid=vagrant ogid=vagrant rdev=00:00 obj=unconfined_u:object_r:user_home_t:s0 objtype=PARENT cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 
node=web type=CWD msg=audit(03/04/21 13:24:48.152:2675) :  cwd=/home/vagrant 
node=web type=SYSCALL msg=audit(03/04/21 13:24:48.152:2675) : arch=x86_64 syscall=rename success=yes exit=0 a0=0x17af960 a1=0x17c0f20 a2=0x7f21ba0e81c8 a3=0x0 items=5 ppid=6509 pid=6510 auid=vagrant uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root tty=pts1 ses=8 comm=python exe=/usr/bin/python2.7 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key=nginx_conf 
----
node=web type=PROCTITLE msg=audit(03/04/21 13:24:48.152:2676) : proctitle=/usr/bin/python /home/vagrant/.ansible/tmp/ansible-tmp-1614853483.113401-199003865889604/AnsiballZ_copy.py 
node=web type=PATH msg=audit(03/04/21 13:24:48.152:2676) : item=0 name=/etc/nginx/nginx.conf inode=101432955 dev=08:01 mode=file,644 ouid=root ogid=root rdev=00:00 obj=unconfined_u:object_r:user_home_t:s0 objtype=NORMAL cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 
node=web type=CWD msg=audit(03/04/21 13:24:48.152:2676) :  cwd=/home/vagrant 
node=web type=SYSCALL msg=audit(03/04/21 13:24:48.152:2676) : arch=x86_64 syscall=lsetxattr success=yes exit=0 a0=0x7f21b4def6d4 a1=0x7f21b5ddcf6a a2=0x17398f0 a3=0x24 items=1 ppid=6509 pid=6510 auid=vagrant uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root tty=pts1 ses=8 comm=python exe=/usr/bin/python2.7 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key=nginx_conf 

```

</details>




