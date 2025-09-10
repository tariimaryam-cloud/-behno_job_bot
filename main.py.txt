# -*- coding: utf-8 -*-
# نیازمندی‌ها: python-telegram-bot==13.15 , Flask==2.3.2
# Run تنظیم: .replit → run = "python3 main.py"
import os, json, time, threading, logging
from flask import Flask
from telegram import InlineKeyboardButton, InlineKeyboardMarkup, Update
from telegram.ext import (
    Updater, CommandHandler, CallbackQueryHandler,
    MessageHandler, Filters, CallbackContext
)
from telegram.error import BadRequest, InvalidToken, NetworkError

# ========== Logging ==========
logging.basicConfig(
    format="%(asctime)s [%(levelname)s] %(message)s", level=logging.INFO
)
log = logging.getLogger("behnojobbot")

# ========== Token ==========
TOKEN = os.environ.get("BOT_TOKEN", "").strip()

# ========== Files ==========
RESUMES_FILE   = "resumes.json"     # رزومه‌ها
JOBS_FILE      = "jobs.json"        # فرصت‌های شغلی
CHALLENGES_FILE= "challenges.json"  # چالش اتاق مصاحبه

# ========== Web server for UptimeRobot ==========
app = Flask(__name__)

@app.route("/")
def home():
    return "Bot is running ✅"

def run_server():
    app.run(host="0.0.0.0", port=8080)

# ========== IO Helpers ==========
def _load(file):
    if not os.path.exists(file):
        return []
    try:
        with open(file, "r", encoding="utf-8") as f:
            return json.load(f)
    except Exception as e:
        log.warning(f"Could not load {file}: {e}")
        return []

