Установка пакетов для SSSD

```
sudo apt update && sudo apt install sssd sssd-tools libnss-sss libpam-sss -y
```

Настройка конфигурационного файла SSSD, например:

`/etc/sssd/sssd.conf`

```
[sssd]
domains = sdr.lan
config_file_version = 2
services = nss, pam

[domain/sdr.lan]
ad_domain = sdr.lan
krb5_realm = SDR.LAN
realmd_tags = manages-system-local-links
id_provider = ad

# Использовать чистые имена без домена
use_fully_qualified_names = False

# Создавать домашние директории при первом входе
fallback_homedir = /home/%d/%u
default_shell = /bin/bash

# Автоматический маппинг ID (генерация одинаковых UID/GID)
ldap_id_mapping = True
```

Изменение глобальных настроек Samba

`/etc/samba/smb.conf`

```
[global]
   workgroup = SDR
   realm = SDR.LAN
   security = ADS
   server role = member server
   winbind use default domain = No
   winbind refresh tickets = Yes
   kerberos method = secrets and keytab
   
   idmap config * : backend = tdb
   idmap config * : range = 10000-99999

   idmap config SDR : backend = sss
   idmap config SDR : range = 200000-2000200000
```

Настройка NSS переход от Winbind к SSSD

`/etc/nsswitch.conf`

```
passwd:         files systemd sss
group:          files systemd sss
shadow:         files systemd sss
gshadow:        files systemd

hosts:          files dns
networks:       files

protocols:      db files
services:       db files sss
ethers:         db files
rpc:            db files

netgroup:       nis sss
automount:  sss
```

Очистка кэша

```
# 1. Тушим службы
sudo systemctl stop smbd nmbd winbind sssd

# 2. Выжигаем старый кэш маппинга Winbind
sudo net cache flush
sudo rm -f /var/lib/samba/idmap_cache.tdb
sudo rm -f /var/lib/samba/winbindd_cache.tdb
sudo rm -f /var/lib/samba/gencache.tdb

# 3. Включаем автозапуск и поднимаем всё в правильном порядке
sudo systemctl enable --now sssd
sudo systemctl enable --now winbind
sudo systemctl restart smbd nmbd
```

Проверка

```
wbinfo -n sa@sdr.lan
wbinfo -S <SID>
getent passwd sa@sdr.lan
```

Повторный выпуск `/etc/krb5.keytab`

```
sudo net ads keytab create -P
```

Для SSSD

```
[sssd]
user = root

[domain/<Домен>]
ad_update_samba_machine_account_password = true
```

Для Samba

```
[global]
machine password timeout = 0
```
