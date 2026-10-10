# OpenWrt Full Backup 2

Резервное копирование системы OpenWrt через LuCI: полный архив `/overlay/upper` и сохранение только конфигурации.

## Что делает

Добавляет в LuCI пункт **Службы → OpenWrt FullBackup 2** с двумя кнопками:

| Кнопка | Что сохраняет | Где взять восстановление |
|---|---|---|
| **Создать полную резервную копию** | Весь `/overlay/upper` — система, пакеты, настройки, данные | Система → Восстановление / Обновление |
| **Сохранить только конфигурацию** | Штатный `sysupgrade -b` — только настройки без системы | Система → Восстановление / Обновление |

Оба архива складываются в `/tmp/backups/`, скачиваются через браузер, на диске роутера не остаются.

## Архитектура

```
LuCI (JS)  --[fs.write]-->  /tmp/backup.request   (флаг)
                                    |
                                    v
                          backup-watcher.sh (демон)
                                    |
                    +---------------+---------------+
                    |                               |
                    v                               v
          backup-manager.sh               backup-config.sh
          (полный /overlay/upper)         (sysupgrade -b)
                    |                               |
                    +---------------+---------------+
                                    |
                                    v
                    /tmp/backup.status  ("RUNNING" -> "OK:<path>" | "ERROR:<msg>")
                                    |
                                    v
                    LuCI (JS) опрашивает статус каждые 3 сек,
                    при OK: скачивает архив через fs.read_direct
```

**Почему так, а не синхронно.** Прямой вызов `fs.exec('/usr/bin/backup-manager.sh')` из LuCI упирается в XHR-таймаут: браузер не готов ждать минуту-две, пока tar сжимает 20+ МБ. Поэтому JS только **пишет флаг** (мгновенно), а демон в фоне делает работу. JS опрашивает `/tmp/backup.status` маленькими запросами — они быстрые, таймаута нет.

## Отличия от OpenWrt FullBackup 1

- Демон-наблюдатель + флаг-запрос вместо синхронного вызова
- Скачивание через браузер вместо складывания в `/overlay/backups`
- Две кнопки: полный архив + только конфигурация
- Имя архива включает hostname и версию прошивки

## Установка

Скопируйте в терминал роутера и нажмите Enter:

```sh
sh <( wget -qO- https://raw.githubusercontent.com/kkkkCampbell/master/refs/heads/fullbackup2/fullbackup2.sh )
```

После установки — **Ctrl+F5** в браузере (сброс кэша JS), затем **Службы → Резервная копия**.

## Файлы

| Файл | Назначение |
|---|---|
| `/usr/bin/backup-manager.sh` | Полный архив `/overlay/upper` в `/tmp/backups/` |
| `/usr/bin/backup-config.sh` | Только конфигурация через `sysupgrade -b` |
| `/usr/bin/backup-watcher.sh` | Демон, следит за флагами и запускает нужный скрипт |
| `/etc/init.d/backup-watcher` | init-скрипт демона (procd, respawn) |
| `/usr/share/luci/menu.d/luci-app-backup.json` | Пункт меню в LuCI |
| `/usr/share/rpcd/acl.d/luci-app-backup.json` | Права на exec/read/write |
| `/www/luci-static/resources/view/backup/backup-manager.js` | Страница с кнопками |

## Временные файлы

| Путь | Назначение |
|---|---|
| `/tmp/backup.request` | Флаг «сделать полный бэкап» |
| `/tmp/backup-config.request` | Флаг «сделать бэкап конфигурации» |
| `/tmp/backup.status` | Статус: `RUNNING`, `OK:<путь>`, `ERROR:<текст>` |
| `/tmp/backup.pid` | PID текущего процесса архивации |
| `/tmp/backups/` | Папка для архивов, пересоздаётся перед каждым запуском |

## Что исключается из полного архива

- `/overlay/upper/backups` — на случай, если кто-то положит бэкапы внутрь upper
- `/overlay/upper/run` — runtime-данные
- `/overlay/upper/etc/os-release`, `/overlay/upper/usr/lib/os-release` — пересоздаются при загрузке
- **whiteout'ы** — символьные устройства `c 0,0`, маркеры удаления файлов из `/rom`. В архиве они не нужны и при восстановлении могут сломать OverlayFS.

Исключение whiteout'ов делается автоматически:

```sh
find /overlay/upper -type c > "$WHITEOUTS"
tar ... -X "$WHITEOUTS" ...
```

## Что сохраняется

- Права (mode), владелец/группа (UID/GID), временные метки
- Обычные файлы, каталоги, симлинки
- Все изменения в `/overlay/upper` (то есть всё, что отличается от `/rom`)

**Не сохраняется:** расширенные атрибуты (xattr, POSIX ACLs). Если они используются в вашей конфигурации — нужно добавить `--xattrs` в `tar`, но с осторожностью к `trusted.overlay.*`.

## Восстановление

Полный архив восстанавливается **штатным механизмом OpenWrt**:

1. **Система → Восстановление / Обновление**.
2. Выбрать файл `backup-full-<hostname>-<version>-<date>.tar.gz`.
3. **Перед восстановлением желательно сбросить роутер до настроек по умолчанию** — иначе файлы из архива лягут поверх текущего upper, что может дать неконсистентное состояние.
4. Загрузить, восстановить, перезагрузить.

Архив конфигурации (`backup-config-...tar.gz`) восстанавливается там же.

## Требования

- OpenWrt с LuCI
- GNU `tar` (устанавливается автоматически, если стоит BusyBox-версия)
- Свободное место в `/tmp` (RAM) — под архив + промежуточные файлы

## Удаление

```bash
/etc/init.d/backup-watcher stop
/etc/init.d/backup-watcher disable
rm -f /etc/init.d/backup-watcher
rm -f /usr/bin/backup-manager.sh /usr/bin/backup-config.sh /usr/bin/backup-watcher.sh
rm -f /usr/share/luci/menu.d/luci-app-backup.json
rm -f /usr/share/rpcd/acl.d/luci-app-backup.json
rm -f /www/luci-static/resources/view/backup/backup-manager.js
rmdir /www/luci-static/resources/view/backup 2>/dev/null
rm -f /tmp/backup.request /tmp/backup-config.request /tmp/backup.status /tmp/backup.pid
rm -rf /tmp/backups
/etc/init.d/rpcd restart
```

## Известные ограничения

- **XHR-таймаут** при прямом вызове архивации — обойдён через демон + флаг.
- **Расширенные атрибуты** (xattr/ACLs) не сохраняются.
- **Восстановление только штатное** — собственная распаковка поверх живого upper опасна и не реализована.
