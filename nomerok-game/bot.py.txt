import json
import os
import sqlite3
import random
from datetime import datetime, timedelta, date
from flask import Flask, request, jsonify
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, WebAppInfo
from telegram.ext import Application, CommandHandler, ContextTypes

# ============================================================
# ЗАМЕНИТЕ ТРИ ЗНАЧЕНИЯ НИЖЕ
# ============================================================
BOT_TOKEN = 8986084844:AAFV8AGFzMSCro-hPBIINCApH0-zZPQ-_Uk  # ← ЗАМЕНИТЕ
ADMIN_ID = 8522347927                          # ← ЗАМЕНИТЕ (ваш ID)
WEBAPP_URL = "https://ваш-проект.railway.app" # ← ЗАМЕНИТЕ после деплоя
# ============================================================

DB_FILE = "game.db"

# ============ РЕДКОСТИ И ЦЕНЫ ============
RARITIES = {
    "common":    {"name": "Обычный",     "price": 100},
    "uncommon":  {"name": "Необычный",   "price": 1000},
    "rare":      {"name": "Редкий",      "price": 5000},
    "epic":      {"name": "Эпический",   "price": 25000},
    "legendary": {"name": "Легендарный", "price": 100000},
    "secret":    {"name": "Секретный",   "price": 1000000},
}

CASES = {
    "basic":    {"name": "Обычный кейс",     "cost": 5000,   "min_rarity": "uncommon"},
    "rare":     {"name": "Редкий кейс",      "cost": 25000,  "min_rarity": "rare"},
    "epic":     {"name": "Эпический кейс",   "cost": 100000, "min_rarity": "epic"},
    "legendary":{"name": "Легендарный кейс", "cost": 500000, "min_rarity": "legendary"},
}

# ============ БАЗА ДАННЫХ ============
def init_db():
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute("""
        CREATE TABLE IF NOT EXISTS players (
            user_id INTEGER PRIMARY KEY,
            username TEXT,
            first_name TEXT,
            balance INTEGER DEFAULT 10000,
            inventory TEXT DEFAULT '[]',
            achievements TEXT DEFAULT '[]',
            daily_tasks TEXT DEFAULT '{}',
            last_bonus TEXT DEFAULT '',
            total_spins INTEGER DEFAULT 0,
            total_earned INTEGER DEFAULT 0,
            banned INTEGER DEFAULT 0,
            clan_id INTEGER DEFAULT 0,
            joined TEXT
        )
    """)
    c.execute("""
        CREATE TABLE IF NOT EXISTS auction (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            seller_id INTEGER,
            number TEXT,
            rarity TEXT,
            price INTEGER,
            created TEXT
        )
    """)
    c.execute("""
        CREATE TABLE IF NOT EXISTS clans (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT UNIQUE,
            owner_id INTEGER,
            balance INTEGER DEFAULT 0,
            members TEXT DEFAULT '[]',
            created TEXT
        )
    """)
    conn.commit()
    conn.close()

def row_to_player(row):
    return {
        "user_id": row[0],
        "username": row[1],
        "first_name": row[2],
        "balance": row[3],
        "inventory": json.loads(row[4]),
        "achievements": json.loads(row[5]),
        "daily_tasks": json.loads(row[6]),
        "last_bonus": row[7],
        "total_spins": row[8],
        "total_earned": row[9],
        "banned": row[10],
        "clan_id": row[11],
        "joined": row[12]
    }

def get_player(user_id, username="", first_name=""):
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute("SELECT * FROM players WHERE user_id=?", (user_id,))
    row = c.fetchone()
    if not row:
        c.execute(
            "INSERT INTO players (user_id, username, first_name, joined) VALUES (?,?,?,?)",
            (user_id, username, first_name, datetime.now().isoformat())
        )
        conn.commit()
        c.execute("SELECT * FROM players WHERE user_id=?", (user_id,))
        row = c.fetchone()
    conn.close()
    return row_to_player(row)

