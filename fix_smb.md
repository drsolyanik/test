## Шаг 1. Точная локализация через strace (Самый быстрый способ)
Чтобы не гадать, на каком именно системном вызове умирает рабочий процесс при попытке подключения, отследите системные вызовы основного процесса smbd:

Узнайте PID главного процесса smbd:

```
pidof smbd
```

Подключитесь утилитой strace с отслеживанием всех дочерних процессов (-f / -ff):

```
strace -ff -s 512 -p <PID_smbd> -o /tmp/smbd_strace.log
```

В другом терминале выполните попытку подключения: smbclient -L 127.0.0.1 -N.

Завершите strace (Ctrl+C) и проверьте созданные файлы в /tmp/smbd_strace.log.*:

```
grep -E 'EACCES|EPERM|SIGSEGV|exit_group' /tmp/smbd_strace.log*
```

Если там виден запуск clone() / fork(), а затем сразу exit_group() или падение при открытии /etc/pam.d/, /etc/parsec/ или сокетов /run/parsec/ — проблема зафиксирована в Astra-специфичном окружении.

## Шаг 2. Проверка конфликта smbd.socket и smbd.service
В Astra Linux 1.8 по умолчанию используется Socket Activation от systemd. Если одновременно запущены и сокет, и сам сервис, порт 445/139 слушает systemd, передает файловый дескриптор в smbd, но тот не может корректно обработать соединение:

Проверьте статус сокетов:

```
systemctl status smbd.socket nmbd.socket
```

Если они активны, отключите их и оставьте только классические сервисы:

```
systemctl stop smbd.socket nmbd.socket
systemctl disable smbd.socket nmbd.socket
systemctl restart smbd nmbd
```

## Шаг 3. Проверка привилегий и Capabilities исполняемых файлов
При сбое astra-modeswitch с бинарников Samba могли слететь POSIX Capabilities (привилегии ядра, позволяющие smbd менять UID/GID и работать с сокетами от имени root при сниженном УЦ):

Проверьте наличиe capabilities на исполняемых файлах:

```
getcap /usr/sbin/smbd /usr/sbin/nmbd
```

Проверьте уровень целостности самих бинарников:

```
getilevel /usr/sbin/smbd /usr/sbin/nmbd
```

Уровень должен быть равен 0.

Если атрибуты сбились или выходы пустые, восстановите базовый уровень целостности бинарных файлов и библиотек Samba:

```
chilevel 0 /usr/sbin/smbd /usr/sbin/nmbd
chilevel -R 0 /usr/lib/x86_64-linux-gnu/samba/
```

## Шаг 4. Проверка PAM-стека (модулей авторизации)
Samba в Astra Linux тесно связана с PAM-модулями (pam_parsec.so, pam_astra.so). Если конфигурация PAM или права на доступ к конфигурации /etc/pam.d/samba нарушены, smbd аварийно завершает рабочий поток:

Проверьте права на конфигурационные файлы PAM:

```
ls -la /etc/pam.d/samba*
getilevel /etc/pam.d/samba
```

Временно проверьте локальную авторизацию через smbclient, отключив PAM-проверку в /etc/samba/smb.conf (только для теста):

```
[global]
obey pam restrictions = no
```

После чего сделайте systemctl reload smbd и проверьте подключение.

## Шаг 5. Чистая переустановка пакетов Samba (Сброс прав и xattr)
Если удаление TDB-баз не помогло, а расширенные атрибуты (xattr) Parsec / DAC повреждены в файловой системе для пакета, проще всего переустановить Samba из штатных репозиториев Astra 1.8. Это пересоздаст все бинарники, права и сопутствующие файлы с правильными системными метками:

```
# Останавливаем службы
systemctl stop smbd nmbd winbind

# Переустанавливаем пакеты
apt-get install --reinstall samba samba-common samba-common-bin libpam-samba2

# Восстанавливаем права на основные каталоги
chilevel -R 0 /var/lib/samba /var/cache/samba /var/log/samba /etc/samba

# Запускаем службы
systemctl start smbd nmbd
```
