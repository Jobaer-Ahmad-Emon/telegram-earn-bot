# telegram-earn-bot
Telegram earning bot with crypto deposit
bot.py
webhook.py
config.py
database.py
requirements.txt
requests
aiogramsaiograrequests
requestsBOT_TOKEN = "8640716147:AAG_Jwq7zsF066rYCuG4SLslC0JiJ0NA12I"
NOWPAY_API_KEY = "3xj1ETEP3jE1iK/NveGO/avyJAulFNaB"

ADMIN_ID = 123456789

WEBHOOK_URL = "https://your-app.onrender.com/webhook"import sqlite3

conn = sqlite3.connect("bot.db", check_same_thread=False)
cursor = conn.cursor()

cursor.execute("""
CREATE TABLE IF NOT EXISTS users (
    user_id INTEGER PRIMARY KEY,
    balance REAL DEFAULT 0
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS withdraws (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER,
    amount REAL,
    address TEXT,
    status TEXT
)
""")

conn.commit()from aiogram import Bot, Dispatcher, types
from aiogram.utils import executor
import requests
from config import *
from database import *

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher(bot)

@dp.message_handler(commands=['start'])
async def start(msg: types.Message):
    cursor.execute("INSERT OR IGNORE INTO users (user_id) VALUES (?)", (msg.from_user.id,))
    conn.commit()
    await msg.answer("👋 Welcome!\n\nCommands:\n/deposit\n/balance\n/withdraw")

@dp.message_handler(commands=['deposit'])
async def deposit(msg: types.Message):
    amount = 1  # USD

    url = "https://api.nowpayments.io/v1/invoice"

    headers = {
        "x-api-key": NOWPAY_API_KEY,
        "Content-Type": "application/json"
    }

    data = {
        "price_amount": amount,
        "price_currency": "usd",
        "pay_currency": "usdttrc20",
        "order_id": str(msg.from_user.id),
        "ipn_callback_url": WEBHOOK_URL
    }

    r = requests.post(url, json=data, headers=headers).json()

    await msg.answer(f"💰 Pay here:\n{r['invoice_url']}")

@dp.message_handler(commands=['balance'])
async def balance(msg: types.Message):
    cursor.execute("SELECT balance FROM users WHERE user_id=?", (msg.from_user.id,))
    bal = cursor.fetchone()[0]
    await msg.answer(f"💰 Balance: {bal} USD")

@dp.message_handler(commands=['withdraw'])
async def withdraw(msg: types.Message):
    await msg.answer("Send: amount address")

@dp.message_handler()
async def handle_withdraw(msg: types.Message):
    try:
        amount, address = msg.text.split()
        amount = float(amount)

        cursor.execute("SELECT balance FROM users WHERE user_id=?", (msg.from_user.id,))
        bal = cursor.fetchone()[0]

        if bal >= amount:
            cursor.execute(
                "INSERT INTO withdraws (user_id, amount, address, status) VALUES (?, ?, ?, ?)",
                (msg.from_user.id, amount, address, "pending")
            )
            conn.commit()
            await msg.answer("✅ Withdraw request submitted")
        else:
            await msg.answer("❌ Not enough balance")
    except:
        pass

executor.start_polling(dp)from flask import Flask, request
from database import *

app = Flask(__name__)

@app.route('/webhook', methods=['POST'])
def webhook():
    data = request.json

    if data["payment_status"] == "finished":
        user_id = int(data["order_id"])
        amount = float(data["price_amount"])

        cursor.execute(
            "UPDATE users SET balance = balance + ? WHERE user_id=?",
            (amount, user_id)
        )
        conn.commit()

        print(f"User {user_id} paid {amount}")

    return "OK"

app.run(host="0.0.0.0", port=10000)
