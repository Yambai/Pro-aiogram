# Урок 1 (Супер-Подробный): Основы aiogram - Начинаем общение! 🤖

Привет еще раз! 👋 Давай вернемся к основам `aiogram` и разберем все по косточкам, ну прямо как для бабушки, только про роботов! 😊 Наша цель – понять, как бот принимает сообщения и как мы можем ему отвечать. Не волнуйся, я буду объяснять все очень-очень подробно!

---

## План урока (Супер-Расширенный):

1.  **Главные детали бота:** `Bot`, `Dispatcher`, `Router` – что это за штуки и зачем они нужны? (Разбираем каждую деталь)
2.  **Сообщение от пользователя (`Message`):** Что внутри этой "посылки"? (Больше полей, подробные примеры)
3.  **Как бот говорит:** `answer`, `reply`, `send_message`, `send_copy` – в чем разница и когда какой способ выбрать? (Подробное сравнение)
4.  **ID - Паспорт и Адрес в Telegram:** Зачем нужны `user_id` и `chat_id`? Как их узнать? (Объясняем на пальцах)
5.  **Письмо по адресу (ID):** Как боту написать Пете или в чат "8-Б"?
6.  **Фильтры – Охранники бота:**
    * Фильтр `Command` (команды с "хвостиками"-аргументами).
    * Магия `F` (ловим фото, текст, стикеры; проверяем, что написано; комбинируем условия "И", "ИЛИ", "НЕ").
7.  **Твой шаблон бота:** Какие файлы пока не трогаем и ***почему***? (Объясняем сложность)
8.  **Украшаем текст:** Как сделать буквы **толстыми** или *наклонными*? (Про `ParseMode`)
9.  **Практика:** Пишем эхо-бота, учим реагировать на фото и стикеры (с объяснением кода).

---

## 1. Главные детали бота: Сердце, Мозг и его Полочки 🧠❤️

Давай представим нашего бота как маленького робота. У него есть важные части:

### `Bot` (Сердце и Руки 🤖):

* Это самая главная часть, которая умеет **общаться с Telegram**. Она использует твой **секретный ключ-токен** (помнишь, от @BotFather? Никому его не давай!).
* Это как "руки" и "рот" робота: он **отправляет** команды и сообщения в Telegram (`bot.send_message(...)`, `bot.send_photo(...)`).
* Он также может **запросить** у Telegram информацию (например, `await bot.get_me()` спросит: "Telegram, расскажи о боте, которым я управляю!").
* **Как использовать:**
    1.  **Импортировать:** `from aiogram import Bot`
    2.  **Создать:** `bot = Bot(token=ТВОЙ_ТОКЕН)`
    3.  **Использовать:** `await bot.send_message(...)`

    ```python
    # В main.py
    from aiogram import Bot
    from config import TOKEN # Импортируем токен из файла config.py

    # Создаем "сердце" бота, даем ему секретный ключ (токен)
    bot = Bot(token=TOKEN)
    print("Объект Bot создан!")
    ```

### `Dispatcher` (Мозг 🧠):

* Это мозг, который **принимает все сигналы** (сообщения, нажатия кнопок) от Telegram и решает, что с ними делать.
* Он работает в цикле: получает пачку обновлений -> разбирает каждое -> ищет подходящий обработчик (функцию) -> передает управление обработчику.
* Команда `dp.start_polling(bot)` запускает этот бесконечный процесс "ожидания и разбора" обновлений. Бот как бы постоянно спрашивает Telegram: "Ну что там? Есть новости?".
* **Аналогия:**
    > Представь сортировочный центр почты. `Dispatcher` получает мешки с письмами (обновления), смотрит на адрес и тему каждого письма (используя фильтры) и отдает его нужному почтальону (обработчику).

    ```python
    # В main.py
    from aiogram import Dispatcher
    # MemoryStorage - это как краткосрочная память для FSM (машины состояний).
    # Нам она пока не нужна по назначению, но Dispatcher требует указать хоть какую-то "память".
    from aiogram.fsm.storage.memory import MemoryStorage

    # Создаем "мозг" бота.
    dp = Dispatcher(storage=MemoryStorage())
    print("Dispatcher (мозг) создан!")
    ```

