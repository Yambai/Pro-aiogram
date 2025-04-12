# Магия Состояний (FSM) - Учим бота вести диалог! 🧠💬

Представь себе, что ты заполняешь анкету на сайте: сначала вводишь имя, потом фамилию, потом email. Сайт *помнит*, что ты уже ввел имя, когда спрашивает фамилию. Он находится в определенном "состоянии" заполнения анкеты.

**Проблема:** Обычные обработчики (`@router.message(...)`) в `aiogram` **не помнят** предыдущие сообщения. Если ты сделаешь команду `/register`, которая спрашивает "Введите ваше имя", а потом пользователь пришлет имя, то как боту понять, что это *именно имя в ответ на регистрацию*, а не просто случайное сообщение? Как потом спросить возраст и связать его с этим же процессом регистрации?

**Решение:** **FSM (Finite State Machine)**! Это механизм, который позволяет боту:

1.  Находиться в **определенных состояниях** (например, `ожидание_имени`, `ожидание_возраста`).
2.  **Запоминать временные данные** пользователя *в рамках этого состояния* (например, введенное имя).
3.  **Переходить** из одного состояния в другое по мере получения нужной информации.

Давай разбираться по порядку!

**План урока (Супер-Расширенный):**

1.  **Аналогия:** FSM - это как... (разные варианты для понимания).
2.  **Зачем вообще нужны состояния?** (Более глубокое погружение в проблему "памяти").
3.  **Ключевые компоненты FSM в `aiogram`:**
    * `StatesGroup` и `State`: Определяем "шаги" нашего диалога.
    * `FSMContext`: "Волшебная сумка" для временных данных и управления состоянием.
    * Хранилище (`Storage`): Где эта "сумка" хранится? (`MemoryStorage` и намек на будущее).
4.  **Шаг за шагом: Пишем простую анкету (Регистрация):**
    * Определение состояний (файл `states/registration.py`).
    * Команда для старта (`/register`) - вход в первое состояние.
    * Обработчик для первого шага (ввод имени) - сохранение данных и переход.
    * Обработчик для второго шага (ввод возраста) - сохранение, валидация и переход.
    * Обработчик для последнего шага (ввод города) - сохранение, получение всех данных и **выход из состояний**.
    * Сборка в отдельном файле (`handlers/registration_handlers.py`).
5.  **"Стоп! Я передумал!" - Отмена состояния:**
    * Команда `/cancel` для выхода из анкеты на любом шаге.
    * Использование `StateFilter("*")`.
    * Метод `state.clear()` - очистка "сумки".
6.  **"Ой, я отправил не то!" - Обработка некорректного ввода:**
    * Что если пользователь прислал фото вместо текста?
    * Добавление обработчиков для "неправильных" типов данных в конкретном состоянии.
7.  **Структура проекта с FSM:** Как организовать файлы?
8.  **Важные моменты и частые ошибки.**
9.  **Практика:** Создаем свою мини-анкету или викторину.

---

## 1. Аналогия: FSM - это как... 🤔

Чтобы лучше понять, что такое FSM, представь:

* **Заполнение анкеты онлайн:** Каждое поле (имя, возраст, город) - это **состояние**. Система помнит, что ты ввел в предыдущие поля, пока ты не нажмешь "Отправить" (финальное состояние).
* **Телефонный робот-помощник:** "Чтобы узнать баланс, нажмите 1" (переход в состояние "запрос баланса"). "Введите номер карты" (состояние "ожидание номера карты"). "Спасибо, ваш баланс..." (конечное состояние).
* **Мастер установки программы (Wizard):** Каждый шаг ("Next", "Choose folder", "Install") - это состояние. Программа помнит твои выборы на предыдущих шагах.
* **Автомат с напитками:**
    1.  Состояние: `Ожидание денег`.
    2.  Вставил купюру -> Переход в состояние: `Ожидание выбора`.
    3.  Нажал кнопку -> Переход в состояние: `Выдача напитка`.
    4.  Напиток выдан -> Переход в состояние: `Ожидание денег`.

