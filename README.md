📁 1. Loyiha strukturasi
Plaintext
my_bot/
│── config.py
│── main.py
│── .env
│── .env.example
│── requirements.txt
│── Procfile
│── Dockerfile
└── docker-compose.yml
📄 2. Konfiguratsiya fayllari
.env.example
Code snippet
BOT_TOKEN=1234567890:ABCdefGHIjklMNOpqrsTUVwxyz
USE_WEBHOOK=True
WEBHOOK_SERVER_HOST=0.0.0.0
WEBHOOK_SERVER_PORT=8080
WEBHOOK_PATH=/webhook
WEBHOOK_SECRET=my-super-secret-token-123
WEBHOOK_URL=https://your-domain.up.railway.app
REDIS_URL=redis://default:password@your-upstash-redis-url.com:6379/0
config.py
Python
import os
from dotenv import load_dotenv

load_dotenv()

BOT_TOKEN = os.getenv("BOT_TOKEN")
USE_WEBHOOK = os.getenv("USE_WEBHOOK", "False").lower() in ("true", "1", "t")

WEBHOOK_SERVER_HOST = os.getenv("WEBHOOK_SERVER_HOST", "0.0.0.0")
WEBHOOK_SERVER_PORT = int(os.getenv("WEBHOOK_SERVER_PORT", 8080))
WEBHOOK_PATH = os.getenv("WEBHOOK_PATH", "/webhook")
WEBHOOK_SECRET = os.getenv("WEBHOOK_SECRET", "secret-key")
WEBHOOK_URL = os.getenv("WEBHOOK_URL", "")

REDIS_URL = os.getenv("REDIS_URL")
🐍 3. Asosiy kod (main.py)
Python
import logging
import sys
from aiohttp import web

from aiogram import Bot, Dispatcher, types
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode
from aiogram.filters import CommandStart
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.fsm.storage.redis import RedisStorage
from aiogram.webhook.aiohttp_server import SimpleRequestHandler, setup_application

import config

# 1. Logging stdout'ga yo'naltirilgan
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    stream=sys.stdout,
)
logger = logging.getLogger(__name__)

# 2. Redis FSM Storage (Upstash yoki mahalliy Redis uchun)
if config.REDIS_URL:
    storage = RedisStorage.from_url(config.REDIS_URL)
    logger.info(" RedisStorage ulandi.")
else:
    storage = MemoryStorage()
    logger.info(" MemoryStorage ishlatilmoqda.")

bot = Bot(
    token=config.BOT_TOKEN,
    default=DefaultBotProperties(parse_mode=ParseMode.HTML)
)
dp = Dispatcher(storage=storage)


# --- ROUTERLAR VA HANDLERLAR ---
@dp.message(CommandStart())
async def cmd_start(message: types.Message):
    await message.answer(f"👋 Salom, <b>{message.from_user.full_name}</b>!\nBot muvaffaqiyatli ishlamoqda.")


# --- HEALTH CHECK ENDPOINT ---
async def health_check(request: web.Request):
    return web.json_response({"status": "ok", "bot": "online"}, status=200)


# --- WEBHOOK HOOKS ---
async def on_startup(bot: Bot):
    webhook_url = f"{config.WEBHOOK_URL}{config.WEBHOOK_PATH}"
    await bot.set_webhook(
        url=webhook_url,
        secret_token=config.WEBHOOK_SECRET,
        drop_pending_updates=True
    )
    logger.info(f" Webhook o'rnatildi: {webhook_url}")


async def on_shutdown(bot: Bot):
    logger.info(" Webhook o'chirilmoqda...")
    await bot.delete_webhook()


# --- MAIN ENGINE ---
def main():
    if config.USE_WEBHOOK:
        logger.info("🚀 Webhook rejimida ishga tushmoqda...")
        
        dp.startup.register(on_startup)
        dp.shutdown.register(on_shutdown)

        app = web.Application()
        
        # Health check marshruti
        app.router.add_get("/health", health_check)

        # SimpleRequestHandler va Secret Token tekshiruvi
        webhook_requests_handler = SimpleRequestHandler(
            dispatcher=dp,
            bot=bot,
            secret_token=config.WEBHOOK_SECRET,
        )
        webhook_requests_handler.register(app, path=config.WEBHOOK_PATH)

        setup_application(app, dp, bot=bot)

        web.run_app(
            app,
            host=config.WEBHOOK_SERVER_HOST,
            port=config.WEBHOOK_SERVER_PORT
        )
    else:
        logger.info("🔄 Long Polling rejimida ishga tushmoqda...")
        import asyncio
        asyncio.run(dp.start_polling(bot))


if __name__ == "__main__":
    main()
📦 4. Deploy va Bo'g'inlar fayllari
requirements.txt
Plaintext
aiogram>=3.4.0
aiohttp>=3.9.0
redis>=5.0.0
python-dotenv>=1.0.0
Procfile
Plaintext
web: python main.py
🐳 5. Docker va Docker-Compose (Bonus)
Dockerfile
Dockerfile
FROM python:3.11-slim

WORKDIR /app

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8080

CMD ["python", "main.py"]
docker-compose.yml
YAML
version: '3.8'

services:
  bot:
    build: .
    env_file:
      - .env
    ports:
      - "8080:8080"
    restart: always
    depends_on:
      - redis

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    restart: always
🚀 6. Railway'da Deploy Qilish Ketma-ketligi
Upstash Redis olish:

Upstash.com saytidan bepul Redis ma'lumotlar bazasini yarating va REDIS_URL manzilini nusxalab oling.

Railway Loyihasi Yaratish:

GitHub omboringizni Railway.app ga ulang.

Loyihangiz sozlamalariga kirib "Generate Domain" tugmasini bosing (Masalan: https://my-bot.up.railway.app).

Variables (O'zgaruvchilar)ni sozlash:
Railway konsolidagi Variables bo'limiga quyidagilarni kiriting:

BOT_TOKEN = BotFather bergan token

USE_WEBHOOK = True

WEBHOOK_URL = https://my-bot.up.railway.app (Railway bergan domen)

WEBHOOK_PATH = /webhook

WEBHOOK_SECRET = Ixtiyoriy xavfsizlik kaliti

WEBHOOK_SERVER_PORT = 8080

REDIS_URL = Upstash Redis URL manzili

Tekshirish:

Brauzerda https://my-bot.up.railway.app/health manziliga kiring. Javob: {"status": "ok", "bot": "online"} chiqishi kerak.

Telegramda botingizga /start yuborib tekshiring!
