import requests
import time
import logging
import schedule
from telegram import Bot

# === ТВОИ НАСТРОЙКИ ===
TELEGRAM_TOKEN = "8216749727:AAHF0yWH4uGiBGvj697eja-FiBrZTs-PtmA"
TELEGRAM_CHAT_ID = 7422789062  # без кавычек — это число
NETWORK = "solana"

bot = Bot(token=TELEGRAM_TOKEN)
logging.basicConfig(level=logging.INFO)

# === ВСПОМОГАТЕЛЬНЫЕ ФУНКЦИИ ===

def is_meme_name(name: str) -> bool:
    name = name.lower()
    keywords = [
        'dog', 'cat', 'pup', 'bear', 'bull', 'troll', 'wojak', 'pepe', 'ai', 
        'trump', 'starbucks', 'santa', 'rizz', 'meme', 'coin', 'moon', 'woof',
        'dick', 'neet', 'pfp', 'christmas', 'flowey', 'elf', 'horse', 'anon',
        'bubble', 'private', 'zero', 'kitkat', 'clippy', 'groo', 'vervle'
    ]
    return any(kw in name for kw in keywords)

def check_lp_lock(lp_token_address: str) -> bool:
    """Проверяет, заблокирован ли LP через Solscan API"""
    try:
        url = f"https://public-api.solscan.io/token/holders?tokenAddress={lp_token_address}&limit=5"
        res = requests.get(url, timeout=10)
        if res.status_code != 200:
            return False
        holders = res.json().get("data", [])
        if not holders:
            return False
        top_holder = holders[0]["owner"].lower()
        lock_indicators = [
            "dead", "lock", "teamfi", "unicrypt",
            "hxro", "pump", "raydium", "orca"
        ]
        return any(ind in top_holder for ind in lock_indicators) or top_holder == "5qpwz9v3qjzu8z7g3x2e5f5l5j5v5u5w5t5r5e5c5o5m5"
    except Exception as e:
        logging.warning(f"Ошибка проверки LP lock: {e}")
        return False

def fetch_top_pairs():
    url = f"https://api.geckoterminal.com/api/v2/networks/{NETWORK}/pools?page=1"
    try:
        res = requests.get(url, timeout=15)
        if res.status_code != 200:
            logging.error(f"GeckoTerminal error: {res.status_code}")
            return []
        data = res.json()
        candidates = []
        for pool in data.get("data", []):
            attrs = pool["attributes"]
            age_sec = time.time() - attrs.get("created_at", 0)
            if not (30 * 60 <= age_sec <= 6 * 3600):  # 30 мин – 6 часов
                continue

            name = attrs.get("name", "Unknown")
            if "usd" in name.lower() or "usdt" in name.lower():
                continue

            mc = attrs.get("market_cap_usd", 0)
            vol = attrs.get("volume_usd", {}).get("h24", 0)
            liquidity = attrs.get("reserve_in_usd", 0)
            txns_1h = attrs.get("transactions", {}).get("h1", {})
            buys = txns_1h.get("buys", 0)
            sells = txns_1h.get("sells", 0)
            price = attrs.get("base_token_price_usd", 0)

            if mc == 0 or vol == 0 or liquidity == 0:
                continue
            vol_mc = vol / mc
            if not (0.5 <= vol_mc <= 3.0):
                continue
            if liquidity < 40_000:
                continue
            if buys + sells < 2000:
                continue
            if buys <= sells:
                continue
            if price > 0.1:
                continue
            if not is_meme_name(name):
                continue

            lp_token = pool["relationships"]["base_token"]["data"]["id"]
            lp_locked = check_lp_lock(lp_token)

            score = vol_mc * 1.5 + (buys / (sells + 1)) * 0.5 + (1 if lp_locked else 0)
            candidates.append({
                "name": name,
                "mc": mc,
                "liquidity": liquidity,
                "vol_mc": vol_mc,
                "buys": buys,
                "sells": sells,
                "lp_locked": lp_locked,
                "score": score,
                "link": f"https://dexscreener.com/{NETWORK}/{pool['id']}"
            })
        candidates.sort(key=lambda x: x["score"], reverse=True)
        return candidates[:10]
    except Exception as e:
        logging.error(f"Ошибка при сборе данных: {e}")
        return []

def send_report():
    try:
        top10 = fetch_top_pairs()
        if not top10:
            bot.send_message(chat_id=TELEGRAM_CHAT_ID, text="🔍 Нет подходящих кандидатов за последние 2 часа.")
            return

        top3 = top10[:3]
        top1 = top3[0]

        message = "🧠 ИИ-АНАЛИТИК МЕМКОИНОВ\n"
        message += "⏱ Обновление каждые 2 часа\n\n"

        message += "🥇 ТОП-1:\n"
        message += f"🪙 {top1['name']}\n"
        message += f"💧 LP: ${top1['liquidity']:,.0f} {'✅' if top1['lp_locked'] else '⚠️'}\n"
        message += f"📊 Vol/MC: {top1['vol_mc']:.2f}\n"
        message += f"📈 Txns: {top1['buys']}/{top1['sells']}\n"
        message += f"🔗 {top1['link']}\n\n"

        for i, coin in enumerate(top3[1:], start=2):
            medal = "🥈" if i == 2 else "🥉"
            message += f"{medal} {coin['name']}\n"
            message += f"  💧 ${coin['liquidity']:,.0f} {'✅' if coin['lp_locked'] else '⚠️'} | Vol/MC: {coin['vol_mc']:.2f}\n"

        bot.send_message(chat_id=TELEGRAM_CHAT_ID, text=message)
        logging.info("Отчёт успешно отправлен!")
    except Exception as e:
        logging.error(f"Ошибка при отправке отчёта: {e}")

# === ЗАПУСК ===
if __name__ == "__main__":
    logging.info("✅ Бот запущен. Первый отчёт через 2 часа.")
    send_report()
    schedule.every(2).hours.do(send_report)

    while True:
        schedule.run_pending()
        time.sleep(60)