### `Router` (Полочки в Мозгу / Папки 🗄️):

* Когда у робота много задач, ему нужны "полочки" в мозгу, чтобы не путаться. `Router` – это такая "полочка" или "папка", где хранятся инструкции (обработчики) для похожих задач.
* Например, одна папка для команд (`/start`, `/help`), другая – для обработки фото, третья – для админских штук.
* Ты создаешь `Router` (`router = Router()`) в отдельном файле (например, `handlers/handler.py`).
* Потом "прикрепляешь" эту папку к главному мозгу (`Dispatcher`) с помощью `dp.include_router(router)`.
* `Dispatcher` сначала ищет нужную инструкцию на подключенных "полочках", а потом уже у себя (если что-то есть). Это помогает держать код в порядке.

    ```python
    # В handlers/handler.py
    from aiogram import Router
    router = Router() # Создаем "полочку" для обычных команд и сообщений
    print("Router создан!")

    # ... тут будут наши обработчики для этой "полочки" ...

    # В main.py
    from handlers import handler # Импортируем файл handler.py, где лежит наш router
    # ... инициализация dp ...
    dp.include_router(handler.router) # Говорим мозгу: "Используй инструкции с этой полочки!"
    print("Роутер handler подключен к Dispatcher!")
    ```

---

## 2. Объект `Message`: Читаем Посылку Внимательно 💌

Когда приходит сообщение, `aiogram` дает нам объект `Message`. Давай изучим его содержимое еще раз, добавив пару деталей:

* `message.message_id`: Уникальный номер этого сообщения *в этом чате*.
* `message.date`: Дата и время отправки (объект `datetime`).
* `message.from_user`: Кто отправил (объект `User`).
    * `id`: **Главный идентификатор пользователя!** ⭐
    * `is_bot`: Бот ли это? (`True`/`False`)
    * `first_name`: Имя.
    * `last_name`: Фамилия (может не быть).
    * `username`: Никнейм (может не быть).
    * `full_name`: Имя + Фамилия.
    * `language_code`: Язык пользователя ('ru', 'en', ...).
* `message.chat`: Где происходит диалог (объект `Chat`).
    * `id`: **Главный идентификатор чата!** ⭐ (В личке с ботом равен `user_id`).
    * `type`: Тип чата ('private', 'group', 'supergroup', 'channel').
    * `title`: Название (для групп/каналов).
* `message.text`: Текст сообщения (если есть).
* `message.photo`: Список `PhotoSize` (если фото). `message.photo[-1]` - самое большое.
    * `file_id`: ID файла фото. **Очень полезно!** 👍
    * `width`, `height`: Размеры.
* `message.sticker`: Объект `Sticker` (если стикер).
    * `file_id`: ID файла стикера. 👍
    * `emoji`: Связанный эмодзи.
* `message.document`: Объект `Document` (если файл).
    * `file_id`: ID файла. 👍
    * `file_name`: Имя файла.
* `message.caption`: Подпись к фото/видео/документу.
* `message.reply_to_message`: Если это ответ, тут будет *то самое* сообщение, на которое ответили (объект `Message`).

**Пример `/info` с пояснениями:**