Во всех этих примерах есть:
* **Четкие состояния**.
* **Переходы** между ними по какому-то событию (ввод данных, нажатие кнопки).
* Часто есть **память** о предыдущих шагах (какой напиток выбран, какие данные введены).

---

## 2. Зачем вообще нужны состояния? 🤷‍♀️

Вернемся к нашему боту. Без FSM, если ты напишешь так:

```python
# НЕПРАВИЛЬНЫЙ ПОДХОД БЕЗ FSM

@router.message(Command("register"))
async def cmd_register(message: types.Message):
    await message.answer("Введите ваше имя:")
    # Как боту узнать, что СЛЕДУЮЩЕЕ сообщение - это имя? Никак!

@router.message(F.text) # Этот хэндлер ловит ВЕСЬ текст
async def process_text(message: types.Message):
    user_text = message.text
    # Это имя? Или возраст? Или просто случайный текст? Бот не знает!
    if ???: # Как проверить, что мы ждем имя?
         await message.answer("Теперь введите возраст:")
    elif ???: # Как проверить, что мы ждем возраст?
         await message.answer("Регистрация завершена!")
    else:
         # Обычное эхо или другая логика
         await message.answer(f"Вы написали: {user_text}")

```

Этот код не сработает как надо. `process_text` будет реагировать на *любое* текстовое сообщение. Он не знает, спросили ли мы имя перед этим.

**FSM решает эту проблему**, позволяя нам создавать обработчики, которые срабатывают **только тогда**, когда пользователь находится в **определенном состоянии**.

---

## 3. Ключевые компоненты FSM в `aiogram` 🛠️

Для работы с FSM в `aiogram` нам понадобятся три основные вещи:

### а) `StatesGroup` и `State`: Определяем наши "шаги"

`StatesGroup` - это класс, который помогает нам логически сгруппировать состояния, относящиеся к одному процессу (например, регистрации).
`State` - это конкретное состояние внутри группы.

```python
# Пример: states/registration.py
from aiogram.fsm.state import State, StatesGroup

class RegistrationStates(StatesGroup):
    waiting_for_name = State()      # Состояние ожидания имени
    waiting_for_age = State()       # Состояние ожидания возраста
    waiting_for_city = State()      # Состояние ожидания города
    # Можно добавлять сколько угодно состояний

# Мы создали "контейнер" RegistrationStates
# и внутри него "полочки": waiting_for_name, waiting_for_age, waiting_for_city
```

Теперь у нас есть имена для каждого шага нашей анкеты.

### б) `FSMContext`: "Волшебная сумка" 🎒

Когда пользователь входит в какое-то состояние (например, начинает регистрацию), `aiogram` предоставляет специальный объект `FSMContext` (часто его называют `state`). Это **временное хранилище данных** и инструмент управления состоянием **для конкретного пользователя в конкретном чате**.

* **Получение в обработчике:** Чтобы получить доступ к этому объекту, добавь параметр `state: FSMContext` в свой обработчик:

    ```python
    async def handle_name(message: types.Message, state: FSMContext):
        # Теперь мы можем работать с состоянием этого пользователя
        ...
    ```

* **Основные методы `FSMContext`:**
    * `await state.set_state(НужноеСостояние)`: **Переводит** пользователя в указанное состояние (например, `await state.set_state(RegistrationStates.waiting_for_age)`).
    * `await state.get_state()`: **Возвращает** текущее состояние пользователя (например, `'RegistrationStates:waiting_for_name'`) или `None`, если пользователь не в состоянии.
    * `await state.update_data(ключ=значение, ...) `: **Добавляет или обновляет** данные во временном хранилище (в "сумке") для этого пользователя. Например, `await state.update_data(name=message.text)`. Данные хранятся как словарь.
    * `await state.get_data()`: **Возвращает** словарь со всеми данными, которые мы сохранили в "сумку" для этого пользователя (`{'name': 'Вася', 'age': 30}`).
    * `await state.clear()`: **Полностью очищает** состояние и все временные данные пользователя. **Очень важно вызывать это в конце процесса!**

