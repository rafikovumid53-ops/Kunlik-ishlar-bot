
# Kunlik Ishlar Toshkent - TO'G'RI MODEL
# E'lon berish BEPUL, Kontakt (telefon) ko'rish PULLIK
# Kutubxona: pip install pyTelegramBotAPI

import telebot
from telebot.types import InlineKeyboardMarkup, InlineKeyboardButton
import sqlite3

# ============ SOZLAMALAR ============
TOKEN = "BU_YERGA_YANGI_TOKENNI_QOY"
ADMIN_ID = 8900786771
VISA_KARTA = "4022 XXXX XXXX XXXX"
KARTA_EGASI = "ISMING"
KONTAKT_NARXI = "15 000 so'm"
# ====================================

bot = telebot.TeleBot(TOKEN)

conn = sqlite3.connect("bot.db", check_same_thread=False)
cur = conn.cursor()
cur.execute("CREATE TABLE IF NOT EXISTS elonlar (id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, matn TEXT, telefon TEXT, status TEXT)")
conn.commit()

user_data = {}

def asosiy_menu():
    kb = InlineKeyboardMarkup()
    kb.add(InlineKeyboardButton("📋 Ishlarni ko'rish", callback_data="ishlar"))
    kb.add(InlineKeyboardButton("📢 E'lon berish - BEPUL", callback_data="elon"))
    kb.add(InlineKeyboardButton("📞 Kontakt olish - 15 000 so'm", callback_data="kontakt"))
    kb.add(InlineKeyboardButton("🆘 Yordam", callback_data="yordam"))
    return kb

@bot.message_handler(commands=['start'])
def start(m):
    bot.send_message(m.chat.id,
        f"Assalomu alaykum, {m.from_user.first_name}! 👋\n\n"
        f"Bu - Kunlik Ishlar Toshkent boti.\n\n"
        f"📢 E'lon berish: BEPUL\n"
        f"📞 Ish beruvchi telefonini ko'rish: {KONTAKT_NARXI}\n\n"
        f"Ishlar ro'yxatida telefon ko'rinmaydi. Telefonni ko'rish uchun to'lov qilasiz, chek yuborasiz, admin tasdiqlaydi.",
        reply_markup=asosiy_menu())

@bot.callback_query_handler(func=lambda c: c.data=="ishlar")
def ishlar(c):
    cur.execute("SELECT id, matn FROM elonlar WHERE status='tasdiqlandi' ORDER BY id DESC LIMIT 10")
    rows = cur.fetchall()
    if not rows:
        bot.send_message(c.message.chat.id, "Hozircha e'lon yo'q.")
    else:
        txt = "📋 So'nggi ishlar (telefon yopiq):\n\n"
        for r in rows:
            txt += f"ID {r[0]}:\n{r[1]}\nTelefonni ko'rish: /kontakt_{r[0]}\n\n---\n\n"
        bot.send_message(c.message.chat.id, txt)
    bot.answer_callback_query(c.id)

@bot.callback_query_handler(func=lambda c: c.data=="elon")
def elon_start(c):
    bot.send_message(c.message.chat.id, "E'lon matnini yozing.\n\nMisol:\nIsh: Yuk tashish kerak\nManzil: Chilonzor\nVaqt: Ertaga 9:00")
    user_data[c.from_user.id] = {'step': 'elon_matn'}
    bot.answer_callback_query(c.id)

@bot.callback_query_handler(func=lambda c: c.data=="kontakt")
def kontakt_info(c):
    bot.send_message(c.message.chat.id,
        f"Kontakt olish uchun:\n\n1. Avval Ishlarni ko'ring, yoqqan ishning ID sini oling. Masalan ID 3\n2. Shu yerga yozing: /kontakt_3\n3. Bot sizga to'lov uchun karta beradi.")
    bot.answer_callback_query(c.id)

@bot.message_handler(commands=['kontakt'])
def kontakt_cmd(m):
    bot.send_message(m.chat.id, "Qaysi ishning kontaktini olmoqchisiz? Masalan: /kontakt_3 deb yozing.")

@bot.message_handler(func=lambda m: m.text and m.text.startswith('/kontakt_'))
def kontakt_talab(m):
    try:
        elon_id = int(m.text.split('_')[1])
    except:
        bot.send_message(m.chat.id, "Xato. Masalan: /kontakt_3")
        return
    cur.execute("SELECT matn FROM elonlar WHERE id=? AND status='tasdiqlandi'", (elon_id,))
    row = cur.fetchone()
    if not row:
        bot.send_message(m.chat.id, "Bunday ID li e'lon topilmadi.")
        return
    user_data[m.from_user.id] = {'step': 'kontakt_chek', 'elon_id': elon_id}
    bot.send_message(m.chat.id,
        f"ID {elon_id} ishning telefonini olish uchun to'lov qiling:\n\n"
        f"💳 Visa: {VISA_KARTA}\n"
        f"👤 Egasi: {KARTA_EGASI}\n"
        f"💵 Summa: {KONTAKT_NARXI}\n\n"
        f"Pulni o'tkazib, chek rasmini shu yerga yuboring.")