```python
# В handlers/handler.py
from aiogram import Router, types, html
from aiogram.filters import Command
from aiogram.enums import ParseMode # Импортируем ParseMode
from datetime import datetime # Импортируем для работы с датой

router = Router()

@router.message(Command("info"))
async def cmd_info(message: types.Message):
    # Получаем данные из объекта message
    user_id = message.from_user.id
    chat_id = message.chat.id
    first_name = message.from_user.first_name
    message_id = message.message_id
    # Конвертируем дату из Unix time в читаемый формат
    # message.date - это уже объект datetime в aiogram 3+
    message_date_str = message.date.strftime("%Y-%m-%d %H:%M:%S") # Формат: ГГГГ-ММ-ДД ЧЧ:ММ:СС

    # Экранируем имя на случай спецсимволов
    safe_first_name = html.quote(first_name)

    # Собираем текст ответа
    response_text = (
        f"ℹ️ <b>Информация о сообщении:</b>\n\n" # Используем жирный шрифт для заголовка
        f"👤 <b>Отправитель:</b> {safe_first_name}\n"
        f"🆔 <b>User ID:</b> <code>{user_id}</code>\n" # Моноширинный для ID
        f"💬 <b>Чат ID:</b> <code>{chat_id}</code>\n"
        f"✉️ <b>ID сообщения:</b> <code>{message_id}</code>\n"
        f"📅 <b>Дата:</b> {message_date_str}\n"
    )

    # Добавляем текст, если он есть
    if message.text:
        response_text += f"📝 <b>Текст:</b> {html.quote(message.text)}\n"

    # Добавляем инфо об ответе, если это ответ
    if message.reply_to_message:
        reply_msg_id = message.reply_to_message.message_id
        response_text += f"↪️ <b>Ответ на сообщение ID:</b> <code>{reply_msg_id}</code>\n"

    # Отправляем ответ, используя HTML разметку
    await message.answer(response_text, parse_mode=ParseMode.HTML)
````

> **Пояснение:** Мы достали разные данные из `message`, включая дату, которую красиво отформатировали с помощью `strftime`. Мы использовали HTML-теги `<b>` (жирный) и `<code>` (код) для лучшего вида. Функция `html.quote()` защищает нас от проблем, если в имени или тексте сообщения встретятся символы `<` или `>` (которые являются частью HTML-разметки).

-----

## 3\. Диалог с пользователем: Говорим правильно 💬

Давай закрепим разницу между способами ответа с помощью таблицы:

| Метод                   | Аналогия                           | Пример кода                               | Результат в Telegram                                   |
| :---------------------- | :--------------------------------- | :---------------------------------------- | :----------------------------------------------------- |
| `message.answer("...")` | Сказать что-то в той же комнате.   | `await message.answer("Понял!")`          | Бот просто пишет "Понял\!" в чат.                       |
| `message.reply("...")`  | Ответить на *конкретную фразу*.     | `await message.reply("На какой вопрос?")` | Сообщение бота будет "прикреплено" к сообщению юзера. |
| `bot.send_message(...)` | Отправить письмо по адресу (ID).   | `await bot.send_message(chat_id, "Привет!")` | Бот отправит "Привет\!" в чат с указанным `chat_id`.   |
| `message.send_copy(...)`| Сделать ксерокопию и отправить. | `await message.send_copy(message.chat.id)` | Бот отправит точную копию сообщения юзера (текст/фото...). |

**Пример последовательности:**

```python
# В handlers/handler.py
# Добавьте в импорты:
import asyncio
from aiogram import Bot # Если bot передается в хэндлер

# ... (другие хэндлеры)

@router.message(Command("test_reply"))
async def cmd_test_reply(message: types.Message, bot: Bot):
    # 1. Отвечаем простым answer
    await message.answer("Это простой ответ (answer)")
    await asyncio.sleep(1) # Маленькая пауза

    # 2. Отвечаем с цитированием (reply)
    await message.reply("А это ответ с цитированием (reply)")
    await asyncio.sleep(1)

    # 3. Отправляем новое сообщение через bot.send_message
    await bot.send_message(message.chat.id, "Это новое сообщение (send_message)")
    await asyncio.sleep(1)

    # 4. Копируем исходное сообщение
    await message.send_copy(message.chat.id)
    await message.answer("А это копия вашего сообщения (send_copy)")

