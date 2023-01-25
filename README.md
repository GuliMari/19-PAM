# Домашнее задание: PAM
Запретить всем пользователям, кроме группы admin логин в выходные (суббота и воскресенье), без учета праздников

##Выполнение
Разрешаем вход по паролю в `/etc/ssh/sshd_config`.
Создаем группу `admin` и 2 пользователей, один из которых входит в эту группу:
```bash
[root@localhost ~]# groupadd admin
[root@localhost ~]# useradd user1
[root@localhost ~]# useradd -G admin user2
[root@localhost ~]# id user1
uid=1001(user1) gid=1002(user1) groups=1002(user1)
[root@localhost ~]# id user2
uid=1002(user2) gid=1003(user2) groups=1003(user2),1001(admin)
```
Воспользуемся модулем `pam_exec`:
```bash
[root@localhost ~]# cat /etc/pam.d/sshd 
#%PAM-1.0
...
account    required     pam_exec.so /usr/local/bin/test_login.sh
...
```
Пишем скрипт:
```bash
#!/bin/bash

user_group=$(id $PAM_USER | grep -ow admin)
day=$(date +%u)

if [[ $day -gt 5 && $user_group == "admin" ]]; then
    exit 0
  else
    exit 1
fi
```
Для проверки меняем дату на удаленном хосте, чтобы был выходной день и пробуем подключиться пользователями по ssh:
```bash
tw4@tw4-mint:~$ ssh user1@192.168.56.77
user1@192.168.56.77's password: 
/usr/local/bin/test_login.sh failed: exit code 1
Connection closed by 192.168.56.77 port 22
tw4@tw4-mint:~$
tw4@tw4-mint:~$ ssh user2@192.168.56.77
user2@192.168.56.77's password: 
Last failed login: Wed Jan 25 17:05:31 UTC 2023 from 192.168.56.1 on ssh:notty
There was 1 failed login attempt since the last successful login.
Last login: Wed Jan 25 17:03:00 2023 from 192.168.56.1
[user2@localhost ~]$
```



