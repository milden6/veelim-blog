---
title: "Unix-way хранение паролей"
date: 2023-06-11T11:24:00+01:00
description: "Менеджер паролей pass + GPG + Git + Termux"
tags: [gpg, open-source, pass, termux]
draft: false
---

Использовать везде один и тот же пароль  - это плохая идея.  Хорошо, когда для каждого сайта есть свой уникальный и надежный пароль, но держать все в голове неудобно. Практичнее всего воспользоваться менеджером паролей.

Если нужен хороший менеджер паролей с открытым исходным кодом, но без лишних сложностей, могу порекомендовать [Bitwarden](https://bitwarden.com/) и [KeePassXC](https://keepassxc.org/).

**Bitwarden** - это самый удобный вариант в плане синхронизации. Пароли хранятся на сервере компании Bitwarden, Inc., но в [хешированном виде](https://bitwarden.com/blog/vault-security-bitwarden-password-manager/), даже если база будет взломана, то пароли будет не так просто заполучить. Бесплатный план без ограничений на количество хранимых паролей. Если не доверяете Bitwarden, Inc. можно развернуть свой сервер. Есть две реализации: [официальная](https://github.com/bitwarden/server) на C#, требовательная к ресурсам и [неофициальная](https://github.com/dani-garcia/vaultwarden), написанная на Rust с минимальными требованиями. Я сам использую Bitwarden и мне он нравится.

С **KeePassXC** о синхронизации нужно думать самому, он хранит пароли в локальной БД.

Теперь перейдем к способу для тех, кто не ищет легких путей.

## pass

Простой менеджер паролей, написанный в [720](https://git.zx2c4.com/password-store/tree/src/password-store.sh#n720) строк на bash. По большому счету это обертка для удобной работы с gpg и git.

Официальный сайт - https://www.passwordstore.org/

## GPG

CLI утилита для шифрования/подписи данных. 

Официальный сайт - https://gnupg.org/

## Git

Система контроля версий.

Официальный сайт - https://git-scm.com/

## Termux

Эмулятор терминала для Android.

Официальный сайт - https://termux.dev/en/

Обычно **gpg** установлена на Unix системах, если ее нет можно поставить через пакетный менеджер.

В первую очередь нужно сгенерировать gpg ключ:

``` bash
# версия gpg
gpg --version
gpg (GnuPG) 2.4.0
libgcrypt 1.10.2-unknown

# генерация ключа
gpg --full-generate-key
# выбираем тип ключа (1) RSA and RSA и жмем Enter
Please select what kind of key you want:
   (1) RSA and RSA
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
   (9) ECC (sign and encrypt) *default*
  (10) ECC (sign only)
  (14) Existing key from card
Your selection? 1

# указываем длину 4096 и жмем Enter
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (3072) 4096

# указываем, что ключ бессрочный и жмем Enter
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 0

# говорим, что все верно и жмем Enter
Is this correct? (y/N) y

GnuPG needs to construct a user ID to identify your key.

# указываем ваше имя
Real name: Veelim
# указываем почту
Email address: veelim@testmail.com
# указываем комментарий
Comment:
You selected this USER-ID:
    "Veelim <veelim@testmail.com>"

# пишем, что все окей
Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O

# на этом этапе нужно ввести секретную фразу, которая нужна для доступа к самим ключам.
# ВАЖНО! Секретная фраза должна быть надежной и больше нигде не должна использоваться.

# ключи сгенерированы
pub   rsa4096 2023-06-11 [SC]
      712C334BC909781880EABFFF1F3E8C0E6D61E711
uid                      Veelim <veelim@testmail.com>
sub   rsa4096 2023-06-11 [E]
```

Посмотреть ключи:

``` bash
# публичные ключи
gpg -k --keyid-format long
-------------------------------------
pub   rsa4096/1F3E8C0E6D61E711 2023-06-11 [SC]
      712C334BC909781880EABFFF1F3E8C0E6D61E711
uid                 [ultimate] Veelim <veelim@testmail.com>
sub   rsa4096/5B978F7103A58A63 2023-06-11 [E]

# приватные ключи
gpg -K --keyid-format long
-------------------------------------
sec   rsa4096/1F3E8C0E6D61E711 2023-06-11 [SC]
      712C334BC909781880EABFFF1F3E8C0E6D61E711
uid                 [ultimate] Veelim <veelim@testmail.com>
ssb   rsa4096/5B978F7103A58A63 2023-06-11 [E]
```

**pub** - публичный мастер-ключ.\
**sec** - секретный мастер-ключ.\
**uid**(user id) - ID пользователя.\
**sub** - публичный саб-ключ.\
**ssb** - секретный саб-ключ.\
**\[SC\]** - предназначение ключа: S - подпись данных, C - сертификация ключей.\
**\[E\]** - шифрование данных.

Создадим отдельный саб-ключ для шифрования паролей:

``` bash
# переходим в режим редактирования мастер ключа
gpg --expert --edit-key veelim@testmail.com

Secret key is available.

sec  rsa4096/1F3E8C0E6D61E711
     created: 2023-06-11  expires: never       usage: SC
     trust: ultimate      validity: ultimate
ssb  rsa4096/5B978F7103A58A63
     created: 2023-06-11  expires: never       usage: E
[ultimate] (1). Veelim <veelim@testmail.com>

# пишем addkey
gpg> addkey

# выбираем (8) RSA (set your own capabilities)
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (12) ECC (encrypt only)
  (13) Existing key
  (14) Existing key from card
Your selection? 8

# выбираем только Encrypt
Possible actions for this RSA key: Sign Encrypt Authenticate
Current allowed actions: Encrypt

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection?

# указываем длину 4096
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (3072) 4096

# ключ будет валидным 1 год
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 1y

# подтверждаем
Is this correct? (y/N) y
Really create? (y/N) y

# сохраняем изменения
gpg> save

# посмотреть созданный ключ
gpg -k --keyid-format long

sub   rsa4096/080EFEAF38155D62 2023-06-11 [E] [expires: 2024-06-10]
```
> Хорошее правило всегда создавать отдельный саб-ключ для разных задач/устройств/программ. Например, можно сделать 3 саб-ключа, один для подписи файлов, второй для шифрования, а третий для SSH соединений.

Для безопасности экспортируем и удалим мастер-ключ, и саб-ключи. Затем импортируем только саб-ключи и оставим один созданный саб-ключ.

``` bash
# экспортируем секретный мастер-ключ
gpg --export-secret-key -a veelim@testmail.com > veelim_secret_key.gpg

# экспортируем публичный мастер-ключ
gpg --export -a veelim@testmail.com > veelim_public_key.gpg

# экспортируем саб-ключи
gpg --export-secret-subkeys -a > veelim_sub_keys.gpg

# удаляем секретную и публичную часть ключа
gpg --delete-secret-and-public-key veelim@testmail.com

# проверяем, что ничего нет
gpg -k --keyid-format long
gpg -K --keyid-format long

# импортируем только саб-ключи
gpg --import veelim_sub_keys.gpg

# проверяем, что саб-ключ добавился
gpg -k --keyid-format long
gpg -K --keyid-format long

# удаляем все ключи кроме созданного
gpg --expert --edit-key veelim@testmail.com

Secret subkeys are available.

pub  rsa4096/1F3E8C0E6D61E711
     created: 2023-06-11  expires: never       usage: SC
     trust: unknown       validity: unknown
ssb  rsa4096/5B978F7103A58A63
     created: 2023-06-11  expires: never       usage: E
ssb  rsa4096/080EFEAF38155D62
     created: 2023-06-11  expires: 2024-06-10  usage: E
[ unknown] (1). Veelim <veelim@testmail.com>

# выбираем ненужный ключ, он выделится звездочкой *
gpg> key 1

# удаляем
gpg> delkey

# после того, как ключ был импортирован нужно сказать gpg, что мы ему доверяем
gpg> trust
Please decide how far you trust this user to correctly verify other users keys
(by looking at passports, checking fingerprints from different sources, etc.)

  1 = I don't know or won't say
  2 = I do NOT trust
  3 = I trust marginally
  4 = I trust fully
  5 = I trust ultimately
  m = back to the main menu

Your decision? 5

# сохраняем изменения
gpg> save
```

Теперь, когда готов gpg ключ переходим к работе с **pass**. На [официальном сайте](https://www.passwordstore.org/) есть инструкция по установке на все поддерживаемые системы.

Инициализируем хранилище pass:

``` bash
# версия pass
pass --version
============================================
= pass: the standard unix password manager =
=                                          =
=                  v1.7.4                  =
=                                          =
=             Jason A. Donenfeld           =
=               Jason@zx2c4.com            =
=                                          =
=      http://www.passwordstore.org/       =
============================================

# указываем идентификатор gpg ключа
# pass создаст директорию /home/user/.password-store/.
pass init 080EFEAF38155D62
```

У pass небольшой manual, поэтому разобраться в нем будет несложно.

Создадим пару паролей:

``` bash
pass insert Social/telegram
Enter password for Social/telegram:
Retype password for Social/telegram:

# pass сгенерирует пароль длиной в 40 символов
pass generate Email/mymail 40

# можно многострочный файл
pass insert Social/redit -m
Enter contents of Social/redit and press Ctrl+D when finished:
# пароль лучше писать всегда первой строкой, чтобы можно было копировать его в буфер
test1234password
# пишем нужную информацию об аккаунте
username: Veelim
email: veelim@testmail.com
```

Вот такая получилась структура репозитория

``` bash
pass                                                                                         
Password Store
├── Email
│   └── mymail
└── Social
    ├── redit
    └── telegram
```

Пара полезных команд:

``` bash
# вывести пароль
pass Social/telegram

# скопировать в буфер
pass Social/telegram -c
Copied Social/telegram to clipboard. Will clear in 45 seconds.

# удалить запись
pass rm Social/telegram
```

Для синхронизации паролей между устройствами воспользуемся git репозиторием. Я использую [gitea](https://gitea.io/en-us/), поднятый на моем сервере.

После создания репозитория нужно запушить в него содержимое директории password-store.

``` bash
#версия git
git version
git version 2.40.1

# инициализируем git репозиторий
pass git init

# добавляем удаленный репозиторий
pass git remote add origin git@my-git-server:user/password-store.git

# пушим в удаленный репозиторий
pass git push -u --all
```

Пароли в удаленном репозитории и их можно синхронизировать между устройствами.

## Windows

Я использую [gopass](https://www.gopass.pw/) под Windows. После загрузки нужно добавить его в `Path`.

``` bash
# версия gopass
gopass> version
gopass 1.15.5 go1.20.2 windows amd64

# склонируем git репозиторий
gopass> clone git@my-git-server:user/password-store.git

# проверим структуру репозитория
gopass> ls
gopass
├── Email/
│   └── mymail
└── Social/
    ├── redit
    └── telegram

# экспортируем саб-ключ с Linux машины
gpg --export-secret-subkeys -a > veelim_pass_sub_key.gpg

# импортируем саб-ключ в Windows машину
gpg --import veelim_pass_sub_key.gpg

# делаем ключ доверенным
gpg --expert --edit-key veelim@testmail.com
gpg> key 1
gpg> trust
Please decide how far you trust this user to correctly verify other users keys
(by looking at passports, checking fingerprints from different sources, etc.)

  1 = I don't know or won't say
  2 = I do NOT trust
  3 = I trust marginally
  4 = I trust fully
  5 = I trust ultimately
  m = back to the main menu

Your decision? 5

# сохраняем изменения
gpg> save
```

Добавим новый пароль и запушим его

``` bash
gopass> generate Email/anotheremail 26
✅ Password for entry "Email/anotheremail" generated
Not printing secrets by default. Use 'gopass show Email/anotheremail' to display the password.
gopass> ls
gopass
├── Email/
│   ├── anotheremail
│   └── mymail
└── Social/
    ├── redit
    └── telegram

gopass> sync
🚥 Syncing with all remotes ...
[<root>]
   gitfs pull and push ... OK (no changes)
   done
✅ All done
```

## Android

Скачиваем из F-Droid:
- [Termux](https://f-droid.org/en/packages/com.termux/)
- [Termux:API](https://f-droid.org/en/packages/com.termux.api/)

Открываем на телефоне Termux и выполняем следующие команды:

``` bash
# обновляем пакеты
pkg upgrade

# ставим полезные termux пакеты
pkg install termux-api

# устанавливаем pass, зависимости подтянутся автоматически
pkg install pass

# закидываем саб-ключ в termux
termux-storage-get veelim_pass_sub_key.gpg

# импортируем саб-ключ в Termux
gpg --import veelim_pass_sub_key.gpg

# делаем ключ доверенным
gpg --expert --edit-key veelim@testmail.com
gpg> key 1
gpg> trust
Please decide how far you trust this user to correctly verify other users keys
(by looking at passports, checking fingerprints from different sources, etc.)

  1 = I don't know or won't say
  2 = I do NOT trust
  3 = I trust marginally
  4 = I trust fully
  5 = I trust ultimately
  m = back to the main menu

Your decision? 5

# сохраняем изменения
gpg> save

# инициализируем директорию pass
pass init 080EFEAF38155D62

# инициализируем git репозиторий
pass git init

# добавляем удаленный репозиторий
pass git remote add origin git@my-git-server:user/password-store.git

# подтягиваем пароли из репозитория
pass git pull origin master

# проверяем
pass
Password Store
├── Email
│   ├── anotheremail
│   └── mymail
└── Social
    ├── redit
    └── telegram

# копируем пароль в буфер устройства
pass Email/anotheremail -c
Copied Email/anotheremail to clipboard. Will clear in 45 seconds.
```

На этом настройка завершена. Теперь pass есть на трех устройствах, пароли синхронизируются через Git. Очень важно придумать надежную секретную фразу для gpg ключа и позаботиться о его безопасности. Мастер ключ лучше забэкапить в надежное место и не забывать продлевать саб-ключи.

## F.A.Q.

1. **Зачем так сложно?**\
   Мне нравится пользоваться терминалом, мне нравится минимализм pass и самое главное - для меня это удобно.
   
2. **Почему не Password Store и OpenKeyChain на Android устройстве?**\
   Я пробовал эту связку, оказалось, что она не работает без танцев с бубном. Первая проблема с которой столкнулся - неспособность OpenKeyChain расшифровать пароли ключом, созданным в актуальной версии gpg. Подробнее можно почитать [тут](https://dev.gnupg.org/T6133), там же и фикс. Вторая проблема - Password Store не может создавать новые ключи, описание и фикс [тут](https://github.com/open-keychain/open-keychain/issues/2588#issuecomment-778886913) . Еще важно, что активная разработка OpenKeyChain [прекращена](https://github.com/open-keychain/open-keychain/commit/6f38af15828e07f0186109a89b7aa57f54101cfa). Автор Password Store вроде как [планирует выпустить новую версию](https://github.com/android-password-store/Android-Password-Store/issues/1523) без OpenKeyChain, но когда и как оно будет работать неизвестно.

3. **Почему не QtPass или Pass4Win на Windows?**\
   Они мне не нравятся. С gopass также удобно работать в терминале.

4. **Почему везде не использовать gopass?**\
   Так исторически сложилось, что с gopass познакомился позже. Думаю, что можно использовать и его.

5. **Что будет если потерять gpg ключи?**\
   Будет грустно. К хранению ключей нужно подходить ответственно.

## Полезные ссылки

[The GNU Privacy Handbook](https://gnupg.org/gph/en/manual.html)\
[pass man](https://git.zx2c4.com/password-store/about/)\
[Quick-start guide to GPG](https://github.com/bfrg/gpg-guide)