```

> **Задание:** Добавь этот обработчик в свой `handlers/handler.py`, не забудь импортировать `asyncio` и `Bot`, а также передать `bot` в `Dispatcher` при запуске (`dp.start_polling(bot)`). Посмотри, как выглядят разные ответы в Telegram.

-----

## 4\. ID - Уникальные номера: Паспорт и Адрес 🏠👤

  * `user_id`: **Номер паспорта** пользователя. Уникален для каждого. Не меняется. Позволяет обратиться к Пете Иванову, даже если он сменит никнейм или фамилию.
  * `chat_id`: **Почтовый адрес** чата. Уникален для каждого чата (личного, группового, канала). Не меняется. Позволяет отправить письмо (сообщение) в конкретный дом (чат), не зная, кто там сейчас живет.

**Зачем так подробно?** Это ключевые понятия\! Без ID ты не сможешь:

  * Отправить личное сообщение конкретному пользователю.
  * Понять, из какой группы пришло сообщение.
  * Сохранить настройки для пользователя (например, выбранный язык) в базе данных.
  * Определить, кто админ, а кто обычный пользователь.

-----

## 5\. Отправка по ID: Адресная рассылка 📮

Ты можешь использовать `user_id` как `chat_id` для отправки сообщения **в личный чат** с пользователем.

```python
# Пример: Отправить сообщение всем админам при каком-то событии
# (Этот код не является полным обработчиком, просто пример логики)
# Добавьте в импорты: from aiogram import Bot
# Где-то определите ADMIN_ID или список админов:
# ADMIN_ID = 123456789 # Ваш ID
# ADMIN_IDS = [123456789, 987654321]

async def notify_admins(bot: Bot, text_to_send: str):
    admin_ids = [123456789, 987654321] # Пример списка ID админов
    for admin_id in admin_ids:
        try:
            await bot.send_message(chat_id=admin_id, text=text_to_send)
            print(f"Уведомление отправлено админу {admin_id}")
        except Exception as e:
            # Если бот не может написать админу (например, заблокирован)
            print(f"Не удалось отправить сообщение админу {admin_id}: {e}")

