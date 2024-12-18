---
title: "DYI Домашний сервер a.k.a. NAS"
date: 2024-12-08T03:54:00+01:00
description: "Как я собрал свой домашний сервер a.k.a NAS"
tags: [nas, open-source, home-server]
draft: false
---
## Предыстория

В декабре 2021 года моя девушка подарила мне Raspberry Pi 4B 2GB (видимо слишком часто рассказывал ей об этой плате 😄). Сразу же были заказаны все необходимые комплектующие т.к. плата из коробки идет даже без блока питания. Raspberry Pi была в роли слабенького ПК на Raspbian OS, ретро консоли на RetroPie, Android TV правда только с программным кодированием. Когда купил внешний HDD на 2 ТБ я подумал, а почему бы не сделать мини-сервер для бэкапов устройств по сети. Так я установил Raspbian OS без UI, поставил S3 хранилище и OwnCloud.

![Вот так это выглядело: Raspberry Pi + внешний HDD](IMG_20230526_105101.jpg)
Вот так это выглядело: Raspberry Pi + внешний HDD
<br>

Для локальных бэкапов такое решение вполне себе, если еще найти корпус, чтобы выглядело красивее. Но есть и минусы. С точки зрения надежности на внешнем диске лучше не хранить ничего важного т.к. нет зеркалирования. Можно было подключить второй такой диск и сделать RAID 1, но выглядело бы это ну очень громоздко и некрасиво. Для архитектуры ARM (да, Raspberry Pi это не x86) бывало не мог найти какое-то ПО. Производительности платы хватало на пару сервисов и расширить RAM было никак нельзя.

![Это все было развернуто на Raspberry Pi](2024-12-08%2013.27.06.jpg)
Это все было развернуто на Raspberry Pi
<br>

До недавнего времени меня все устраивало, пока я не получил уведомление от Google, что облачное хранилище скоро заполнится. Я бы мог оплатить подписку и получить дополнительные 100GB, но это не мой путь.

![Это, конечно, должно было произойти когда-нибудь](SCR-20241208-mcqg.png)
Это, конечно, должно было произойти когда-нибудь
<br>

## Начало пути

Сначала я хотел купить готовый NAS (Network-attached storage), но увидев цены передумал. На тот момент я не понимал почему они такие дорогие. За очень слабое железо и корпус такие деньги. Дело в том, что это цена за готовое решение. Пришел домой, включил, вставил диски, протыкал "далее" и у тебя готовое хранилище с доступом из любой точки мира. Если железо позволяет, можно даже поднять сервисы для стриминга и умного хранения фото.

![Да, цена, конечно, космос](dns_synology_scr.png)
Да, цена, конечно, космос
<br>

Есть NAS'ы от TerraMaster, они сильно дешевле Synology, но по отзывам ОС там не дотягивает до ОС от Synology, но по железу они вполне неплохие.

<center><img src="../../static/posts/dyi-home-server-aka-nas/dns_terramaster_scr.png" ...></center>
<br>

![](dns_terramaster_scr.png)
<br>

Но я не хотел готовое решение, хотелось собрать самому и желательно за небольшие деньги. Сначала я планировал NAS из мини ПК и DAS (Direct-attached storage). DAS отличается от NAS тем, что это просто коробка куда можно вставить диски и подключить ее к ПК по USB.

![](das_scr.png)
![](IMG_20241109_110636.jpg)
<br>

И да, я купил мини ПК, но вовремя понял, что мини пк + DAS мне не подойдет. Хоть недорогой мини ПК и на архитектуре x86, он все равно будет слабым по производительности + высокие температуры в таком маленьком корпусе. В том, что я брал стоял Intel N5095 и 8GB RAM. Подключать диски по USB не хотелось. Ну и выглядело бы это не так аккуратно, как я бы хотел. Мини ПК был продан. Я решил собирать NAS сам, брать БУ железо я не планировал, даже с учетом нового железа, сборка будет дешевле и сильно мощнее чем в готовых NAS, а также можно будет легко заменить любой компонент.

