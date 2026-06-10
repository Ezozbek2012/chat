# chat
mening 1- loyiham
"""
Telegram AI Agent Bot
- Shaxsiy chat: har xabarga javob beradi
- Gurux: har xabarga javob beradi
- O'zbekcha, Ruscha, Inglizcha
"""

import logging
from anthropic import Anthropic
from telegram import Update
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    MessageHandler,
    ContextTypes,
    filters,
)

# ─── Sozlamalar ───────────────────────────────────────────────────────────────
TELEGRAM_TOKEN = "TELEGRAM_BOT_TOKEN_NI_SHU_YERGA_QOYING"
ANTHROPIC_API_KEY = "ANTHROPIC_API_KEY_NI_SHU_YERGA_QOYING"

SYSTEM_PROMPT = """You are a helpful AI assistant. You can communicate fluently in Uzbek, Russian, and English.
Always detect the language of the user's message and respond in the same language.
Be concise, clear, and helpful.
"""

# ─── Logging ──────────────────────────────────────────────────────────────────
logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    level=logging.INFO,
)
logger = logging.getLogger(__name__)

# ─── Anthropic client ─────────────────────────────────────────────────────────
client = Anthropic(api_key=ANTHROPIC_API_KEY)

# Har bir chat (shaxsiy yoki gurux) uchun tarix
conversation_history: dict[int, list] = {}


# ─── /start ───────────────────────────────────────────────────────────────────
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat_id = update.effective_chat.id
    conversation_history[chat_id] = []
    await update.message.reply_text(
        "👋 Salom! Men AI agentman. Har bir xabarga javob beraman!\n"
        "🇷🇺 Привет! Отвечаю на каждое сообщение.\n"
        "🇬🇧 Hello! I reply to every message.\n\n"
        "➡️ /reset — suhbatni tozalash"
    )


# ─── /reset ───────────────────────────────────────────────────────────────────
async def reset(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat_id = update.effective_chat.id
    conversation_history[chat_id] = []
    await update.message.reply_text("🔄 Suhbat tozalandi!")


# ─── Xabarni qayta ishlash ────────────────────────────────────────────────────
async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat_id = update.effective_chat.id
    user = update.effective_user
    user_text = update.message.text

    # Guruxda kim yozganini ko'rsatish uchun
    is_group = update.effective_chat.type in ("group", "supergroup")
    if is_group:
        display_name = user.first_name or user.username or "Foydalanuvchi"
        content = f"{display_name}: {user_text}"
    else:
        content = user_text

    if chat_id not in conversation_history:
        conversation_history[chat_id] = []

    conversation_history[chat_id].append({
        "role": "user",
        "content": content,
    })

    await update.message.chat.send_action("typing")

    try:
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            system=SYSTEM_PROMPT,
            messages=conversation_history[chat_id],
        )

        assistant_reply = response.content[0].text

        conversation_history[chat_id].append({
            "role": "assistant",
            "content": assistant_reply,
        })

        # Tarixni 40 xabar bilan cheklash (gurux uchun ko'proq)
        if len(conversation_history[chat_id]) > 40:
            conversation_history[chat_id] = conversation_history[chat_id][-40:]

        await update.message.reply_text(assistant_reply)

    except Exception as e:
        logger.error(f"Xato: {e}")
        await update.message.reply_text("⚠️ Xatolik yuz berdi. Qayta urinib ko'ring.")


# ─── Ishga tushirish ──────────────────────────────────────────────────────────
def main():
    app = ApplicationBuilder().token(TELEGRAM_TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("reset", reset))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))

    logger.info("Bot ishga tushdi...")
    app.run_polling()


if __name__ == "__main__":
    main()