### в) Хранилище (`Storage`): Где лежит "сумка"? 💾

Объекту `FSMContext` нужно где-то хранить информацию о текущем состоянии и временные данные для каждого пользователя. За это отвечает **хранилище состояний (storage)**.

* `MemoryStorage`: Самый простой вариант. Хранит все данные **в оперативной памяти** бота.
    * **Плюс:** Легко настроить (мы уже использовали его в `main.py`: `dp = Dispatcher(storage=MemoryStorage())`).
    * **Минус:** **Все состояния и данные теряются при перезапуске бота!** Подходит для простых ботов или для тестирования.
* **Другие хранилища (для реальных ботов):**
    * `RedisStorage`: Хранит данные во внешней базе данных Redis (быстрая key-value база). Данные **сохраняются** между перезапусками. Рекомендуется для большинства ботов.
    * `MongoStorage`: Хранит данные в MongoDB.
    * `SQLAlchemyStorage` (и другие для SQL): Хранят данные в реляционных базах данных (PostgreSQL, SQLite).
    * `FileStorage`: Хранит данные в файлах (менее надежно, чем Redis/DB).

> **На этом уроке мы будем использовать `MemoryStorage`, так как он проще всего для старта. Но помни, что для серьезного бота нужно будет настроить что-то более надежное, например, `RedisStorage`.**

---

## 4. Шаг за шагом: Пишем простую анкету (Регистрация) 📝

Давай создадим процесс регистрации, где бот спрашивает имя, возраст и город.

### Шаг 1: Определяем состояния

Создадим файл `states/registration.py`:

```python
# Файл: states/registration.py
from aiogram.fsm.state import State, StatesGroup

class RegistrationStates(StatesGroup):
    waiting_for_name = State()
    waiting_for_age = State()
    waiting_for_city = State()

print("States defined!") # Для проверки при запуске
```

Не забудь создать пустой файл `states/__init__.py`, чтобы Python мог импортировать из этой папки.

### Шаг 2: Создаем обработчики

Создадим файл `handlers/registration_handlers.py`:

```python
# Файл: handlers/registration_handlers.py
from aiogram import Router, F, types, html
from aiogram.filters import Command, StateFilter
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import default_state # Импортируем default_state

# Импортируем наши состояния
from states.registration import RegistrationStates

router = Router()

# --- Обработчик команды /register ---
# Он будет работать ТОЛЬКО если пользователь НЕ находится в каком-либо состоянии
@router.message(Command("register"), StateFilter(default_state))
async def start_registration(message: types.Message, state: FSMContext):
    await message.answer("Давай начнем регистрацию! ✨\nПожалуйста, введите ваше имя:")
    # Устанавливаем пользователю состояние "ожидания имени"
    await state.set_state(RegistrationStates.waiting_for_name)
    print(f"User {message.from_user.id} started registration, set state -> waiting_for_name")

# --- Обработчик для ввода имени ---
# Сработает ТОЛЬКО если пользователь в состоянии waiting_for_name И прислал текст
@router.message(RegistrationStates.waiting_for_name, F.text)
async def process_name(message: types.Message, state: FSMContext):
    user_name = message.text
    # Сохраняем имя в FSMContext ("волшебную сумку")
    await state.update_data(name=user_name)
    print(f"User {message.from_user.id} entered name: {user_name}, saved to state data")
    await message.answer(f"Отлично, {html.quote(user_name)}!\nТеперь введите ваш возраст:")
    # Устанавливаем состояние "ожидания возраста"
    await state.set_state(RegistrationStates.waiting_for_age)
    print(f"User {message.from_user.id} set state -> waiting_for_age")

# --- Обработчик для ввода возраста ---
# Сработает ТОЛЬКО если пользователь в состоянии waiting_for_age
@router.message(RegistrationStates.waiting_for_age)
async def process_age(message: types.Message, state: FSMContext):
    # Проверяем, является ли введенный текст числом
    if not message.text or not message.text.isdigit():
        await message.answer("Пожалуйста, введите возраст цифрами.")
        print(f"User {message.from_user.id} entered invalid age: {message.text}")
        # Остаемся в том же состоянии waiting_for_age
        return # Выходим из обработчика

    age = int(message.text)
    # Простая проверка на адекватность возраста
    if age < 4 or age > 120:
        await message.answer("Хм, такой возраст кажется маловероятным. Попробуйте еще раз.")
        print(f"User {message.from_user.id} entered unlikely age: {age}")
        # Остаемся в том же состоянии waiting_for_age
        return # Выходим из обработчика

    # Сохраняем возраст в FSMContext
    await state.update_data(age=age)
    print(f"User {message.from_user.id} entered age: {age}, saved to state data")
    await message.answer(f"Записал возраст: {age}.\nТеперь введите ваш город:")
    # Устанавливаем состояние "ожидания города"
    await state.set_state(RegistrationStates.waiting_for_city)
    print(f"User {message.from_user.id} set state -> waiting_for_city")


# --- Обработчик для ввода города ---
# Сработает ТОЛЬКО если пользователь в состоянии waiting_for_city И прислал текст
@router.message(RegistrationStates.waiting_for_city, F.text)
async def process_city(message: types.Message, state: FSMContext):
    city = message.text
    # Сохраняем город
    await state.update_data(city=city)
    print(f"User {message.from_user.id} entered city: {city}, saved to state data")

    # Получаем все сохраненные данные из "сумки"
    user_data = await state.get_data()
    print(f"User {message.from_user.id} finished registration. Data: {user_data}")

    # Формируем итоговое сообщение
    name = user_data.get('name', 'Не указано')
    age = user_data.get('age', 'Не указан')
    city_saved = user_data.get('city', 'Не указан')

    response_text = (
        f"🎉 Регистрация завершена! 🎉\n\n"
        f"Вот ваши данные:\n"
        f"👤 Имя: <b>{html.quote(name)}</b>\n"
        f"🎂 Возраст: <b>{age}</b>\n"
        f"🏙️ Город: <b>{html.quote(city_saved)}</b>\n\n"
        f"Спасибо!"
    )
    await message.answer(response_text, parse_mode="HTML")

    # ОЧЕНЬ ВАЖНО: Очищаем состояние и данные пользователя
    await state.clear()
    print(f"User {message.from_user.id} state cleared.")
    # Можно также установить default_state явно, если нужно
    # await state.set_state(default_state)

print("Registration handlers registered!") # Для проверки
```

Не забудь создать пустой файл `handlers/__init__.py`.

### Шаг 3: Подключаем состояния и обработчики

Теперь нужно "рассказать" нашему `Dispatcher` об этих состояниях и обработчиках. Обновим `main.py`:

```python
# Файл: main.py
import asyncio
import logging
from aiogram import Bot, Dispatcher, types
from aiogram.fsm.storage.memory import MemoryStorage # Используем MemoryStorage

from config import TOKEN # Ваш токен из config.py

# Импортируем роутеры из handlers
from handlers import handler as common_handler # Наш старый роутер
from handlers import registration_handlers # Наш новый роутер для регистрации

# Настройка логирования (опционально, но полезно)
logging.basicConfig(level=logging.INFO)

async def main():
    # Создаем объекты Bot и Dispatcher
    bot = Bot(token=TOKEN)
    # Используем MemoryStorage для FSM
    storage = MemoryStorage()
    dp = Dispatcher(storage=storage)
    print("Bot and Dispatcher created")

    # Подключаем роутеры
    dp.include_router(common_handler.router)
    print("Common handlers router included")
    dp.include_router(registration_handlers.router)
    print("Registration handlers router included")

    # Пропускаем накопленные апдейты (если бот был выключен)
    await bot.delete_webhook(drop_pending_updates=True)
    print("Webhook deleted, pending updates dropped")

    # Запускаем polling
    print("Starting polling...")
    await dp.start_polling(bot)

if __name__ == "__main__":
    try:
        asyncio.run(main())
    except (KeyboardInterrupt, SystemExit):
        print("Bot stopped")
```

