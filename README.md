# -*- coding: utf-8 -*-
"""
=====================================================================
  KIMYO-AU-24 REYTING BOTI  —  BITTA FAYLLIK, WEBHOOKSIZ (POLLING) VERSIYA
=====================================================================
"""

import os
import re
import io
import csv
import time
import html
import json
import logging
import random
import threading
from datetime import datetime
from difflib import SequenceMatcher

import requests

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s: %(message)s")
log = logging.getLogger("reyting-bot")


# =====================================================================
#  SOZLAMALAR — SHU YERNI O'ZINGIZNIKIGA ALMASHTIRING
# =====================================================================

BOT_TOKEN = "8682692827:AAH6kIoAsGyoCCJY10uJu1M9kZ9H44bu4kU"
GROUP_ID = -1004316949734
ADMIN_IDS = [7173988985 , 7473207424]
FIREBASE_DB_URL = "https://reytingkimyo-default-rtdb.firebaseio.com"

STUDENTS_WHITELIST = {
    "451241100454": "MAMADJANOVA AZIZAXON RAXMONJON QIZI",
    "451241100436": "YOQUBJONOVA MADINA ULUG'BEK QIZI",
    "451241100420": "AHMADALIYEVA MAHFURAXON AHMADALI QIZI",
    "451241100409": "SIDIQALIYEVA RUHSHONA NURIDDIN QIZI",
    "451241100426": "MAHAMMADJONOVA MOHIRA ABDULMAHMUD QIZI",
    "451241100493": "TURSUNALIYEVA GAVHAROY KELDIBOY QIZI",
    "451241100492": "XOLBOYEVA XILOLA RAHIMJON QIZI",
    "451241100495": "ABDURASHIDOV FOZILBEK LUTFILLO O'G'LI",
    "451241100432": "SADRIDDINOVA SHAHNOZA FAZLIDDIN QIZI",
    "451241100486": "MAXAMMATOV MUXAMMADYUSUF IXVOLDIN O'G'LI",
    "451241100412": "SAYDULLAYEVA MARJONA MURODILLA QIZI",
    "451241100423": "SOTIBOLDIYEVA KAMOLA MAXMUDJON QIZI",
    "451241100421": "JABBOROVA GULYUZ G'IYOSIDDIN QIZI",
    "451241100408": "DILMURODOVA MUXTARAMXON MIRODILJON QIZI",
    "451241101717": "YUSUBALIYEVA NAVBAHOR O'TKIRALI QIZI",
    "451241100475": "ERGASHOVA MAFTUNA SHUHRATJON QIZI",
    "451241100446": "ORIFJONOVA ZEBOXON ZOXIDJON QIZI",
    "451241100429": "ABDUPATTAYEVA ZUHRAXON OTABEK QIZI",
    "451241100444": "XUSANOVA SHAXLO XUSNIDDIN QIZI",
    "451241100443": "SADRIDDINOV YUNUSBEK RIVOJIDDIN O'G'LI",
    "451241100489": "KENJAYEVA OYDINOY MADAMINJON QIZI",
    "451241100415": "G'OFUROVA DILRABOXON ZOKIRJON QIZI",
    "451241100467": "NE'MATOV XOJIAKBAR QUDRATILLO O'G'LI",
}

NAME_MATCH_THRESHOLD = 0.72

BASE_DIR = os.path.dirname(os.path.abspath(__file__))
MEMBERS_FILE = os.path.join(BASE_DIR, "data_members.json")
MANUAL_MAP_FILE = os.path.join(BASE_DIR, "data_manual_map.json")
STATE_FILE = os.path.join(BASE_DIR, "data_state.json")
TIMER_FILE = os.path.join(BASE_DIR, "data_timer.json")

# =====================================================================
#  SOZLAMALAR TUGADI
# =====================================================================

API_BASE = f"https://api.telegram.org/bot{BOT_TOKEN}"
FILE_BASE = f"https://api.telegram.org/file/bot{BOT_TOKEN}"
TMP_DIR = os.path.join(BASE_DIR, "tmp_uploads")
os.makedirs(TMP_DIR, exist_ok=True)

# ---------------------------------------------------------------------
#  Kichik JSON-fayl ombori
# ---------------------------------------------------------------------