def save_player(p):
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute("""
        UPDATE players SET balance=?, inventory=?, achievements=?, daily_tasks=?,
        last_bonus=?, total_spins=?, total_earned=?, banned=?, clan_id=?
        WHERE user_id=?
    """, (
        p["balance"], json.dumps(p["inventory"]), json.dumps(p["achievements"]),
        json.dumps(p["daily_tasks"]), p["last_bonus"], p["total_spins"],
        p["total_earned"], p["banned"], p["clan_id"], p["user_id"]
    ))
    conn.commit()
    conn.close()

# ============ БОТ: КОМАНДЫ ============
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    p = get_player(user.id, user.username or "", user.first_name)
    
    if p["banned"]:
        await update.message.reply_text("⛔ Вы забанены в игре.")
        return
    
    keyboard = [[
        InlineKeyboardButton("🎰 Играть в Номерной Магнат",
                             web_app=WebAppInfo(url=WEBAPP_URL))
    ]]
    await update.message.reply_text(
        f"Привет, {user.first_name}! 👋\n\n"
        "Добро пожаловать в **Номерной Магнат**!\n\n"
        "🎯 Крути номера, собирай редкие комбинации, продавай их и соревнуйся с другими!\n\n"
        "**Команды:**\n"
        "/play — открыть игру\n"
        "/top — топ-10 игроков\n"
        "/me — мой профиль\n"
        "/clan — кланы\n"
        "/help — помощь",
        reply_markup=InlineKeyboardMarkup(keyboard),
        parse_mode="Markdown"
    )

async def help_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = (
        "📖 **Помощь**\n\n"
        "**Основные команды:**\n"
        "/start — перезапустить бота\n"
        "/play — открыть игру\n"
        "/me — ваш профиль и баланс\n"
        "/top — рейтинг игроков\n"
        "/clan — кланы (создать/вступить)\n"
        "/daily — ежедневный бонус\n\n"
        "**Игра:**\n"
        "🎰 Крутите номера — 500 ₽ за крутку\n"
        "📦 Продавайте номера из инвентаря\n"
        "🏛 Выставляйте на аукцион\n"
        "🌍 Открывайте новые регионы\n"
        "🎁 Забирайте бонус раз в 24 часа\n"
    )
    await update.message.reply_text(text, parse_mode="Markdown")