**Что мы сделали:**

1.  Импортировали `registration_handlers`.
2.  Передали `MemoryStorage()` в `Dispatcher`.
3.  Подключили `registration_handlers.router` к `Dispatcher` с помощью `dp.include_router()`. **Важно:** Порядок подключения роутеров может иметь значение, если у них есть пересекающиеся фильтры. Обычно специфичные (как регистрация) идут раньше общих (как эхо).

**Запускай и тестируй!**

* Отправь `/register`. Бот должен спросить имя.
* Отправь имя. Бот должен спросить возраст.
* Попробуй отправить текст вместо возраста. Бот должен попросить ввести цифры.
* Введи корректный возраст. Бот спросит город.
* Введи город. Бот покажет все данные и завершит регистрацию.
* Попробуй снова ввести город. Бот должен отреагировать как обычный эхо-бот (если он есть), потому что состояние уже очищено.
* Попробуй снова отправить `/register`. Процесс должен начаться заново.

---

## 5. "Стоп! Я передумал!" - Отмена состояния 🚫

Что если пользователь начал регистрацию, а потом передумал? Ему нужен способ отменить процесс на любом шаге.

Добавим команду `/cancel` в `handlers/registration_handlers.py`.

```python
# Файл: handlers/registration_handlers.py
# ... (импорты как раньше)
from aiogram.fsm.state import default_state # Добавлен default_state

router = Router()

# --- Команда отмены ---
# Будет работать В ЛЮБОМ СОСТОЯНИИ (StateFilter("*"))
# или можно указать конкретные состояния: StateFilter(RegistrationStates.waiting_for_name, ...)
@router.message(Command("cancel"), StateFilter("*"))
async def cancel_registration(message: types.Message, state: FSMContext):
    current_state = await state.get_state()
    if current_state is None:
        # Если пользователь не в состоянии, просто сообщаем ему об этом
        await message.answer("Вы сейчас не в процессе регистрации.")
        return

    print(f"User {message.from_user.id} cancelled state {current_state}")
    # Очищаем состояние и данные
    await state.clear()
    # await state.set_state(default_state) # Можно и так, если нужно точно вернуть в None
    await message.answer("Действие отменено. Вы вышли из процесса регистрации.")

# --- Обработчик команды /register ---
# Указываем, что он работает только из default_state (когда нет состояния)
# чтобы /register не сработал во время самой регистрации
@router.message(Command("register"), StateFilter(default_state))
async def start_registration(message: types.Message, state: FSMContext):
    # ... (код как раньше)

# --- Обработчик для ввода имени ---
# ... (код как раньше)

# --- Обработчик для ввода возраста ---
# ... (код как раньше)

# --- Обработчик для ввода города ---
# ... (код как раньше)

print("Registration handlers (with cancel) registered!")

```

**Ключевые моменты:**

1.  `StateFilter("*")`: Этот фильтр означает, что обработчик сработает, если пользователь находится **в любом состоянии**, отличном от `None` (состояния по умолчанию).
2.  Проверка `current_state is None`: Мы добавили проверку на случай, если пользователь введет `/cancel`, когда он и так не был ни в каком состоянии.
3.  `await state.clear()`: Снова используем для очистки состояния и данных.

**Порядок важен!** Обработчик для `/cancel` должен быть зарегистрирован в `aiogram` так, чтобы он проверялся *до* обработчиков для конкретных состояний, если они оба могут сработать (например, если пользователь отправит текст `/cancel` в состоянии ожидания текста). В `aiogram 3.x` порядок внутри одного `Router` обычно соответствует порядку написания функций. Если `/cancel` в отдельном роутере, то порядок `dp.include_router()` влияет.

**Тестируй:** Начни регистрацию (`/register`), введи имя, а потом отправь `/cancel`. Бот должен подтвердить отмену. Попробуй отправить текст после этого - он должен обрабатываться как обычно (эхо).

---

## 6. "Ой, я отправил не то!" - Обработка некорректного ввода 🖼️➡️🔢

