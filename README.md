import base64
import sqlite3
import datetime
import os
import sys
from dotenv import load_dotenv

# Load .env
load_dotenv()

from telegram import Update, ReplyKeyboardMarkup
from telegram.ext import ApplicationBuilder, MessageHandler, filters, ContextTypes, CommandHandler
from openai import OpenAI

print("=" * 50)
print("🚀 BOT INIT BOSHLANDI...")
print("=" * 50)

# Get tokens
TOKEN = os.getenv("TELEGRAM_TOKEN")
OPENAI_KEY = os.getenv("OPENAI_KEY")
ADMIN_ID = int(os.getenv("ADMIN_ID", "0"))

print(f"✅ TOKEN: {TOKEN[:20]}...")
print(f"✅ OPENAI_KEY: {OPENAI_KEY[:20]}...")
print(f"✅ ADMIN_ID: {ADMIN_ID}")

if not TOKEN or not OPENAI_KEY:
    print("❌ .env faylda TOKEN yoki OPENAI_KEY yo'q!")
    sys.exit(1)

CHANNELS = ["@HashtagMasterUZ"]

try:
    client = OpenAI(api_key=OPENAI_KEY)
    print("✅ OpenAI client yaratildi")
except Exception as e:
    print(f"❌ OpenAI client xatosi: {e}")
    sys.exit(1)

# ===== DB =====
conn = sqlite3.connect("bot.db", check_same_thread=False)
cursor = conn.cursor()

cursor.execute("""
CREATE TABLE IF NOT EXISTS users (
    user_id INTEGER PRIMARY KEY,
    username TEXT,
    limit_count INTEGER,
    used INTEGER,
    last_date TEXT,
    vip INTEGER,
    vip_expire TEXT
)
""")
conn.commit()
print("✅ Database initialized")

# ===== MENU =====
menu = [
    ["💬 Chat", "✍️ Yozish"],
    ["🎨 Rasm", "🖼 Tahlil"],
    ["🆙 4K Rasm", "✨ Tiniqlashtirish"],
    ["🔊 Ovoz tanlash"]
]

voices = ["alloy", "nova", "echo", "shimmer"]

# ===== USER =====
def get_user(user):
    today = str(datetime.date.today())
    cursor.execute("SELECT * FROM users WHERE user_id=?", (user.id,))
    data = cursor.fetchone()

    if not data:
        cursor.execute(
            "INSERT INTO users VALUES (?, ?, ?, ?, ?, ?, ?)",
            (user.id, user.username, 10, 0, today, 0, None)
        )
        conn.commit()
        return get_user(user)

    if data[4] != today:
        cursor.execute(
            "UPDATE users SET used=0, last_date=? WHERE user_id=?",
            (today, user.id)
        )
        conn.commit()

    return cursor.execute("SELECT * FROM users WHERE user_id=?", (user.id,)).fetchone()

def is_vip(user_id):
    user = cursor.execute("SELECT vip, vip_expire FROM users WHERE user_id=?", (user_id,)).fetchone()
    if user and user[0] == 1:
        if user[1] and datetime.datetime.fromisoformat(user[1]) < datetime.datetime.now():
            cursor.execute("UPDATE users SET vip=0 WHERE user_id=?", (user_id,))
            conn.commit()
            return False
        return True
    return False

def can_use(user_id):
    if is_vip(user_id):
        return True
    user = cursor.execute("SELECT * FROM users WHERE user_id=?", (user_id,)).fetchone()
    if user[3] >= user[2]:
        return False
    cursor.execute("UPDATE users SET used=used+1 WHERE user_id=?", (user_id,))
    conn.commit()
    return True

# ===== CHANNEL CHECK =====
async def check_sub(update, context):
    uid = update.message.from_user.id
    for ch in CHANNELS:
        try:
            member = await context.bot.get_chat_member(ch, uid)
            if member.status not in ["member", "administrator", "creator"]:
                return False
        except:
            return False
    return True

# ===== START =====
async def start(update, context):
    print(f"📍 START: {update.message.from_user.id}")
    user = update.message.from_user
    get_user(user)
    msg = await update.message.reply_text(
        "👋 Salom! Menuni tanlang:",
        reply_markup=ReplyKeyboardMarkup(menu, resize_keyboard=True)
    )
    print(f"✅ START javob yuborildi")

