## Чек-лист быстрых действий (Экспресс-диагностика)

Выполняйте команды последовательно в командной строке (`cmd`), запущенной **от имени Администратора**:

```
:: 1. Проверка и запуск драйвера HTTP
sc config http start= auto
net start http

:: 2. Проверка службы WMI
net start winmgmt
winmgmt /verifyrepository

:: 3. Запуск и настройка WinRM
sc config winrm start= auto
net start winrm
winrm quickconfig -q
```

---

## Пошаговая инструкция по устранению неполадок

### Шаг 1: Проверка и запуск службы/драйвера HTTP (`http.sys`)
WinRM напрямую зависит от драйвера режим ядра `http.sys`. Если он остановлен или отключен, WinRM не сможет запуститься (ошибка `1068`).

1. Откройте `cmd` от имени администратора.
2. Проверьте состояние службы:
   ```
   sc query http
   ```
3. Если статус `STOPPED` или тип запуска `DISABLED`, выполните:
   ```
   sc config http start= auto
   net start http
   ```
4. Если `http.sys` не запускается, перезагрузите сервер.

---

### Шаг 2: Проверка репозитория WMI (Winmgmt)
Служба WinRM использует инфраструктуру WMI для своей работы.

1. Проверьте статус службы WMI:
   ```
   net start winmgmt
   ```
2. Выполните проверку целостности репозитория:
   ```
   winmgmt /verifyrepository
   ```
3. **Возможные результаты:**
   * **`Repository is consistent`** — репозиторий исправен, переходите к Шагу 3.
   * **`Repository is inconsistent`** — репозиторий поврежден. Выполните восстановление:
     ```
     winmgmt /salvagerepository
     ```
   * Если восстановление не помогло, сбросьте репозиторий:
     ```
     winmgmt /resetrepository
     ```

---

### Шаг 3: Настройка сетевого профиля (Network Location)
Служба `winrm quickconfig` блокирует работу, если хотя бы один сетевой адаптер имеет профиль **Public (Общественная сеть)**.

1. Откройте **PowerShell** от имени Администратора.
2. Проверьте текущий профиль:
   ```
   Get-NetConnectionProfile
   ```
3. Если статус `NetworkCategory` указан как `Public`, смените его на `Private`:
   ```
   Set-NetConnectionProfile -NetworkCategory Private
   ```

---

### Шаг 4: Очистка зависших URL ACL привязок
При сбоях в прошлых конфигурациях порт `5985` может быть заблокирован старыми правилами `http.sys`.

1. Удалите старые правила резервации URL в `cmd`:
   ```
   netsh http delete urlacl url=http://+:5985/wsman/
   netsh http delete urlacl url=https://+:5986/wsman/
   ```
2. Повторите попытку настройки WinRM:
   ```
   winrm quickconfig
   ```

---

### Шаг 5: Проверка прав доступа к папке MachineKeys
WinRM использует системные криптографические ключи RSA для аутентификации.

1. Перейдите в каталог: `C:\ProgramData\Microsoft\Crypto\RSA\`
2. Нажмите правой кнопкой мыши по папке **MachineKeys** -> **Свойства** -> **Безопасность**.
3. Проверьте наличие прав:
   * **Администраторы (Administrators):** Полный доступ (`Full Control`).
   * **Все (Everyone):** Чтение и запись (`Read / Write`) или специальное разрешение (для создания файлов).
4. Если права нарушены, назначьте владельцем группы **Администраторы** и восстановите доступ.

---

## Решение специфических ситуаций и кодов ошибок

### Сценарий А: Ошибка 1068 (Зависимая служба не запущена)
* **Причина:** Не запущена одна из базовых служб (`HTTP`, `RPC`, `EventLog`).
* **Решение:**
  1. Откройте оснастку `services.msc`.
  2. Проверьте и запустите службы:
     * **Удаленный вызов процедур (RPC)** (`RpcSs`)
     * **Журнал событий Windows** (`eventlog`)
     * **Драйвер протокола HTTP** (`http`)

### Сценарий Б: Ошибка 5 / Access is Denied (Отказано в доступе)
* **Причина:** Ограничения UAC для локальных учетных записей при удаленном управлении.
* **Решение:** Добавьте параметр реестра, отключающий фильтрацию токенов UAC для сетевых входов:
  ```
  reg add HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v LocalAccountTokenFilterPolicy /t REG_DWORD /d 1 /f
  ```

### Сценарий В: Ошибка `0x80090014` или `0x80070005` при `winrm quickconfig`
* **Причина:** Повреждение сертификатов или ключей SSL/TLS.
* **Решение:**
  1. Re-initialize WinRM прослушиватели:
     ```
     winrm invoke Restore winrm/config @{}
     ```
  2. Пересоздайте HTTP listener вручную:
     ```
     winrm create winrm/config/listener?Address=*+Transport=HTTP
     ```

---

## Обходной путь: Установка IIS в обход Server Manager / WinRM

Если запустить WinRM оперативно не удается, вы можете установить IIS напрямую через командную строку с помощью **DISM** (данный способ не зависит от службы WinRM).

### Вариант 1: Через DISM в `cmd`
```
dism /online /enable-feature /featurename:IIS-WebServerRole /featurename:IIS-WebServer /featurename:IIS-CommonHttpFeatures /featurename:IIS-StaticContent /featurename:IIS-DefaultDocument /featurename:IIS-DirectoryBrowsing /featurename:IIS-HttpErrors /featurename:IIS-ManagementConsole
```

### Вариант 2: Через PowerShell
```
Install-WindowsFeature -Name Web-Server -IncludeManagementTools
```

---

## Проверка успешности установки IIS
После выполнения всех шагов убедитесь, что IIS функционирует:
1. Откройте браузер на сервере и перейдите по адресу: `http://localhost` или `http://127.0.0.1`.
2. Должна отобразиться стандартная страница-заставка **IIS Windows Server**.
3. Проверьте доступность оснастки управления IIS: `Win + R` -> `inetmgr`.