# Вызвать эту функцию можно из другого хэндлера:
# await notify_admins(bot, "Внимание! Произошло важное событие!")
```

-----

## 6\. Фильтры – Охранники бота на входе 💂‍♂️

Фильтры решают, какой обработчик сработает для пришедшего обновления (чаще всего, сообщения).

### `Command("команда", ...)`

  * Ловит команды типа `/start`, `/help`.

  * **Параметры:**

      * `ignore_case=True`: Неважно, `/help` или `/HELP`.
      * `prefix="/!."`: Сработает на `/cmd`, `!cmd`, `.cmd`.

  * **Аргументы:** Если команда пришла как `/weather Москва` и фильтр `Command("weather")`, то получить аргумент "Москва" можно через специальный объект `CommandObject`, который передается в хэндлер:

    ```python
    from aiogram.filters import CommandObject # Не забудь импортировать

    @router.message(Command("weather"))
    async def cmd_weather(message: types.Message, command: CommandObject):
        if command.args:
            city = command.args
            await message.answer(f"Узнаю погоду для города: {html.quote(city)}")
            # Здесь могла бы быть логика запроса погоды
        else:
            await message.answer("Пожалуйста, укажите город после команды /weather")
    ```

### Магический фильтр `F`

Он "смотрит" на объект события (чаще `Message`) и проверяет его поля. Очень гибкий\!

  * **Типы контента:**

      * `F.text` - только текстовое сообщение.
      * `F.photo` - только фото.
      * `F.audio` - только аудиофайл.
      * `F.document` - только документ/файл.
      * `F.sticker` - только стикер.
      * `F.voice` - только голосовое сообщение.
      * `F.video` - только видео.
      * `F.video_note` - только видео-кружочек.
      * `F.contact` - только контакт.
      * `F.location` - только локация.
      * `F.animation` - только GIF-анимация (без звука).

  * **Проверки текста:**

      * `F.text == "привет"` - точное совпадение.
      * `F.text.lower() == "привет"` - без учета регистра.
      * `F.text.startswith("купить")` - начинается с "купить".
      * `F.text.endswith("?")` - заканчивается на "?".
      * `F.text.contains("важно")` - содержит слово "важно".
      * `F.text.in_({"да", "нет", "yes", "no"})` - текст равен одному из элементов набора.

  * **Проверки пользователя/чата:**

      * `F.from_user.id == ADMIN_ID` - сообщение от конкретного пользователя (админа).
      * `F.chat.type == "private"` - сообщение из личного чата.
      * `F.chat.type.in_({"group", "supergroup"})` - сообщение из группы или супергруппы.

  * **Логические операции:**

      * `&` (**И**): Оба условия верны.
          * `F.chat.type == "group" & F.text.lower() == "правила"` (Сообщение "правила" в группе).
      * `|` (**ИЛИ**): Хотя бы одно условие верно.
          * `F.photo | F.document` (Пришло или фото, или документ).
      * `~` (**НЕ**): Условие НЕ верно.
          * `~F.text` (Любое сообщение, КРОМЕ текстового).

  * **Скобки важны для порядка операций\!**

      * `(F.photo | F.video) & F.caption` - (Пришло Фото ИЛИ Видео) И при этом есть подпись (`caption`).
      * `F.photo | (F.video & F.caption)` - Пришло Фото (без разницы, есть ли подпись) ИЛИ пришло Видео с подписью.

-----

## 7\. Твой шаблон бота: Почему откладываем? 🤔

Почему мы пока не используем все файлы из более сложного шаблона бота?

  * ❌ `database.py` / `db_utils.py`: **База данных (БД)** хранит информацию надолго (пользователей, их настройки, статистику), даже после выключения бота. Это как записная книжка или склад. Но работа с ней (SQL-запросы или ORM, подключение, структура таблиц) – это отдельная большая и важная тема. Пока обойдемся без "долгосрочной памяти".
  * ❌ `states/States.py` или `fsm/`: **Машина состояний (FSM - Finite State Machine)** нужна, чтобы бот помнил, на каком *этапе диалога* он с пользователем. Например:
    1.  Бот спросил имя (перешел в состояние "ожидаю имя").
    2.  Пользователь прислал имя, бот его запомнил и спросил возраст (перешел в состояние "ожидаю возраст").
        Без FSM бот не помнит предыдущие шаги в рамках одной сессии. Это мощный инструмент для анкет, заказов, игр, но усложняет начальный код. Начнем с простых реакций "вопрос-ответ".
  * ❌ `keyboards/`: **Кнопки** (инлайн и обычные) делают бота удобным, но их создание (`InlineKeyboardMarkup`, `ReplyKeyboardMarkup`) и обработка нажатий (требует фильтра `F.callback_query` и обработки объекта `CallbackQuery`) – это тоже отдельные шаги, которые требуют своих обработчиков и логики. Сначала освоим текст и базовые медиа.
  * ❌ `handlers/admin_handlers.py`: **Админские команды** часто требуют *проверки прав* (кто админ, а кто нет). Это обычно делается либо через сравнение `message.from_user.id` со списком админов, либо через проверку роли в базе данных. Упростим для начала.
  * ❌ `filters/`: **Создание своих фильтров** (например, `IsAdminFilter`) нужно, когда встроенных (`Command`, `F`) не хватает для сложной логики проверки. Это продвинутая техника. Нам пока с головой хватит `Command` и `F`.

> **Итог:** Сосредоточимся на `main.py` (инициализация и запуск), `config.py` (токен) и `handlers/handler.py` (простые реакции на команды и типы сообщений). Меньше файлов – меньше путаницы на старте\!

-----

## 8\. Немного форматирования: `ParseMode` 🎨

Мы уже видели `parse_mode=ParseMode.HTML` в примере `/info`. Этот параметр говорит Telegram: "Эй, в моем тексте есть специальные теги, не показывай их как текст, а примени форматирование\!".

**Основные HTML-теги, поддерживаемые Telegram:**

  * `<b>Текст</b>` или `<strong>Текст</strong>` -\> **Жирный**
  * `<i>Текст</i>` или `<em>Текст</em>` -\> *Курсив*
  * `<u>Текст</u>` или `<ins>Текст</ins>` -\> <u>Подчеркнутый</u>
  * `<s>Текст</s>` или `<strike>Текст</strike>` или `<del>Текст</del>` -\> <s>Зачеркнутый</s>
  * `<span class="tg-spoiler">Текст</span>` -\> <span class="tg-spoiler">Спойлер</span> (скрытый текст)
  * `<code>Текст</code>` -\> `Моноширинный текст` (удобно для кода, ID)
  * `<pre>Блок текста</pre>` -\> Блок с моноширинным текстом, сохраняет пробелы и переносы строк
  * `<pre><code class="language-python">print("hello")</code></pre>` -\> Блок кода с подсветкой синтаксиса (если клиент поддерживает)
  * `<a href="URL">Текст ссылки</a>` -\> [Текст ссылки](https://www.google.com/search?q=URL)

**Пример использования:**

```python
# В handlers/handler.py
# Добавьте в импорты:
from aiogram.enums import ParseMode
from aiogram import html

