# Скрипт для управления общими папками Samba

Этот скрипт предназначен для удобного и надёжного управления общими папками (шарами) Samba. Он позволяет создавать, удалять, измениять, запускать, останавливать и просматривать Samba-шары, а также управлять соответствующими службами Samba.

---

## Основные возможности

- **Создание и удаление общих папок (шар) Samba** с настройкой:
  - Имя шары (`share_name`)
  - Путь к каталогу (`share_path`)
  - Пользователь или группа, которым предоставляется доступ (`access_user`)
  - Тип доступа: только чтение (`ro`) или чтение/запись (`rw`)
- **Проверка корректности конфигурации Samba** перед применением изменений (`testparm`)
- **Перезапуск и управление службами Samba** (`smb.service` и `nmb.service`)
- **Развёртывание и свёртывание Samba** — включает или отключает необходимые службы
- **Вывод списка всех существующих Samba-шар** в удобном JSON-формате
- **Логирование всех операций** в файл для аудита и отладки

---

## Расположение файлов

| Файл                 | Описание                                    | Путь                                 |
|----------------------|---------------------------------------------|-------------------------------------|
| Основной скрипт       | Скрипт управления Samba-шарами               | `/usr/bin/service-samba_shares`            |
| Backend-файл         | Вспомогательные функции и логика              | `/usr/share/alterator-service-samba/service-samba-shares.backend` |
| systemd unit-файл    | Сервис systemd для автоматического запуска скрипта | `/etc/systemd/system/service-samba-shares.service`          |
| Файл блокировки      | Для исключения одновременного запуска операций | `/var/lock/samba-shares.lock`         |

---

## Как работает скрипт — подробности

### 1. Блокировка

При запуске скрипт захватывает эксклюзивный лок-файл `/var/lock/samba-shares.lock`, чтобы исключить одновременное выполнение нескольких инстансов, предотвращая конфликтные изменения.

### 2. Чтение входных данных

Скрипт может принимать входные параметры двумя способами:

- Через стандартный ввод — JSON с параметрами и командой `mode`.
- Через первый аргумент командной строки, если JSON не был передан.

### 3. Обработка параметров

Обязательные параметры зависят от действия (`operation`):

- Для `create` нужны: `share_name`, `share_path`, `access_user`, `access_type`
- Для `delete` нужны: `share_name`, `share_path`
- Для остальных команд — параметры не обязательны

Опционально:  
- `allowed_users` — дополнительные пользователи с доступом (через запятую)

- `backup_name` — файл кофигурации для копирования/восстановления


### 4. Работа с конфигурацией Samba

- При создании:  
  Добавляется блок с настройками шары в конец файла `/etc/samba/smb.conf`.  
  Права на папку устанавливаются согласно типу доступа (`ro` — 2750, `rw` — 2770).  
  Пользователь назначается владельцем каталога.  

- При удалении:  
  Удаляется блок, соответствующий шары, из конфигурационного файла.  
  При этом удаляется и каталог с диска, если он существует.

- Конфигурация всегда проверяется через `testparm -s` перед применением, чтобы избежать ошибок и падений службы.

### 5. Управление службами

Скрипт использует `systemctl` для:

- Включения (`enable`) и запуска (`start`) служб `smb.service` и `nmb.service`
- Отключения (`disable`) и остановки (`stop`) тех же служб
- Перезапуска службы Samba после изменения конфигурации

---

## Пример вызова

Создание новой шары:


```bash
echo '{"operation":"create","share_name":"data","share_path":"/srv/shares/data","access_user":"alice","access_type":"rw","allowed_users":"bob,carol"}' | /usr/bin/service-samba-shares
```

## Команда `status`:

Команда `status` служит для получения текущего состояния Samba-сервисов и списка всех созданных Samba-шар.

### Что делает `status`:

- Проверяет состояние системных служб `smb.service` и `nmb.service` через `systemctl is-active`.
- Считывает и парсит текущий конфигурационный файл Samba `/etc/samba/smb.conf`, извлекая информацию о всех шарах.
- Возвращает JSON-объект с подробной информацией о статусе служб и списком доступных шар с параметрами:
  - `share_name` — имя шары
  - `share_path` — путь к расшаренной папке
  - `access_type` — тип доступа (`ro` или `rw`)
  - `access_user` — пользователь или группа с доступом (если задан)



