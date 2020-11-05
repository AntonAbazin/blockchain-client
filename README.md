Добрый день, сегодня я хотел бы поделится с Вами проблемами и их необычными решениями, которые встретились при написании небольших IT проектов. Сразу скажу, что статья для тех, кто хоть немного разбирается в разработке телеграмм ботов, баз данных, SQL и в языке программировании python. 

Весь проект выложен на github, ссылка будет в конце статьи.

<img src="https://www.vamtlgrm.com/wp-content/uploads/2018/04/y-pDbyfXZqI.jpg" alt="image"/>

<h3>Основная проблема:</h3>
Изначально я хотел для себя написать простенького телеграмм бота счетчика калорий, который получает число от пользователя и возвращает сколько калорий осталось до нормы на день. То есть нужно хранить грубо говоря пару переменных.
<cut />
В итоге нужно было выбрать способ хранить эти данные.

 <ol>
<li>Вариант - <b>глобальные переменные</b>, оперативная память. Вариант сразу провальный, так как при падении программы мы теряем все</li>
<li>Вариант - <b>запись в файл</b> на диске. Для такого проекта может и пойдет, но я планировал деплой бота на heroku, который каждый день стирает все данные с диска. Так что этот вариант не подошел </li>
<li>Вариант - <b>Google-таблицы</b>. Изначально я хотел остановится на этом варианте, но начал разбираться и понял, что есть ограничение на количество запросов к таблице, и чтобы только начать использовать таблицу нужно написать кучу строк кода и разобраться в их не самом простом апи </li>
<li>Вариант - <b>база данных</b>. Да, это наилучший вариант во всем. Но для такого проекта это даже смешно использовать. Также развертывание и поддержка базы данных на стороннем сервере обойдется в копеечку. </li>
</ol>
В итоге ни один из этих вариантов не подошел. Конечно же есть и десятки других способов, но мне хотелось бы, чтобы было бесплатно, быстро и минимум кода.
<cut />
<h3>Решение</h3>
Идея очень простая, для хранения данных мы будем использовать in memory базу данных sqllite, так как она уже встроена в python 3 и будем делать бэкапы нашей таблицы на сервера Telegram с небольшим интервалом (примерно каждые 30 секунд) и бэкап при закрытие процесса программы.

Если сервер упал, то при первом запросе мы автоматически загрузим нашу таблицу с сервера Telegram и восстановим данные в sqllite. 

Можно использовать и любую другую in memory бд, кому как нравится.

<h3>Плюсы</h3>
<ol>
	<li> <b>Быстродействие </b>- за счет работы с данными в оперативной памяти скорость выполнения программы даже быстрее, чем при использовании бд на стороннем сервере (графики скорости выполнения и тестирования будут в конце)</li>
<li> <b>Бесплатно</b> - не нужно покупать сторонние сервера для баз данных и все данные хранятся в виде бэкапа бесплатно на серверах Telegramа </li>
<li> <b>Относительно надежно</b> - если сервер падает по непонятным причинам, то мы максимум теряем данные за последние 30 секунд (время интервала бэкапов), для рабочего прототипа или небольшого проекта будет достаточно. </li>
<li> <b>Минимальные затраты</b> при переходе на обычную бд - нужно заменить данные подключения, убрать код бекапов и перенести данные таблицы из бэкапа на новую бд. </li>
</ol>
<h3>Минусы</h3>
<ol>
	<li> Отсутствие горизонтального масштабирования</li>
<li>Нужно два аккаунта в Telegramе (один для администратора, другой для тестирования пользователя) </li>
<li> Сервер не будет работать в России из-за блокировок</li>
<li> В комментариях я думаю Вы найдете еще десяток других нюансов.</li>
</ol><cut />
<h3>Время говнокодить</h3>
Напишем простой кликер и проведем тесты на скорость выполнения.

Бот будет написан на языке программирования python с использованием асинхронной библиотеки взаимодействия с api телеграмма aiogram. 

Первым делом нужно заполнить настройки бота, не буду рассказывать как получить токен от BotFather, уже сотни статей есть на эту тему.

Также нам нужен второй аккаунт в телеграмме для админа, в котором будут сохраняться наши бекапы.

Для того, чтобы получить admin_id и config_id нам нужно запустить бота с аккаунта администратора и написать боту "admin", после чего он создаст первый бекап, и напишет ваш admin_id, config_id. Заменяем и запускаем бота заново.<cut />
<source lang="python">#--------------------Настройки бота-------------------------
# Ваш токен от BotFather
TOKEN = '1234567:your_token'

