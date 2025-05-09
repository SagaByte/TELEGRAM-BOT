import telebot
import json # سيبقى للاستخدام الداخلي إذا لزم الأمر، ولكن ليس للحفظ في ملف
from telebot.types import InlineKeyboardMarkup, InlineKeyboardButton
import os

# يمنع تغيير الحقوق مـايكي>>
#https://t.me/SSUU_R

TOKEN = os.environ.get('TELEGRAM_BOT_TOKEN')
ADMIN_ID = int(os.environ.get('TELEGRAM_ADMIN_ID', 7261987706)) # القيمة الافتراضية إذا لم يتم تعيين متغير البيئة

if not TOKEN:
    raise ValueError("لم يتم تعيين TELEGRAM_BOT_TOKEN كمتغير بيئة.")

bot = telebot.TeleBot(TOKEN)

sessions = {} # لتتبع الرسائل للردود
banned_users = set()
admins = {ADMIN_ID}

# بيانات المستخدمين ستكون في الذاكرة فقط
users_data_in_memory = {} # تم تغيير الاسم للإشارة إلى أنها في الذاكرة

forced_channel = None
forced_subscription = False
communication_enabled = True

# لا حاجة لدوال save_users_data() و load_users_data()

@bot.message_handler(commands=['start'])
def start_cmd(m):
    u_id = m.from_user.id
    # إضافة المستخدم إلى القائمة الموجودة في الذاكرة لهذه الجلسة
    if str(u_id) not in users_data_in_memory:
        users_data_in_memory[str(u_id)] = {
            'first_name': m.from_user.first_name,
            'username': m.from_user.username,
            'id': u_id
        }
        # لا يوجد حفظ في ملف هنا

    if forced_subscription and forced_channel:
        try:
            member = bot.get_chat_member(forced_channel, u_id)
            if member.status not in ['member', 'administrator', 'creator']:
                bot.send_message(u_id, f"الرجاء الاشتراك في القناة أولاً: {forced_channel}")
                return
        except Exception as e:
            print(f"Error checking channel membership for {u_id} in {forced_channel}: {e}")
            # يمكنك إرسال رسالة للمستخدم هنا إذا أردت إعلامه بوجود مشكلة في التحقق

    bot.send_message(u_id, "أهلاً بك عزيزي في بوت التواصل ارسل رسالتك ليتم ارسالها للمالك .")

@bot.message_handler(commands=['admin'])
def admin_cmd(m):
    if m.chat.id not in admins:
        bot.send_message(m.chat.id, "ليس لديك صلاحيات الوصول إلى لوحة التحكم.")
        return

    markup = InlineKeyboardMarkup()
    markup.row_width = 2
    markup.add(
        InlineKeyboardButton("🚫 حظر", callback_data='ban'),
        InlineKeyboardButton("✅ فك حظر", callback_data='unban'),
        InlineKeyboardButton("➕ إضافة أدمن", callback_data='add_admin'),
        InlineKeyboardButton("➖ حذف أدمن", callback_data='remove_admin'),
        InlineKeyboardButton("📢 إذاعة", callback_data='broadcast'),
        InlineKeyboardButton("📊 إحصائيات (جلسة حالية)", callback_data='stats'), # تم تحديث النص
        InlineKeyboardButton("🔒 تفعيل الاشتراك الإجباري", callback_data='enable_forced_sub'),
        InlineKeyboardButton("🔓 إيقاف الاشتراك الإجباري", callback_data='disable_forced_sub'),
        InlineKeyboardButton("➕ إضافة قناة اشتراك", callback_data='add_channel'),
        InlineKeyboardButton("📬 تفعيل التواصل", callback_data='enable_communication'),
        InlineKeyboardButton("📴 إيقاف التواصل", callback_data='disable_communication')
    )
    # تمت إزالة زر "عرض بيانات المستخدمين"
    bot.send_message(m.chat.id, "اهلا بك عزيزي المالك في لوحة التحكم الخاصه بك ", reply_markup=markup)

