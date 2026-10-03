# ДЗ №1 Изучение средств защиты ОС GNU/Linux

**Бухгалтерский шифр**: 372694

## 1. Создание пользователя

Создаём группу студентов:

```shell
$ sudo groupadd students
```

Создаём пользователя *user1* с домашней директорией, с *UID* 1234 и дополнительной группой *students*. Задаём пароль.

```shell
$ sudo useradd -m -u 1234 -G students user1
$ sudo passwd user1
```

Настройка необходимости изменения пароля пользователя *user1* каждые три месяца:

```shell
$ sudo chage -d 0 -m 0 -M 90 user1
```

Проверка:

```shell
$ groups user1
user1 : user1 students

$ sudo chage -l user1
Last password change                                 : password must be changed
Password expires                                     : password must be changed
Password inactive                                    : password must be changed
Account expires                                      : never
Minimum number of days between password change       : 0
Maximum number of days between password change       : 90
Number of days of warning before password expires    : 7

$ sudo grep user1 /etc/shadow
user1:$y$j9T$uwnWZ/YwhVUk6qNc.OgPF1$wZhu4dYrjb.jY5sqg2Bb5eGDKpizUVP8zqBHMA40MVA:0:0:90:7:::
```

## 2. Мониторинг файлов и процессов

Поиск всех файлов с установленным битом set-UID:

```shell
$ find / -perm -4000 2>/dev/null
/usr/sbin/mount.nfs
/usr/sbin/grub2-set-bootflag
/usr/sbin/unix_chkpwd
/usr/sbin/userhelper
/usr/sbin/pam_timestamp_check
/usr/bin/umount
/usr/bin/sudo
/usr/bin/gpasswd
/usr/bin/crontab
/usr/bin/newgrp
/usr/bin/loginfo.sh
/usr/bin/chage
/usr/bin/su
/usr/bin/at
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/mount
/usr/bin/chsh
/usr/bin/pkexec
/usr/lib/polkit-1/polkit-agent-helper-1
```

Поиск всех процессов, у которых эффективный UID равен 0, а реальный UID относится к обычным пользователям:

```shell
$ ps -eo euid,ruid,pid,comm | awk '$1 == 0 && $2 != 0'
    0  1000    1430 passwd
```

## 3. Изучение механизма set-UID

Команда, требующая права *root*:

```shell
$ cat /etc/shadow
cat: /etc/shadow: Permission denied
```

Копируем утилиту в текущую директорию, устанавливаем в качестве владельца пользователя *root* и выставляем set-UID:

```shell
$ which cat
/usr/bin/cat

$ cp /usr/bin/cat .
$ ls -l
total 36
-rwxr-xr-x. 1 max max 36568 Oct  3 15:33 cat

$ sudo chown root cat
$ sudo chmod u+s cat

$ ls -l
total 36
-rwsr-xr-x. 1 root max 36568 Oct  3 15:33 cat
```

Проверка работы утилиты:

```shell
$ ./cat /etc/shadow
root:!::0:99999:7:::
bin:*:20047:0:99999:7:::
daemon:*:20047:0:99999:7:::
...
```

## 4. Изучение механизма привилегий

Для изменения владельца требуются права *root*:

```shell
$ touch tmp_file
$ ls -l | grep tmp_file
-rw-r--r--. 1 max  max     0 Oct  3 15:41 tmp_file

$ chown user1 tmp_file 
chown: changing ownership of 'tmp_file': Operation not permitted
```

Копируем утилиту в текущую директорию, устанавливаем привилегию для изменения *UID* и *GID* файлов:

```shell
$ which chown
/usr/bin/chown

$ cp /usr/bin/chown .
$ ls -l | grep chown
-rwxr-xr-x. 1 max  max 65840 Oct  3 15:44 chown

$ sudo setcap "cap_chown=ep" chown
$ getcap ./chown 
./chown cap_chown=ep
```

Проверка работы утилиты:

```shell
$ ./chown user1 tmp_file
$ ls -l | grep tmp_file
-rw-r--r--. 1 user1 max     0 Oct  3 15:41 tmp_file
```

## 5. Изучение механизма sudo 

Проверяем, что по умолчанию пользовать *user1* не может менять время:

```shell
$ date
Sat Oct  3 04:18:16 PM MSK 2026

$ date -s "2026-10-03 06:00:00"
date: cannot set date: Operation not permitted
```

Добавляем запись в */etc/sudoers* (используя утилиту *visudo*):

```
user1 ALL(ALL) NOPASSWD: /bin/date
```

Проверяем, что мы можем изменить время с помощью *sudo*:

```shell
$ sudo date -s "2026-10-03 06:00:00"
Sat Oct  3 06:00:00 AM MSK 2026

$ date
Sat Oct  3 06:00:02 AM MSK 2026
```

Проверяем, что другие команды через *sudo* не работают:

```shell
$ sudo cat /etc/shadow
Sorry, user user1 is not allowed to execute '/bin/cat /etc/shadow' as root on localhost.localdomain.
```

