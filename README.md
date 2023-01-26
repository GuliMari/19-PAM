# Домашнее задание: PAM
## Запретить всем пользователям, кроме группы admin логин в выходные (суббота и воскресенье), без учета праздников

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

## Дать конкретному пользователю права работать с докером и возможность рестартить докер сервис

Создаем пользователя `nonroot` и даем ему права на работу с Docker согласно инструкции на официальном сайте:
```bash
tw4@tw4-mint:~/1$ sudo useradd -m nonroot
w4@tw4-mint:~/1$ sudo passwd nonroot 
New password: 
Retype new password: 
passwd: password updated successfully
tw4@tw4-mint:~/1$ sudo usermod -aG docker nonroot
tw4@tw4-mint:~/1$ id nonroot
uid=1001(nonroot) gid=1001(nonroot) groups=1001(nonroot),999(docker)
tw4@tw4-mint:~/1$ newgrp docker
```
Заходим под `nonroot` и проверяем:
```bash
tw4@tw4-mint:~/1$ su - nonroot 
Password: 
$ docker run -d --rm nginx
9b0c810ccabdf54df75d881b88f1e58910b7bd6f47cb18a187dec6fe239d0999
$ sudo systemctl restart docker
[sudo] password for nonroot: 
sudo: a password is required
$ exit
```
Пользователь может запускать контейнеры, но не перезагружать.
Первый вариант решения проблемы - добавить соответствующие разрешения в `/etc/sudoers.d`:
```bash
tw4@tw4-mint:~/1$ sudo visudo  /etc/sudoers.d/nonroot 
tw4@tw4-mint:~/1$ sudo cat /etc/sudoers.d/nonroot 
                  
nonroot ALL= NOPASSWD:/usr/bin/systemctl restart docker
```
Снова заходим под пользователем и перезапускаем Docker:
```bash
tw4@tw4-mint:~/1$ su - nonroot
Password: 
$ sudo systemctl restart docker
$ 
```

Второй способ - использовать правила `polkit`.
В `/etc/polkit-1/rules.d` создаем правило:
```
polkit.addRule(function(action, subject) {
    if (action.id == "org.freedesktop.systemd1.manage-units") &&
        action.lookup("unit") == "docker.service") &&
        action.lookup("verb") == "restart") &&
        subject.user == "nonroot"
     {
        return polkit.Result.YES;
      }
});
```