@bot.callback_query_handler(func=lambda call: True)
def callback_query(call):
    if call.message.chat.id not in admins:
        bot.answer_callback_query(call.id, "ليس لديك صلاحيات.")
        return

    action = call.data
    chat_id = call.message.chat.id

    if action == 'ban':
        msg = bot.send_message(chat_id, "أرسل آيدي المستخدم لحظره:")
        bot.register_next_step_handler(msg, ban_user)
    elif action == 'unban':
        msg = bot.send_message(chat_id, "أرسل آيدي المستخدم لفك حظره:")
        bot.register_next_step_handler(msg, unban_user)
    elif action == 'add_admin':
        msg = bot.send_message(chat_id, "أرسل آيدي المستخدم لإضافته كأدمن:")
        bot.register_next_step_handler(msg, add_admin_user)
    elif action == 'remove_admin':
        msg = bot.send_message(chat_id, "أرسل آيدي المستخدم لحذفه من الأدمن:")
        bot.register_next_step_handler(msg, remove_admin_user)
    elif action == 'broadcast':
        msg = bot.send_message(chat_id, "أرسل الرسالة التي تريد إذاعتها:")
        bot.register_next_step_handler(msg, broadcast_message_to_users) # تم تغيير اسم الدالة
    elif action == 'stats':
        show_current_session_stats(chat_id) # تم تغيير اسم الدالة
    elif action == 'enable_forced_sub':
        global forced_subscription
        forced_subscription = True
        bot.send_message(chat_id, "تم تفعيل الاشتراك الإجباري.")
    elif action == 'disable_forced_sub':
        forced_subscription = False
        bot.send_message(chat_id, "تم إيقاف الاشتراك الإجباري.")
    elif action == 'add_channel':
        msg = bot.send_message(chat_id, "أرسل معرف القناة (@username أو ID) لإضافتها:")
        bot.register_next_step_handler(msg, set_forced_channel_id) # تم تغيير اسم الدالة
    elif action == 'enable_communication':
        global communication_enabled
        communication_enabled = True
        bot.send_message(chat_id, "تم تفعيل التواصل.")
    elif action == 'disable_communication':
        communication_enabled = False
        bot.send_message(chat_id, "تم إيقاف التواصل.")
    
    bot.answer_callback_query(call.id)


def ban_user(m):
    if m.chat.id not in admins: return
    u = m.text.strip()
    try:
        user_id_to_ban = int(u)
        banned_users.add(user_id_to_ban)
        bot.send_message(m.chat.id, f"تم حظر المستخدم صاحب الآيدي: {user_id_to_ban}.")
    except ValueError:
        bot.send_message(m.chat.id, "يرجى إرسال آيدي رقمي للمستخدم لحظره.")

def unban_user(m):
    if m.chat.id not in admins: return
    u = m.text.strip()
    try:
        user_id_to_unban = int(u)
        banned_users.discard(user_id_to_unban)
        bot.send_message(m.chat.id, f"تم فك حظر المستخدم صاحب الآيدي: {user_id_to_unban}.")
    except ValueError:
        bot.send_message(m.chat.id, "يرجى إرسال آيدي رقمي للمستخدم لفك حظره.")

def add_admin_user(m):
    if m.chat.id not in admins: return
    u = m.text.strip()
    try:
        admin_to_add = int(u)
        admins.add(admin_to_add)
        bot.send_message(m.chat.id, f"تمت إضافة الأدمن صاحب الآيدي: {admin_to_add} بنجاح.")
    except ValueError:
        bot.send_message(m.chat.id, "تعذر إضافة الأدمن. يرجى إرسال آيدي رقمي صحيح.")
    except Exception as e:
        bot.send_message(m.chat.id, f"حدث خطأ: {e}")

def remove_admin_user(m):
    if m.chat.id not in admins: return
    u = m.text.strip()
    try:
        admin_to_remove = int(u)
        if admin_to_remove == ADMIN_ID:
             bot.send_message(m.chat.id, "لا يمكن حذف الأدمن الأساسي.")
             return
        admins.discard(admin_to_remove)
        bot.send_message(m.chat.id, f"تمت إزالة الأدمن صاحب الآيدي: {admin_to_remove}.")
    except ValueError:
        bot.send_message(m.chat.id, "تعذر إزالة الأدمن. يرجى إرسال آيدي رقمي صحيح.")
    except Exception as e:
        bot.send_message(m.chat.id, f"حدث خطأ: {e}")

def broadcast_message_to_users(m): # تم تغيير الاسم
    if m.chat.id not in admins: return
    msg_text = m.text.strip()
    if not msg_text:
        bot.send_message(m.chat.id, "لا يمكن إرسال رسالة فارغة.")
        return
    
    count = 0
    failed_users_ids = []
    # الإذاعة للمستخدمين الموجودين في الذاكرة فقط
    for u_id_str in list(users_data_in_memory.keys()):
        try:
            bot.send_message(int(u_id_str), msg_text)
            count +=1
        except Exception as e:
            print(f"Failed to send broadcast to {u_id_str}: {e}")
            failed_users_ids.append(u_id_str)
    
    bot.send_message(m.chat.id, f"تم إرسال الرسالة إلى {count} مستخدمًا (من الجلسة الحالية).")
    if failed_users_ids:
        bot.send_message(m.chat.id, f"فشل الإرسال إلى (IDs): {', '.join(failed_users_ids)}")

def show_current_session_stats(chat_id): # تم تغيير الاسم
    num_users_current_session = len(users_data_in_memory)
    stats_message = f"عدد المستخدمين في الجلسة الحالية: {num_users_current_session}\n"
    
    if num_users_current_session > 0:
        stats_message += "مستخدمو الجلسة الحالية:\n"
        display_count = 0
        for u_id_str, u_info in users_data_in_memory.items():
            if display_count < 20: # عرض أول 20 كحد أقصى
                stats_message += f"الاسم: {u_info.get('first_name', 'N/A')}, المعرف: @{u_info.get('username', 'N/A')}, آيدي: {u_info.get('id', 'N/A')}\n"
                display_count += 1
            else:
                stats_message += "والمزيد...\n"
                break
    else:
        stats_message += "لم يتفاعل أي مستخدم مع البوت في هذه الجلسة بعد."
        
    bot.send_message(chat_id, stats_message)

