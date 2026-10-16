# План: Скрипт автоматической установки extfilter

## Обзор

Создать универсальный bash-скрипт `install.sh`, который автоматически:
1. Определяет ОС и устанавливает зависимости
2. Скачивает и компилирует DPDK
3. Настраивает hugepages и isolcpus на основе ресурсов сервера
4. Скачивает и компилирует peafowl и marisa-trie
5. Компилирует и устанавливает extfilter
6. Настраивает systemd сервис и конфигурацию

## Структура скрипта

### Этапы выполнения:

#### 1. Инициализация и проверка прав
- Проверка запуска от root (id -u == 0)
- Определение ОС (CentOS/RHEL, Ubuntu/Debian)
- Определение архитектуры (x86_64)
- Логирование всех действий в файл
- Проверка наличия необходимых утилит (git, make, gcc)

#### 2. Установка системных зависимостей
- **CentOS/RHEL 7/8/9:**
  - gcc, gcc-c++, make, autoconf, automake, git
  - libpcap-devel
  - python3 (для DPDK)
  - tuned (опционально)
- **Ubuntu/Debian:**
  - build-essential, gcc, g++, make
  - autoconf, automake
  - git
  - libpcap-dev
  - python3

#### 3. Сборка Poco Library
- Скачивание Poco SDK с GitHub (версия 1.12.x или 1.13.x)
- Компиляция с флагами: Foundation, Net, Util
- Установка в /usr/local

#### 4. Сборка DPDK
- Скачивание DPDK v22.11 LTS (стабильная версия, совместима с проектом)
- Компиляция DPDK с UIO драйвером
- Установка в /usr/local

#### 5. Настройка системы (требует reboot)
- **Анализ ресурсов:**
  - CPU cores (nproc)
  - Total RAM (free -g)
  - Available hugepages
- **Расчёт hugepages:**
  - Формула: 1G страницы для DPDK
  - Минимум: 4 страницы (4GB)
  - Максимум: 50% от доступной RAM
  - Формула: min(4, max(ceil(RAM_GB * 0.5), 4))
- **Настройка isolcpus:**
  - Оставляем ядро 0 для системы
  - Изолируем все остальные ядра
  - Формат: isolcpus=1,2,3,...,N-1
- **Применение настроек:**
  - Обновление GRUB_CMDLINE_LINUX
  - Настройка hugepages через sysfs или tuned
  - **ПРЕДУПРЕЖДЕНИЕ о необходимости reboot**

#### 6. Сборка зависимостей extfilter
- **Peafowl (DPI library):**
  - git clone https://github.com/max197616/Peafowl.git
  - make
- **Marisa-trie:**
  - git clone https://github.com/max197616/marisa-trie.git
  - autoreconf -i && ./configure --enable-native-code && make

#### 7. Сборка extfilter
- ./autogen.sh
- ./configure с параметрами:
  - --with-dpdk_home=/usr/local
  - --with-dpdk_target=x86_64-native-linuxapp-gcc
  - --enable-native-code
- make -j$(nproc)

#### 8. Установка
- make install (или ручное копирование)
- Копирование конфигурации в /usr/local/etc/extfilter.ini
- Создание директорий для списков блокировки
- Настройка systemd сервисов

#### 9. Финальная настройка
- Создание примеров файлов whitelist, domains, urls
- Настройка прав доступа
- Вывод инструкции по завершению

## Формат вывода в консоль

Каждый этап сопровождается:
```
============================================================
[ЭТАП N/N] Название этапа
============================================================
[INFO] Описание действия
[SUCCESS] Успешное завершение
[WARNING] Предупреждение
[ERROR] Ошибка (с выходом)
```

## Параметры скрипта (опционально)

```bash
./install.sh [OPTIONS]

Options:
  --skip-reboot       Пропустить reboot (если уже выполнен)
  --skip-hugepages    Не настраивать hugepages
  --skip-isolcpus     Не настраивать isolcpus
  --dpdk-version      Версия DPDK (по умолчанию: 22.11.1)
  --poco-version      Версия Poco (по умолчанию: 1.13.1)
  --debug             Включить отладочный вывод
  --help              Показать справку
```

## Обработка ошибок

- Каждая команда проверяется на успех
- При ошибке — вывод сообщения и выход с кодом 1
- Возможность продолжить после исправления ошибки
- Лог всех действий в /var/log/extfilter-install.log

## Требования к системе

- Linux (CentOS/RHEL 7+, Ubuntu 18.04+)
- x86_64 архитектура
- Минимум 2 сетевых интерфейса, поддерживаемых DPDK
- Минимум 8GB RAM
- Минимум 2 CPU ядра

## Пример использования

```bash
# Полный автоматический режим
sudo ./install.sh

# С указанием версии DPDK
sudo ./install.sh --dpdk-version 22.11.1

# Если reboot уже был выполнен
sudo ./install.sh --skip-reboot
```

## Файлы, которые создаёт скрипт

1. `/usr/local/bin/extFilter` — бинарник
2. `/usr/local/etc/extfilter.ini` — конфигурация
3. `/usr/local/etc/extfilter/domains` — пример доменов
4. `/usr/local/etc/extfilter/urls` — пример URL
5. `/usr/local/etc/extfilter/whitelist` — белый список
6. `/etc/systemd/system/extfilter.service` — systemd сервис
7. `/etc/systemd/system/igb_uio_module.service` — UIO драйвер
8. `/etc/systemd/system/dpdk-devbind.service` — binding NIC

## Потенциальные проблемы и решения

1. **Отсутствие сетевых интерфейсов DPDK**
   - Проверка через `lspci | grep -i ethernet`
   - Предупреждение если нет совместимых карт

2. **Недостаточно RAM для hugepages**
   - Автоматический расчёт безопасного значения
   - Минимум 4GB для hugepages

3. **Конфликт с существующей настройкой GRUB**
   - Бэкап текущего grub.cfg
   - Аккуратное добавление параметров

4. **Долгая компиляция DPDK**
   - Показ прогресса
   - Оценка оставшегося времени

5. **Проблемы с сетью при настройке**
   - Предупреждение что сеть может быть недоступна
   - Рекомендация использовать консоль
