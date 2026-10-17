# tbd
"""
Techsub International Shop Bot — Telegram shop bot with manual USDT orders.
Setup: copy .env.example to .env, fill BOT_TOKEN / ADMIN_ID / USDT_ADDRESS, run.
"""
import logging
import os

from dotenv import load_dotenv
from telegram import InlineKeyboardButton, InlineKeyboardMarkup, Update
from telegram.ext import (
    Application,
    CallbackQueryHandler,
    CommandHandler,
    ContextTypes,
    ConversationHandler,
    MessageHandler,
    filters,
)

load_dotenv()

BOT_TOKEN = os.getenv("BOT_TOKEN", "")
ADMIN_ID = int(os.getenv("ADMIN_ID", "0"))
USDT_ADDRESS = os.getenv("USDT_ADDRESS", "")
SUPPORT_USER = os.getenv("SUPPORT_USER", "")

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

PRODUCTS = {
    "Muse": {
        "name": "Muse Shared — 1 Month",
        "price": "$5.99",
        "desc": "Muse (Anthropic) shared subscription.\n"
                "Full chat access for 1 month.\n"
                "12-hour replacement warranty.",
    },
    "gemini": {
        "name": "Google AI Pro Shared — 1 Month",
        "price": "$3.99",
        "desc": "Google AI Pro shared slot (family invitation).\n"
                "Gemini Advanced access for 1 month.\n"
                "12-hour replacement warranty.",
    },
    "claude_api": {
        "name": "Claude API — $100 Credits",
        "price": "$9.99",
        "desc": "$100 Claude API credits.\n"
                "API key delivered after payment.\n"
                "Beginner usage guide included.",
    },
    "flow": {
        "name": "Google Flow + Omni Combo",
        "price": "$4.49",
        "desc": "Google Flow + Google Omni combo.\n"
                "1050 credits included.\n"
                "AI video creation made easy.",
    },
}

ASK_TXID = 1


def catalog_keyboard() -> InlineKeyboardMarkup:
    rows = [
        [InlineKeyboardButton(f"🛒 {p['name']} — {p['price']}", callback_data=f"buy:{key}")]
        for key, p in PRODUCTS.items()
    ]
    rows.append([InlineKeyboardButton("💬 Support", callback_data="support")])
    return InlineKeyboardMarkup(rows)


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = (
        "🤖 *Welcome to Techsub International Shop!*\n\n"
        "AI subscriptions with instant delivery\\.\n"
        "💳 Payment: *USDT \\(TRC\\-20\\)*\n"
        "⚡ Delivery after payment confirmation\n"
        "🛡️ 12\\-hour replacement warranty\n\n"
        "Choose a product below 👇"
    )
    if update.message:
        await update.message.reply_text(text, reply_markup=catalog_keyboard(), parse_mode="MarkdownV2")
    elif update.callback_query:
        await update.callback_query.edit_message_text(text, reply_markup=catalog_keyboard(), parse_mode="MarkdownV2")


async def on_button(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    data = query.data

    if data == "support":
        await query.edit_message_text(
            f"💬 Need help\\? Contact support: {SUPPORT_USER}\n\nWe reply within a few hours\\.",
            reply_markup=InlineKeyboardMarkup(
                [[InlineKeyboardButton("⬅️ Back to shop", callback_data="back")]]
            ),
            parse_mode="MarkdownV2",
        )
        return

    if data == "back":
        await start(update, context)
        return

    if data.startswith("buy:"):
        key = data.split(":", 1)[1]
        product = PRODUCTS.get(key)
        if not product:
            return
        context.user_data["product"] = key
        text = (
            f"🛒 *{product['name']}*\n"
            f"💰 Price: *{product['price']}*\n\n"
            f"{product['desc']}\n\n"
            f"💳 Send exactly *{product['price']}* USDT \\(TRC\\-20\\) to:\n"
            f"`{USDT_ADDRESS}`\n\n"
            "After sending, tap *✅ I've Paid* and send your transaction ID \\(TXID\\)\\."
        )
        await query.edit_message_text(
            text,
            reply_markup=InlineKeyboardMarkup(
                [
                    [InlineKeyboardButton("✅ I've Paid", callback_data="paid")],
                    [InlineKeyboardButton("⬅️ Back to shop", callback_data="back")],
                ]
            ),
            parse_mode="MarkdownV2",
        )


async def paid_pressed(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    await query.edit_message_text(
        "📝 Please send your *USDT transaction ID* \\(TXID / hash\\) as a message\\.\n\n"
        "You can find it in your wallet's transaction history\\.",
        parse_mode="MarkdownV2",
    )
    return ASK_TXID


async def txid_received(update: Update, context: ContextTypes.DEFAULT_TYPE):
    txid = update.message.text.strip()
    key = context.user_data.get("product", "?")
    product = PRODUCTS.get(key, {"name": key, "price": "?"})
    user = update.effective_user

    # Notify admin
    admin_text = (
        "💰 *New order!*\n\n"
        f"Product: {product['name']}\n"
        f"Price: {product['price']}\n"
        f"Buyer: @{user.username or 'no\\_username'} \\(ID: {user.id}\\)\n"
        f"TXID: `{txid}`\n\n"
        "Verify the payment, then deliver the credentials to the buyer\\."
    )
    try:
        await context.bot.send_message(chat_id=ADMIN_ID, text=admin_text, parse_mode="MarkdownV2")
    except Exception as e:
        logger.error("Could not notify admin: %s", e)

    await update.message.reply_text(
        "✅ *TXID received!*\n\n"
        "We're verifying your payment now\\.\n"
        "Your login details will arrive here within a few hours\\.\n\n"
        "Thank you for shopping with Techsub! 🙏",
        parse_mode="MarkdownV2",
        reply_markup=catalog_keyboard(),
    )
    context.user_data.pop("product", None)
    return ConversationHandler.END


async def cancel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Cancelled. Use /start to shop again. 🛒")
    context.user_data.clear()
    return ConversationHandler.END


def main():
    if not BOT_TOKEN or not ADMIN_ID or not USDT_ADDRESS:
        raise SystemExit("Fill BOT_TOKEN, ADMIN_ID and USDT_ADDRESS in .env first.")

    app = Application.builder().token(BOT_TOKEN).build()

    order_flow = ConversationHandler(
        entry_points=[CallbackQueryHandler(paid_pressed, pattern="^paid$")],
        states={ASK_TXID: [MessageHandler(filters.TEXT & ~filters.COMMAND, txid_received)]},
        fallbacks=[CommandHandler("cancel", cancel)],
        per_message=False,
    )

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CallbackQueryHandler(on_button, pattern="^(buy:|support|back)"))
    app.add_handler(order_flow)

    logger.info("Bot is running…")
    app.run_polling()


if __name__ == "__main__":
    main()