# Логирование
logging.basicConfig(level=logging.INFO)

bot = Bot(token=TOKEN)
dp = Dispatcher(bot)

# Ваш айди аккаунта администратора и айди сообщения где хранится файл с данными
admin_id=12345678
config_id=12345

conn = sqlite3.connect(":memory:")  # настройки in memory бд
cursor = conn.cursor()
</source>
Так теперь пройдемся по основной логике бота:

Если боту приходит сообщение  со словом "admin", то мы создаем таблицу пользователей с такой моделью данных:

<ul>
	<li>chatid - уникальный чат айди пользователя</li>
<li>name - имя  пользователя</li>
<li>click - количество кликов</li>
<li>state - значение для машины состояний, в данном проекте не используется, но в более сложных без него не обойтись</li>
</ul>
Добавляем тестового пользователя, и отправляем документ на сервер Telegram с нашей таблицей. Так же отправляем admin_id и config_id администратору в виде сообщений. После получения айдишников, данный код нужно закомментировать.
<source lang="python">
# Логика для администратора
    if message.text == 'admin':
        cursor.execute("CREATE TABLE users (chatid INTEGER , name TEXT, click INTEGER, state INTEGER)")
        cursor.execute("INSERT INTO users VALUES (1234, 'eee', 1,0)")
        conn.commit()
        sql = "SELECT * FROM users "
        cursor.execute(sql)
        data = cursor.fetchall()
        str_data = json.dumps(data)
        await bot.send_document(message.chat.id, io.StringIO(str_data))
        await bot.send_message(message.chat.id, 'admin_id = {}'.format(message.chat.id))
        await bot.send_message(message.chat.id, 'config_id = {}'.format(message.message_id+1))
</source><cut />
Логика для пользователя:

Первым делом пытаемся получить из in memory бд данные пользователя, который отправил сообщение. Если ловим ошибку, то загружаем данные с бекапа сервера Telergam, заполняем нашу бд данными с бекапа и повторно пытаемся найти пользователя.

<source lang="python"># Логика для пользователя
    try:
        sql = "SELECT * FROM users where chatid={}".format(message.chat.id)
        cursor.execute(sql)
        data = cursor.fetchone()  # or use fetchone()
    except Exception:
        data = await get_data()
        cursor.execute("CREATE TABLE users (chatid INTEGER , name TEXT, click INTEGER, state INTEGER)")
        cursor.executemany("INSERT INTO users VALUES (?,?,?,?)", data)
        conn.commit()
        sql = "SELECT * FROM users where chatid={}".format(message.chat.id)
        cursor.execute(sql)
        data = cursor.fetchone()  # or use fetchone()
</source>
Если мы нашли пользователя в бд, то обрабатываем кнопки:

 <ul>
	<li>При нажатие "Клик" мы обновляем количество кликов у данного пользователя</li>
<li>При нажатие "Рейтинг" мы выводим список пятнадцати человек у которых наибольшее количество кликов.</li>
</ul>
Если не нашли пользователя, то написать ему ошибку.<cut />
<source lang="python"> #При нажатии кнопки клик увеличиваем значение click на один и сохраняем
    if data is not None:
        if message.text == 'Клик':
            sql = "UPDATE users SET click = {} WHERE chatid = {}".format(data[2]+1,message.chat.id)
            cursor.execute(sql)
            conn.commit()
            await bot.send_message(message.chat.id, 'Кликов: {} 🏆'.format(data[2]+1))

        # При нажатии кнопки Рейтинг выводим пользователю топ 10
        if message.text == 'Рейтинг':
            sql = "SELECT * FROM users ORDER BY click DESC LIMIT 15"
            cursor.execute(sql)
            newlist = cursor.fetchall()  # or use fetchone()
            sql_count = "SELECT COUNT(chatid) FROM users"
            cursor.execute(sql_count)
            count=cursor.fetchone()
            rating='Всего: {}\n'.format(count[0])
            i=1
            for user in newlist:
                rating=rating+str(i)+': '+user[1]+' - '+str(user[2])+'🏆\n'
                i+=1
            await bot.send_message(message.chat.id, rating)
    else:
        await bot.send_message(message.chat.id, 'Вы не зарегистрированы')</source>
Напишем логику для регистрации пользователя:

