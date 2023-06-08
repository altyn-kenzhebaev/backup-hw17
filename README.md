# Borg Backup Stand
Для выполнения этого действия требуется установить приложением git:
`git clone https://github.com/altyn-kenzhebaev/backup-hw17.git`
В текущей директории появится папка с именем репозитория. В данном случае backup-hw17. Ознакомимся с содержимым:
```
cd backup-hw17
ls -l
ansible
README.md
Vagrantfile
```
Здесь:
- ansible - папка с плэйбуками
- README.md - файл с данным руководством
- Vagrantfile - файл описывающий виртуальную инфраструктуру для `Vagrant`
Запускаем ВМ:
```
vagrant up
```
Стенд должен подняться с 2-мя виртуальными машиами. Дальнейшие работы будет проведены на сервере clint под пользователем root:
```
vagrant ssh client
sudo -i
```
## Логи процесса бэкапа
В целях логирования процесса было включено логирование в отдельный файл /var/log/borg/borg.log
Данные настройки логирования (файл /etc/borg/logging.conf) были применены в файле systemd:
```
$ cat /etc/systemd/system/borg-backup.service
[Unit]
Description=Borg Backup

[Service]
Type=oneshot
Environment="BORG_PASSPHRASE=vagrant"
Environment="BORG_LOGGING_CONF=/etc/borg/logging.conf"      # Включение логирования
Environment=REPO=borg@192.168.50.10:/var/backup/
Environment=BACKUP_TARGET=/etc

ExecStart=/bin/borg create \
    --stats                \
    ${REPO}::etc-{now:%%Y-%%m-%%d_%%H:%%M:%%S} ${BACKUP_TARGET}

ExecStart=/bin/borg check ${REPO}

ExecStart=/bin/borg prune \
    --keep-daily  90      \
    --keep-monthly 12     \
    --keep-yearly  1       \
    ${REPO}
$ cat /etc/borg/logging.conf
[loggers]
keys=root

[handlers]
keys=logfile

[formatters]
keys=logfile

[logger_root]
level=DEBUG
handlers=logfile

[handler_logfile]
class=FileHandler
level=DEBUG
formatter=logfile
args=('/var/log/borg/borg.log', 'w')

[formatter_logfile]
format=%(asctime)s %(levelname)s %(message)s
datefmt=
class=logging.Formatter
```
Вывод логов:
```
$ cat /var/log/borg/borg.log 
2023-06-08 08:51:25,688 DEBUG using logging configuration read from "/etc/borg/logging.conf"
2023-06-08 08:51:25,775 DEBUG 33 self tests completed in 0.09 seconds
2023-06-08 08:51:25,776 DEBUG SSH command line: ['ssh', 'borg@192.168.50.10', 'borg', 'serve', '--debug']
2023-06-08 08:51:26,175 DEBUG Remote: using builtin fallback logging configuration
2023-06-08 08:51:26,259 DEBUG Remote: 33 self tests completed in 0.08 seconds
2023-06-08 08:51:26,260 DEBUG Remote: using builtin fallback logging configuration
2023-06-08 08:51:26,261 DEBUG Remote: Initialized logging system for JSON-based protocol
2023-06-08 08:51:26,263 DEBUG Remote: Resolving repository path b'/var/backup'
2023-06-08 08:51:26,263 DEBUG Remote: Resolved repository path to '/var/backup'
2023-06-08 08:51:26,266 DEBUG Remote: Verified integrity of /var/backup/index.9
2023-06-08 08:51:26,339 DEBUG TAM-verified manifest
2023-06-08 08:51:26,341 DEBUG security: read previous location 'ssh://borg@192.168.50.10/var/backup'
2023-06-08 08:51:26,342 DEBUG security: read manifest timestamp '2023-06-08T02:51:22.489679'
2023-06-08 08:51:26,342 DEBUG security: determined newest manifest timestamp as 2023-06-08T02:51:22.489679
2023-06-08 08:51:26,342 DEBUG security: repository checks ok, allowing access
2023-06-08 08:51:26,346 DEBUG Verified integrity of /root/.cache/borg/954bbf3360015dce812059ed9a70e02d91349ba3fe4085de5e378c47fb4c5bec/chunks
2023-06-08 08:51:26,346 DEBUG security: read previous location 'ssh://borg@192.168.50.10/var/backup'
2023-06-08 08:51:26,347 DEBUG security: read manifest timestamp '2023-06-08T02:51:22.489679'
2023-06-08 08:51:26,347 DEBUG security: determined newest manifest timestamp as 2023-06-08T02:51:22.489679
2023-06-08 08:51:26,347 DEBUG security: repository checks ok, allowing access
2023-06-08 08:51:26,348 DEBUG RemoteRepository: 220 B bytes sent, 2.59 kB bytes received, 5 messages sent
$ systemctl status borg-backup
○ borg-backup.service - Borg Backup
     Loaded: loaded (/etc/systemd/system/borg-backup.service; static)
     Active: inactive (dead) since Thu 2023-06-08 08:51:26 +06; 11min ago
TriggeredBy: ○ borg-backup.timer
    Process: 6747 ExecStart=/bin/borg create --stats ${REPO}::etc-{now:%Y-%m-%d_%H:%M:%S} ${BACKUP_TARGET} (code=exited, status=0/SUCCESS)
    Process: 6749 ExecStart=/bin/borg check ${REPO} (code=exited, status=0/SUCCESS)
    Process: 6751 ExecStart=/bin/borg prune --keep-daily 90 --keep-monthly 12 --keep-yearly 1 ${REPO} (code=exited, status=0/SUCCESS)
   Main PID: 6751 (code=exited, status=0/SUCCESS)
        CPU: 2.044s

Jun 08 08:51:19 client systemd[1]: Starting Borg Backup...
Jun 08 08:51:26 client systemd[1]: borg-backup.service: Deactivated successfully.
Jun 08 08:51:26 client systemd[1]: Finished Borg Backup.
Jun 08 08:51:26 client systemd[1]: borg-backup.service: Consumed 2.044s CPU time.
```
Также конечно был включен logrotate:
```
$ cat /etc/logrotate.d/borg
/var/log/borg/*.log {
    weekly
    rotate 4
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
}
```
## Описание процесса восстановления
Проверим выполнились ли бэкапы:
```
$ borg list borg@192.168.50.10:/var/backup/
Enter passphrase for key ssh://borg@192.168.50.10/var/backup: 
etc-2023-06-08_08:41:15              Thu, 2023-06-08 08:41:16 [eab9ec257b7ce37b11d8a000dea0f88194282afdb7870c07c436ede55385e3ca]
etc-2023-06-08_08:51:19              Thu, 2023-06-08 08:51:20 [d457f9c6c8f949bafd30184d451ec98b50995a7614e83d1a3ff3dd49c745ede8]
```
Проверим работает ли таймер:
```
$ systemctl list-timers
NEXT                        LEFT          LAST                        PASSED       UNIT                         ACTIVATES                     
Thu 2023-06-08 08:56:19 +06 3min 31s left -                           -            borg-backup.timer            borg-backup.service
Thu 2023-06-08 09:04:02 +06 11min left    -                           -            systemd-tmpfiles-clean.timer systemd-tmpfiles-clean.service
Thu 2023-06-08 09:21:28 +06 28min left    -                           -            dnf-makecache.timer          dnf-makecache.service
Fri 2023-06-09 00:00:00 +06 15h left      Thu 2023-06-08 08:49:17 +06 3min 30s ago logrotate.timer              logrotate.service
$ systemctl status borg-backup.timer
● borg-backup.timer - Borg Backup
     Loaded: loaded (/etc/systemd/system/borg-backup.timer; enabled; preset: disabled)
     Active: active (waiting) since Thu 2023-06-08 08:30:58 +06; 10min ago
      Until: Thu 2023-06-08 08:30:58 +06; 10min ago
    Trigger: Thu 2023-06-08 08:46:15 +06; 4min 48s left
```
Если все в порядке, проводим процесс восстановления, останавливаем таймер, удаляем к примеру папку с /etc/logrotate.d, и восстанавливаем с помощью borg:
```
[root@client /]# rm -rf /etc/logrotate.d/
[root@client /]# borg extract borg@192.168.50.10:/var/backup/::etc-2021-10-15_23:00:15 etc/logrotate.d/
Enter passphrase for key ssh://borg@192.168.50.10/var/backup: 
Archive etc-2021-10-15_23:00:15 does not exist
[root@client /]# 
[root@client /]# borg extract borg@192.168.50.10:/var/backup/::etc-2023-06-08_08:51:19 etc/logrotate.d/
Enter passphrase for key ssh://borg@192.168.50.10/var/backup: 
```
Проверяем, успешно, ли восстановлена директория:
```
[root@client /]# ls -l /etc/logrotate.d/
total 36
-rw-r--r--. 1 root root 124 Jun  8 08:51 borg
-rw-r--r--. 1 root root 130 Oct 14  2019 btmp
-rw-r--r--. 1 root root 160 Aug 29  2022 chrony
-rw-r--r--. 1 root root  88 Sep  9  2022 dnf
-rw-r--r--. 1 root root  93 Apr  7 07:13 firewalld
-rw-r--r--. 1 root root 162 May  9 16:13 kvm_stat
-rw-r--r--. 1 root root 226 May 10 13:44 rsyslog
-rw-r--r--. 1 root root 237 Apr 21 19:18 sssd
-rw-r--r--. 1 root root 145 Oct 14  2019 wtmp
```