При выполнении действия `status` скрипт возвращает специальные коды выхода, которые можно использовать для автоматической обработки состояния Samba-сервисов:

- **128** — обе службы (`smb.service` и `nmb.service`) активны и работают нормально.  
  Скрипт считает, что Samba полностью развернута и функционирует.

- **127** — только одна из служб активна (либо `smb.service`, либо `nmb.service`).  
  Это состояние указывает на частичное развертывание, когда одна из служб еще не запущена или остановлена.

- **0** — ни одна из служб не активна.  
  Система считает, что Samba не развернута (undeployed).

- **1** — возникла ошибка при чтении или парсинге конфигурационного файла Samba (`smb.conf`).

- **3** — общая внутренняя ошибка, например, отсутствие прав, проблемы с файлами или системные сбои.

Использование этих кодов позволяет интегрировать скрипт с системами мониторинга, автоматического деплоя и контроля состояния сервисов.

---

### Пример вывода:

```json
{
  "status": "started",
  "shares": [
    {
      "share_name": "data",
      "share_path": "/srv/data",
      "access_type": "rw",
      "access_user": "alice"
    },
    {
      "share_name": "public",
      "share_path": "/srv/public",
      "access_type": "ro",
      "access_user": ""
    }
  ],
  "services": {
    "smb": "active",
    "nmb": "active"
  }
}
```

## Описание файла `.service` (описание сервиса в системе управления)

Файл `.service` описывает сервис **samba_shares**, предоставляющий единый интерфейс для управления Samba-шарами через систему конфигурации.

### Основные поля и их назначение

- `type = "Service"` — определяет, что это сервисный объект.
- `name = "samba_shares"` — уникальное системное имя сервиса.
- `category = "X-Alterator-Servers"` — категория для группировки в интерфейсе управления.
- `persistent = true` — сервис хранит свое состояние и параметры между перезагрузками.
- `display_name` и `comment` — отображаемые названия и описания на английском и русском языках.
- `icon = "network-server"` — иконка для UI.

### Параметры (parameters)

Параметры сервиса позволяют управлять конфигурацией отдельных Samba-шар.

- `share_name` — уникальное имя шары (обязательно при создании).
- `share_path` — путь к расшариваемой папке.
- `access_user` — пользователь, которому предоставлен доступ.
- `access_type` — тип доступа: `ro` (только чтение) или `rw` (чтение и запись).
- `enabled` — флаг включения/отключения шары.

Эти параметры можно использовать в различных контекстах: настройка (`configure`), резервное копирование (`backup`), восстановление (`restore`), и просмотр статуса (`status`).

### Ресурсы (resources)

- `smb_conf` — путь к файлу конфигурации Samba `/etc/samba/smb.conf`.
- `smb_service` и `nmb_service` — ссылки на системные службы Samba и NetBIOS, управляемые через systemd (`smb.service`, `nmb.service`).

### Массив `shares`

Содержит список всех текущих Samba-шар с их параметрами:

- `share_name`
- `share_path`
- `access_user`
- `access_type`
- `enabled`

Этот массив отображается в режиме `status`, позволяя удобно видеть и мониторить все созданные шары в интерфейсе.

---


## Функции резервного копирования и восстановления

### Резервное копирование (`backup`) (restore`))

Функция `backup` `restore` предназначена для создания копии текущей конфигурации Samba-шар и связанных данных. Это позволяет сохранить состояние настроек и при необходимости быстро восстановить их после сбоев, обновлений или других изменений.

- **Что сохраняется:**
  - Файл конфигурации Samba `/etc/samba/smb.conf`
  - Каталоги, связанные с шарами (опционально, если включено)
  - Пользовательские настройки доступа и права на папки

- **Где хранится резервная копия:**
  - Резервная копия помещается в каталог `/etc/samba/backups/smb-b.conf` с отметкой времени создания
  - Формат резервной копии — архив `.tar.gz` с полным содержимым конфигурации и данных

- **Как вызывается:**
  - Через команду с параметром `mode=backup`
  - Может принимать дополнительный параметр `backup_path` для указания альтернативного каталога

  ### Развертывание и его отмена (deploy/undeploy)

  - При развертывании включаются службы smb и nmb
  - Создается оригинальный файл по пути: `etc/samba/smb-original.conf`