# ...

@router.message(Command("style"))
async def cmd_style(message: types.Message):
    # Используем html.quote для имени, чтобы обезопасить от вставки тегов пользователем
    safe_user_name = html.quote(message.from_user.first_name)

    styled_text = f"<b>{safe_user_name}</b>, смотри, как можно форматировать:\n\n" \
                  f"<i>Курсив</i>, <u>Подчеркнутый</u>, <s>Зачеркнутый</s>, <span class='tg-spoiler'>Спойлер</span>.\n" \
                  f"<code>Моноширинный код (inline)</code>\n\n" \
                  f"<pre>Блок\n  моноширинного\n    текста</pre>\n" \
                  f'<pre><code class="language-python"># Python код\ndef hello():\n    print("Hello world!")</code></pre>\n' \
                  f'А вот <a href="[https://aiogram.dev/](https://aiogram.dev/)">ссылка на документацию aiogram</a>.'

    await message.answer(styled_text, parse_mode=ParseMode.HTML)

```

> **Важно\!** Если ты вставляешь в форматированный текст данные от пользователя (имя, текст сообщения и т.д.) и используешь `ParseMode.HTML`, **всегда** оборачивай эти данные в `html.quote(...)`. Иначе пользователь сможет отправить `<b>` в своем имени или сообщении, и разметка вашего ответа сломается или приведет к нежелательному форматированию.

-----

## 9\. Практика: Добавим жизни боту\! ✨

Теперь давай соберем все вместе в `handlers/handler.py`.

  * Убедимся, что есть обработчик `/start`.
  * Добавим `/info` и `/style` из примеров выше.
  * Добавим обработчики для фото и стикеров.
  * Добавим эхо-бот для текста, который должен стоять **после** всех остальных текстовых обработчиков (команд).

<!-- end list -->

```python
# Файл: handlers/handler.py

import asyncio
from datetime import datetime
from aiogram import Router, F, types, html, Bot
from aiogram.filters import Command, StateFilter, CommandObject
from aiogram.fsm.context import FSMContext
from aiogram.enums import ParseMode
from aiogram.types import Message # Убедимся, что Message импортирован

router = Router()

# --- Обработчик команды /start ---
# StateFilter("*") означает, что команда сработает из любого состояния FSM
@router.message(Command('start'), StateFilter("*"))
async def process_start_command(message: Message, state: FSMContext):
    # await state.clear() # На всякий случай очищаем состояние (об этом позже)
    # await state.set_state(None) # Устанавливаем пустое состояние
    user_name = message.from_user.first_name
    await message.answer(f"Привет, {html.quote(user_name)}! 👋\nЯ эхо-бот с некоторыми дополнениями.")

# --- Обработчик команды /info ---
@router.message(Command("info"))
async def cmd_info(message: types.Message):
    user_id = message.from_user.id
    chat_id = message.chat.id
    first_name = message.from_user.first_name
    message_id = message.message_id
    message_date_str = message.date.strftime("%Y-%m-%d %H:%M:%S")
    safe_first_name = html.quote(first_name)
    response_text = (
        f"ℹ️ <b>Информация о сообщении:</b>\n\n"
        f"👤 <b>Отправитель:</b> {safe_first_name}\n"
        f"🆔 <b>User ID:</b> <code>{user_id}</code>\n"
        f"💬 <b>Чат ID:</b> <code>{chat_id}</code>\n"
        f"✉️ <b>ID сообщения:</b> <code>{message_id}</code>\n"
        f"📅 <b>Дата:</b> {message_date_str}\n"
    )
    if message.text:
        response_text += f"📝 <b>Текст:</b> {html.quote(message.text)}\n"
    if message.reply_to_message:
        reply_msg_id = message.reply_to_message.message_id
        response_text += f"↪️ <b>Ответ на сообщение ID:</b> <code>{reply_msg_id}</code>\n"
    await message.answer(response_text, parse_mode=ParseMode.HTML)

