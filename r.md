## Шаг 1. Жесткое резервное копирование (Обязательно)
Поскольку файлы конфигураций хранятся внутри директории проекта (storage/), перед любыми командами git мы должны скопировать их в безопасное место, чтобы они не пострадали при слиянии веток.

Выполните от root (или через sudo):

```
# Создаем папку для бэкапа в корне или домашней директории
mkdir -p /root/rconfig_backup

# 1. Бэкап конфигураций и всех данных (самое важное!)
cp -a /var/www/html/rconfig/storage /root/rconfig_backup/

# 2. Бэкап файла с паролями от БД и ключами
cp /var/www/html/rconfig/.env /root/rconfig_backup/

# 3. Бэкап базы данных (замените логин/пароль на свои, если они отличаются)
# Посмотреть их можно в файле .env (DB_USERNAME и DB_PASSWORD)
mysqldump -u root -p rconfig > /root/rconfig_backup/rconfig_db.sql
```

## Шаг 2. Остановка фоновых задач
Чтобы во время обновления rConfig не попытался скачать новый конфиг и не записал его в наполовину обновленную базу:

```
cd /var/www/html/rconfig
php artisan down
```

(Это переведет веб-интерфейс в режим обслуживания).

## Шаг 3. Обновление исходного кода и зависимостей
Здесь мы прячем локальные изменения (если они были), скачиваем новый код и сначала обновляем зависимости PHP.

```
cd /var/www/html/rconfig

# Прячем любые изменения, чтобы git pull не выдал конфликт
git stash

# Скачиваем последнюю версию (8.2.17)
git pull

# СНАЧАЛА устанавливаем зависимости (используем флаг --no-dev для production)
composer install --no-dev --optimize-autoloader
```

## Шаг 4. Миграции и синхронизация
Теперь, когда все новые пакеты загружены, безопасно обновлять базу данных.

```
# Обновляем структуру базы данных
php artisan migrate --force

# Синхронизируем системные задачи
php artisan rconfig:sync-tasks
```

## Шаг 5. Восстановление прав и доступов к файлам
Это тот самый этап, который решает проблему «файлы не найдены».

```
# Возвращаем владельца веб-серверу (иначе файлы будут принадлежать root)
chown -R www-data:www-data /var/www/html/rconfig/storage
chown -R www-data:www-data /var/www/html/rconfig/bootstrap/cache

# Выставляем права 775, чтобы Apache мог читать и писать
chmod -R 775 /var/www/html/rconfig/storage
chmod -R 775 /var/www/html/rconfig/bootstrap/cache

# Пересоздаем символическую ссылку для веб-интерфейса (ОЧЕНЬ ВАЖНО)
# Если спросит, перезаписать ли текущую - отвечайте Yes.
php artisan storage:link

# Запускаем встроенный скрипт прав rConfig
php artisan rconfig:set-config-permissions
```

## Шаг 6. Глубокая очистка кэша
У Laravel агрессивное кэширование путей. Если не выполнить эти команды, система будет искать старые маршруты.

```
php artisan optimize:clear
php artisan rconfig:clear-all
```

## Шаг 7. Перезапуск сервисов и запуск системы

```
systemctl restart apache2

# Если вы используете supervisor для фоновых задач rConfig, перезапустите и его:
systemctl restart supervisor 2>/dev/null || true

# Выключаем режим обслуживания
php artisan up
```

Если после этого файлы конфигураций все равно не отображаются:
Зайдите в /root/rconfig_backup/storage/app/ (ваш бэкап) и убедитесь, что файлы там есть.

В версии 8.2 rConfig мог изменить структуру папок внутри storage/. Посмотрите, где лежат ваши конфиги в бэкапе, и скопируйте их вручную поверх новой структуры с сохранением прав:

```
# Пример копирования из бэкапа обратно в рабочую директорию (с заменой файлов)
cp -a /root/rconfig_backup/storage/app/* /var/www/html/rconfig/storage/app/
chown -R www-data:www-data /var/www/html/rconfig/storage/app/
```
# Пример копирования из бэкапа обратно в рабочую директорию (с заменой файлов)
cp -a /root/rconfig_backup/storage/app/* /var/www/html/rconfig/storage/app/
chown -R www-data:www-data /var/www/html/rconfig/storage/app/
