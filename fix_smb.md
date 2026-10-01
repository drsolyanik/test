1. Проверка уровней целостности через pdpl-file
В Astra Linux за мандатный контроль и уровни целостности отвечает утилита pdpl-file (а не стандартный chilevel).

Если astra-modeswitch прервался, каталоги Samba могли остаться с высокими атрибутами мандатного уровня, и smbd при старте не может создать в них файлы IPC.

Проверьте текущие уровни целостности:

Bash
pdpl-file /run/samba /var/lib/samba /var/cache/samba
(На рабочем сервере при отключенных/нулевых уровнях должно быть 0:0:0:0 или 0).

Принудительно сбросьте уровень целостности в 0:

Bash
systemctl stop smbd nmbd
pdpl-file 0 /run/samba
pdpl-file 0 /var/lib/samba
pdpl-file 0 /var/cache/samba
pdpl-file 0 /var/log/samba
2. Проверка разделяемой памяти (Shared Memory / /dev/shm)
Модуль messaging_init в Samba 4 активно использует POSIX shared memory и семафоры в /dev/shm. Если при сбое astra-modeswitch сбились права на /dev/shm или /tmp, создание IPC-сокетов блокируется.

Проверьте права и мандатный уровень /dev/shm:

Bash
ls -ld /dev/shm /tmp
pdpl-file /dev/shm /tmp
Права должны быть drwxrwxrwt (1777), а уровень целостности — 0.

Если там остались старые файлы блокировок Samba от предыдущего режима, очистите их:

Bash
rm -rf /dev/shm/sem.smb* /dev/shm/smb*
3. Разблокировка strace или ручной запуск smbd
strace в Astra Linux заблокирован механизмом ptrace_scope в ядре. Чтобы временно разрешить трассировку для диагностики:

Снимите ограничение ptrace:

Bash
sysctl -w kernel.yama.ptrace_scope=0
Запустите трассировку инициализации IPC:

Bash
strace -f -e trace=file,ipc,network smbd -F -i -d 3
Вы увидите точную строчку, где Samba пытается сделать mkdir("/run/samba/ncalrpc") или shm_open() / bind() и получает от ядра ошибку EACCES (Permission denied) или EPERM.

4. Сравнение systemd-юнитов и PARSEC-директив
В Astra Linux 1.8 службы systemd запускаются с учётом контекста безопасности PARSEC.

Сравните файл /lib/systemd/system/smbd.service на сломанном и рабочем сервере.

Проверьте, нет ли переопределений в /etc/systemd/system/smbd.service.d/ или параметров вида:

ParsecPrivileged=yes

CapabilityBoundingSet=...

Выполните перезагрузку конфигурации systemd:

Bash
systemctl daemon-reload
systemctl restart smbd
5. Точечный тест ручного запуска
Остановите службу и запустите smbd вручную напрямую из консоли root:

Bash
systemctl stop smbd nmbd
/usr/sbin/smbd -F -i -d 10 | grep -iE 'messaging|socket|lock|failed|error'
Поскольку служба не будет уходить в фоновый режим, в первых 20–30 строках вывода отладки появится конкретная причина, почему smbd пропустил создание /run/samba/ncalrpc и /run/samba/msg.lock.