def _load_json(path, default):
    if not os.path.exists(path): return default
    try:
        with open(path, "r", encoding="utf-8") as f: return json.load(f)
    except Exception: return default

def _save_json(path, data):
    with open(path, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

def upsert_member(user_id, first_name="", last_name="", username=""):
    members = _load_json(MEMBERS_FILE, {})
    full_name = f"{first_name or ''} {last_name or ''}".strip()
    members[str(user_id)] = {
        "first_name": first_name or "", "last_name": last_name or "",
        "username": username or "", "full_name": full_name,
    }
    _save_json(MEMBERS_FILE, members)

def load_members(): return _load_json(MEMBERS_FILE, {})

def add_manual_mapping(doc_name_key, user_id):
    mapping = _load_json(MANUAL_MAP_FILE, {})
    mapping[doc_name_key] = user_id
    _save_json(MANUAL_MAP_FILE, mapping)

def load_manual_map(): return _load_json(MANUAL_MAP_FILE, {})

def get_state(admin_id):
    all_states = _load_json(STATE_FILE, {})
    return all_states.get(str(admin_id), {})

def set_state(admin_id, state_dict):
    all_states = _load_json(STATE_FILE, {})
    all_states[str(admin_id)] = state_dict
    _save_json(STATE_FILE, all_states)

def clear_state(admin_id): set_state(admin_id, {})

# ---------------------------------------------------------------------
#  Telegram Bot API bilan ishlash
# ---------------------------------------------------------------------

def _post(method, payload=None):
    try:
        resp = requests.post(f"{API_BASE}/{method}", json=payload, timeout=30)
        return resp.json()
    except requests.exceptions.RequestException:
        # Tizim tarmog'i xatolari ignor qilinadi (qotib qolmasligi uchun)
        return {"ok": False, "description": "Network/Proxy Error"}

def send_message(chat_id, text, reply_markup=None):
    payload = {"chat_id": chat_id, "text": text, "parse_mode": "HTML", "disable_web_page_preview": True}
    if reply_markup: payload["reply_markup"] = reply_markup
    return _post("sendMessage", payload)

def edit_message_text(chat_id, message_id, text, reply_markup=None):
    payload = {"chat_id": chat_id, "message_id": message_id, "text": text, "parse_mode": "HTML"}
    if reply_markup: payload["reply_markup"] = reply_markup
    return _post("editMessageText", payload)

def pin_chat_message(chat_id, message_id):
    payload = {"chat_id": chat_id, "message_id": message_id}
    return _post("pinChatMessage", payload)

def answer_callback_query(cq_id, text=None, show_alert=False):
    payload = {"callback_query_id": cq_id, "show_alert": show_alert}
    if text: payload["text"] = text
    return _post("answerCallbackQuery", payload)

def get_file_path(file_id):
    data = _post("getFile", {"file_id": file_id})
    if data.get("ok"): return data["result"]["file_path"]
    return None

def download_telegram_file(file_path, save_to):
    try:
        resp = requests.get(f"{FILE_BASE}/{file_path}", timeout=60)
        with open(save_to, "wb") as f: f.write(resp.content)
        return save_to
    except requests.exceptions.RequestException:
        return None

def inline_keyboard(rows):
    return {"inline_keyboard": [[{"text": text, "callback_data": data} for (text, data) in row] for row in rows]}

def get_updates(offset=None, timeout=25):
    payload = {"timeout": timeout}
    if offset is not None: payload["offset"] = offset
    try:
        resp = requests.post(f"{API_BASE}/getUpdates", json=payload, timeout=timeout + 10)
        return resp.json()
    except requests.exceptions.ProxyError:
        # PythonAnywhere 503 xatosi uchun jim turuvchi yechim
        return {"ok": False, "result": []}
    except requests.exceptions.RequestException:
        return {"ok": False, "result": []}
    except Exception:
        return {"ok": False, "result": []}

# ---------------------------------------------------------------------
#  Taymer jarayoni (Gibrid aqlli interval bilan)
# ---------------------------------------------------------------------
def timer_worker():
    """Taymer qolgan vaqtga qarab yangilanish tezligini avtomatik sozlaydi"""
    while True:
        timer_data = _load_json(TIMER_FILE, {})
        target_str = timer_data.get("target")
        msg_id = timer_data.get("message_id")
        sleep_interval = 10 # Standart uxlash vaqti

        if target_str and msg_id:
            try:
                target_date = datetime.strptime(target_str, "%Y-%m-%d %H:%M:%S")
                now = datetime.now()
                diff = target_date - now

                if diff.total_seconds() > 0:
                    days = diff.days
                    hours, rem = divmod(diff.seconds, 3600)
                    minutes, seconds = divmod(rem, 60)

                    text = (
                        f"📣 <b>Diqqat!</b>\n\n"
                        f"⏳ <b>Taxminiy natija e'lon qilinishiga:</b>\n\n"
                        f"🗓 <b>{days}</b> kun, <b>{hours:02d}</b> soat, <b>{minutes:02d}</b> minut, <b>{seconds:02d}</b> sekund qoldi."
                    )
                    _post("editMessageText", {"chat_id": GROUP_ID, "message_id": msg_id, "text": text, "parse_mode": "HTML"})

                    # Dinamik interval: Vaqt qisqargan sari tezlashamiz
                    if diff.total_seconds() > 3600:
                        sleep_interval = 10 # 1 soatdan ko'p qolsa 10 sek.
                    elif diff.total_seconds() > 60:
                        sleep_interval = 5  # 1 daqiqadan ko'p qolsa 5 sek.
                    else:
                        sleep_interval = 1  # Oxirgi 1 minut qolganda SUPER TEZ (1 sek) sanaydi
                else:
                    text = "🎉 <b>Kutilgan vaqt keldi! Natijalar e'lon qilinmoqda!</b>"
                    _post("editMessageText", {"chat_id": GROUP_ID, "message_id": msg_id, "text": text, "parse_mode": "HTML"})
                    timer_data["target"] = None
                    _save_json(TIMER_FILE, timer_data)
            except Exception:
                pass

        time.sleep(sleep_interval)

# ---------------------------------------------------------------------
#  Hujjat parser funksiyalari
# ---------------------------------------------------------------------

NAME_KEYWORDS = ["ism", "familiya", "familya", "f.i.sh", "fio", "talaba", "ism-sharif"]
BALL_KEYWORDS = ["ball", "umumiy", "reyting", "score"]
TYPE_KEYWORDS = ["ta'lim", "talim", "grant", "kontrakt", "toifa", "turi"]

def _find_column(header_cells, keywords):
    for idx, cell in enumerate(header_cells):
        if not cell: continue
        low = str(cell).strip().lower()
        for kw in keywords:
            if kw in low: return idx
    return None

def _rows_from_table(table):
    if not table or len(table) < 2: return []
    header = table[0]
    name_idx = _find_column(header, NAME_KEYWORDS)
    ball_idx = _find_column(header, BALL_KEYWORDS)
    type_idx = _find_column(header, TYPE_KEYWORDS)

    if name_idx is None:
        name_idx = 0
        first_col_vals = [row[0] for row in table[1:6] if len(row) > 0]
        if all(str(v).strip().isdigit() for v in first_col_vals if v):
            name_idx = 1 if len(header) > 1 else 0

    rows = []
    for raw_row in table[1:]:
        if not raw_row or all((c is None or str(c).strip() == "") for c in raw_row): continue
        try: full_name = str(raw_row[name_idx]).strip() if name_idx < len(raw_row) else ""
        except Exception: full_name = ""
        if not full_name or full_name.isdigit(): continue
        ball = str(raw_row[ball_idx]).strip() if (ball_idx is not None and ball_idx < len(raw_row) and raw_row[ball_idx]) else ""
        talim_turi = str(raw_row[type_idx]).strip() if (type_idx is not None and type_idx < len(raw_row) and raw_row[type_idx]) else ""
        rows.append({"full_name": full_name, "ball": ball, "talim_turi": talim_turi})
    return rows

def parse_docx(path):
    from docx import Document
    doc = Document(path)
    all_rows = []
    for table in doc.tables:
        grid = [[cell.text.strip() for cell in row.cells] for row in table.rows]
        all_rows.extend(_rows_from_table(grid))
    return all_rows

def parse_xlsx(path):
    import openpyxl
    wb = openpyxl.load_workbook(path, data_only=True)
    all_rows = []
    for ws in wb.worksheets:
        grid = [["" if c is None else str(c) for c in row] for row in ws.iter_rows(values_only=True)]
        rows = _rows_from_table(grid)
        if rows: all_rows.extend(rows)
    return all_rows

def parse_csv(path):
    with open(path, "r", encoding="utf-8-sig", errors="ignore") as f: content = f.read()
    delimiter = ";" if content.count(";") > content.count(",") else ","
    grid = list(csv.reader(io.StringIO(content), delimiter=delimiter))
    return _rows_from_table(grid)

def _parse_pdf_text_fallback(text):
    rows = []
    pattern = re.compile(r"^\s*\d*[\.\)]?\s*([A-Za-zА-Яа-яЎўҚқҒғҲҳ'\.\s]{4,60}?)\s{1,}(\d{1,3}(?:[.,]\d+)?)\s{1,}(.+)$")
    for raw_line in text.splitlines():
        line = raw_line.strip()
        if not line: continue
        m = pattern.match(line)
        if m:
            name = m.group(1).strip(" .-")
            ball = m.group(2).replace(",", ".")
            talim_turi = m.group(3).strip()
            if len(name.split()) >= 2:
                rows.append({"full_name": name, "ball": ball, "talim_turi": talim_turi})
    return rows

def parse_pdf(path):
    import pdfplumber
    all_rows, full_text_parts = [], []
    with pdfplumber.open(path) as pdf:
        for page in pdf.pages:
            for table in page.extract_tables(): all_rows.extend(_rows_from_table(table))
            full_text_parts.append(page.extract_text() or "")
    if not all_rows: all_rows = _parse_pdf_text_fallback("\n".join(full_text_parts))
    return all_rows

def parse_document(path, filename):
    lower = filename.lower()
    try:
        if lower.endswith(".pdf"): rows = parse_pdf(path)
        elif lower.endswith(".docx"): rows = parse_docx(path)
        elif lower.endswith((".xlsx", ".xls", ".xlsm")): rows = parse_xlsx(path)
        elif lower.endswith(".csv"): rows = parse_csv(path)
        else: return [], "Bu formatni o'qiy olmayman. PDF, DOCX, XLSX yoki CSV yuboring."
    except Exception as e:
        return [], f"Hujjatni o'qishda xatolik: {e}"
    if not rows: return [], "Hujjatdan jadval topa olmadim."
    return rows, None

# ---------------------------------------------------------------------
#  Saytdan ma'lumot olish, Ism moslashtirish, Matnlar
# ---------------------------------------------------------------------
def _key_for(name):
    for ch in [".", "#", "$", "[", "]"]: name = name.replace(ch, "_")
    return name

def fetch_ranking():
    try:
        resp = requests.get(f"{FIREBASE_DB_URL}/students.json", timeout=20)
        resp.raise_for_status()
        students_data = resp.json() or {}
    except Exception as e:
        return None, f"Firebase bazasiga ulanib bo'lmadi: {e}"

    ranking = []
    for hemis_id, full_name in STUDENTS_WHITELIST.items():
        entry = students_data.get(_key_for(full_name), {}) or {}
        gpa = float(entry.get("gpa") or 0)
        criteria = entry.get("criteria") or [0] * 11
        try: criteria_sum = sum(float(x or 0) for x in criteria)
        except Exception: criteria_sum = 0.0
        total = round(gpa * 16 + criteria_sum / 5, 2)
        ranking.append({"hemis_id": hemis_id, "full_name": full_name, "gpa": gpa, "total": total})

    ranking.sort(key=lambda x: x["total"], reverse=True)
    for idx, item in enumerate(ranking, start=1): item["place"] = idx
    return ranking, None

def normalize_name(name):
    if not name: return ""
    name = name.lower()
    name = name.replace("’", "'").replace("ʻ", "'").replace("`", "'")
    name = re.sub(r"[^a-zа-яёўқғҳ'\s]", " ", name)
    return re.sub(r"\s+", " ", name).strip()

def _similarity(a, b): return SequenceMatcher(None, a, b).ratio()

def match_name_to_member(doc_full_name):
    norm_doc = normalize_name(doc_full_name)
    if not norm_doc: return None

    manual_map = load_manual_map()
    if norm_doc in manual_map:
        user_id = manual_map[norm_doc]
        member = load_members().get(str(user_id), {})
        return {"user_id": user_id, "matched_name": member.get("full_name", doc_full_name), "score": 1.0, "source": "manual"}

    members = load_members()
    best, best_score = None, 0.0
    doc_parts = norm_doc.split()
    doc_reversed = " ".join(reversed(doc_parts))

    for user_id, info in members.items():
        member_norm = normalize_name(info.get("full_name", ""))
        if not member_norm: continue
        score = max(_similarity(norm_doc, member_norm), _similarity(doc_reversed, member_norm))
        member_parts = set(member_norm.split())
        if set(doc_parts).issubset(member_parts) and doc_parts: score = max(score, 0.95)
        if score > best_score:
            best_score = score
            best = {"user_id": int(user_id), "matched_name": info.get("full_name", ""), "score": round(score, 3), "source": "auto"}

    if best and best_score >= NAME_MATCH_THRESHOLD: return best
    return None

CONGRATS_LINES = [
    "Zo'r natija! Mehnatingiz albatta o'z samarasini beradi. 🎉",
    "Tabriklaymiz! Bu g'alaba siz uchun yangi marralarga bir qadam. 🏆",
    "Ajoyib natija! Shunday davom eting, kelajak sizniki! ✨",
]

def _esc(text): return html.escape(str(text or "").strip())

def classify_grant(talim_turi_text):
    raw = (talim_turi_text or "").strip()
    low = raw.lower().replace("’", "'").replace("ʻ", "'").replace("`", "'")
    if "to'liq bo'lmagan" in low or "toliq bolmagan" in low: return raw, "50%"
    if "to'liq" in low or "toliq" in low: return raw, "100%"
    return raw, None

def mention_html(user_id, display_name):
    return f'<a href="tg://user?id={user_id}">{_esc(display_name)}</a>'

def format_result_message(display_name_html, ball, talim_turi_text, place=None):
    talim_matn, foiz = classify_grant(talim_turi_text)
    talim_line = f"🎓 <b>Ta'lim turi:</b> {_esc(talim_matn)}" + (f" <i>({foiz} lik grant)</i>" if foiz else "")
    place_line = f"📍 <b>Reytingdagi o'rni:</b> {place}\n" if place else ""
    ball_esc = _esc(ball) if ball not in (None, "") else "—"
    return (f"🔔 <b>Natijalar e'lon qilindi!</b>\n\n👤 {display_name_html}\n{place_line}"
            f"📊 <b>Umumiy ball:</b> {ball_esc}\n{talim_line}\n\n🎉 <b>Tabriklaymiz!</b> {random.choice(CONGRATS_LINES)}")

def format_no_grant_message(display_name_html, place=None):
    place_line = f"📍 <b>Reytingdagi o'rni:</b> {place}\n\n" if place else "\n"
    return (f"📢 <b>Natijalar bo'yicha ma'lumot</b>\n\n👤 {display_name_html}\n{place_line}"
            f"😔 Afsuski, siz 2026/2027 - o'quv yilida <b>GRANT</b> asosida tahsil olish uchun yig'gan ballaringiz yetmadi.\n\n"
            f"💪 Keyingi o'quv yilida bardosh bering, mehnat albatta natija beradi. 🌱")

def format_summary_message(total_count, matched_count, unmatched_count):
    return (f"✅ <b>Yuborish yakunlandi!</b>\n\n👥 Jami talabalar: <b>{total_count}</b>\n"
            f"✔️ Guruhda topilib zikr qilingan: <b>{matched_count}</b>\n✏️ Faqat ism bilan yozilgan (topilmadi): <b>{unmatched_count}</b>")

def is_admin(user_id): return int(user_id) in ADMIN_IDS

def show_admin_panel(chat_id):
    text = "🛠 <b>Admin Boshqaruv Paneli</b>\n\nQuyidagi amallardan birini tanlang:"
    kb = inline_keyboard([[("📢 Natija e'lon qilish", "adm:doc")], [("🌐 Saytdan integratsiya", "adm:site")]])
    send_message(chat_id, text, reply_markup=kb)

def parse_range(text, max_val=None):
    try:
        parts = [p.strip() for p in text.replace(" ", "").split(",")]
        if len(parts) != 2: return None
        a, b = int(parts[0]), int(parts[1])
        if a > b: a, b = b, a
        if max_val is not None and (a < 0 or b > max_val): return None
        return (a, b)
    except Exception: return None

def build_ranking_preview(ranking, limit=30):
    lines = ["🌐 <b>Saytdan olingan reyting:</b>\n"]
    for item in ranking[:limit]: lines.append(f"{item['place']}. {_esc(item['full_name'])} — <b>{item['total']}</b> ball")
    if len(ranking) > limit: lines.append(f"\n... va yana {len(ranking) - limit} ta talaba.")
    lines.append(f"\n👥 Jami: <b>{len(ranking)}</b> talaba.")
    return "\n".join(lines)

def build_preview_text(prepared_rows, limit=15):
    matched = sum(1 for r in prepared_rows if r.get("match"))
    unmatched = len(prepared_rows) - matched
    lines = ["🔎 <b>Tayyorlangan natijalar:</b>\n"]
    for i, r in enumerate(prepared_rows[:limit], start=1):
        status = "✅" if r.get("match") else "⚠️"
        grant = r.get("talim_turi") if r.get("talim_turi") is not None else "GRANT YETMADI"
        lines.append(f"{i}. {_esc(r['full_name'])} — {r.get('ball','')} ball — {_esc(str(grant))} {status}")
    if len(prepared_rows) > limit: lines.append(f"\n... va yana {len(prepared_rows) - limit} ta qator.")
    lines.append("\n✅ Hammasi to'g'ri bo'lsa, pastdagi tugmani bosing.")
    return "\n".join(lines)

def download_document(document):
    file_id = document["file_id"]
    filename = document.get("file_name") or f"hujjat_{file_id[:8]}"
    file_path = get_file_path(file_id)
    if not file_path: return None, None
    local_path = os.path.join(TMP_DIR, f"{int(time.time())}_{filename}")
    if download_telegram_file(file_path, local_path): return local_path, filename
    return None, None

# ---------------------------------------------------------------------
#  Guruhga yuborish (Interval bilan)
# ---------------------------------------------------------------------

def broadcast_to_group(admin_chat_id, rows):
    matched, unmatched, failed = 0, 0, 0
    delay_seconds = _load_json(STATE_FILE, {}).get("global_delay", 15)

    for row in rows:
        match = row.get("match")
        if match:
            display = mention_html(match["user_id"], match["matched_name"])
            matched += 1
        else:
            display = f"<b>{_esc(row['full_name'])}</b>"
            unmatched += 1

        talim_turi, place = row.get("talim_turi"), row.get("place")
        if talim_turi is None: text = format_no_grant_message(display, place=place)
        else: text = format_result_message(display, row.get("ball"), talim_turi, place=place)

        result = send_message(GROUP_ID, text)
        if not result.get("ok"): failed += 1

        time.sleep(delay_seconds)

    summary = format_summary_message(len(rows), matched, unmatched)
    if failed: summary += f"\n\n⚠️ <b>{failed}</b> ta xabar yuborishda xatolik yuz berdi."
    send_message(admin_chat_id, summary)

# ---------------------------------------------------------------------
#  Callback va Xabarlar
# ---------------------------------------------------------------------

def handle_callback(cq):
    cq_id = cq["id"]
    user_id = cq.get("from", {}).get("id")
    message = cq.get("message", {})
    chat_id = message.get("chat", {}).get("id")
    message_id = message.get("message_id")
    data = cq.get("data", "")

    if not is_admin(user_id):
        answer_callback_query(cq_id, "Sizda ruxsat yo'q.", show_alert=True)
        return
    answer_callback_query(cq_id)

    if data == "adm:doc":
        set_state(user_id, {"mode": "await_document"})
        edit_message_text(chat_id, message_id, "📎 Yaxshi! Endi hujjatni (PDF, DOCX, XLSX yoki CSV) shu yerga yuboring.\nHujjatda: <b>Ism Familya</b>, <b>Umumiy ball</b>, <b>Ta'lim turi</b> bo'lishi kerak.")

    elif data == "adm:site":
        edit_message_text(chat_id, message_id, "⏳ Saytdan ma'lumotlar olinmoqda, kuting...")
        ranking, err = fetch_ranking()
        if err:
            send_message(chat_id, f"❌ {err}"); clear_state(user_id); return
        if not ranking:
            send_message(chat_id, "❌ Saytdan hech qanday talaba topilmadi."); clear_state(user_id); return
        set_state(user_id, {"mode": "await_range_100", "payload": {"ranking": ranking}})
        text = (build_ranking_preview(ranking) + "\n\n✏️ <b>100% lik (to'liq) grant</b> uchun o'rinlar oralig'ini kiriting.\nMasalan: <code>1,8</code>")
        send_message(chat_id, text)

    elif data == "confirm:send":
        state = get_state(user_id)
        rows = (state.get("payload") or {}).get("rows", [])
        if not rows:
            edit_message_text(chat_id, message_id, "❌ Xatolik. Qaytadan boshlang."); clear_state(user_id); return

        delay = _load_json(STATE_FILE, {}).get("global_delay", 15)
        edit_message_text(chat_id, message_id, f"📤 Guruhga yuborilmoqda... (Har {delay} soniyada bitta)\nBiroz kuting.")
        clear_state(user_id)
        broadcast_to_group(chat_id, rows)

    elif data == "confirm:cancel":
        clear_state(user_id)
        edit_message_text(chat_id, message_id, "❌ Bekor qilindi.")

def handle_group_message(msg):
    new_members = msg.get("new_chat_members")
    if new_members:
        for m in new_members:
            if not m.get("is_bot"): upsert_member(m["id"], m.get("first_name", ""), m.get("last_name", ""), m.get("username", ""))
        return
    from_user = msg.get("from", {})
    if from_user and not from_user.get("is_bot"):
        upsert_member(from_user["id"], from_user.get("first_name", ""), from_user.get("last_name", ""), from_user.get("username", ""))

def handle_map_command(chat_id, text):
    body = text[len("/map"):].strip()
    if "|" not in body:
        send_message(chat_id, "❗️ Format xato."); return
    doc_name, target = [p.strip() for p in body.split("|", 1)]
    try:
        user_id = int(target)
        add_manual_mapping(normalize_name(doc_name), user_id)
        send_message(chat_id, f"✅ Bog'landi: <b>{_esc(doc_name)}</b> → <code>{user_id}</code>")
    except ValueError:
        send_message(chat_id, "❌ user_id noto'g'ri formatda.")

def handle_private_message(msg, chat_id, user_id):
    if not is_admin(user_id):
        send_message(chat_id, "Bu bot faqat ma'muriyat uchun."); return

    text = msg.get("text", "") or ""
    document = msg.get("document")

    if text == "/start":
        clear_state(user_id); show_admin_panel(chat_id); return

    if text.startswith("/map"):
        handle_map_command(chat_id, text); return

    if text.startswith("/delay"):
        try:
            val = int(text.split()[1])
            all_states = _load_json(STATE_FILE, {})
            all_states["global_delay"] = val
            _save_json(STATE_FILE, all_states)
            send_message(chat_id, f"✅ Xabarlar orasidagi interval <b>{val} soniya</b> qilib belgilandi.")
        except Exception:
            send_message(chat_id, "❌ Noto'g'ri format. Foydalanish: <code>/delay 20</code>")
        return

    if text.startswith("/timer"):
        try:
            target_str = text[len("/timer"):].strip()
            datetime.strptime(target_str, "%Y-%m-%d %H:%M:%S")
            resp = send_message(GROUP_ID, "⏳ Taymer ishga tushirildi...")
            if resp.get("ok"):
                msg_id = resp["result"]["message_id"]
                pin_chat_message(GROUP_ID, msg_id)
                timer_conf = _load_json(TIMER_FILE, {})
                timer_conf["target"] = target_str
                timer_conf["message_id"] = msg_id
                _save_json(TIMER_FILE, timer_conf)
                send_message(chat_id, f"✅ Taymer guruhga yuborildi va pin qilindi!\nVaqt: {target_str}")
            else:
                send_message(chat_id, "❌ Guruhga xabar yuborishda xatolik.")
        except ValueError:
            send_message(chat_id, "❌ Sana formati noto'g'ri. \nFoydalanish: <code>/timer 2026-08-15 14:00:00</code>")
        return

    state = get_state(user_id)
    mode = state.get("mode")

    if mode == "await_document":
        if document:
            send_message(chat_id, "⏳ Hujjat qabul qilindi, tahlil qilinmoqda...")
            local_path, filename = download_document(document)
            if not local_path:
                send_message(chat_id, "❌ Faylni yuklab bo'lmadi."); return
            rows, err = parse_document(local_path, filename)
            try: os.remove(local_path)
            except OSError: pass

            if err:
                send_message(chat_id, f"❌ {err}"); return
            prepared = [{"full_name": r["full_name"], "ball": r["ball"], "talim_turi": r["talim_turi"] or "Noma'lum", "place": None, "match": match_name_to_member(r["full_name"])} for r in rows]
            set_state(user_id, {"mode": "confirm_broadcast", "payload": {"rows": prepared}})
            kb = inline_keyboard([[("✅ Guruhga yuborish", "confirm:send"), ("❌ Bekor qilish", "confirm:cancel")]])
            send_message(chat_id, build_preview_text(prepared), reply_markup=kb)
        else: send_message(chat_id, "📎 Iltimos, hujjat (PDF/DOCX/XLSX/CSV) yuboring.")
        return

    if mode == "await_range_100":
        payload = state.get("payload", {})
        ranking = payload.get("ranking", [])
        rng = parse_range(text, max_val=len(ranking))
        if not rng:
            send_message(chat_id, "❌ Format noto'g'ri. Masalan: <code>1,8</code>"); return
        payload["range100"] = rng
        set_state(user_id, {"mode": "await_range_50", "payload": payload})
        send_message(chat_id, "✏️ Endi <b>50% lik (to'liq bo'lmagan) grant</b> uchun oralig'ni kiriting.\nMasalan: <code>9,20</code> yoki <code>0,0</code>")
        return

    if mode == "await_range_50":
        payload = state.get("payload", {})
        ranking = payload.get("ranking", [])
        range100 = payload.get("range100")
        rng50 = parse_range(text, max_val=len(ranking))
        if not rng50:
            send_message(chat_id, "❌ Format noto'g'ri."); return
        prepared = []
        for item in ranking:
            place = item["place"]
            if range100[0] <= place <= range100[1]: talim_turi = "To'liq ta'lim granti"
            elif rng50 != (0, 0) and rng50[0] <= place <= rng50[1]: talim_turi = "To'liq bo'lmagan ta'lim granti"
            else: talim_turi = None
            prepared.append({"full_name": item["full_name"], "ball": item["total"], "talim_turi": talim_turi, "place": place, "match": match_name_to_member(item["full_name"])})
        set_state(user_id, {"mode": "confirm_broadcast", "payload": {"rows": prepared}})
        kb = inline_keyboard([[("✅ Guruhga yuborish", "confirm:send"), ("❌ Bekor qilish", "confirm:cancel")]])
        send_message(chat_id, build_preview_text(prepared), reply_markup=kb)
        return

    show_admin_panel(chat_id)

# ---------------------------------------------------------------------
#  ASOSIY POLLING SIKLI
# ---------------------------------------------------------------------

def main():
    if "BU_YERGA" in BOT_TOKEN:
        print("❌ Iltimos, SOZLAMALAR bo'limini to'ldiring!")
        return

    print("✅ Bot ishga tushdi... (to'xtatish uchun Ctrl+C)")
    threading.Thread(target=timer_worker, daemon=True).start()

    offset = None
    while True:
        try:
            resp = get_updates(offset=offset)
            if not resp.get("ok"):
                time.sleep(3)
                continue
            for update in resp.get("result", []):
                offset = update["update_id"] + 1
                try:
                    if "callback_query" in update: handle_callback(update["callback_query"])
                    elif "message" in update:
                        msg = update["message"]
                        chat_id = msg["chat"]["id"]
                        chat_type = msg["chat"]["type"]
                        if chat_id == GROUP_ID or chat_type in ("group", "supergroup"): handle_group_message(msg)
                        else: handle_private_message(msg, chat_id, msg.get("from", {}).get("id"))
                except Exception:
                    pass
        except KeyboardInterrupt:
            print("\n⏹ Bot to'xtatildi.")
            break
        except Exception:
            time.sleep(5)

if __name__ == "__main__":
    main()
