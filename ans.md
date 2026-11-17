```
#!/usr/bin/env bash
# ==============================================================================
# [HEADER]
# Назначение: Автоматический запуск Ansible-плейбука в виртуальном окружении
#             с динамическим формированием списка хостов, сохранением ANSI-цветов 
#             в лог-файл и полным циклом обработки ошибок.
# Особенности:
#   - Строгий режим bash (set -euo pipefail) + перехват ошибок (trap)
#   - Пересоздание лог-файла при каждом запуске
#   - Сохранение цветовой палитры Ansible в файл логов через утилиту `script`
#   - Чтение списка хостов из соседнего файла с автоматическим добавлением localhost
# Зависимости: bash, coreutils, ansible, util-linux (утилита script)
# Примеры запуска:
#  ./run_playbook.sh
# ==============================================================================

# --- СТРОГИЙ РЕЖИМ И TRAP ---
set -euo pipefail
trap 'log_error "Скрипт аварийно завершился на строке $LINENO с кодом ошибки $?"' ERR

# --- ГЛОБАЛЬНЫЕ КОНСТАНТЫ (ТОЛЬКО ЧТЕНИЕ) ---
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly VENV_PATH="${SCRIPT_DIR}/venv"
readonly PLAYBOOK_NAME="playbook.yml"
readonly PLAYBOOK_PATH="${SCRIPT_DIR}/${PLAYBOOK_NAME}"
readonly HOSTS_FILE="${SCRIPT_DIR}/hosts.list"
readonly LOG_FILE="${SCRIPT_DIR}/ansible_run.log"

# --- ЦВЕТА ДЛЯ ВЫВОДА В ТЕРМИНАЛ ---
readonly NC='\033[0m'
readonly GREEN='\033[0;32m'
readonly YELLOW='\033[0;33m'
readonly RED='\033[0;31m'
readonly BLUE='\033[0;34m'
readonly CYAN='\033[0;36m'

# --- ФУНКЦИИ ЛОГИРОВАНИЯ ---
log_info()  { echo -e "${GREEN}[INFO]  [$(date +'%Y-%m-%d %H:%M:%S')] $1${NC}"; }
log_warn()  { echo -e "${YELLOW}[WARN]  [$(date +'%Y-%m-%d %H:%M:%S')] $1${NC}"; }
log_error() { echo -e "${RED}[ERROR] [$(date +'%Y-%m-%d %H:%M:%S')] $1${NC}" >&2; }
log_banner(){ 
    echo -e "${CYAN}======================================================================${NC}"
    echo -e "${CYAN} $1${NC}"
    echo -e "${CYAN}======================================================================${NC}"
}

# --- ПРОВЕРКА ОКРУЖЕНИЯ И ЗАВИСИМОСТЕЙ ---
check_dependencies() {
    log_info "Проверка наличия необходимых файлов и окружения..."

    [[ -f "${PLAYBOOK_PATH}" ]] \vert{}\vert{} { log_error "Плейбук не найден по пути: ${PLAYBOOK_PATH}"; exit 1; }
    [[ -f "${VENV_PATH}/bin/activate" ]] \vert{}\vert{} { log_error "Виртуальное окружение не найдено в: ${VENV_PATH}"; exit 1; }
    command -v script &>/dev/null || { log_error "Утилита 'script' не установлена в системе (пакет util-linux)."; exit 1; }
    
    log_info "Все предварительные проверки пройдены успешно."
}

# --- ГЕНЕРАЦИЯ СТРОКИ LIMIT ИЗ ФАЙЛА ХОСТОВ ---
get_limit_argument() {
    local hosts_list=()

    if [[ -f "${HOSTS_FILE}" ]]; then
        while IFS= read -r line || [[ -n "$line" ]]; do
            # Убираем пробелы и пропускаем комментарии/пустые строки
            line="$(echo "$line" | xargs)"
            [[ -z "$line" \vert{}\vert{} "$line" == \#* ]] && continue
            hosts_list+=("$line")
        done < "${HOSTS_FILE}"
    else
        log_warn "Файл хостов ${HOSTS_FILE} не найден. Будет использован только localhost."
    fi

    # Всегда добавляем localhost в конец списка
    hosts_list+=("localhost")

    # Объединяем массив через запятую
    local limit_str
    limit_str=$(IFS=,; echo "${hosts_list[*]}")
    echo "${limit_str}"
}

# --- ОСНОВНОЙ ПРОЦЕСС ЗАПУСКА ANSIBLE ---
run_ansible() {
    log_banner "СТАРТ ВЫПОЛНЕНИЯ ANSIBLE ПЛЕЙБУКА"
    
    # 1. Пересоздаем/очищаем лог-файл при каждом запуске
    : > "${LOG_FILE}" \vert{}\vert{} { log_error "Не удалось создать/очистить лог-файл: ${LOG_FILE}"; exit 1; }
    log_info "Лог-файл инициализирован: ${LOG_FILE}"

    # 2. Активируем виртуальное окружение
    # shellcheck source=/dev/null
    source "${VENV_PATH}/bin/activate" || { log_error "Не удалось активировать venv."; exit 1; }

    # 3. Формируем список хостов для параметра -l
    local target_limit
    target_limit=$(get_limit_argument)
    log_info "Целевые хосты (-l): ${target_limit}"

    # 4. Запуск плейбука с сохранением ANSI-цветов через утилиту script
    # ANSIBL_FORCE_COLOR=1 заставляет Ansible генерировать цвета в не-TTY окружении
    log_info "Запуск ansible-playbook..."
    
    set +e # Временно отключаем errexit для перехвата кода возврата ansible
    script -q -c "ANSIBLE_FORCE_COLOR=1 ansible-playbook '${PLAYBOOK_PATH}' -l '${target_limit}'" /dev/null \vert{} tee "${LOG_FILE}"
    local ansible_exit_code=$?
    set -e

    log_banner "ЗАВЕРШЕНИЕ ВЫПОЛНЕНИЯ ANSIBLE ПЛЕЙБУКА"

    if [[ ${ansible_exit_code} -eq 0 ]]; then
        log_info "Плейбук успешно выполнен без ошибок."
    else
        log_error "Плейбук завершился с ошибкой (код возврата: ${ansible_exit_code}). Проверьте лог: ${LOG_FILE}"
        exit ${ansible_exit_code}
    fi
}

# --- ИТОГОВАЯ ПРОВЕРКА / СТАТУС ---
post_check() {
    log_info "Проверка результатов работы..."
    if [[ -f "${LOG_FILE}" ]]; then
        local log_size
        log_size=$(du -h "${LOG_FILE}" | awk '{print $1}')
        log_info "Размер файла логов: ${log_size}"
        log_info "Последние строки выполнения:"
        tail -n 10 "${LOG_FILE}"
    fi
}

# --- ТОЧКА ВХОДА (MAIN) ---
main() {
    check_dependencies
```

```
0 6 * * 1,3,5 /bin/bash /home/ansible/run_playbook.sh >/dev/null 2>&1
```