# --- Обработчик команды /style ---
@router.message(Command("style"))
async def cmd_style(message: types.Message):
    safe_user_name = html.quote(message.from_user.first_name)
    styled_text = f"<b>{safe_user_name}</b>, смотри, как можно форматировать:\n\n" \
                  f"<i>Курсив</i>, <u>Подчеркнутый</u>, <s>Зачеркнутый</s>, <span class='tg-spoiler'>Спойлер</span>.\n" \
                  f"<code>Моноширинный код (inline)</code>\n\n" \
                  f"<pre>Блок\n  моноширинного\n    текста</pre>\n" \
                  f'<pre><code class="language-python"># Python код\ndef hello():\n    print("Hello world!")</code></pre>\n' \
                  f'А вот <a href="[https://aiogram.dev/](https://aiogram.dev/)">ссылка на документацию aiogram</a>.'
    await message.answer(styled_text, parse_mode=ParseMode.HTML, disable_web_page_preview=True) # Отключаем превью ссылки

# --- Обработчик команды /test_reply (для демонстрации) ---
@router.message(Command("test_reply"))
async def cmd_test_reply(message: types.Message, bot: Bot):
    await message.answer("Это простой ответ (answer)")
    await asyncio.sleep(1)
    await message.reply("А это ответ с цитированием (reply)")
    await asyncio.sleep(1)
    await bot.send_message(message.chat.id, "Это новое сообщение (send_message)")
    await asyncio.sleep(1)
    try:
        await message.send_copy(message.chat.id)
        await message.answer("А это копия вашего сообщения (send_copy)")
    except Exception as e:
        await message.answer(f"Не могу скопировать это сообщение. Ошибка: {e}")


# --- Обработчик фото ---
# Ловим только фото (F.photo)
@router.message(F.photo)
async def photo_handler(message: types.Message):
    # message.photo - это список разных размеров фото, берем последний (самый большой)
    photo_big = message.photo[-1]
    # Отвечаем с цитированием на сообщение с фото
    await message.reply(
        f"Ого, какое фото! ✨\n"
        f"Размер: {photo_big.width}x{photo_big.height}\n"
        f"ID файла (для повторной отправки): <code>{photo_big.file_id}</code>",
        parse_mode=ParseMode.HTML
    )
    # Можно еще и сохранить file_id куда-нибудь или переслать фото
    # await message.send_copy(chat_id=message.chat.id) # Перешлет копию фото

# --- Обработчик стикеров ---
# Ловим только стикеры (F.sticker)
@router.message(F.sticker)
async def sticker_handler(message: types.Message):
    sticker_id = message.sticker.file_id
    emoji = message.sticker.emoji or "нет" # Если эмодзи нет, будет "нет"
    # Отвечаем на стикер
    await message.reply(
        f"Классный стикер! 👍\n"
        f"Связанный эмодзи: {emoji}\n"
        f"ID файла: <code>{sticker_id}</code>",
        parse_mode=ParseMode.HTML
    )
    # Отправляем этот же стикер в ответ, используя ID
    await message.answer_sticker(sticker_id)

# --- Обработчик команды /dice (Игральный кубик) ---
@router.message(Command("dice"))
async def cmd_dice(message: types.Message):
    # Просто отправляем анимированный кубик
    await message.answer_dice()
    # Можно отправить с конкретным эмодзи
    # await message.answer_dice(emoji="🎲") # или 🎯, 🏀, ⚽, 🎳, 🎰

# --- Обработчик голосовых сообщений ---
@router.message(F.voice)
async def voice_handler(message: types.Message):
    duration = message.voice.duration # Длительность в секундах
    await message.reply(f"Получил голосовое сообщение!\nДлительность: {duration} сек.\nID файла: <code>{message.voice.file_id}</code>", parse_mode=ParseMode.HTML)