@bot.message_handler(content_types=['text'])
def matnlar(m):
    uid = m.from_user.id
    if uid not in user_data:
        return
    step = user_data[uid].get('step')
    if step == 'elon_matn':
        user_data[uid]['matn'] = m.text
        user_data[uid]['step'] = 'elon_tel'
        bot.send_message(m.chat.id, "Endi telefon raqamingizni yozing. Bu raqam faqat to'lov qilganlarga ko'rinadi.\nMasalan: +99890 123-45-67")
    elif step == 'elon_tel':
        matn = user_data[uid]['matn']
        tel = m.text
        cur.execute("INSERT INTO elonlar (user_id, matn, telefon, status) VALUES (?, ?, ?, 'kutilmoqda')", (uid, matn, tel))
        elon_id = cur.lastrowid
        conn.commit()
        bot.send_photo(ADMIN_ID, m.text) if False else None
        bot.send_message(ADMIN_ID,
            f"🆕 Yangi BEPUL e'lon!\n\nID: {elon_id}\nFoydalanuvchi: @{m.from_user.username} ({uid})\nMatn:\n{matn}\nTelefon: {tel}",
            reply_markup=InlineKeyboardMarkup().add(
                InlineKeyboardButton("✅ Tasdiqlash", callback_data=f"tasdiq_{elon_id}"),
                InlineKeyboardButton("❌ Rad etish", callback_data=f"rad_{elon_id}")
            ))
        bot.send_message(m.chat.id, "✅ E'loningiz qabul qilindi! Admin tasdiqlagach ro'yxatga chiqadi. E'lon berish BEPUL.")
        del user_data[uid]

@bot.message_handler(content_types=['photo'])
def chek(m):
    uid = m.from_user.id
    if uid not in user_data or user_data[uid].get('step') != 'kontakt_chek':
        return
    elon_id = user_data[uid]['elon_id']
    bot.send_photo(ADMIN_ID, m.photo[-1].file_id,
        caption=f"💰 Kontakt uchun to'lov!\n\nE'lon ID: {elon_id}\nFoydalanuvchi: @{m.from_user.username} ({uid})",
        reply_markup=InlineKeyboardMarkup().add(
            InlineKeyboardButton("✅ Tasdiqlash - telefonni ber", callback_data=f"kontakt_tasdiq_{elon_id}_{uid}"),
            InlineKeyboardButton("❌ Rad etish", callback_data=f"kontakt_rad_{uid}")
        ))
    bot.send_message(m.chat.id, "✅ Chek qabul qilindi! Admin tasdiqlagach telefon raqam yuboriladi.")
    del user_data[uid]

@bot.callback_query_handler(func=lambda c: c.data.startswith('tasdiq_'))
def tasdiq(c):
    elon_id = int(c.data.split('_')[1])
    cur.execute("UPDATE elonlar SET status='tasdiqlandi' WHERE id=?", (elon_id,))
    cur.execute("SELECT user_id FROM elonlar WHERE id=?", (elon_id,))
    user_id = cur.fetchone()[0]
    conn.commit()
    bot.send_message(user_id, f"🎉 E'loningiz (ID {elon_id}) tasdiqlandi va ro'yxatga chiqdi! BEPUL.")
    bot.answer_callback_query(c.id, "Tasdiqlandi")

@bot.callback_query_handler(func=lambda c: c.data.startswith('rad_'))
def rad(c):
    elon_id = int(c.data.split('_')[1])
    cur.execute("UPDATE elonlar SET status='rad' WHERE id=?", (elon_id,))
    cur.execute("SELECT user_id FROM elonlar WHERE id=?", (elon_id,))
    user_id = cur.fetchone()[0]
    conn.commit()
    bot.send_message(user_id, "❌ E'loningiz rad etildi.")
    bot.answer_callback_query(c.id, "Rad etildi")

@bot.callback_query_handler(func=lambda c: c.data.startswith('kontakt_tasdiq_'))
def kontakt_tasdiq(c):
    parts = c.data.split('_')
    elon_id = int(parts[2]); uid = int(parts[3])
    cur.execute("SELECT telefon, matn FROM elonlar WHERE id=?", (elon_id,))
    row = cur.fetchone()
    if row:
        bot.send_message(uid, f"🎉 To'lovingiz tasdiqlandi!\n\nID {elon_id} ishning telefoni:\n📞 {row[0]}\n\nIsh matni:\n{row[1]}")
    bot.answer_callback_query(c.id, "Telefon yuborildi")

@bot.callback_query_handler(func=lambda c: c.data.startswith('kontakt_rad_'))
def kontakt_rad(c):
    uid = int(c.data.split('_')[2])
    bot.send_message(uid, "❌ To'lovingiz tasdiqlanmadi. Adminga yozing.")
    bot.answer_callback_query(c.id, "Rad etildi")

@bot.callback_query_handler(func=lambda c: c.data=="yordam")
def yordam(c):
    bot.send_message(c.message.chat.id, "Savol bo'lsa adminga yozing: @aristokrat6561")
    bot.answer_callback_query(c.id)

print("Bot ishga tushdi...")
bot.infinity_polling()