def set_forced_channel_id(m): # تم تغيير الاسم
    if m.chat.id not in admins: return
    global forced_channel
    channel_id_text = m.text.strip()
    if not channel_id_text:
        bot.send_message(m.chat.id, "لم يتم إرسال معرف القناة.")
        return
    
    try:
        chat_info = bot.get_chat(channel_id_text)
        forced_channel = channel_id_text
        bot.send_message(m.chat.id, f"تم تعيين قناة الاشتراك الإجباري: {forced_channel} (اسمها: {chat_info.title})")
        bot.send_message(m.chat.id, "تأكد من أن البوت مشرف في هذه القناة ليتمكن من التحقق من عضوية المستخدمين.")
    except telebot.apihelper.ApiTelegramException as e:
        if "chat not found" in e.description:
            bot.send_message(m.chat.id, "لم يتم العثور على القناة. تأكد من صحة المعرف وأن البوت عضو فيها.")
        elif "bot is not a member" in e.description:
             bot.send_message(m.chat.id, "البوت ليس عضواً في هذه القناة. يرجى إضافته أولاً.")
        else:
            bot.send_message(m.chat.id, f"تعذر إضافة القناة. خطأ: {e.description}.")
    except Exception as e:
        bot.send_message(m.chat.id, f"خطأ غير متوقع: {e}")
 # يمنع تغيير الحقوق مـايكي>>
 #https://t.me/SSUU_R
@bot.message_handler(func=lambda m: True, content_types=['text', 'photo', 'video', 'document', 'audio', 'voice', 'sticker'])
def handle_message(m):
    u_id = m.chat.id
    user_info = m.from_user

    if u_id in banned_users or (isinstance(user_info.username, str) and user_info.username in banned_users):
        print(f"Blocked message from banned user: {u_id} or @{user_info.username}")
        return

    if not communication_enabled:
        is_admin_message = any(u_id == admin_id_val for admin_id_val in admins)
        if not is_admin_message:
            try:
                admin_user = bot.get_chat(ADMIN_ID) # استخدام ADMIN_ID المعرف عالمياً
                bot.send_message(u_id, f"تم إيقاف استلام الرسائل من قبل المالك. @{admin_user.username}")
            except Exception as e:
                print(f"Error getting admin username: {e}")
                bot.send_message(u_id, "تم إيقاف استلام الرسائل من قبل المالك.")
            return
    
    # إضافة المستخدم إلى القائمة الموجودة في الذاكرة إذا لم يكن موجودًا
    if str(u_id) not in users_data_in_memory:
        users_data_in_memory[str(u_id)] = {
            'first_name': user_info.first_name,
            'username': user_info.username,
            'id': u_id
        }
        # لا يوجد حفظ في ملف هنا
    
    is_sender_admin = any(u_id == admin_id_val for admin_id_val in admins)

    if not is_sender_admin:
         if forced_subscription and forced_channel:
             try:
                 member = bot.get_chat_member(forced_channel, u_id)
                 if member.status not in ['member', 'administrator', 'creator']:
                     bot.send_message(u_id, f"عذراً، يجب عليك الاشتراك في القناة أولاً لمتابعة استخدام البوت: {forced_channel}")
                     return
             except Exception as e:
                 print(f"Error checking channel subscription for {u_id} in handle_message: {e}")
        
         forward_message_text = f"رسالة جديدة من المستخدم: {user_info.first_name} (@{user_info.username}, ID: {u_id})\n"
         for admin_id_val in admins:
             try:
                 bot.send_message(admin_id_val, forward_message_text)
                 forwarded_msg = bot.forward_message(admin_id_val, u_id, m.message_id)
                 sessions[forwarded_msg.message_id] = u_id
             except Exception as e:
                 print(f"Failed to forward message to admin {admin_id_val}: {e}")
         bot.send_message(u_id, "تم إرسال رسالتك إلى المالك. انتظر الرد منه.")

    else: # إذا كان المرسل هو أحد الأدمنز
        if m.reply_to_message:
            original_sender_id = sessions.get(m.reply_to_message.message_id)
            if original_sender_id:
                try:
                    # إعادة توجيه الرد بكافة أنواعه
                    bot.copy_message(original_sender_id, m.chat.id, m.message_id)
                    bot.send_message(m.chat.id, "تم إرسال ردك للمستخدم.")
                except Exception as e:
                    bot.send_message(m.chat.id, f"فشل إرسال الرد للمستخدم {original_sender_id}. الخطأ: {e}")
                    print(f"Error sending reply to {original_sender_id}: {e}")
            else:
                bot.send_message(m.chat.id, "تعذر العثور على المستخدم الأصلي لهذه الرسالة (ربما تم الرد على رسالة قديمة أو تم إعادة تشغيل البوت).")



def main():
    print("Bot is starting with polling (no persistent storage)...")
    bot.infinity_polling(skip_pending=True)

if __name__ == "__main__":
    main()