Пытаемся найти пользователя в бд, если его нет, то добавляем новую строку в таблицу и делаем бэкап.
Если ловим ошибку, то подгружаем последний бэкап, заполняем таблицу и повторяем попытку регистрации.<cut />
<source lang="python">sql_select = "SELECT * FROM users where chatid={}".format(message.chat.id)
    sql_insert = "INSERT INTO users VALUES ({}, '{}', {},{})".format(message.chat.id,message.chat.first_name, 0, 0)
    try:
        cursor.execute(sql_select)
        data = cursor.fetchone()
        if data is None:
            cursor.execute(sql_insert)
            conn.commit()
            await save_data()
    except Exception:
        data = await get_data()
        cursor.execute("CREATE TABLE users (chatid INTEGER , name TEXT, click INTEGER, state INTEGER)")
        cursor.executemany("INSERT INTO users VALUES (?,?,?,?)", data)
        conn.commit()
        cursor.execute(sql_select)
        data = cursor.fetchone()
        if data is  None:
            cursor.execute(sql_insert)
            conn.commit()
            await save_data()
        # Создаем кнопки
    button = KeyboardButton('Клик')
    button2 = KeyboardButton('Рейтинг')
    # Добавляем
    kb = ReplyKeyboardMarkup(resize_keyboard=True).add(button).add(button2)
    # Отправляем сообщение с кнопкой
    await bot.send_message(message.chat.id,'Приветствую {}'.format(message.chat.first_name),reply_markup=kb)</source>
Так, ну и самое интересное:
Сохранение и получение данных с сервера Telergam.

Мы выгружаем все данные с таблицы пользователей, переводим словарь  в строку и изменяем наш файл, который хранится на серверах Telegram.<cut />
<source lang="python"> #--------------------Сохранение данных-------------------------
async def save_data():

    sql = "SELECT * FROM users "
    cursor.execute(sql)
    data = cursor.fetchall()  # or use fetchone()
    try:
        # Переводим словарь в строку
        str_data=json.dumps(data)

        # Обновляем  наш файл с данными
        await bot.edit_message_media(InputMediaDocument(io.StringIO(str_data)), admin_id, config_id)

    except Exception as ex:
        print(ex)
</source><cut />
Для того, чтобы получить бэкап нам нужно переслать сообщение с файлом от админа к админу. Затем получить путь к файл, считать данные по url и вернуть весь бэкап.
<source lang="python"># #--------------------Получение данных-------------------------
async def get_data():
    # Пересылаем сообщение в данными от админа к админу
    forward_data = await bot.forward_message(admin_id, admin_id, config_id)

    # Получаем путь к файлу, который переслали
    file_data = await bot.get_file(forward_data.document.file_id)

    # Получаем файл по url
    file_url_data = bot.get_file_url(file_data.file_path)

    # Считываем данные с файла
    json_file= urlopen(file_url_data).read()

    # Переводим данные из json в словарь и возвращаем
    return json.loads(json_file)</source>
Ну вот почти и все, осталось только написать таймер, чтобы делал бэкапы и протестировать бота.
Создаем поток, который каждые 30 секунд выполняет наш метод save_data()
<source lang="python">def timer_start():
    threading.Timer(30.0, timer_start).start()
    try:
        asyncio.run_coroutine_threadsafe(save_data(),bot.loop)
    except Exception as exc:
        pass</source>
Ну и в главной программе мы запускаем таймер и самого бота.
<source lang="python">#--------------------Запуск бота-------------------------
if __name__ == '__main__':
    timer_start()
    executor.start_polling(dp, skip_updates=True)</source><cut />
Так с кодом вроде бы разобрались, вот ссылка рабочего проект на github: https://github.com/akashkinKV/click-bot-telegram

<h3>Как запустить</h3>
<ol>
	<li>Скачиваем проект с гитхаба. Запускаем проект в любой среде разработки для python (Например: PyCharm).</li>
	<li>Среда разработки автоматически подгрузит необходимые библиотеки с файла requirements.</li>
	<li>Заменяем Token от BotFather в файле main.py  
<img src="https://habrastorage.org/webt/7c/jz/75/7cjz75bpkulsulwhaefjh4h_go0.png" /></li>
<li>Запускаем проект</li>
<li>Со второго аккаунта нажимаем /start и пишем слово "admin" <img src="https://habrastorage.org/webt/gq/jk/ru/gqjkru3lwkehvo5bwwgqkdvdxgw.jpeg" /></li>
<li>Выключаем проект и заполняем admin_id и config_id в файле main.py</li>
<li>Запускаем проект и с аккаунта пользователя нажимаем старт 
<img src="https://habrastorage.org/webt/qw/f8/q6/qwf8q6x6t981g8hxbrnaxcnbabi.jpeg" /></li>
<li><b>Профит</b></li>
</ol>
<h3>Тестирование и графики</h3>

