"""
IQ-тест для Telegram бота (инлайн-кнопки + reply-клавиатура)
Команды:
/start - приветствие
/iqtest - начать тест
/iq - показать текущий IQ
/hint - подсказка к текущему вопросу
/reset - сбросить свой IQ до 100
"""

import os
import sqlite3
from pyrogram import Client, filters
from pyrogram.types import (
    Message,
    InlineKeyboardMarkup,
    InlineKeyboardButton,
    CallbackQuery,
    ReplyKeyboardMarkup,
    KeyboardButton
)

# Импортируем настройки из config.py
from config import API_ID, API_HASH, BOT_TOKEN

# --- Функции для работы с БД ---
DB_PATH = 'databaze/scores.db'

def setup_database():
    os.makedirs(os.path.dirname(DB_PATH), exist_ok=True)
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS scores (
            user_id INTEGER PRIMARY KEY,
            score INTEGER DEFAULT 0
        )
    ''')
    conn.commit()
    conn.close()

def update_score(user_id, score_increment):
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute('''
        INSERT INTO scores (user_id, score) VALUES (?, ?)
        ON CONFLICT(user_id) DO UPDATE SET score = score + ?
    ''', (user_id, score_increment, score_increment))
    conn.commit()
    conn.close()

def get_score(user_id):
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute('SELECT score FROM scores WHERE user_id = ?', (user_id,))
    result = cursor.fetchone()
    conn.close()
    return result[0] if result else 0

def reset_score(user_id):
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute('INSERT OR REPLACE INTO scores (user_id, score) VALUES (?, ?)', (user_id, 0))
    conn.commit()
    conn.close()
# --------------------------------

# Хранилище данных пользователей (активные сессии)
user_sessions = {}

# Вопросы теста с подсказками
QUESTIONS = [
    {
        "question": "Сколько месяцев в году имеют 28 дней?",
        "options": ["1", "2", "12", "Все"],
        "correct": 3,
        "hint": "Подумайте, есть ли месяцы, в которых меньше 28 дней? Например, февраль обычно имеет 28, а в високосный год — 29. Но любой месяц имеет хотя бы 28 дней."
    },
    {
        "question": "Что тяжелее: килограмм ваты или килограмм железа?",
        "options": ["Вата", "Железо", "Одинаково"],
        "correct": 2,
        "hint": "Вспомните, что такое килограмм — это мера массы. Один килограмм чего угодно весит одинаково."
    },
    {
        "question": "Если перевернуть эту цифру, она уменьшится на 3. Что это за цифра?",
        "options": ["6", "8", "9", "3"],
        "correct": 2,
        "hint": "Попробуйте перевернуть каждую цифру. 9 переворачивается в 6, а 6 в 9. Какая уменьшится?"
    },
    {
        "question": "У человека было 10 овец. Все, кроме 9, умерли. Сколько осталось?",
        "options": ["1", "9", "10", "0"],
        "correct": 1,
        "hint": "Фраза «все, кроме 9» означает, что 9 остались живы, а остальные умерли."
    },
    {
        "question": "Что можно приготовить, но нельзя съесть?",
        "options": ["Суп", "Уроки", "Кофе", "Завтрак"],
        "correct": 1,
        "hint": "Здесь игра слов: приготовить можно еду, а можно подготовить уроки."
    }
]

# Инициализация бота
bot = Client("iq_test_bot", api_id=API_ID, api_hash=API_HASH, bot_token=BOT_TOKEN)

# --- Функция для reply-клавиатуры (постоянные кнопки команд) ---
def get_main_keyboard():
    """Возвращает клавиатуру с основными командами."""
    keyboard = [
        [KeyboardButton("/start"), KeyboardButton("/iqtest")],
        [KeyboardButton("/iq"), KeyboardButton("/reset")],
        [KeyboardButton("/hint")]
    ]
    return ReplyKeyboardMarkup(keyboard, resize_keyboard=True, one_time_keyboard=False)

# --- Функции для отправки вопросов с инлайн-кнопками ---
def get_question_keyboard(step):
    """Создаёт инлайн-клавиатуру для текущего вопроса."""
    q = QUESTIONS[step]
    buttons = []
    for i, opt in enumerate(q["options"]):
        buttons.append([InlineKeyboardButton(opt, callback_data=f"answer_{i}")])
    buttons.append([InlineKeyboardButton("💡 Подсказка", callback_data="hint")])
    return InlineKeyboardMarkup(buttons)

async def send_question(chat_id, user_id):
    step = user_sessions[user_id]["step"]
    if step < len(QUESTIONS):
        q = QUESTIONS[step]
        # Отправляем вопрос с инлайн-кнопками; reply-клавиатура остаётся внизу
        await bot.send_message(
            chat_id,
            f"❓ Вопрос {step+1}/{len(QUESTIONS)}:\n\n{q['question']}",
            reply_markup=get_question_keyboard(step)
        )
    else:
        # Тест окончен
        score = user_sessions[user_id]["score"]
        total = len(QUESTIONS)
        update_score(user_id, score)
        total_score = get_score(user_id)
        iq = 100 + total_score * 10
        await bot.send_message(
            chat_id,
            f"✅ Тест завершён!\n\n"
            f"Правильных ответов в этом тесте: {score} из {total}\n"
            f"🧠 Ваш общий IQ (с учётом всех попыток): {iq}\n\n"
            f"Чтобы пройти заново, нажмите кнопку /iqtest или кнопку ниже.",
            reply_markup=InlineKeyboardMarkup([
                [InlineKeyboardButton("🔄 Пройти заново", callback_data="restart_test")]
            ])
        )
        del user_sessions[user_id]

# --- Обработчики команд ---
@bot.on_message(filters.command("start"))
async def start_cmd(client: Client, message: Message):
    await message.reply(
        "👋 Привет! Я бот для проверки IQ.\n\n"
        "Используйте кнопки ниже для управления:\n"
        "• /iqtest - начать тест\n"
        "• /iq - показать текущий IQ\n"
        "• /reset - сбросить IQ до 100\n"
        "• /hint - подсказка (во время теста)",
        reply_markup=get_main_keyboard()
    )

@bot.on_message(filters.command("iqtest"))
async def iqtest_cmd(client: Client, message: Message):
    user_id = message.from_user.id
    if user_id in user_sessions:
        await message.reply(
            "Вы уже проходите тест. Хотите начать заново? Отправьте /iqtest ещё раз.",
            reply_markup=get_main_keyboard()
        )
        return
    user_sessions[user_id] = {"step": 0, "score": 0}
    await message.reply("Начинаем тест!", reply_markup=get_main_keyboard())
    await send_question(message.chat.id, user_id)

@bot.on_message(filters.command("iq"))
async def iq_cmd(client: Client, message: Message):
    user_id = message.from_user.id
    total_score = get_score(user_id)
    iq = 100 + total_score * 10
    await message.reply(f"🧠 Ваш текущий IQ: {iq}", reply_markup=get_main_keyboard())

@bot.on_message(filters.command("hint"))
async def hint_cmd(client: Client, message: Message):
    user_id = message.from_user.id
    if user_id not in user_sessions:
        await message.reply(
            "Сейчас вы не проходите тест. Начните его с помощью /iqtest",
            reply_markup=get_main_keyboard()
        )
        return
    step = user_sessions[user_id]["step"]
    if step >= len(QUESTIONS):
        return
    hint_text = QUESTIONS[step].get("hint", "Нет подсказки.")
    await message.reply(f"💡 Подсказка к вопросу {step+1}:\n\n{hint_text}")

@bot.on_message(filters.command("reset"))
async def reset_cmd(client: Client, message: Message):
    user_id = message.from_user.id
    current_score = get_score(user_id)
    if current_score == 0:
        await message.reply(
            "Ваш IQ уже равен 100 (очков нет).",
            reply_markup=get_main_keyboard()
        )
        return
    # Запрашиваем подтверждение
    keyboard = InlineKeyboardMarkup([
        [
            InlineKeyboardButton("✅ Да, сбросить", callback_data="reset_confirm"),
            InlineKeyboardButton("❌ Нет", callback_data="reset_cancel")
        ]
    ])
    await message.reply(
        "⚠️ Вы уверены, что хотите сбросить все свои очки и вернуть IQ к 100? "
        "Это действие нельзя отменить.",
        reply_markup=keyboard
    )

# --- Обработчик инлайн-кнопок (callback query) ---
@bot.on_callback_query()
async def handle_callback(client: Client, callback_query: CallbackQuery):
    user_id = callback_query.from_user.id
    data = callback_query.data

    # Подтверждение сброса
    if data == "reset_confirm":
        reset_score(user_id)
        await callback_query.answer("✅ Очки сброшены! Ваш IQ теперь 100.", show_alert=True)
        await callback_query.message.edit_text(
            "✅ Очки сброшены. Ваш IQ теперь 100.\n\n"
            "Можете начать тест заново командой /iqtest."
        )
        # Отправляем новое сообщение с клавиатурой, чтобы вернуть reply-кнопки
        await bot.send_message(
            callback_query.message.chat.id,
            "Выберите действие:",
            reply_markup=get_main_keyboard()
        )
        return
    elif data == "reset_cancel":
        await callback_query.answer("❌ Сброс отменён.")
        await callback_query.message.edit_text("Сброс отменён.")
        await bot.send_message(
            callback_query.message.chat.id,
            "Выберите действие:",
            reply_markup=get_main_keyboard()
        )
        return

    # Если пользователь не в сессии, обрабатываем кнопку рестарта
    if user_id not in user_sessions:
        if data == "restart_test":
            user_sessions[user_id] = {"step": 0, "score": 0}
            await send_question(callback_query.message.chat.id, user_id)
            await callback_query.answer()
        else:
            await callback_query.answer("Сначала начните тест командой /iqtest", show_alert=True)
        return

    step = user_sessions[user_id]["step"]
    if step >= len(QUESTIONS):
        await callback_query.answer("Тест уже завершён.", show_alert=True)
        return

    if data == "hint":
        hint_text = QUESTIONS[step].get("hint", "Нет подсказки.")
        await callback_query.answer()
        await callback_query.message.reply(f"💡 Подсказка к вопросу {step+1}:\n\n{hint_text}")
        return

    if data.startswith("answer_"):
        try:
            answer_idx = int(data.split("_")[1])
        except ValueError:
            return

        q = QUESTIONS[step]
        if answer_idx == q["correct"]:
            user_sessions[user_id]["score"] += 1
            await callback_query.answer("✅ Верно!")
        else:
            await callback_query.answer("❌ Неверно!")

        user_sessions[user_id]["step"] += 1
        await callback_query.message.edit_text(
            callback_query.message.text,
            reply_markup=None
        )
        await send_question(callback_query.message.chat.id, user_id)

# Запуск бота
if __name__ == "__main__":
    setup_database()
    print("Бот запущен...")
    bot.run()
