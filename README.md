import asyncio
import sqlite3
from datetime import datetime, timedelta
from aiogram import Bot, Dispatcher, Router, types
from aiogram.filters import Command
import re

API_TOKEN = ""

bot = Bot(token=API_TOKEN)
dp = Dispatcher()
router = Router()

conn = sqlite3.connect("birthdays.db")
cursor = conn.cursor()
cursor.execute("""
CREATE TABLE IF NOT EXISTS users (
    user_id INTEGER PRIMARY KEY,
    birth_date TEXT
)
""")
conn.commit()


@router.message(Command("start"))
async def start_command(message: types.Message):
    await message.answer(
        "Привет! Введи свою дату рождения в формате ДД.ММ.ГГГГ, чтобы я мог запомнить её."
    )


@router.message(
    lambda message: re.match(r"^\d{2}\.\d{2}\.\d{4}$", message.text))
async def save_birth_date(message: types.Message):
    try:
        birth_date = datetime.strptime(message.text, "%d.%m.%Y").date()
        cursor.execute(
            "REPLACE INTO users (user_id, birth_date) VALUES (?, ?)",
            (message.from_user.id, birth_date))
        conn.commit()
        await message.answer("Дата рождения успешно сохранена!")
    except ValueError:
        await message.answer(
            "Неверный формат даты. Попробуй ещё раз в формате ДД.ММ.ГГГГ.")


@router.message(Command("dr"))
async def days_to_birthday(message: types.Message):
    cursor.execute("SELECT birth_date FROM users WHERE user_id = ?",
                   (message.from_user.id, ))
    result = cursor.fetchone()
    if result:
        birth_date = datetime.strptime(result[0], "%Y-%m-%d").date()
        today = datetime.now().date()
        next_birthday = birth_date.replace(year=today.year)
        if next_birthday < today:
            next_birthday = next_birthday.replace(year=today.year + 1)
        days_left = (next_birthday - today).days
        await message.answer(
            f"До твоего дня рождения осталось {days_left} дней! 🎉")
    else:
        await message.answer(
            "Сначала введи свою дату рождения с помощью команды /start.")


@router.message()
async def handle_unknown_messages(message: types.Message):
    await message.answer(
        "Неверный формат даты. Попробуй ещё раз в формате ДД.ММ.ГГГГ или используй команды /start и /dr."
    )


async def weekly_notifications():
    while True:
        cursor.execute("SELECT user_id, birth_date FROM users")
        users = cursor.fetchall()
        today = datetime.now().date()
        for user_id, birth_date in users:
            birth_date = datetime.strptime(birth_date, "%Y-%m-%d").date()
            next_birthday = birth_date.replace(year=today.year)
            if next_birthday < today:
                next_birthday = next_birthday.replace(year=today.year + 1)
            days_left = (next_birthday - today).days
            try:
                await bot.send_message(
                    user_id,
                    f"Напоминание: до твоего дня рождения осталось {days_left} дней! 🎂"
                )
            except Exception as e:
                print(
                    f"Ошибка при отправке сообщения пользователю {user_id}: {e}"
                )
        await asyncio.sleep(7 * 24 * 60 * 60)


async def main():
    dp.include_router(router)
    asyncio.create_task(weekly_notifications())
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
