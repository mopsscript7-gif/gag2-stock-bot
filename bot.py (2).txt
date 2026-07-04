import json
import os
import time
from datetime import datetime
import requests

# ---------- НАСТРОЙКИ ----------
BOT_TOKEN = os.getenv("BOT_TOKEN", "8821616249:AAHf7B5d9WpON5859bVAmm9wtLB_-0p41U8")
CHECK_INTERVAL = 30  # секунд между проверками стока
CATEGORIES = ["seeds", "gear", "eggs"]  # какие категории мониторим
BASE_STOCK_URL = "https://api.growagarden2stock.com/stock"
SUBS_FILE = "subscriptions.json"  # {chat_id: [item_name, ...]}
STATE_FILE = "last_stock.json"  # {category: {item_name: {...}}}
TELEGRAM_API = "https://api.telegram.org/bot{token}/{method}"

# --------------------------------
# ---------- Хранилище подписок ----------
def load_json(path: str, default):
    if os.path.exists(path):
        with open(path, "r", encoding="utf-8") as f:
            return json.load(f)
    return default

def save_json(path: str, data):
    with open(path, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

def load_subs() -> dict:
    return load_json(SUBS_FILE, {})

def save_subs(subs: dict):
    save_json(SUBS_FILE, subs)

# ---------- Telegram API ----------
def tg_call(method: str, **params):
    url = TELEGRAM_API.format(token=BOT_TOKEN, method=method)
    try:
        r = requests.post(url, json=params, timeout=15)
        r.raise_for_status()
        return r.json()
    except requests.RequestException as e:
        print(f"[{datetime.now()}] Ошибка Telegram API ({method}): {e}")
        return None

def send_message(chat_id, text: str):
    tg_call(
        "sendMessage",
        chat_id=chat_id,
        text=text,
        parse_mode="HTML",
        disable_web_page_preview=True,
    )

# ---------- Сток ----------
def fetch_category_stock(category: str) -> dict:
    resp = requests.get(
        BASE_STOCK_URL,
        params={"category": category},
        timeout=10,
        headers={"User-Agent": "Mozilla/5.0", "Accept": "application/json"},
    )
    resp.raise_for_status()
    data = resp.json()
    stock = {}
    for item in data.get("items", []):
        name = item.get("item_name")
        if not name:
            continue
        stock[name] = {
            "quantity": item.get("quantity", 0),
            "in_stock": bool(item.get("in_stock")),
            "price": item.get("price"),
            "rarity": item.get("rarity"),
        }
    return stock

def diff_stock(old: dict, new: dict) -> list:
    """Предметы, только что перешедшие in_stock: False -> True."""
    changed = []
    for name, info in new.items():
        was_in_stock = old.get(name, {}).get("in_stock", False)
        if info["in_stock"] and not was_in_stock:
            changed.append((name, info))
    return changed

def get_all_known_items(all_stock: dict) -> list:
    names = set()
    for cat_stock in all_stock.values():
        names.update(cat_stock.keys())
    return sorted(names)

# ---------- Обработка команд от пользователей ----------
def handle_update(update: dict, subs: dict, last_all_items: list):
    message = update.get("message")
    if not message:
        return
    chat_id = str(message["chat"]["id"])
    text = (message.get("text") or "").strip()
    if not text.startswith("/"):
        return
    parts = text.split(maxsplit=1)
    command = parts[0].lower()
    arg = parts[1].strip() if len(parts) > 1 else ""
    user_subs = set(subs.get(chat_id, []))

    if command == "/start":
        send_message(
            chat_id,
            "🌱 <b>GAG2 Stock Bot</b>\n\n"
            "Я пришлю уведомление, когда в магазине Grow a Garden 2 "
            "появится нужное тебе семя.\n\n"
            "Команды:\n"
            "/list — список всех известных семян/предметов\n"
            "/sub Название — подписаться на предмет\n"
            "/unsub Название — отписаться\n"
            "/mysubs — мои подписки\n"
            "/suball — подписаться на всё\n"
            "/clear — очистить все подписки",
        )
    elif command == "/list":
        if not last_all_items:
            send_message(chat_id, "Список пока не загружен, попробуй через минуту.")
        else:
            send_message(chat_id, "📋 Доступные предметы:\n" + ", ".join(last_all_items))
    elif command == "/sub":
        if not arg:
            send_message(chat_id, "Укажи название, например: /sub Carrot")
        elif last_all_items and arg not in last_all_items:
            send_message(chat_id, f"Не нашёл «{arg}». Проверь /list на точное написание.")
        else:
            user_subs.add(arg)
            subs[chat_id] = sorted(user_subs)
            save_subs(subs)
            send_message(chat_id, f"✅ Подписал на «{arg}»")
    elif command == "/unsub":
        if arg in user_subs:
            user_subs.discard(arg)
            subs[chat_id] = sorted(user_subs)
            save_subs(subs)
            send_message(chat_id, f"❌ Отписал от «{arg}»")
        else:
            send_message(chat_id, f"Ты и так не подписан на «{arg}»")
    elif command == "/mysubs":
        if user_subs:
            send_message(chat_id, "🔔 Твои подписки:\n" + ", ".join(sorted(user_subs)))
        else:
            send_message(chat_id, "У тебя пока нет подписок. Используй /sub Название")
    elif command == "/suball":
        subs[chat_id] = list(last_all_items)
        save_subs(subs)
        send_message(chat_id, "✅ Подписал на все известные предметы")
    elif command == "/clear":
        subs[chat_id] = []
        save_subs(subs)
        send_message(chat_id, "🗑 Все подписки очищены")
    else:
        send_message(chat_id, "Не знаю такую команду. Смотри /start")

def poll_updates(offset: int, subs: dict, last_all_items: list) -> int:
    result = tg_call("getUpdates", offset=offset, timeout=1)
    if not result or not result.get("ok"):
        return offset
    for update in result.get("result", []):
        handle_update(update, subs, last_all_items)
    offset = update["update_id"] + 1
    return offset

# ---------- Основной цикл ----------
def notify_subscribers(subs: dict, category: str, changes: list):
    if not changes:
        return
    for chat_id, items in subs.items():
        items_set = set(items)
        relevant = [(n, i) for n, i in changes if n in items_set]
        if not relevant:
            continue
        lines = [f"🌱 <b>Новый сток ({category})!</b>"]
        for name, info in relevant:
            rarity = (info.get("rarity") or "").capitalize()
            lines.append(
                f"• <b>{name}</b> [{rarity}] — x{info['quantity']}, цена: {info['price']}"
            )
        send_message(chat_id, "\n".join(lines))

def main():
    print("GAG2 stock bot с подписками запущен...")
    last_stock_by_category = load_json(STATE_FILE, {})
    subs = load_subs()
    update_offset = 0
    last_all_items = get_all_known_items(last_stock_by_category)
    last_check = 0

    while True:
        # Обрабатываем команды пользователей почти непрерывно
        update_offset = poll_updates(update_offset, subs, last_all_items)
        # Проверяем сток раз в CHECK_INTERVAL секунд
        if time.time() - last_check >= CHECK_INTERVAL:
            last_check = time.time()
            for category in CATEGORIES:
                try:
                    current = fetch_category_stock(category)
                    old = last_stock_by_category.get(category, {})
                    changes = diff_stock(old, current)
                    if changes:
                        notify_subscribers(subs, category, changes)
                        print(f"[{datetime.now()}] {category}: {changes}")
                    last_stock_by_category[category] = current
                except Exception as e:
                    print(f"[{datetime.now()}] Ошибка ({category}): {e}")
            save_json(STATE_FILE, last_stock_by_category)
            last_all_items = get_all_known_items(last_stock_by_category)
        time.sleep(1)

if __name__ == "__main__":
    main()
