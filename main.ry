import os
import sqlite3
import asyncio
from datetime import datetime, timedelta
from aiogram import Bot, Dispatcher, F, types
from aiogram.filters import Command
from aiogram.types import InlineKeyboardButton, InlineKeyboardMarkup, LabeledPrice, PreCheckoutQuery
from google import genai

BOT_TOKEN = os.environ.get("BOT_TOKEN")
GEMINI_KEY = os.environ.get("GEMINI_API_KEY")

FREE_DAILY_LIMIT = 3

# Тарифы подписки: срок указывается в днях, цена — в Telegram Stars (XTR).
SUBSCRIPTION_PLANS = {
    "month": {"title": "1 месяц", "days": 30, "stars": 150},
    "quarter": {"title": "3 месяца", "days": 90, "stars": 350},
    "year": {"title": "1 год", "days": 365, "stars": 1000},
}

client = genai.Client(api_key=GEMINI_KEY)

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()

# Системный промт для автоэксперта
SYSTEM_PROMPT = """
Ты — бот-автоэксперт. Твоя задача — выдавать ТОЛЬКО точную стоимость машины. Тебе запрещено писать длинные тексты, давать советы по торгу и рассуждать.

Критически важные правила:
1. ТОЛЬКО 4 ПАРАМЕТРОВ: Тебе нужны — Марка/Модель, Год, Пробег, Состояние.
2. НЕДОСТАТОЧНО ДАННЫХ: Если нет хотя бы ОДНОГО из 4 параметров — НЕ выдавай цену! НЕ придумывай марку сам .
3. ВСЕ ДАННЫЕ ЕСТЬ: Выдавай ТОЛЬКО ШАБЛОН 2. Никаких доп. разделов вроде "Аргументы для торга" или "Болячки".

⛔ ШАБЛОН 1 (Запрос данных):
⚠️ Не хватает данных для оценки!
Пожалуйста, укажи:
• 🚘 Модель и год: [?]
• 🛣 Пробег: [?]
• 🛠 Состояние: [?]
👉 Отправь недостающие данные одним сообщением!
"""

# База данных
def init_db():
    conn = sqlite3.connect("bot_database.db")
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS users (
            user_id INTEGER PRIMARY KEY,
            requests_count INTEGER DEFAULT 0,
            last_request_date TEXT,
            unlimited_until TEXT
        )
    ''')
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS payments (
            telegram_payment_charge_id TEXT PRIMARY KEY,
            user_id INTEGER NOT NULL,
            payload TEXT NOT NULL,
            stars_amount INTEGER NOT NULL,
            plan_key TEXT NOT NULL,
            paid_at TEXT NOT NULL
        )
    ''')
    conn.commit()
    conn.close()

init_db()

def check_access(user_id):
    conn = sqlite3.connect("bot_database.db")
    cursor = conn.cursor()
    cursor.execute("SELECT requests_count, last_request_date, unlimited_until FROM users WHERE user_id = ?", (user_id,))
    row = cursor.fetchone()
    
    today_str = datetime.now().strftime("%Y-%m-%d")
    
    if not row:
        cursor.execute("INSERT INTO users (user_id, requests_count, last_request_date) VALUES (?, 0, ?)", (user_id, today_str))
        conn.commit()
        conn.close()
        return True, "free"
        
    requests_count, last_request_date, unlimited_until = row
    
    if unlimited_until:
        unlimited_date = datetime.strptime(unlimited_until, "%Y-%m-%d %H:%M:%S")
        if datetime.now() < unlimited_date:
            conn.close()
            return True, "unlimited"
            
    if last_request_date != today_str:
        cursor.execute("UPDATE users SET requests_count = 0, last_request_date = ? WHERE user_id = ?", (today_str, user_id))
        conn.commit()
        requests_count = 0
        
    if requests_count < FREE_DAILY_LIMIT:
        conn.close()
        return True, "free"
        
    conn.close()
    return False, "limit_exceeded"

def register_request(user_id, access_type):
    if access_type == "unlimited":
        return
    conn = sqlite3.connect("bot_database.db")
    cursor = conn.cursor()
    cursor.execute("UPDATE users SET requests_count = requests_count + 1 WHERE user_id = ?", (user_id,))
    conn.commit()
    conn.close()

def get_plan(payload):
    prefix = "subscription_"
    if not payload.startswith(prefix):
        return None, None

    plan_key = payload[len(prefix):]
    plan = SUBSCRIPTION_PLANS.get(plan_key)
    return plan_key, plan

def activate_subscription(user_id, payment, plan_key, plan):
    conn = sqlite3.connect("bot_database.db")
    cursor = conn.cursor()

    try:
        cursor.execute(
            """
            INSERT OR IGNORE INTO payments (
                telegram_payment_charge_id,
                user_id,
                payload,
                stars_amount,
                plan_key,
                paid_at
            ) VALUES (?, ?, ?, ?, ?, ?)
            """,
            (
                payment.telegram_payment_charge_id,
                user_id,
                payment.invoice_payload,
                payment.total_amount,
                plan_key,
                datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
            ),
        )

        if cursor.rowcount != 1:
            conn.rollback()
            return False, None

        today_str = datetime.now().strftime("%Y-%m-%d")
        cursor.execute(
            """
            INSERT OR IGNORE INTO users (
                user_id, requests_count, last_request_date
            ) VALUES (?, 0, ?)
            """,
            (user_id, today_str),
        )
        cursor.execute(
            "SELECT unlimited_until FROM users WHERE user_id = ?",
            (user_id,),
        )
        row = cursor.fetchone()

        current_until = datetime.now()
        if row and row[0]:
            try:
                stored_until = datetime.strptime(row[0], "%Y-%m-%d %H:%M:%S")
                if stored_until > current_until:
                    current_until = stored_until
            except ValueError:
                pass

        unlimited_date = current_until + timedelta(days=plan["days"])
        unlimited_date_str = unlimited_date.strftime("%Y-%m-%d %H:%M:%S")
        cursor.execute(
            """
            UPDATE users
            SET unlimited_until = ?
            WHERE user_id = ?
            """,
            (unlimited_date_str, user_id),
        )
        conn.commit()
        return True, unlimited_date
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()

