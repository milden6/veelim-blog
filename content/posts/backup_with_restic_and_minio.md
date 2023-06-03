---
title: "Restic и MinIO (S3)"
date: 2023-06-03T11:35:00+01:00
description: "Как делать бэкапы без проводов"
tags: [backup, open-source]
draft: false
---

До недавнего времени я делал бэкапы на внешний HDD. Если делать бэкапы нечасто, то в целом это не проблема, а если речь о ежедневных бэкапах на разных устройствах, то бегать с внешним HDD крайне неудобно. Хочу рассказать про удобное (по крайней мере для меня) решение для ежедневных бекапов без проводов.

## restic

Для создания самих бэкапов я пользуюсь программой [restic](https://github.com/restic/restic). Это простая CLI программа, работает на всех популярных ОС т.к написана на Go. Позволяет делать бэкапы локально, на внешние устройства и в облачные сервисы из коробки. Довольно эффективно сжимает файлы, например, бэкап моей домашней директории размером в ~111GB получился всего в 20GB. При первом запуске он делает полный снапшот указанной директории, при след. запусках уже бэкапит дельту и сохраняет отдельными снапшотами только изменения. Снапшот - это слепок данных в определенный момент времени.

## MinIO

Бэкапы нужно где-то хранить, я выбрал для этого S3 хранилище [MinIO](https://github.com/minio/minio). Его довольно легко развернуть на своем сервере, в веб-интерфейсе несложно разобраться и вообще это production ready решение, его успешно используют во многих крупных компаниях для хранения данных.

## Установка и настройка

Скачать restic можно из github репозитория или с помощью пакетного менеджера.

В первую очередь инициализируем репозиторий:

```bash
restic -r RESTIC_REPOSITORY init
```

Репозиторием может быть директория или адрес облачного сервиса, например, если делать бэкап прямо на том же компьютере в папку `backup-repo`, то команда будет выглядеть так:

```bash
restic -r /home/backup-repo init
enter password for new repository:
enter password again:
```

restic попросить ввести пароль от репозитория, если пароль потеряется, то его никак нельзя будет восстановить и данные будут утеряны.

Команда для запуска процесса бэкапа выглядит следующим образом:

```bash
restic -r /home/backup-repo --verbose backup /documents
```

Флаг `--verbose` нужен для подробного вывода информации. Этой коммандой мы запустили бэкап директории `documents` в директорию  `backup-repo`.

Восстановить данные из бэкапа можно следующей командой:

```bash
restic -r /home/backup-repo restore latest --target /restore_folder
```

`latest` - это самый последний снапшот, можно указать id нужного снапшота, если требуется. restic позволяет восстанавливать отдельные файлы и директории для этого после флага `--path` нужно указать нужный путь.

Развернуть MinIO удобнее всего в docker контейнере:

```bash
docker run \
   -p 9000:9000 \
   -p 9090:9090 \
   --name minio \
   -v /mnt/my-storage:/data \
   -e "MINIO_ROOT_USER=ROOTNAME" \
   -e "MINIO_ROOT_PASSWORD=CHANGEME123" \
   quay.io/minio/minio server /data --console-address ":9090"
```

- `-p` пробрасывает порты между контейнером и машиной на которой запускается контейнер. 
- `9000` - это порт самого сервиса, а по порту `9090` можно попасть в веб-интерфейс.
- `-v` указание директории в которой будут храниться данные на хостовой машине, сервис будет зеркалировать данные в директорию  `/data`
- `-e` переменные окружения, нужны для доступа к веб-интерфейсу.
-  `--console-address` - адрес по которому будет доступен веб-интерфейс.

У меня MinIO крутится на мое сервере.

После установки MinIO нужно зайти в консоль и создать ключ доступа (access key) для того, чтобы restic имел доступ к хранилищу.

Теперь можно сделать бэкап в хранилище: 

```bash
# указываем ID и SECRET, созданные ранее в веб-интерфейсе
export AWS_ACCESS_KEY_ID=my_key_id
export AWS_SECRET_ACCESS_KEY=my_key_secret

# создаем новый репозиторий и указываем пароль
restic -r s3:http://localhost:9000/backup-repo init
enter password for new repository:
enter password again:

# запускаем процесс бэкапа
restic backup -r s3:http://localhost:9000/backup-repo --verbose /documents
```

Вместо `http://localhost:9000` будет ваш адрес на котором поднят MinIO.

В целом так и работает резервное копирование с restic и MinIO, но каждый день делать это  руками быстро надоест, поэтому этот процесс нужно автоматизировать, чтобы процесс бэкапа был полностью автономным.

## systemd

Я использую систему инициализации sytstemd для этих целей, но вместо systemd можно использовать cron на Linux или планировщик заданий на Windows для запуска скрипта по расписанию.

В первую очередь нужно создать файл в котором будут переменные окружения для работы restic. У меня он лежит в `~/.config/restic-backup.conf`

```
# доступы к MinIO
AWS_ACCESS_KEY_ID=my_key_id
AWS_SECRET_ACCESS_KEY=my_key_secret
# адрес репозитория MinIO
RESTIC_REPOSITORY=s3:http://localhost:9000/backup-repo
# пароль от репозитория
RESTIC_PASSWORD=restic_pass
# путь к директории, которую бэкапим
BACKUP_PATH="/documents"
# за сколько дней нужно хранить бэкапы
RETENTION_DAYS=7
```

Теперь сам сервис systemd. У меня он лежит в `~/.config/systemd/user/restic-backup.service`

```
[Unit]
Description=Restic backup service
[Service]
Type=oneshot
ExecStart=restic backup --verbose $BACKUP_PATH
ExecStartPost=restic forget --verbose --keep-daily $RETENTION_DAYS
EnvironmentFile=%h/.config/restic-backup.conf
```

Тут появляется новая комманда  `restic forget --verbose` и флаг `--keep-daily`. Она нужна для очистки старых снапшотов, а флаг позволяет настроить политику хранения снапшотов т.е. за последние n дней, которые имеют один или более снапшотов, сохранять только самый последний для каждого дня. `forget` удаляет только снапшоты, но не сами данные.

Для удаления данных сделаем другой сервис. Возможно возникнет вопрос, а почему не добавить `prune` в первый сервис, можно и так, но дело в том, что команды будут запускаться с разной периодичностью, поэтому они разделены.

```
[Unit]
Description=Restic backup service (data pruning)
[Service]
Type=oneshot
ExecStart=restic prune
EnvironmentFile=%h/.config/restic-backup.conf
```

Команда `prune` будет удалять данные на которые ссылаются удаленные снапшоты.

Теперь сделаем таймеры для сервисов в той же директории.

`~/.config/systemd/user/restic-backup.timer`

```
[Unit]
Description=Backup with restic daily
[Timer]
OnCalendar=daily
Persistent=true
[Install]
WantedBy=timers.target
```

`OnCalendar` задает время запуска, у меня сервис запускается ежедневно в 12 ночи, поэтому стоит значение `daily`. 

`~/.config/systemd/user/restic-prune.timer`

```
[Unit]
Description=Prune data from the restic repository monthly
[Timer]
# This will run on the 1st of every month at 2AM
OnCalendar=*-*-01 02:00:00
Persistent=true
[Install]
WantedBy=timers.target
```

Сервис для очистки будет запускаться раз в месяц каждое первое числа в 2 часа ночи . Я указал 2 часа ночи, чтобы не было конфликта с доступом к репозиторию у сервисов.

Для работы сервисов нужно перезапустить менджер systemd, чтобы они подхватились:

```bash
systemctl --user daemon-reload
```

И запустить таймеры:

``` bash
systemctl --user enable --now restic-backup.timer
systemctl --user enable --now restic-prune.timer
```

На этом этапе все настройки завершены и если все сделано правильно restic будет делать ежедневные бэкапы и очищать старые раз в месяц. Вместо MinIO можно использовать любое другое хранилище, список поддерживаемых restic'ом можно посмотреть [здесь](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html).

## Полезные ссылки

[Restic Documentation](https://restic.readthedocs.io/en/stable/index.html)\
[MinIO docker install documentation](https://min.io/docs/minio/container/index.html)\
[systemd service](https://www.freedesktop.org/software/systemd/man/systemd.service.html)\
[systemd timer](https://www.freedesktop.org/software/systemd/man/systemd.timer.html)