def _save(file, obj):
    data = _load(file)
    data.append(obj)
    with open(file, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

def norm(x): return (x or "").strip()

# ========== Constant Data ==========
PROVINCES = [
    "آذربایجان شرقی","آذربایجان غربی","اردبیل","اصفهان","البرز","ایلام","بوشهر","تهران",
    "چهارمحال و بختیاری","خراسان جنوبی","خراسان رضوی","خراسان شمالی","خوزستان","زنجان","سمنان",
    "سیستان و بلوچستان","فارس","قزوین","قم","کردستان","کرمان","کرمانشاه","کهگیلویه و بویراحمد",
    "گلستان","گیلان","لرستان","مازندران","مرکزی","هرمزگان","همدان","یزد"
]

FIELDS = [
    "منابع انسانی و اداری",
    "حسابداری و مالی و انبار",
    "بازرگانی و فروش و بازاریابی",
    "تولید، برنامه‌ریزی و کنترل کیفیت",
    "فنی و مهندسی (تعمیرات و نگهداری)",
    "کلیه حوزه‌های رشته فناوری اطلاعات",
    "طراحی (لباس، گرافیست، صنعتی و سایر…)",
    "مدیرعامل، مدیر کارخانه، مدیر اجرایی، مدیر داخلی",
    "کلیه حوزه‌های رشته فنی و مهندسی",
    "نگهبان، حراست، خدمات، کارگر ساده و ماهر",
    "کلیه حوزه‌های رشته پزشکی و پیراپزشکی",
    "سایر"
]

LEVELS = ["کارشناس", "کارشناس ارشد", "مدیر ارشد به بالا"]

# ========== Keyboards ==========
def main_menu_kb():
    return InlineKeyboardMarkup([
        [InlineKeyboardButton("کارجو 🧑‍💼", callback_data="jobseeker")],
        [InlineKeyboardButton("کارفرما 🏢", callback_data="employer")],
        [InlineKeyboardButton("📢 فرصت‌های شغلی", callback_data="see_jobs")],
        [InlineKeyboardButton("🎤 چالش اتاق مصاحبه", callback_data="challenge")],
        [InlineKeyboardButton("ℹ️ درباره ما", callback_data="about_us")]
    ])

def provinces_kb(prefix):
    kb, row = [], []
    for i, prov in enumerate(PROVINCES):
        row.append(InlineKeyboardButton(prov, callback_data=f"{prefix}_prov_{i}"))
        if len(row) == 2:
            kb.append(row); row = []
    if row: kb.append(row)
    kb.append([InlineKeyboardButton("🏠 خانه", callback_data="home")])
    return InlineKeyboardMarkup(kb)

def fields_kb(prefix):
    kb = [[InlineKeyboardButton(f, callback_data=f"{prefix}_field_{i}")] for i, f in enumerate(FIELDS)]
    kb.append([InlineKeyboardButton("🏠 خانه", callback_data="home")])
    return InlineKeyboardMarkup(kb)

def levels_kb(prefix):
    kb = [[InlineKeyboardButton(l, callback_data=f"{prefix}_level_{i}")] for i, l in enumerate(LEVELS)]
    kb.append([InlineKeyboardButton("🏠 خانه", callback_data="home")])
    return InlineKeyboardMarkup(kb)

# ========== Utilities ==========
def safe_edit(qmsg, text, **kwargs):
    """Avoid BadRequest: Message is not modified."""
    try:
        qmsg.edit_text(text, **kwargs)
    except BadRequest as e:
        if "Message is not modified" in str(e):
            # تغییر کوچیک برای رد شدن از ارور
            qmsg.edit_text(text + "‌", **kwargs)
        else:
            raise

def send_items(bot, chat_id, items, kind_caption):
    """ارسال لیست رزومه/آگهی به صورت فایل یا عکس، با کپشن خلاصه؛ در چند پیام پشت‌سرهم."""
    for it in items:
        cap = f"{kind_caption}\n📍 {it.get('province','')}\n🔹 {it.get('field','')}\n📊 {it.get('level','-')}"
        try:
            if it.get("file_type") == "photo":
                bot.send_photo(chat_id, it["file_id"], caption=cap)
            else:
                bot.send_document(chat_id, it["file_id"], caption=cap)
        except Exception as e:
            log.warning(f"send_items error: {e}")

def clear_wait_flags(context: CallbackContext):
    for k in ["awaiting_resume", "awaiting_jobfile", "awaiting_challenge"]:
        context.user_data.pop(k, None)

# ========== /start ==========
WELCOME = (
    "🎉 *به بهنو جاب خوش آمدید!* \n"
    "اینجا جایی است که کارجویان حرفه‌ای مسیر شغلی آینده خود را می‌سازند و "
    "کارفرمایان هوشمند بهترین استعدادها را کشف می‌کنند.\n\n"
    "🔹 برای کارجویان → کشف بهترین فرصت‌های شغلی متناسب با مهارت‌ها و علایق شما\n"
    "🔹 برای کارفرمایان → جذب سریع و هدفمند بهترین نیروهای متخصص و متعهد\n"
    "🔹 برای سازمان‌ها → دسترسی به ابزارهای نوین ارزیابی و انتخاب استعدادها\n\n"
    "✨ آینده شغلی شما از همین‌جا آغاز می‌شود"
)

ABOUT = (
    "✨ *بِهنو* ✨\n"
    "یک مجموعه تخصصی آموزش، مشاوره و بهبود سیستم‌های سازمانی با رویکردی نوین "
    "در حوزه‌های منابع انسانی، مالی، فروش، بازاریابی و برنامه‌ریزی تولید و سیستم‌ها و روش‌ها.\n\n"
    "🎯 رسالت: ارتقای بهره‌وری، توسعه سرمایه انسانی و ساختاردهی هوشمندانه کسب‌وکارها.\n\n"
    "🔗 کانال رسمی بهنو: https://t.me/behnosyschannel\n"
    "👥 گروه تخصصی منابع انسانی بهنو: https://t.me/behnosys\n"
    "💼 گروه تخصصی مالی بهنو: https://t.me/behnosysfinancial\n"
    "🔹 لینکدین: https://www.linkedin.com/company/behnosys/\n"
    "📸 اینستاگرام: https://www.instagram.com/behnosys\n"
    "☎️ تماس: 09128483679"
)

def start(update: Update, context: CallbackContext):
    clear_wait_flags(context)
    if update.message:
        update.message.reply_text(WELCOME, reply_markup=main_menu_kb(), parse_mode="Markdown", disable_web_page_preview=True)
    else:
        # فالس‌بک در صورت صدا شدن از جای دیگر
        context.bot.send_message(update.effective_chat.id, WELCOME, reply_markup=main_menu_kb(), parse_mode="Markdown", disable_web_page_preview=True)

# ========== Button Router ==========
def button(update: Update, context: CallbackContext):
    q = update.callback_query
    data = q.data
    q.answer()
    clear_wait_flags(context)

    # خانه
    if data == "home":
        safe_edit(q.message, "🏠 منوی اصلی:", reply_markup=main_menu_kb())
        return

    # درباره ما
    if data == "about_us":
        safe_edit(q.message, ABOUT, parse_mode="Markdown", disable_web_page_preview=True,
                  reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🏠 خانه", callback_data="home")]]))
        return

    # ===== چالش اتاق مصاحبه =====
    if data == "challenge":
        items = _load(CHALLENGES_FILE)
        if items:
            last = items[-10:]
            lines = ["🎤 *چالش‌های اتاق مصاحبه (ناشناس):*", ""]
            for i, c in enumerate(last, 1):
                t = norm(c.get("text"))
                if t:
                    # حداکثر 500 کاراکتر هر مورد
                    if len(t) > 500: t = t[:500] + "…"
                    lines.append(f"{i}. {t}")
            lines.append("")
        else:
            lines = ["🎤 هنوز تجربه‌ای ثبت نشده. شما اولین نفر باشید!"]
        lines.append("✍️ «ثبت تجربه جدید» را بزنید و متنتان را بفرستید (بدون نمایش نام/آیدی).")
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("➕ ثبت تجربه جدید", callback_data="add_challenge")],
            [InlineKeyboardButton("🏠 خانه", callback_data="home")]
        ])
        safe_edit(q.message, "\n".join(lines), reply_markup=kb, parse_mode="Markdown")
        return

    if data == "add_challenge":
        context.user_data["awaiting_challenge"] = True
        safe_edit(q.message, "📝 لطفاً تجربه/چالش مصاحبه خود را بنویسید.\n(ناشناس منتشر می‌شود) ",
                  reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🏠 خانه", callback_data="home")]]))
        return

    # ===== کارجو: ارسال رزومه =====
    if data == "jobseeker":
        safe_edit(q.message, "📍 استان خود را انتخاب کنید:", reply_markup=provinces_kb("js"))
        return

    if data.startswith("js_prov_"):
        context.user_data["province"] = PROVINCES[int(data.split("_")[-1])]
        safe_edit(q.message, "🔹 حوزه شغلی را انتخاب کنید:", reply_markup=fields_kb("js"))
        return

    if data.startswith("js_field_"):
        context.user_data["field"] = FIELDS[int(data.split("_")[-1])]
        safe_edit(q.message, "📊 سطح شغلی را انتخاب کنید:", reply_markup=levels_kb("js"))
        return

    if data.startswith("js_level_"):
        context.user_data["level"] = LEVELS[int(data.split("_")[-1])]
        context.user_data["awaiting_resume"] = True
        safe_edit(q.message, "📄 لطفاً رزومه را *فایل PDF/Word یا عکس* ارسال کنید.", parse_mode="Markdown")
        return

    # ===== کارفرما =====
    if data == "employer":
        kb = [
            [InlineKeyboardButton("➕ ثبت فرصت شغلی", callback_data="emp_addjob")],
            [InlineKeyboardButton("📂 مشاهده رزومه کارجویان", callback_data="emp_resumes")],
            [InlineKeyboardButton("🏠 خانه", callback_data="home")]
        ]
        safe_edit(q.message, "کارفرما عزیز، انتخاب کنید:", reply_markup=InlineKeyboardMarkup(kb))
        return

    # ثبت فرصت شغلی
    if data == "emp_addjob":
        safe_edit(q.message, "📍 استان آگهی را انتخاب کنید:", reply_markup=provinces_kb("empjob"))
        return

    if data.startswith("empjob_prov_"):
        context.user_data["province"] = PROVINCES[int(data.split("_")[-1])]
        safe_edit(q.message, "🔹 حوزه شغلی آگهی:", reply_markup=fields_kb("empjob"))
        return

    if data.startswith("empjob_field_"):
        context.user_data["field"] = FIELDS[int(data.split("_")[-1])]
        safe_edit(q.message, "📊 سطح مورد نظر:", reply_markup=levels_kb("empjob"))
        return

    if data.startswith("empjob_level_"):
        context.user_data["level"] = LEVELS[int(data.split("_")[-1])]
        context.user_data["awaiting_jobfile"] = True
        safe_edit(q.message, "📄 لطفاً فایل/عکس آگهی را ارسال کنید.")
        return

    # دیدن رزومه‌ها (کارفرما)
    if data == "emp_resumes":
        safe_edit(q.message, "📍 استان مورد نظر را انتخاب کنید:", reply_markup=provinces_kb("empres"))
        return

    if data.startswith("empres_prov_"):
        context.user_data["province"] = PROVINCES[int(data.split("_")[-1])]
        safe_edit(q.message, "🔹 حوزه مورد نظر:", reply_markup=fields_kb("empres"))
        return

    if data.startswith("empres_field_"):
        context.user_data["field"] = FIELDS[int(data.split("_")[-1])]
        safe_edit(q.message, "📊 سطح مورد نظر:", reply_markup=levels_kb("empres"))
        return

    if data.startswith("empres_level_"):
        level = norm(LEVELS[int(data.split("_")[-1])])
        prov  = norm(context.user_data.get("province"))
        field = norm(context.user_data.get("field"))
        resumes = _load(RESUMES_FILE)
        found = [r for r in resumes
                 if norm(r.get("province"))==prov and norm(r.get("field"))==field and norm(r.get("level"))==level]
        if not found:
            # فالبک: بدون توجه به سطح
            found = [r for r in resumes if norm(r.get("province"))==prov and norm(r.get("field"))==field]
        if found:
            send_items(context.bot, q.message.chat_id, found, "📄 رزومه")
        else:
            safe_edit(q.message, "❌ رزومه‌ای یافت نشد.",
                      reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🏠 خانه", callback_data="home")]]))
        return

    # ===== کارجو: دیدن فرصت‌های شغلی =====
    if data == "see_jobs":
        safe_edit(q.message, "📍 استان مورد نظر را انتخاب کنید:", reply_markup=provinces_kb("seejob"))
        return

    if data.startswith("seejob_prov_"):
        context.user_data["province"] = PROVINCES[int(data.split("_")[-1])]
        safe_edit(q.message, "🔹 حوزه مورد نظر:", reply_markup=fields_kb("seejob"))
        return

    if data.startswith("seejob_field_"):
        context.user_data["field"] = FIELDS[int(data.split("_")[-1])]
        safe_edit(q.message, "📊 سطح مورد نظر:", reply_markup=levels_kb("seejob"))
        return

    if data.startswith("seejob_level_"):
        level = norm(LEVELS[int(data.split("_")[-1])])
        prov  = norm(context.user_data.get("province"))
        field = norm(context.user_data.get("field"))
        jobs = _load(JOBS_FILE)
        found = [j for j in jobs
                 if norm(j.get("province"))==prov and norm(j.get("field"))==field and norm(j.get("level"))==level]
        if not found:
            # فالبک: بدون توجه به سطح
            found = [j for j in jobs if norm(j.get("province"))==prov and norm(j.get("field"))==field]
        if found:
            send_items(context.bot, q.message.chat_id, found, "📢 فرصت شغلی")
        else:
            safe_edit(q.message, "❌ فرصت شغلی موجود نیست.",
                      reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🏠 خانه", callback_data="home")]]))
        return

# ========== Handlers for file & text ==========
def handle_file(update: Update, context: CallbackContext):
    chat_id = update.effective_chat.id
    file_id, file_type = None, None

    if update.message.document:
        file_id = update.message.document.file_id
        file_type = "document"
    elif update.message.photo:
        file_id = update.message.photo[-1].file_id
        file_type = "photo"
    else:
        update.message.reply_text("لطفاً فایل PDF/Word یا عکس ارسال کنید.")
        return

    # رزومه کارجو
    if context.user_data.get("awaiting_resume"):
        obj = {
            "province": norm(context.user_data.get("province")),
            "field":    norm(context.user_data.get("field")),
            "level":    norm(context.user_data.get("level")),
            "file_id":  file_id,
            "file_type":file_type,
            "ts": int(time.time())
        }
        _save(RESUMES_FILE, obj)
        context.user_data["awaiting_resume"] = False
        update.message.reply_text("✅ رزومه شما ثبت شد.")
        return

    # آگهی کارفرما
    if context.user_data.get("awaiting_jobfile"):
        obj = {
            "province": norm(context.user_data.get("province")),
            "field":    norm(context.user_data.get("field")),
            "level":    norm(context.user_data.get("level")),
            "file_id":  file_id,
            "file_type":file_type,
            "ts": int(time.time())
        }
        _save(JOBS_FILE, obj)
        context.user_data["awaiting_jobfile"] = False
        update.message.reply_text("✅ آگهی شما ثبت شد.")
        return

    # اگر هیچ انتظاری نبود:
    update.message.reply_text("برای ثبت رزومه/آگهی ابتدا از منو مسیر را انتخاب کنید. /start")

def handle_text(update: Update, context: CallbackContext):
    if context.user_data.get("awaiting_challenge"):
        txt = norm(update.message.text)
        if not txt:
            update.message.reply_text("متن خالی است 🙂")
            return
        _save(CHALLENGES_FILE, {"text": txt, "ts": int(time.time())})
        context.user_data["awaiting_challenge"] = False
        update.message.reply_text("✅ تجربه شما به صورت ناشناس ثبت شد و برای همه قابل مشاهده است.")
        return

    # سایر متن‌ها
    update.message.reply_text("برای استفاده از ربات از منو استفاده کنید. /start")

# ========== Extra Commands ==========
def cmd_ping(update: Update, context: CallbackContext):
    update.message.reply_text("pong ✅")

def cmd_cancel(update: Update, context: CallbackContext):
    clear_wait_flags(context)
    update.message.reply_text("لغو شد. /start")

def error_handler(update: object, context: CallbackContext):
    log.error("Exception in handler", exc_info=context.error)

# ========== Run ==========
def main():
    # Flask برای بیدار ماندن
    threading.Thread(target=run_server, daemon=True).start()

    if not TOKEN:
        log.error("BOT_TOKEN ست نشده. Tools → Secrets → BOT_TOKEN")
        return

    try:
        updater = Updater(TOKEN, use_context=True)
        me = updater.bot.get_me()
        log.info(f"Bot OK: @{me.username} ({me.id})")
    except InvalidToken:
        log.error("InvalidToken: توکن نامعتبر است یا باطل شده.")
        return
    except Exception as e:
        log.error(f"Bot init error: {e}")
        return

    dp = updater.dispatcher
    dp.add_handler(CommandHandler("start", start))
    dp.add_handler(CommandHandler("ping", cmd_ping))
    dp.add_handler(CommandHandler("cancel", cmd_cancel))
    dp.add_handler(CallbackQueryHandler(button))
    dp.add_handler(MessageHandler(Filters.document | Filters.photo, handle_file))
    dp.add_handler(MessageHandler(Filters.text & ~Filters.command, handle_text))
    dp.add_error_handler(error_handler)

    try:
        updater.start_polling(clean=True)
        log.info("Bot started. Waiting for updates…")
        updater.idle()
    except NetworkError as e:
        log.error(f"NetworkError: {e}")
    except Exception as e:
        log.error(f"Runtime error: {e}")

if __name__ == "__main__":
    main()