@dp.message(Command("start"))
async def cmd_start(message: types.Message):
    await message.answer(
        "🚗 Привет! Я бот-автоэксперт.\n\n"
        f"У тебя есть {FREE_DAILY_LIMIT} бесплатных запроса на сегодня. Отправь мне данные машины (Марка/Модель, год, пробег, состояние), и я назову её стоимость!"
    )

@dp.message(Command("buy"))
async def cmd_buy(message: types.Message):
    keyboard = InlineKeyboardMarkup(inline_keyboard=[
        [
            InlineKeyboardButton(
                text="1 месяц — 150 ⭐",
                callback_data="buy_month",
            )
        ],
        [
            InlineKeyboardButton(
                text="3 месяца — 350 ⭐",
                callback_data="buy_quarter",
            )
        ],
        [
            InlineKeyboardButton(
                text="1 год — 1000 ⭐",
                callback_data="buy_year",
            )
        ],
    ])
    await message.answer(
        "Выберите срок безлимитного доступа:",
        reply_markup=keyboard,
    )

@dp.callback_query(F.data.startswith("buy_"))
async def process_buy(callback: types.CallbackQuery):
    plan_key = callback.data.removeprefix("buy_") if callback.data else ""
    plan = SUBSCRIPTION_PLANS.get(plan_key)
    if plan is None or callback.message is None:
        await callback.answer("Тариф недоступен.", show_alert=True)
        return

    await callback.message.answer_invoice(
        title=f"Безлимитный доступ — {plan['title']}",
        description=f"Безлимитные запросы на {plan['days']} дней",
        payload=f"subscription_{plan_key}",
        currency="XTR",
        prices=[
            LabeledPrice(
                label=f"Подписка на {plan['title']}",
                amount=plan["stars"],
            )
        ],
        provider_token="",
    )
    await callback.answer()

@dp.pre_checkout_query()
async def process_pre_checkout(pre_checkout_query: PreCheckoutQuery):
    plan_key, plan = get_plan(pre_checkout_query.invoice_payload)
    if (
        plan is None
        or pre_checkout_query.currency != "XTR"
        or pre_checkout_query.total_amount != plan["stars"]
    ):
        await pre_checkout_query.answer(
            ok=False,
            error_message="Тариф недействителен. Откройте /buy и выберите его заново.",
        )
        return

    await pre_checkout_query.answer(ok=True)

@dp.message(F.successful_payment)
async def successful_payment(message: types.Message):
    payment = message.successful_payment
    if payment is None or message.from_user is None:
        return

    plan_key, plan = get_plan(payment.invoice_payload)
    if (
        plan is None
        or payment.currency != "XTR"
        or payment.total_amount != plan["stars"]
    ):
        await message.answer(
            "Платёж получен, но его параметры не удалось проверить. "
            "Обратитесь к администратору."
        )
        return

    credited, unlimited_date = activate_subscription(
        user_id=message.from_user.id,
        payment=payment,
        plan_key=plan_key,
        plan=plan,
    )
    if not credited:
        await message.answer("Этот платёж уже был обработан ранее.")
        return

    await message.answer(
        f"Оплата прошла успешно! Безлимитный доступ на {plan['title']} "
        f"активирован до {unlimited_date.strftime('%d.%m.%Y %H:%M')}."
    )

@dp.message(F.text)
async def handle_message(message: types.Message):
    user_id = message.from_user.id
    has_access, access_type = check_access(user_id)
    
    if not has_access:
        await message.answer(
            "🛑 **Бесплатный дневной лимит исчерпан!**\n\n"
            f"Вы использовали все {FREE_DAILY_LIMIT} бесплатные попытки на сегодня.\n"
            "Лимит обновится завтра, либо вы можете оформить безлимит прямо сейчас через /buy",
            parse_mode="Markdown"
        )
        return
        
    await bot.send_chat_action(chat_id=message.chat.id, action="typing")
    
    try:
        # Передаем системный промт вместе с сообщением пользователя в Gemini
        response = client.models.generate_content(
            model="gemini-3.6-flash",
            contents=f"{SYSTEM_PROMPT}\n\nЗапрос пользователя: {message.text}"
        )
        
        register_request(user_id, access_type)
        await message.answer(response.text)
        
    except Exception as e:
        await message.answer(f"Произошла ошибка при обращении к нейросети: {e}")

@dp.message(Command("start"))
async def cmd_start(message: types.Message):
    admin_id = 5142358103
    await bot.send_message(admin_id, f"Новый пользователь: @{message.from_user.username}")

                        
async def main():
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
