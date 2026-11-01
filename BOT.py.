"""Bot Telegram privato per offerte con sconto alto.
Legge i feed RSS in config.json, filtra, ti scrive su Telegram e poi
tiene aggiornato lo stato (ATTIVA / SCADUTA) dello stesso messaggio."""
import os, re, json, html, sys
import xml.etree.ElementTree as ET
from datetime import datetime, timedelta
from zoneinfo import ZoneInfo
import requests

BASE = os.path.dirname(os.path.abspath(__file__))
CFG = json.load(open(os.path.join(BASE, "config.json"), encoding="utf-8"))
STATE_PATH = os.path.join(BASE, "state.json")
TZ = ZoneInfo("Europe/Rome")
UA = {"User-Agent": "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 Chrome/124 Safari/537.36"}


def now():
    return datetime.now(TZ)


def load_state():
    try:
        return json.load(open(STATE_PATH, encoding="utf-8"))
    except Exception:
        return {"seen": [], "deals": {}, "started": False}


def save_state(st):
    st["seen"] = st["seen"][-3000:]
    json.dump(st, open(STATE_PATH, "w", encoding="utf-8"), ensure_ascii=False, indent=1)


def tg(method, **data):
    token = os.environ["TELEGRAM_TOKEN"]
    r = requests.post(f"https://api.telegram.org/bot{token}/{method}", data=data, timeout=20)
    j = r.json()
    if not j.get("ok"):
        raise RuntimeError(f"Telegram {method}: {j.get('description')}")
    return j["result"]


def clean(t):
    return html.unescape(re.sub(r"<[^>]+>", " ", t or "")).strip()


def read_feed(url):
    r = requests.get(url, headers=UA, timeout=25)
    r.raise_for_status()
    root = ET.fromstring(r.content)
    out = []
    for it in root.iter("item"):
        g = lambda tag: (it.findtext(tag) or "").strip()
        link = g("link")
        out.append({
            "id": g("guid") or link,
            "title": clean(g("title")),
            "link": link,
            "desc": clean(g("description")),
        })
    return out


def to_num(s):
    s = s.strip()
    s = s.replace(".", "").replace(",", ".") if "," in s else s
    try:
        return float(s)
    except ValueError:
        return None


def find_discount(text):
    """Ritorna (percentuale, stimata?) oppure (None, False)."""
    pcts = [int(x) for x in re.findall(r"(\d{2,3})\s?%", text)]
    pcts = [p for p in pcts if 10 <= p <= 99]
    if pcts:
        return max(pcts), False
    if re.search(r"invece|listino|prima|anziché|anziche|era\b", text, re.I):
        N = r"(\d{1,3}(?:\.\d{3})+(?:,\d{1,2})?|\d+(?:[.,]\d{1,2})?)"
        nums = [a or b for a, b in re.findall(rf"{N}\s?(?:€|euro)|€\s?{N}", text)]
        vals = [v for v in (to_num(n) for n in nums) if v]
        if len(vals) >= 2 and max(vals) > 0:
            return round((1 - min(vals) / max(vals)) * 100), True
    return None, False


def find_coupon(text):
    m = re.search(r"(?:codice|coupon|code)\s*:?\s*([A-Z0-9]{4,20})\b", text)
    return m.group(1) if m else None


def accept(item):
    text = f'{item["title"]} {item["desc"]}'
    low = text.lower()
    if any(w.lower() in low for w in CFG.get("exclude_words", [])):
        return None
    inc = [w.lower() for w in CFG.get("include_words", [])]
    if inc and not any(w in low for w in inc):
        return None
    pct, est = find_discount(text)
    coupon = find_coupon(text)
    if pct is not None and pct >= CFG["min_discount"]:
        return pct, est, coupon
    if coupon and pct is not None and pct >= CFG.get("coupon_min_discount", 30):
        return pct, est, coupon
    return None


def render(d, status, extra=""):
    icon = {"active": "🟢 ATTIVA", "expired": "🔴 SCADUTA", "unknown": "⚪️ NON PIÙ CONTROLLATA"}[status]
    lines = [f"<b>{icon}</b>", html.escape(d["title"])]
    sconto = f'-{d["pct"]}%' + (" (stimato)" if d["est"] else "")
    lines.append(f"💸 Sconto: {sconto}")
    if d.get("coupon"):
        lines.append(f'🏷 Codice: <code>{html.escape(d["coupon"])}</code>')
    lines.append(d["link"])
    stamp = now().strftime("%d/%m %H:%M")
    lines.append(f"🕐 Ultimo controllo: {stamp}" + (f" {extra}" if extra else ""))
    return "\n".join(lines)


def is_expired(link):
    """True/False, oppure None se non riesce a controllare."""
    try:
        r = requests.get(link, headers=UA, timeout=25)
    except requests.RequestException:
        return None
    if r.status_code in (404, 410):
        return True
    if r.status_code != 200:
        return None
    return any(re.search(p, r.text, re.I) for p in CFG["expired_patterns"])


def edit(d, text):
    try:
        tg("editMessageText", chat_id=os.environ["TELEGRAM_CHAT_ID"], message_id=d["msg_id"],
           text=text, parse_mode="HTML", disable_web_page_preview="true")
    except RuntimeError as e:
        if "not modified" not in str(e):
            print("edit fallita:", e)


def update_open_deals(st):
    max_age = timedelta(hours=CFG.get("max_age_hours", 48))
    for key, d in list(st["deals"].items()):
        if d["status"] != "active":
            continue
        age = now() - datetime.fromisoformat(d["sent_at"])
        if age > max_age:
            d["status"] = "unknown"
            edit(d, render(d, "unknown"))
            continue
        res = is_expired(d["link"])
        if res is True:
            d["status"] = "expired"
            edit(d, render(d, "expired"))
        elif res is False:
            edit(d, render(d, "active"))
        else:
            edit(d, render(d, "active", "(controllo non riuscito)"))


def main():
    chat = os.environ["TELEGRAM_CHAT_ID"]
    st = load_state()
    if not st.get("started"):
        tg("sendMessage", chat_id=chat, text="✅ Bot offerte attivo. Da ora ti scrivo solo le offerte nuove.")
        first_run = True
        st["started"] = True
    else:
        first_run = False

    update_open_deals(st)

    seen = set(st["seen"])
    failures = 0
    for url in CFG["feeds"]:
        try:
            items = read_feed(url)
        except Exception as e:
            print("feed non leggibile:", url, e)
            failures += 1
            continue
        for it in items:
            if not it["id"] or it["id"] in seen:
                continue
            seen.add(it["id"])
            st["seen"].append(it["id"])
            if first_run:
                continue
            res = accept(it)
            if not res:
                continue
            pct, est, coupon = res
            d = {"title": it["title"], "link": it["link"], "pct": pct, "est": est,
                 "coupon": coupon, "status": "active", "sent_at": now().isoformat()}
            msg = tg("sendMessage", chat_id=chat, text=render(d, "active"),
                     parse_mode="HTML", disable_web_page_preview="true")
            d["msg_id"] = msg["message_id"]
            st["deals"][it["id"]] = d

    # tieni nello stato solo le offerte recenti
    if len(st["deals"]) > 200:
        for k in list(st["deals"])[:-200]:
            del st["deals"][k]
    save_state(st)
    if failures == len(CFG["feeds"]):
        sys.exit("Nessun feed leggibile: controlla gli indirizzi in config.json")


if __name__ == "__main__":
    main()