## DYI NAS

Самое первое с чего стоит начать это корпус. Он должен вмещать в себя много HDD 3.5 дисков. Мне порекомендовали Jonsbo N4. В него можно поставить 8 дисков (6x 3.5, 2x 2.5), Карл! Производитель позиционирует решение, как раз для сборок под NAS. А еще он выглядит стильно на мой взгляд и достаточно компактный.

![](jonsbo_n4_case.png)
<br>

Блок питания и материнскую плату я взял просто качественные. CPU Intel Core i3 12100 4 core, 8 threads. Более производительный CPU я не вижу смысла пока ставить. 16 GB RAM, 240 GB SSD NVMe под систему, башенный кулер ну и пара дисков Seagate IronWolf 4TB. У меня еще был старый HDD от Western Digital на 750GB, которому уже много лет. Но и его можно использовать.

![Диски приехали позже](IMG_20241114_211348.jpg)
Диски приехали позже
<br>

Собрал за вечер, никаких сложностей в процессе не было.

![](IMG_20241115_003737.jpg)
<br>

![](IMG_20241115_082621.jpg)
<br>

По температурам CPU все было супер, кулера более чем хватало. Но когда я установил все диски, то заметил, что температура дисков высоковата для работы в режиме 24/7. А я планирую еще ставить диски...

![](scrutiny_high_temp.jpg)
<br>

Проблема этого корпуса на мой взгляд в том, что если поставить все 8 (а я поставил пока только 3) дисков, то они будут выделять тепло, которое плохо вытягивает задний вентилятор. На Reddit нашел пост, где была такая же проблема с высокими температурами дисков. В комментариях один пользователь рассказал, как он решил эту проблему. Он спроектировал и распечатал на 3D принтере панель на котороую можно установить 2 вентилятора на вдув и они будут охлаждать диски. И это никакой не колхоз, все выглядит красиво и крепиться на магнитах. Я заказал на Авито печать самой панели, заказал 2 кулера и набор магнитов. Через пару дней все было готово. После сборки и установки это выглядит вот так (снял крышку для фото)

![](IMG_20241123_121518.jpg)
<br>

После установки панели с доп. охлаждением, температуры упали.

![](scrutiny_low_temp.jpg)
<br>

В качестве операционной системы я выбрал Ubuntu Server, я с ней хорошо знаком, есть много гайдов и софта. Сделал программный RAID 1 из двух дисков по 4TB для важных данных. Все остальное, что нежалко потерять храню на старом диске, как он помрет, возьму, наверное тоже что-то от Western Digital. Сейчас NAS стоит в прихожей и не занимает места. К нему я планирую еще купить ИБП, больше для того, чтобы он выключался безопасно и затем включался при появлении питания. Это же все таки сервер, он должен жить с минимальным вмешательством с моей стороны.

Немного расскажу про мои сценарии использования. Во первых доступ к серверу есть извне, это очень круто. Во вторых я перенес все свои данные и данные близких из Google, все храниться теперь на нем. Фильмы/сериалы теперь можно смотреть через свой собственный стриминг где угодно и на чем угодно. И управлять процессом хоть со смартфона. Слушать свою коллекцию треков со школьных времен в любое время. Настроил регулярный бэкап через TimeMachine по сети, это гораздо удобнее чем бегать с внешним диском к ноутбуку. Вообще имея домашний сервер можно делать много интересного и решать разные задачи.

## Сервисы, которые использую

![](dashboard.png)
<br>

1. **NextCloud** - аналог Google Drive. Свое облако для хранения файлов.

2. **Immich** - аналог Google Photo. Умеет распознавать лица, локации, искать по фото (на англ.). Применяется ML для индексации фото/видео. 

3. **Minio** - S3 совместимое хранилище. Здесь храню большие файлы - архивы и бэкапы.

4. **Scrutiny** - мониторинг дисков. Позволяет отслеживать состояние дисков, получать уведомления, если с каким-то из дисков есть проблемы.

5. **Pi-hole** - блокировка рекламы по всей внутренней сети и локальный DNS.