В нашей анкете мы уже добавили проверку на то, что возраст введен цифрами. Но что если пользователь вместо имени пришлет фото? Или стикер вместо города?

Наш текущий код просто проигнорирует это, потому что фильтры настроены на `F.text` там, где мы ждем текст. Но лучше сообщить пользователю, что он ввел что-то не то.

Добавим "ловушки" для неправильных типов данных в `handlers/registration_handlers.py`:

```python
# Файл: handlers/registration_handlers.py
# ... (импорты как раньше)

router = Router()

# --- Команда отмены ---
# ... (код как раньше)

# --- Обработчик команды /register ---
# ... (код как раньше)

# --- Обработчик для ввода имени (ждем ТЕКСТ) ---
@router.message(RegistrationStates.waiting_for_name, F.text)
async def process_name(message: types.Message, state: FSMContext):
    # ... (код как раньше)

# --- Ловушка для НЕ текста в состоянии waiting_for_name ---
@router.message(RegistrationStates.waiting_for_name)
async def process_name_invalid(message: types.Message):
    await message.answer("Пожалуйста, введите ваше имя текстом.")
    print(f"User {message.from_user.id} sent invalid data type for name: {message.content_type}")


# --- Обработчик для ввода возраста (ждем ТЕКСТ с цифрами) ---
@router.message(RegistrationStates.waiting_for_age, F.text) # Уточняем, что ждем текст
async def process_age(message: types.Message, state: FSMContext):
    # ... (код как раньше, с проверкой isdigit)

# --- Ловушка для НЕ текста в состоянии waiting_for_age ---
@router.message(RegistrationStates.waiting_for_age)
async def process_age_invalid(message: types.Message):
    await message.answer("Пожалуйста, введите возраст текстом (цифрами).")
    print(f"User {message.from_user.id} sent invalid data type for age: {message.content_type}")


# --- Обработчик для ввода города (ждем ТЕКСТ) ---
@router.message(RegistrationStates.waiting_for_city, F.text)
async def process_city(message: types.Message, state: FSMContext):
    # ... (код как раньше)

# --- Ловушка для НЕ текста в состоянии waiting_for_city ---
@router.message(RegistrationStates.waiting_for_city)
async def process_city_invalid(message: types.Message):
    await message.answer("Пожалуйста, введите название города текстом.")
    print(f"User {message.from_user.id} sent invalid data type for city: {message.content_type}")

print("Registration handlers (with cancel and validation) registered!")
```

**Как это работает:**

1.  Для каждого шага (состояния) мы теперь имеем **два** обработчика:
    * Первый: `RegistrationStates.ИмяСостояния, F.text` - ловит **правильный** тип ввода (текст).
    * Второй: `RegistrationStates.ИмяСостояния` (без `F.text`) - ловит **все остальное**, что пришло в этом состоянии (фото, стикеры, документы и т.д.).
2.  Aiogram проверяет обработчики по порядку. Если пришел текст, сработает первый. Если пришло не текст, первый не подойдет, и сработает второй (наша "ловушка").
3.  В "ловушке" мы просто просим пользователя ввести данные корректно и **не меняем состояние**. Пользователь остается на том же шаге анкеты.

**Тестируй:** Начни регистрацию, и когда бот попросит имя, отправь ему фото или стикер. Он должен вежливо попросить ввести текст.

---

## 7. Структура проекта с FSM 📂

Когда проект растет, важно держать его организованным. Для FSM рекомендуется такая структура:

```
your_bot_project/
├── venv/                 # Виртуальное окружение
├── main.py               # Точка входа, инициализация Bot, Dispatcher, запуск
├── config.py             # Токен и другие настройки
├── requirements.txt      # Зависимости (aiogram, etc.)
├── states/               # Папка для определения состояний FSM
│   ├── __init__.py       # Пустой файл
│   └── registration.py   # Состояния для регистрации (RegistrationStates)
│   └── quiz.py           # Состояния для викторины (QuizStates) ...
├── handlers/             # Папка для обработчиков
│   ├── __init__.py       # Пустой файл
│   ├── handler.py        # Общие обработчики (start, help, echo...)
│   ├── registration_handlers.py # Обработчики для регистрации FSM
│   └── quiz_handlers.py  # Обработчики для викторины FSM ...
└── utils/                # (Опционально) Вспомогательные функции
    └── ...
└── keyboards/            # (Опционально) Кнопки
    └── ...
```

