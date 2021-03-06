Как наши студенты создали MVP: инструкция по сбору и отправке сообщений из Telegram в Slack

[Яндекс.Практикум](https://m.vk.com/yandex.practicum)·16 сен в 17:11

В этой статье расскажем о том, как за относительно небольшой срок (около трех недель) студенты бэкенд-факультета Практикума решили прикладную задачу, сочетающую в себе целый букет технологий.

Решение не претендует на эксклюзивность и эталонность и будет интересно тем, кто только задумался над тем, как транслировать данные между двумя мессенджерами.

Отметим, что несмотря на свою простоту и минималистичность, приложение без сбоев работает больше 8 месяцев и делает ровно то, что от него требовалось.

Задача

У Яндекс.Практикума есть сообщество выпускников в слаке. Одна из важных задач сообщества в том, чтобы регулярно оповещать участников о разных IT-мероприятиях ― хакатонах, конференциях, митапах и т.д.

Сначала эту задачу вручную выполняли админы сообщества. Затем, в конце 2020 года, мы решили попробовать автоматизировать её.

Чтобы совместить полезное (высвободить ресурс админов) с ещё более полезным (превратить выполнение боевой задачи в учебный проект), основной фронт работы по проектированию программы и написанию кода был отдан студентам бэкенд-факультета. Наставником на проекте был один из членов команды Практикума, сам в свое время отучившийся на бэкенд-факультете.

В общих чертах задача выглядела так:

дан список телеграм-каналов, в которых публикуют, в том числе, сообщения об IT-мероприятиях;

есть универсальные каналы (постят сообщения для широкой IT-аудитории), есть специализированные каналы для конкретного IT-направления, например Data Science;

нужно в режиме 24/7 собирать только релевантную информацию из этих каналов и ежедневно в 12.00 постить её в сообществе выпускников в слаке;

поскольку у каждого факультета практикума есть свой отдельный канал внутри сообщества, сообщения должны отправляться в соответствующий канал;

исключение ― сообщения о хакатонах и прочих соревнованиях ― независимо от тематики мероприятия сообщения должны отправляться в один канал.

Инструменты

Язык разработки ― Python.

Стандартные библиотеки:

datetime ― для работы с датой и временем публикации сообщений в ТГ-каналах;

json ― запись релевантных сообщений из ТГ-каналов в json-файл и чтение из него при отправке сообщений в сообщество выпускников;

re ― фильтрация текстов сообщений в ТГ-каналах через регулярные выражения;

\3. Сторонние библиотеки:

[Pyrogram](https://m.vk.com/away.php?to=https%3A%2F%2Fdocs.pyrogram.org%2F) ― ведущая библиотека проекта, используем для работы с Telegram Client API;

requests ― для отправки в слак POST-запросов с сообщениями о мероприятиях.

\4. Развертывание и запуск приложения ― Docker.

\5. Удалённый сервер ― виртуальная машина на Ubuntu в Yandex.Cloud.

Начинаем разработку: Telegram-клиент

Ядро нашего приложения ― Telegram-клиент, который в режиме 24/7 слушает интересующие каналы.

В отличие от бота, который не добавишь в публичные ТГ-каналы, клиент является клоном пользователя и все действия совершает от его имени. Пользователю достаточно вступить в канал, тогда клиент сможет фиксировать всё происходящее в канале.

Что нужно:

На сайте telegram [зарегистрировать приложение](https://m.vk.com/away.php?to=https%3A%2F%2Fmy.telegram.org%2Fapps), получить API ID и API hash.

С помощью модуля Client библиотеки pyrogram создать клиента.

Создание Telegram-клиента

Собираем исходные данные

В отдельный файл мы собрали следующую информацию:

ID всех интересующих ТГ-каналов.

Списки каналов по темам, список универсальных каналов, список всех каналов.

ID всех каналов в слаке, куда будут перенаправляться сообщения из ТГ-каналов.

Стоп-слова, отсекающие рекламный контент.

Ключевые слова, по которым клиент будет отбирать релевантные сообщения из каналов. Например, 'collaborations': r'хакатон|соревн|олимпиад|отборочный тур'.

Настраиваем слушателя событий

Слушателя событий в Pyrogram можно объявить с помощью [декоратора](https://m.vk.com/away.php?to=https%3A%2F%2Fdocs.pyrogram.org%2Fapi%2Fdecorators). Один из них ― on_message ― позволяет перехватывать сообщения.

Разумеется, нам не нужно, чтобы наше приложение реагировало вообще на абсолютно все сообщения, которые проходят в аккаунте юзера, от имени которого работает клиент.

Гибко настроить то, на какие сообщения реагировать, помогают многочисленные [фильтры](https://m.vk.com/away.php?to=https%3A%2F%2Fdocs.pyrogram.org%2Ftopics%2Fuse-filters) Pyrogram. Фильтры можно комбинировать.

Для нашего приложения мы подобрали следующую комбинацию фильтров:

Комбинация фильтров

Разберём, что означает этот комбинированный фильтр.

Фильтр chat указывает, что клиент реагирует на сообщения не во всех чатах, группах, каналах подряд, а только в списке определенных каналов, который мы обозначили переменной ALL_CHANNELS.

Фильтр edited реагирует на сообщения, которые редактировали после создания. С помощью оператора ~ мы инвертируем этот фильтр, т.е. клиент не будет реагировать на отредактированные сообщения. Главным образом, этот фильтр появился из-за сообщений с кнопками для голосования. Каждое голосование порождало у сообщения статус «отредактированное», а значит, оно могло неопределенное количество раз улавливаться слушателем.

С точки зрения оформления интересующие нас сообщения в ТГ-каналах можно разделить на две группы: классические текстовые сообщения и сообщения в виде картинки, под которой расположен текст. Второй вид сообщений не являются текстовыми, их тип caption. Поэтому фильтр filters.text | filters.caption ловит и те, и другие сообщения (оператор | означает логическое ИЛИ).

Содержание сообщений мы фильтруем по регулярным выражениям, для этого предназначен [фильтр regex](https://m.vk.com/away.php?to=https%3A%2F%2Fdocs.pyrogram.org%2Fapi%2Ffilters%3Fhighlight%3Dregex%23pyrogram.filters.regex). Мы используем его дважды ― чтобы отловить сообщения с подходящим содержанием и чтобы отсечь сообщения с неподходящим содержанием.

Таким образом, по результатам обработки перехваченного сообщения у нас есть:

Список из одной или нескольких тем, к которым относится сообщение.

Словарь с данными о сообщении.

Заключительный шаг - записать полученную информацию. На уровне MVP мы не стали создавать базу данных, тем более какая-то аналитика по истории сообщений в общем-то и не нужна. Решили записывать сообщения в json-файл. Изначально он выглядит так:

{

"QA_events": [],

"DS_events": [],

"backend_events": [],

"frontend_events": [],

"another_events": [],

"collaborations": []

}

Названия ключей в json полностью совпадают с названиями тем, к которым может быть отнесено сообщение. Мы запускаем цикл по списку тем, который сформирован для сообщения, и записываем данные в массив по соответствующему ключу в json-файле.

После отправки сообщений из json-файла списки по ключам очищаются.

Настраиваем прием сообщений от telegram-клиента в Slack

Чтобы сообщения попадали в сообщество выпускников в слаке, нужно настроить “принимающую сторону” - слак-бота, который будет постить сообщения. Для этого необходимо:

Создать приложение [https://api.slack.com/apps?new_app=1](https://m.vk.com/away.php?to=https%3A%2F%2Fapi.slack.com%2Fapps%3Fnew_app%3D1)

Достаточно задать имя и указать workspace, в котором будет жить приложение.

В разделе Add features and functionality настроек приложения выбрать Bot, т.е. наше приложение будет ботом.

В разделе Install your app настроек приложения нажать Install to Workspace и установить приложение в workspace. Для бесплатного тарифного плана слака есть ограничение - не более 10 приложений.

Ознакомиться с доступными API-методами [https://api.slack.com/methods](https://m.vk.com/away.php?to=https%3A%2F%2Fapi.slack.com%2Fmethods) и выбрать нужные. Например, для отправки ботом сообщений в канал понадобится метод [chat.postMessage.](https://m.vk.com/away.php?to=https%3A%2F%2Fapi.slack.com%2Fmethods%2Fchat.postMessage)

Посмотреть в описании метода, какие разрешения (required scopes) нужны для того, чтобы бот смог его использовать. Например, для использования метода chat.postMessage нужно разрешение chat:write.

Наделить бота необходимым разрешением.

Для этого в настройках приложения (все приложения воркспейса доступны тут [https://api.slack.com/apps](https://m.vk.com/away.php?to=https%3A%2F%2Fapi.slack.com%2Fapps)) нужно:

выбрать раздел OAuth & Permissions

далее Bot Token Scopes

нажать Add an OAuth Scope и выбрать нужное разрешение

В этом же разделе доступен токен бота - Bot User OAuth Access Token.

Программируем отправку сообщений в Slack

Логика отправки сообщений в слак достаточно проста.

Все каналы-получатели объединяются в словарь, ключами в котором являются темы. Нетрудно заметить, что эти ключи совпадают с ключами в json-файле.

SLACK_CHANNELS = {

'QA_events': 'C0172LTCDR8',

'DS_events': 'C01714KE2CF',

'backend_events': 'C01BPFF6CHJ',

'frontend_events': 'C01F01MKGER',

'another_events': 'G01H3EM2HCN',

'collaborations': 'C018H32RTCN',

}

По этим ключам запускаем цикл и на каждой итерации мы смотрим, есть ли что-то по этому ключу в json-файле. Если есть, то запускается вложенный цикл, в рамках которого каждое сообщение по ключу отправляется в соответствующий тематический канал в slack.

Непосредственно отправка сообщения в слак реализуется с помощью POST-запроса со следующими характеристиками:

url - [https://slack.com/api/chat.postMessage](https://m.vk.com/away.php?to=https%3A%2F%2Fslack.com%2Fapi%2Fchat.postMessage)

в заголовке запроса передаем токен бота

в json передаем данные о сообщении.

Запускаем приложение на сервере

На удаленном сервере в Яндекс.Облаке мы обернули наше приложение в докер-контейнер и запустили в нем скрипт с телеграм-клиентом. Благодаря pyrogram-методу [run](https://m.vk.com/away.php?to=https%3A%2F%2Fdocs.pyrogram.org%2Fapi%2Fmethods%2Frun%3Fhighlight%3Dapp%20run) телеграм-клиент работает до тех пор, пока не будет принудительно остановлен.

Вопрос с тем, как ежедневно в 12.00 “дергать” скрипт, отвечающий за извлечение сообщений из json-файла и их отправку в слак, решили просто - запрограммировали задачу для линукс-планировщика cron.

Программирование задачи

Каждый день в 12.00 он из докер-контейнера alumni_client запускает python-скрипт sender.py, который и отправляет сообщения в слак.

Вот как выглядит сообщение в мессенджере:

280 просмотров·8 поделились