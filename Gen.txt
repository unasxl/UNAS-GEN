import telebot
import random
from telebot.types import InlineKeyboardMarkup, InlineKeyboardButton, InputFile
import io
import requests

token = '8031553250:AAHEczFhkRWWBD7hUeRxYdqwaY74M5ZUSwM'
bot = telebot.TeleBot(token, parse_mode="HTML")

CARD_LENGTH = 16

def generate_cards(bin_number, amount=10, exp_month=None, exp_year=None):
    cards = []
    for _ in range(amount):
        card_number = bin_number + ''.join(random.choices('0123456789', k=CARD_LENGTH - len(bin_number)))
        month = exp_month if exp_month else random.randint(1, 12)
        year = exp_year if exp_year else random.randint(25, 32)
        cvv = ''.join(random.choices('0123456789', k=3))

        # Format the card details
        card = f"{card_number}|{month:02d}|20{year}|{cvv}"
        cards.append(card)
    return cards

def validate_bin(bin_number):
    try:
        response = requests.get(f'https://bins.antipublic.cc/bins/{bin_number}')
        if response.status_code == 200:
            return True
        else:
            return False
    except requests.RequestException as e:
        print(f"Error validating BIN: {e}")
        return False

def parse_message(input_text):
    parts = input_text.split()
    bin_number = parts[0].replace('x', '') if parts else ''

    exp_month, exp_year = None, None
    if len(parts) > 1 and '/' in parts[1]:  # Check for MM/YY format
        month_year = parts[1].split('/')
        if len(month_year) == 2 and month_year[0].isdigit() and month_year[1].isdigit():
            exp_month = int(month_year[0])
            exp_year = int(month_year[1])

    amount = int(parts[2]) if len(parts) > 2 and parts[2].isdigit() else 10
    return bin_number, exp_month, exp_year, amount

def generate_markup():
    markup = InlineKeyboardMarkup()
    markup.add(
        InlineKeyboardButton("👨‍💻 Support", url="https://t.me/UNASL"),
        InlineKeyboardButton("📢 Channel", url="https://t.me/UNASQ")
    )
    return markup

@bot.message_handler(commands=['start'])
def send_welcome(message):
    welcome_message = """
    👋 Welcome to UNAS Generator Bot!

    I can help you generate credit card details for testing purposes.

    Here are the available commands:

    ➤ /start - Get started with the bot and see available commands.
    ➤ /gen <BIN> <MM/YY> <amount> - Generate fake credit card numbers (example: /gen 123456 12/25 5).

    🌐 You can generate credit cards using a BIN (Bank Identification Number). For example, use `/gen 123456 12/25 5` to generate 5 fake cards.

    If you need help, use the Support button below or visit our channel.
    """
    bot.send_message(message.chat.id, welcome_message, parse_mode="HTML", reply_markup=generate_markup())

@bot.message_handler(func=lambda message: message.text.startswith('/gen'))
def check_and_generate_cards(message):
    try:
        input_text = message.text.replace('/gen', '').strip()
        bin_number, exp_month, exp_year, amount = parse_message(input_text)

        if len(bin_number) < 6 or len(bin_number) > CARD_LENGTH:
            bot.send_message(message.chat.id, "❌ Invalid BIN length.")
            return

        if validate_bin(bin_number):
            cards = generate_cards(bin_number, amount, exp_month, exp_year)

            # Wrapping each card in <code> tags for easy copying
            formatted_cards = "\n".join([f"<code>{card}</code>" for card in cards])
            expiry_info = f"{exp_month:02d}/{exp_year}" if exp_month and exp_year else "Random"

            if amount <= 10:
                # Send cards directly in message for small amounts
                result_message = f"""
<b>🎉 Generated Cards</b>

<b>BIN</b> ➔ <code>{bin_number}</code>
<b>Amount</b> ➔ {amount}
<b>Expiry</b> ➔ {expiry_info}

{formatted_cards}
"""
                bot.send_message(message.chat.id, result_message, parse_mode="HTML", reply_markup=generate_markup())
            else:
                # Send cards as a file if the amount is more than 10
                file_content = "\n".join(cards)
                file = io.StringIO(file_content)
                file.name = "generated_cards.txt"
                bot.send_document(message.chat.id, InputFile(file), caption=f"Generated {amount} cards for BIN {bin_number}")
        else:
            bot.send_message(message.chat.id, "❌ Invalid BIN.")
    except Exception as e:
        print(e)
        bot.send_message(message.chat.id, "❌ An error occurred while processing the request.")

@bot.callback_query_handler(func=lambda call: True)
def callback_query(call):
    if call.data == "regenerate":
        bot.answer_callback_query(call.id, "Generating new cards...")
        check_and_generate_cards(call.message)  # Regenerates the card based on the original message

bot.polling()