**Преимущества:**

* **Логическое разделение:** Состояния отдельно, обработчики для них отдельно.
* **Масштабируемость:** Легко добавлять новые FSM-сценарии (викторины, заказы), не захламляя один файл.
* **Читаемость:** Понятно, где искать код, относящийся к определенной функции.

---

## 8. Важные моменты и частые ошибки ⚠️

* **ВСЕГДА очищайте состояние!** Забытый `await state.clear()` в конце FSM-цепочки приведет к тому, что пользователь "застрянет" в последнем состоянии и не сможет нормально пользоваться ботом.
* **Не забывайте `await`!** Все методы `FSMContext` (`set_state`, `update_data`, `get_data`, `clear`) - асинхронные, им нужен `await`.
* **Порядок обработчиков важен.** Особенно для команды отмены (`StateFilter("*")`) и для обработчиков на разные типы контента внутри одного состояния. Более конкретные фильтры должны идти раньше менее конкретных.
* **Используйте `StateFilter` в декораторах.** Первый аргумент декоратора `@router.message()` - это фильтр состояния.
* **Валидируйте ввод.** Не доверяйте данным от пользователя. Проверяйте тип, диапазон, формат там, где это возможно.
* **Давайте пользователю обратную связь.** Если ввод неверный, скажите ему об этом. Если состояние отменено - подтвердите.
* **Выбирайте правильное хранилище.** `MemoryStorage` хорош для начала, но для реального бота нужен `RedisStorage` или аналог.
* **Не усложняйте без надобности.** Если нужно просто получить ответ на один вопрос, FSM может быть избыточен. Но для последовательностей из 2+ шагов он идеален.

---

## 9. Практика: Твоя очередь! 🚀

Теперь, когда ты вооружен знаниями FSM, попробуй создать что-то свое!

**Идея 1: Простая форма обратной связи**

1.  Создай состояния: `FeedbackStates`: `waiting_for_feedback_message`, `waiting_for_confirmation`.
2.  Команда `/feedback` начинает процесс, просит ввести сообщение.
3.  После ввода сообщения, бот показывает его пользователю и спрашивает: "Отправить это сообщение? (да/нет)". Устанавливает состояние `waiting_for_confirmation`.
4.  Если пользователь отвечает "да", бот пишет "Спасибо за отзыв!" и очищает состояние. (В реальном боте он бы отправил отзыв админу).
5.  Если пользователь отвечает "нет", бот пишет "Отменено" и очищает состояние.
6.  Добавь команду `/cancel`.

**Идея 2: Мини-викторина**

1.  Создай состояния: `QuizStates`: `question_1`, `question_2`, `results`.
2.  Создай словарь с вопросами и правильными ответами.
3.  Команда `/quiz` начинает викторину, задает первый вопрос и ставит состояние `question_1`. Сохраняй счет пользователя в `state.update_data(score=0)`.
4.  В обработчике для `question_1` проверь ответ, обнови счет (`user_data = await state.get_data(); current_score = user_data.get('score', 0); await state.update_data(score=current_score + 1)`), задай второй вопрос и переведи в `question_2`.
5.  В обработчике для `question_2` сделай то же самое, а затем покажи итоговый счет и очисти состояние (`state.clear()`).
6.  Добавь обработку неправильных ответов и команду `/cancel`.

---

Фух! Это был действительно большой урок! 😅 Машина состояний - это мощный инструмент, и поначалу он может показаться сложным. Но как только ты поймешь основной принцип (состояние -> сбор данных -> переход -> ... -> очистка), ты сможешь создавать гораздо более умных и полезных ботов.

Не бойся экспериментировать, заглядывать в документацию `aiogram` по FSM и задавать вопросы! Удачи!