# ===== TEXT =====
async def handle_text(update, context):
    user = update.message.from_user
    text = update.message.text
    
    print(f"📝 TEXT: {user.id} -> {text[:50]}")
    
    get_user(user)

    # Channel check
    if not await check_sub(update, context):
        await update.message.reply_text("❌ Avval @HashtagMasterUZ kanaliga obuna bo'ling!")
        return

    # Voice select
    if text == "🔊 Ovoz tanlash":
        keyboard = [["alloy", "nova"], ["echo", "shimmer"]]
        await update.message.reply_text("🎤 Ovoz tanlang:", reply_markup=ReplyKeyboardMarkup(keyboard, resize_keyboard=True))
        return

    if text in voices:
        context.user_data["voice"] = text
        await update.message.reply_text(f"✅ Ovoz tanlandi: {text}")
        return

    # Mode select
    if text in ["💬 Chat", "✍️ Yozish", "🎨 Rasm", "🆙 4K Rasm", "✨ Tiniqlashtirish"]:
        context.user_data["mode"] = text
        await update.message.reply_text(f"✏️ {text} uchun matn yozing...")
        return

    mode = context.user_data.get("mode")

    # 🎨 RASM
    if mode == "🎨 Rasm":
        if not can_use(user.id):
            await update.message.reply_text("⚠️ Bugunlik limitingiz tugadi!")
            return
        try:
            await update.message.reply_text("⏳ Rasm yaratilmoqda...")
            img = client.images.generate(
                model="dall-e-3",
                prompt=text,
                size="1024x1024",
                quality="standard",
                n=1
            )
            img_url = img.data[0].url
            await update.message.reply_photo(photo=img_url, caption="🎨 Sizning rasmingiz")
            print(f"✅ RASM: {user.id}")
        except Exception as e:
            await update.message.reply_text(f"❌ Xato: {str(e)[:100]}")
            print(f"❌ RASM ERROR: {e}")
        return

    # 🆙 4K RASM
    if mode == "🆙 4K Rasm":
        if not can_use(user.id):
            await update.message.reply_text("⚠️ Bugunlik limitingiz tugadi!")
            return
        try:
            await update.message.reply_text("⏳ 4K rasm yaratilmoqda...")
            img = client.images.generate(
                model="dall-e-3",
                prompt="4K ultra HD " + text,
                size="1024x1024",
                quality="hd",
                n=1
            )
            img_url = img.data[0].url
            await update.message.reply_photo(photo=img_url, caption="🆙 4K Rasm")
            print(f"✅ 4K RASM: {user.id}")
        except Exception as e:
            await update.message.reply_text(f"❌ Xato: {str(e)[:100]}")
            print(f"❌ 4K RASM ERROR: {e}")
        return

    # ✨ TINIQLASHTIRISH
    if mode == "✨ Tiniqlashtirish":
        if not can_use(user.id):
            await update.message.reply_text("⚠️ Bugunlik limitingiz tugadi!")
            return
        try:
            await update.message.reply_text("⏳ Rasm tiniqlanmoqda...")
            img = client.images.generate(
                model="dall-e-3",
                prompt="enhance sharp high quality " + text,
                size="1024x1024",
                quality="hd",
                n=1
            )
            img_url = img.data[0].url
            await update.message.reply_photo(photo=img_url, caption="✨ Tiniqlantirilgan")
            print(f"✅ TINIQLASHTIRISH: {user.id}")
        except Exception as e:
            await update.message.reply_text(f"❌ Xato: {str(e)[:100]}")
            print(f"❌ TINIQLASHTIRISH ERROR: {e}")
        return

    # 💬 CHAT / ✍️ YOZISH
    if mode in ["💬 Chat", "✍️ Yozish"]:
        try:
            await update.message.reply_text("⏳ Javob tayyorlanmoqda...")
            res = client.chat.completions.create(
                model="gpt-3.5-turbo",
                messages=[{"role": "user", "content": text}],
                max_tokens=1000
            )
            answer = res.choices[0].message.content
            await update.message.reply_text(answer)
            print(f"✅ CHAT: {user.id}")
        except Exception as e:
            await update.message.reply_text(f"❌ Xato: {str(e)[:100]}")
            print(f"❌ CHAT ERROR: {e}")
        return

    await update.message.reply_text("📝 Iltimos, avval menyu variantini tanlang!")