# --- Эхо для текстовых сообщений (ВАЖНО: должен быть последним среди текстовых!) ---
# Ловим только текст (F.text) и убеждаемся, что это не команда (дополнительно)
# Хотя порядок регистрации важнее, можно добавить и такую проверку для ясности:
# from aiogram.filters import CommandStart, CommandHelp и т.д.
# @router.message(F.text, ~CommandStart(), ~Command("help"), ...)
# Но проще просто разместить его последним в этом роутере.
@router.message(F.text)
async def echo_message(message: types.Message):
    # Используем send_copy, чтобы сохранить форматирование, ссылки и т.д.
    try:
        await message.send_copy(chat_id=message.chat.id)
    except Exception as e:
        # Если не получилось скопировать (например, для служебных сообщений)
        print(f"Ошибка при копировании сообщения: {e}")
        # Просто отвечаем текстом
        safe_text = html.quote(message.text) # Экранируем на всякий случай
        await message.reply(f"Я получил текст: {safe_text}")

# --- Обработчик для "не пойманных" типов сообщений ---
# Ставится САМЫМ ПОСЛЕДНИМ в роутере, чтобы ловить все остальное
@router.message()
async def unknown_message_type(message: types.Message):
    # Определяем тип контента для информации
    content_type = message.content_type
    await message.reply(f"Я пока не знаю, как обрабатывать сообщения типа: `{content_type}` 🤔")

```

> **Задание:** Замени весь код в `handlers/handler.py` на этот обновленный код. Запусти бота (`python main.py`) и протестируй все команды (`/start`, `/info`, `/style`, `/test_reply`, `/dice`) и отправку разного контента: обычный текст, **жирный текст**, *курсив*, [ссылки](https://google.com), фото, стикеры, голосовые сообщения, файлы. Проверь, как работает эхо и обработка неизвестных типов.

-----

## Заключение первого урока (Супер-Подробное)

Фух\! Вот это мы поработали\! 😅 Надеюсь, теперь стало гораздо понятнее. Мы очень подробно разобрали:

  * Кто за что отвечает: `Bot` (общение с API), `Dispatcher` (мозг, маршрутизация), `Router` (организация обработчиков).
  * Из чего состоит `Message` и как доставать оттуда данные (`from_user`, `chat`, `text`, `photo`, `sticker`, `file_id` и т.д.).
  * Как и когда использовать `answer`, `reply`, `send_message`, `send_copy`, `answer_sticker`, `answer_dice`.
  * Что такое `user_id`, `chat_id` и зачем они **критически важны**.
  * Как использовать фильтр `Command`, в том числе с аргументами (`CommandObject`).
  * Как работает магия `F` для разных типов сообщений и логических условий (`&`, `|`, `~`).
  * Почему мы пока **осознанно отложили** работу с базами данных, состояниями (FSM) и кнопками.
  * Как делать текст красивым и безопасным с помощью `ParseMode.HTML` и `html.quote()`.
  * Написали и протестировали обработчики для команд, текста, фото, стикеров, голосовых сообщений и даже для неизвестных типов.

**Главное – не бойся экспериментировать\!** Лучший способ научиться – это писать код самому, ломать его и чинить. 😊

**Идеи для самостоятельной практики:**

1.  Попробуй поймать видео (`F.video`) и написать его длительность и размеры.
2.  Поймай документ (`F.document`) и выведи его имя файла (`file_name`) и размер (`file_size`).
3.  Сделай команду `/me`, которая будет выводить `user_id`, `first_name` и `username` пользователя, который ее вызвал.
4.  Попробуй использовать `MarkdownV2` вместо `HTML` для форматирования (синтаксис немного другой, например, `*жирный*`, `_курсив_`, `` `код` ``, `[ссылка](url)`). Не забудь импортировать `ParseMode.MARKDOWNV2` и экранировать спецсимволы Markdown с помощью `html.quote()` (да, он работает и для Markdown).

Если что-то неясно – смело спрашивай\! Движемся дальше\! 💪

```