6. **Beszel** - мониторинг системы.

7. **Portainer** - управление Docker контейнерами.

8. **Filebrowser** - простой браузер файлов. Иногда это удобно, особенно с телефона

9. **Nginx Proxy Manager** - прокси сервер. Использую для роутинга запросов.

10. **Glances** - аналог top, только в браузере.

11. **Cockpit** - панель управления для ОС, если не хочется лезть в терминал.

12. **SpeedTest** - сервис для замера скорости к серверу.

13. **Excalidraw** - здесь рисую всякие разные блок схемы.

14. **Navidrome** - стримминг аудио. Слушаю музыку из школьных времен.

15. **Jellyfin** - стримминг медиа контента.

16. **QBittorrent** - торрент клиент для загрузки любимых Linux дистрибутивов.

## F.A.Q.

1. **Зачем в 2024 году HDD, есть же SSD?**\
    Скорость SSD избыточна, если только у вас нет 10 Gbit'ной сети, в лучшем случае у вас 1Gbit, а это 125 мбайт. Скорость HDD дисков где-то 200 мбайт. И 125 мбайт не будет даже по проводу (я максимум смог получить 111 мбайт). Цена за гигабайт у HDD все еще ниже чем у SSD, 2 HDD диска по 4 TB можно купить дешевле чем 1 SSD на 4 TB. Ну и важно понимать вид нагрузки, если часто происходит перезаписть, то SSD здесь не подойдет.

2. **Безопасно ли так хранить свои данные?**\
    Безопасное хранение достигается зеркалированием дисков (см. RAID 1). Регулярными бэкапами. В идеале по стратегии [3-2-1](https://www.backblaze.com/blog/the-3-2-1-backup-strategy/). Да, всегда что-то может пойти не так, но если все делать правильно, то шанс потерять свои данные не сильно выше чем потерять их в облаке. Я сделал RAID 1 на случай, если один из дисков выйдет из строя и делаю регулярные бэкапы на внешний диск. В будущем я куплю готовый недорогой NAS, поставлю его вообще в другой дом/город и буду делать дополнительно бэкапы и на него.

3. **Есть смысл переплачивать за готовый NAS?**\
    Да, есть. Вы получаете готовое решение, вам не надо заморачиваться с выбором и сборкой железа, где можно допустить ошибку. Не надо настраивать ОС и софт. Если хочется получить свое персональное облако здесь и сейчас не теряя время, то лучше заплатить и купить готовое решение, оно того стоит.

4. **Почему в качестве ОС не выбрал TrueNAS, Xpenology, OpenMediaVault?**\
    На мой взгляд эти ОС создавались под определенную задачу и возможно решают ее хорошо, но я не хотел быть ограничен ею и мне комфортнее взять general purpose OS и настроить все, что мне необходимо с нуля.

5. **Почему RAID 1 программный, а не аппаратный?**\
    Материнская плата в моем NAS поддерживает аппартный RAID 1, но в случае ее выхода из строя, мне придется искать точно такую же материнскую плату, чтобы восстановить работу RAID. Программный RAID 1 позволяет независеть от железа.

6. **Почему не собрал из БУ железа?**\
    Не хотел тратить время на поиск, ну и БУ железо это такое, никогда не знаешь, где выстрелит. Но если очень хочется и понятны все риски, то почему бы и да.

7. **Сколько это все стоит и можно список?**\
    Список комплектующих:

    **Case:**	Jonsbo N4 Black\
    **Motherboard:**	ASRock B760M Pro RS\
    **PSU:**	Chieftec COMPACT 450W\
    **CPU:**	Intel Core i3-12100 OEM\
    **CPU Fan:**	ID-COOLING IS-47-XT\
    **RAM:**	Kingston Fury Beast Black AMD 16 GB (2x8 GB)\
    **SSD:**	Kingston NV2 250 ГБ M.2 NVMe\
    **HDD:**	Seagate IronWolf 4 TB (ST4000VN006) x2

    На 12.11.2024 стоимость без дисков `550$`, с дисками `810$`