# ===== 🖼 RASM TAHLIL =====
async def handle_image(update, context):
    user = update.message.from_user
    
    print(f"📸 IMAGE: {user.id}")

    if not await check_sub(update, context):
        await update.message.reply_text("❌ Avval @HashtagMasterUZ kanaliga obuna bo'ling!")
        return

    if not can_use(user.id):
        await update.message.reply_text("⚠️ Bugunlik limitingiz tugadi!")
        return

    try:
        await update.message.reply_text("⏳ Rasm tahlil qilinmoqda...")
        photo = await update.message.photo[-1].get_file()
        file_bytes = await photo.download_as_bytearray()
        base64_image = base64.b64encode(file_bytes).decode()

        res = client.chat.completions.create(
            model="gpt-4-vision-preview",
            messages=[{
                "role": "user",
                "content": [
                    {"type": "text", "text": "Ushbu rasmni tahlil qil va tushuntir:"},
                    {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{base64_image}"}}
                ]
            }],
            max_tokens=500
        )

        answer = res.choices[0].message.content
        await update.message.reply_text(answer)
        print(f"✅ IMAGE TAHLIL: {user.id}")
    except Exception as e:
        await update.message.reply_text(f"❌ Xato: {str(e)[:100]}")
        print(f"❌ IMAGE ERROR: {e}")

# ===== 🎤 VOICE =====
async def handle_voice(update, context):
    user = update.message.from_user
    
    print(f"🎤 VOICE: {user.id}")

    if not await check_sub(update, context):
        await update.message.reply_text("❌ Avval @HashtagMasterUZ kanaliga obuna bo'ling!")
        return

    if not can_use(user.id):
        await update.message.reply_text("⚠️ Bugunlik limitingiz tugadi!")
        return

    try:
        await update.message.reply_text("⏳ Ovoz qayta ishlanmoqda...")
        voice = await update.message.voice.get_file()
        await voice.download_to_drive("voice.ogg")

        with open("voice.ogg", "rb") as f:
            transcript = client.audio.transcriptions.create(
                model="whisper-1",
                file=f
            )

        res = client.chat.completions.create(
            model="gpt-3.5-turbo",
            messages=[{"role": "user", "content": transcript.text}],
            max_tokens=500
        )

        reply = res.choices[0].message.content
        voice_type = context.user_data.get("voice", "alloy")

        speech = client.audio.speech.create(
            model="tts-1",
            voice=voice_type,
            input=reply
        )

        with open("reply.mp3", "wb") as f:
            f.write(speech.content)

        with open("reply.mp3", "rb") as f:
            await update.message.reply_voice(voice=f)

        if os.path.exists("voice.ogg"):
            os.remove("voice.ogg")
        if os.path.exists("reply.mp3"):
            os.remove("reply.mp3")

        print(f"✅ VOICE: {user.id}")
    except Exception as e:
        await update.message.reply_text(f"❌ Xato: {str(e)[:100]}")
        print(f"❌ VOICE ERROR: {e}")

# ===== ADMIN =====
async def addlimit(update, context):
    if update.message.from_user.id != ADMIN_ID:
        await update.message.reply_text("❌ Admin ruxsati yo'q!")
        return
    
    if len(context.args) < 2:
        await update.message.reply_text("/addlimit <user_id> <amount>")
        return
    
    try:
        uid = int(context.args[0])
        amount = int(context.args[1])
        cursor.execute("UPDATE users SET limit_count=limit_count+? WHERE user_id=?", (amount, uid))
        conn.commit()
        await update.message.reply_text(f"✅ {uid} ga {amount} limit qo'shildi")
        print(f"✅ ADDLIMIT: {uid} +{amount}")
    except Exception as e:
        await update.message.reply_text(f"❌ Xato: {str(e)}")

async def vip(update, context):
    if update.message.from_user.id != ADMIN_ID:
        await update.message.reply_text("❌ Admin ruxsati yo'q!")
        return
    
    if len(context.args) < 1:
        await update.message.reply_text("/vip <user_id>")
        return
    
    try:
        uid = int(context.args[0])
        expire = datetime.datetime.now() + datetime.timedelta(days=30)
        cursor.execute("UPDATE users SET vip=1, vip_expire=? WHERE user_id=?", (str(expire), uid))
        conn.commit()
        await update.message.reply_text(f"✅ {uid} ga 30 kun VIP berildi")
        print(f"✅ VIP: {uid} +30 days")
    except Exception as e:
        await update.message.reply_text(f"❌ Xato: {str(e)}")

async def stats(update, context):
    if update.message.from_user.id != ADMIN_ID:
        return
    
    try:
        cursor.execute("SELECT COUNT(*) FROM users")
        users_count = cursor.fetchone()[0]
        cursor.execute("SELECT COUNT(*) FROM users WHERE vip=1")
        vip_count = cursor.fetchone()[0]
        
        msg = f"""
📊 BOT STATISTIKASI:
👥 Jami users: {users_count}
👑 VIP users: {vip_count}
📅 Bugun: {datetime.date.today()}
        """
        await update.message.reply_text(msg)
    except Exception as e:
        await update.message.reply_text(f"❌ Xato: {str(e)}")

# ===== RUN =====
if __name__ == "__main__":
    print("=" * 50)
    print("🤖 BOT ISHGA TUSHMOQDA...")
    print("=" * 50)
    
    try:
        app = ApplicationBuilder().token(TOKEN).build()
        
        print("✅ Handlers qo'shilmoqda...")
        app.add_handler(CommandHandler("start", start))
        app.add_handler(CommandHandler("addlimit", addlimit))
        app.add_handler(CommandHandler("vip", vip))
        app.add_handler(CommandHandler("stats", stats))

        app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_text))
        app.add_handler(MessageHandler(filters.PHOTO, handle_image))
        app.add_handler(MessageHandler(filters.VOICE, handle_voice))
        
        print("=" * 50)
        print("✅ BOT MUVAFFAQIYATLI ISHGA TUSHDI!")
        print("=" * 50)
        print("🎯 Telegramda /start yuboring")
        print("=" * 50)
        
        app.run_polling()
    except Exception as e:
        print(f"❌ BOT XATOSI: {e}")
        import traceback
        traceback.print_exc()
        