async def play(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [[
        InlineKeyboardButton("🎰 Открыть игру",
                             web_app=WebAppInfo(url=WEBAPP_URL))
    ]]
    await update.message.reply_text(
        "Нажми кнопку ниже, чтобы играть:",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )

async def me(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    p = get_player(user.id)
    text = (
        f"👤 **{p['first_name']}**\n\n"
        f"💰 Баланс: {p['balance']:,} ₽\n"
        f"📦 Номеров: {len(p['inventory'])}\n"
        f"🎰 Круток: {p['total_spins']}\n"
        f"💎 Заработано: {p['total_earned']:,} ₽\n"
        f"🏆 Достижений: {len(p['achievements'])}\n"
    )
    await update.message.reply_text(text, parse_mode="Markdown")

async def top(update: Update, context: ContextTypes.DEFAULT_TYPE):
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute("SELECT first_name, balance, total_spins FROM players WHERE banned=0 ORDER BY balance DESC LIMIT 10")
    rows = c.fetchall()
    conn.close()
    
    text = "🏆 **ТОП-10 ИГРОКОВ**\n\n"
    medals = ["🥇", "🥈", "🥉"] + ["▪️"] * 7
    for i, (name, bal, spins) in enumerate(rows):
        text += f"{medals[i]} {name} — {bal:,} ₽ ({spins} круток)\n"
    await update.message.reply_text(text, parse_mode="Markdown")

async def daily(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    p = get_player(user.id)
    now = datetime.now()
    
    if p["last_bonus"]:
        last = datetime.fromisoformat(p["last_bonus"])
        if now - last < timedelta(hours=24):
            wait = timedelta(hours=24) - (now - last)
            h, m = int(wait.total_seconds() // 3600), int((wait.total_seconds() % 3600) // 60)
            await update.message.reply_text(f"⏰ Следующий бонус через {h}ч {m}м")
            return
    
    bonus = 5000
    p["balance"]+= bonus
    p["last_bonus"] = now.isoformat()
    save_player(p)
    await update.message.reply_text(f"🎁 Ежедневный бонус: +{bonus:,} ₽\n💰 Ваш баланс: {p['balance']:,} ₽")

async def clan_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    p = get_player(user.id)
    
    args = context.args
    if not args:
        if p["clan_id"]:
            conn = sqlite3.connect(DB_FILE)
            c = conn.cursor()
            c.execute("SELECT name, balance, members FROM clans WHERE id=?", (p["clan_id"],))
            row = c.fetchone()
            conn.close()
            if row:
                members = json.loads(row[2])
                await update.message.reply_text(
                    f"🛡 Клан: **{row[0]}**\n"
                    f"💰 Казна: {row[1]:,} ₽\n"
                    f"👥 Участников: {len(members)}",
                    parse_mode="Markdown"
                )
                return
        await update.message.reply_text(
            "Вы не в клане.\n\n"
            "**Команды:**\n"
            "/clan create Название — создать клан (10 000 ₽)\n"
            "/clan join ID — вступить\n"
            "/clan list — список кланов\n"
            "/clan leave — выйти",
            parse_mode="Markdown"
        )
        return
    
    action = args[0].lower()
    
    if action == "create" and len(args) >= 2:
        name = " ".join(args[1:])[:30]
        if p["clan_id"]:
            await update.message.reply_text("Вы уже в клане.")
            return
        if p["balance"] < 10000:
            await update.message.reply_text("Нужно 10 000 ₽ для создания клана.")
            return
        try:
            conn = sqlite3.connect(DB_FILE)
            c = conn.cursor()
            c.execute(
                "INSERT INTO clans (name, owner_id, balance, members, created) VALUES (?,?,?,?,?)",
                (name, user.id, 0, json.dumps([user.id]), datetime.now().isoformat())
            )
            clan_id = c.lastrowid
            conn.commit()
            conn.close()
            p["balance"] -= 10000
            p["clan_id"] = clan_id
            save_player(p)
            await update.message.reply_text(f"✅ Клан **{name}** создан!", parse_mode="Markdown")
        except sqlite3.IntegrityError:
            await update.message.reply_text("Клан с таким названием уже существует.")
    
    elif action == "join" and len(args) >= 2:
        if p["clan_id"]:
            await update.message.reply_text("Вы уже в клане.")
            return
        try:
            clan_id = int(args[1])
            conn = sqlite3.connect(DB_FILE)
            c = conn.cursor()
            c.execute("SELECT members FROM clans WHERE id=?", (clan_id,))
            row = c.fetchone()
            if not row:
                conn.close()
                await update.message.reply_text("Клан не найден.")
                return
            members = json.loads(row[0])
            members.append(user.id)
            c.execute("UPDATE clans SET members=? WHERE id=?", (json.dumps(members), clan_id))
            conn.commit()
            conn.close()
            p["clan_id"] = clan_id
            save_player(p)
            await update.message.reply_text("✅ Вы вступили в клан!")
        except ValueError:
            await update.message.reply_text("Укажите ID клана числом.")
    
    elif action == "list":
        conn = sqlite3.connect(DB_FILE)
        c = conn.cursor()
        c.execute("SELECT id, name, balance, members FROM clans ORDER BY balance DESC LIMIT 20")
        rows = c.fetchall()
        conn.close()
        if not rows:
            await update.message.reply_text("Кланов пока нет.")
            return
        text = "🛡 **Кланы:**\n\n"
        for cid, name, bal, members in rows:
            text += f"`{cid}`. {name} — {len(json.loads(members))} чел. — {bal:,} ₽\n"
        await update.message.reply_text(text, parse_mode="Markdown")
    
    elif action == "leave":
        if not p["clan_id"]:
await update.message.reply_text("Вы не в клане.")
            return
        conn = sqlite3.connect(DB_FILE)
        c = conn.cursor()
        c.execute("SELECT members, owner_id FROM clans WHERE id=?", (p["clan_id"],))
        row = c.fetchone()
        if row:
            members = json.loads(row[0])
            if user.id in members:
                members.remove(user.id)
            if row[1] == user.id or not members:
                c.execute("DELETE FROM clans WHERE id=?", (p["clan_id"],))
            else:
                c.execute("UPDATE clans SET members=? WHERE id=?", (json.dumps(members), p["clan_id"]))
            conn.commit()
        conn.close()
        p["clan_id"] = 0
        save_player(p)
        await update.message.reply_text("Вы вышли из клана.")

# ============ АДМИН-КОМАНДЫ ============
def is_admin(user_id):
    return user_id == ADMIN_ID

async def admin(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_admin(update.effective_user.id):
        await update.message.reply_text("⛔ Нет доступа.")
        return
    
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute("SELECT COUNT(*) FROM players")
    total_players = c.fetchone()[0]
    c.execute("SELECT COUNT(*) FROM players WHERE banned=1")
    banned = c.fetchone()[0]
    c.execute("SELECT SUM(balance) FROM players")
    total_money = c.fetchone()[0] or 0
    c.execute("SELECT SUM(total_spins) FROM players")
    total_spins = c.fetchone()[0] or 0
    c.execute("SELECT COUNT(*) FROM clans")
    total_clans = c.fetchone()[0]
    c.execute("SELECT COUNT(*) FROM auction")
    total_auctions = c.fetchone()[0]
    conn.close()
    
    text = (
        "📊 **АДМИН-ПАНЕЛЬ**\n\n"
        f"👥 Игроков: {total_players}\n"
        f"🚫 Забанено: {banned}\n"
        f"💰 Денег в игре: {total_money:,} ₽\n"
        f"🎰 Всего круток: {total_spins}\n"
        f"🛡 Кланов: {total_clans}\n"
        f"🏛 Лотов на аукционе: {total_auctions}\n\n"
        "**Команды:**\n"
        "/give ID СУММА — выдать деньги\n"
        "/setbal ID СУММА — установить баланс\n"
        "/ban ID — забанить\n"
        "/unban ID — разбанить\n"
        "/broadcast ТЕКСТ — рассылка всем\n"
        "/top — топ игроков\n"
    )
    await update.message.reply_text(text, parse_mode="Markdown")

async def give(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_admin(update.effective_user.id):
        return
    args = context.args
    if len(args) < 2:
        await update.message.reply_text("Использование: /give ID СУММА")
        return
    try:
        uid, amount = int(args[0]), int(args[1])
        p = get_player(uid)
        p["balance"] += amount
        p["total_earned"] += amount
        save_player(p)
        await update.message.reply_text(f"✅ Игроку {uid} выдано {amount:,} ₽. Новый баланс: {p['balance']:,} ₽")
        try:
            await context.bot.send_message(uid, f"🎁 Вам выдали {amount:,} ₽!")
        except: pass
    except Exception as e:
        await update.message.reply_text(f"Ошибка: {e}")

async def setbal(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_admin(update.effective_user.id):
        return
    args = context.args
    if len(args) < 2:
        await update.message.reply_text("Использование: /setbal ID СУММА")
        return
    try:
        uid, amount = int(args[0]), int(args[1])
        p = get_player(uid)
        p["balance"] = amount
        save_player(p)
        await update.message.reply_text(f"✅ Баланс игрока {uid} = {amount:,} ₽")
    except Exception as e:
        await update.message.reply_text(f"Ошибка: {e}")

async def ban(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_admin(update.effective_user.id):
        return
    if not context.args:
        await update.message.reply_text("Использование: /ban ID")
        return
    uid = int(context.args[0])
    p = get_player(uid)
    p["banned"] = 1
    save_player(p)
    await update.message.reply_text(f"🚫 Игрок {uid} забанен.")

async def unban(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_admin(update.effective_user.id):
        return
    if not context.args:
        await update.message.reply_text("Использование: /unban ID")
        return
    uid = int(context.args[0])
    p = get_player(uid)
    p["banned"] = 0
    save_player(p)
    await update.message.reply_text(f"✅ Игрок {uid} разбанен.")

async def broadcast(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_admin(update.effective_user.id):
        return
    if not context.args:
        await update.message.reply_text("Использование: /broadcast ТЕКСТ")
        return
    text = " ".join(context.args)
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute("SELECT user_id FROM players WHERE banned=0")
    ids = [r[0] for r in c.fetchall()]
    conn.close()
    
    sent, failed = 0, 0
    for uid in ids:
        try:
            await context.bot.send_message(uid, f"📢 **Объявление:**\n\n{text}", parse_mode="Markdown")
            sent += 1
        except:
            failed += 1
    await update.message.reply_text(f"✅ Отправлено: {sent}\n❌ Ошибок: {failed}")

# ============ FLASK API ============
app = Flask(__name__)

@app.route("/health")
def health():
    return "OK"

@app.route("/api/player", methods=["POST"])
def api_player():
    data = request.json
    uid = data.get("user_id")
    if not uid: return jsonify({"error": "no user_id"}), 400
    p = get_player(uid, data.get("username", ""), data.get("first_name", ""))
    return jsonify(p)

@app.route("/api/save", methods=["POST"])
def api_save():
    data = request.json
    uid = data.get("user_id")
    if not uid: return jsonify({"error": "no user_id"}), 400
    p = get_player(uid)
    if p["banned"]:
        return jsonify({"error": "banned"}), 403
    p["balance"] = data.get("balance", p["balance"])
    p["inventory"] = data.get("inventory", p["inventory"])
    p["total_spins"] = data.get("total_spins", p["total_spins"])
    p["achievements"] = data.get("achievements", p["achievements"])
    p["daily_tasks"] = data.get("daily_tasks", p["daily_tasks"])
    p["total_earned"] = data.get("total_earned", p["total_earned"])
    save_player(p)
    return jsonify({"ok": True})

@app.route("/api/bonus", methods=["POST"])
def api_bonus():
    data = request.json
    uid = data.get("user_id")
    p = get_player(uid)
    last = p["last_bonus"]
    now = datetime.now()
    if last:
        last_dt = datetime.fromisoformat(last)
        if now - last_dt < timedelta(hours=24):
            wait = timedelta(hours=24) - (now - last_dt)
            h = int(wait.total_seconds() // 3600)
            m = int((wait.total_seconds() % 3600) // 60)
            return jsonify({"ok": False, "message": f"Через {h}ч {m}м"})
    bonus = 5000
    p["balance"] += bonus
    p["last_bonus"] = now.isoformat()
    save_player(p)
    return jsonify({"ok": True, "bonus": bonus, "balance": p["balance"]})

@app.route("/api/case", methods=["POST"])
def api_case():
    data = request.json
    uid = data["user_id"]
    case_key = data["case"]
    if case_key not in CASES:
        return jsonify({"ok": False, "message": "Неизвестный кейс"})
    case = CASES[case_key]
    p = get_player(uid)
    if p["balance"] < case["cost"]:
        return jsonify({"ok": False, "message": "Недостаточно денег"})
    
    # Определяем редкость от min_rarity и выше
    order = ["common", "uncommon", "rare", "epic", "legendary", "secret"]
    min_idx = order.index(case["min_rarity"])
    weights = {
        "uncommon":  [0, 60, 25, 10, 4, 1],
        "rare":      [0, 0, 65, 25, 8, 2],
        "epic":      [0, 0, 0, 70, 25, 5],
        "legendary":[0, 0, 0, 0, 88, 12],
    }[case["min_rarity"]]
    rarity = random.choices(order, weights=weights)[0]
    
    letters = "АВЕКМНОРСТУХ"
    l1 = random.choice(letters)
    l2 = random.choice(letters)
    digits = "".join(str(random.randint(0, 9)) for _ in range(3))

region = random.choice(["77", "78", "16", "66", "25"])
    number = f"{l1} {digits} {l2}{l2} {region}"
    price = RARITIES[rarity]["price"] * random.randint(9, 15) // 10
    
    p["balance"] -= case["cost"]
    save_player(p)
    return jsonify({"ok": True, "number": number, "rarity": rarity, "price": price, "balance": p["balance"]})

@app.route("/api/auction/list")
def api_auction_list():
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute("SELECT id, seller_id, number, rarity, price FROM auction ORDER BY id DESC LIMIT 50")
    rows = c.fetchall()
    conn.close()
    return jsonify([{"id": r[0], "seller_id": r[1], "number": r[2], "rarity": r[3], "price": r[4]} for r in rows])

@app.route("/api/auction/add", methods=["POST"])
def api_auction_add():
    data = request.json
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute("INSERT INTO auction (seller_id, number, rarity, price, created) VALUES (?,?,?,?,?)",
              (data["user_id"], data["number"], data["rarity"], data["price"], datetime.now().isoformat()))
    conn.commit()
    conn.close()
    return jsonify({"ok": True})

@app.route("/api/auction/buy", methods=["POST"])
def api_auction_buy():
    data = request.json
    buyer_id, auc_id = data["user_id"], data["auction_id"]
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute("SELECT seller_id, number, rarity, price FROM auction WHERE id=?", (auc_id,))
    row = c.fetchone()
    if not row:
        conn.close()
        return jsonify({"ok": False, "message": "Лот не найден"})
    seller_id, number, rarity, price = row
    if seller_id == buyer_id:
        conn.close()
        return jsonify({"ok": False, "message": "Нельзя купить свой лот"})
    buyer = get_player(buyer_id)
    if buyer["balance"] < price:
        conn.close()
        return jsonify({"ok": False, "message": "Недостаточно денег"})
    seller = get_player(seller_id)
    buyer["balance"] -= price
    seller["balance"] += price
    seller["total_earned"] += price
    buyer["inventory"].append({"number": number, "rarity": rarity, "price": price})
    save_player(buyer)
    save_player(seller)
    c.execute("DELETE FROM auction WHERE id=?", (auc_id,))
    conn.commit()
    conn.close()
    return jsonify({"ok": True})

@app.route("/api/top")
def api_top():
    conn = sqlite3.connect(DB_FILE)
    c = conn.cursor()
    c.execute("SELECT first_name, balance, total_spins FROM players WHERE banned=0 ORDER BY balance DESC LIMIT 20")
    rows = c.fetchall()
    conn.close()
    return jsonify([{"name": r[0], "balance": r[1], "spins": r[2]} for r in rows])

# ============ ЗАПУСК ============
def main():
    init_db()
    application = Application.builder().token(BOT_TOKEN).build()
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("help", help_cmd))
    application.add_handler(CommandHandler("play", play))
    application.add_handler(CommandHandler("me", me))
    application.add_handler(CommandHandler("top", top))
    application.add_handler(CommandHandler("daily", daily))
    application.add_handler(CommandHandler("clan", clan_cmd))
    application.add_handler(CommandHandler("admin", admin))
    application.add_handler(CommandHandler("give", give))
    application.add_handler(CommandHandler("setbal", setbal))
    application.add_handler(CommandHandler("ban", ban))
    application.add_handler(CommandHandler("unban", unban))
    application.add_handler(CommandHandler("broadcast", broadcast))
    
    import threading
    def run_flask():
        app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
    threading.Thread(target=run_flask, daemon=True).start()
    
    print("✅ Бот запущен!")
    application.run_polling()

if __name__ == "__main__":
    main()
