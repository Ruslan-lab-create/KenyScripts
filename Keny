import asyncio
import logging
import secrets
import string
import json
import time

import aiosqlite
import httpx
from aiogram import Bot, Dispatcher, Router, F, BaseMiddleware
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode
from aiogram.filters import Command, CommandStart, CommandObject
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.types import (
    Message, CallbackQuery, Update,
    InlineKeyboardMarkup, InlineKeyboardButton,
)
from aiogram.utils.keyboard import InlineKeyboardBuilder

# ======================================================================
#  НАСТРОЙКИ — тут все твои данные уже вписаны
# ======================================================================

BOT_TOKEN = "8771434852:AAE7SY5_E0bfw5qyG15_-fPnnwba_5jCV98"
ADMIN_IDS = {8061549073}          # твой Telegram ID, всегда админ
BOT_USERNAME = "Kenyyt_bot"       # юзернейм бота без @
TRAFSLY_TOKEN = "at_0b48947db56ac43395be0edda91d4895"
TRAFSLY_BASE_URL = "https://api.trafsly.com"
CHANNEL_URL = "https://t.me/kenyaga"   # ссылка на твой канал
DB_PATH = "bot.db"

logging.basicConfig(level=logging.INFO, format="%(asctime)s | %(levelname)s | %(message)s")
logger = logging.getLogger(__name__)

_ALPHABET = string.ascii_letters + string.digits


# ======================================================================
#  БАЗА ДАННЫХ
# ======================================================================

async def init_db() -> None:
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("""
            CREATE TABLE IF NOT EXISTS scripts (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                code TEXT UNIQUE NOT NULL,
                title TEXT NOT NULL,
                content TEXT NOT NULL,
                created_by INTEGER NOT NULL,
                created_at INTEGER NOT NULL,
                clicks INTEGER NOT NULL DEFAULT 0,
                unlocks INTEGER NOT NULL DEFAULT 0
            )
        """)
        await db.execute("""
            CREATE TABLE IF NOT EXISTS pending_checks (
                user_id INTEGER NOT NULL,
                script_code TEXT NOT NULL,
                sponsors_json TEXT NOT NULL,
                PRIMARY KEY (user_id, script_code)
            )
        """)
        await db.execute("""
            CREATE TABLE IF NOT EXISTS users (
                user_id INTEGER PRIMARY KEY,
                username TEXT,
                first_seen INTEGER NOT NULL,
                is_blocked INTEGER NOT NULL DEFAULT 0
            )
        """)
        await db.execute("""
            CREATE TABLE IF NOT EXISTS admins (
                user_id INTEGER PRIMARY KEY,
                username TEXT
            )
        """)
        await db.commit()


def gen_code(length: int = 8) -> str:
    return "".join(secrets.choice(_ALPHABET) for _ in range(length))


async def upsert_user(user_id: int, username: str | None) -> None:
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute(
            "INSERT INTO users (user_id, username, first_seen) VALUES (?, ?, ?) "
            "ON CONFLICT(user_id) DO UPDATE SET username = excluded.username",
            (user_id, username, int(time.time())),
        )
        await db.commit()


async def mark_blocked(user_id: int, blocked: bool) -> None:
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("UPDATE users SET is_blocked = ? WHERE user_id = ?", (int(blocked), user_id))
        await db.commit()


async def create_script(title: str, content: str, created_by: int) -> str:
    code = gen_code()
    async with aiosqlite.connect(DB_PATH) as db:
        while True:
            cur = await db.execute("SELECT 1 FROM scripts WHERE code = ?", (code,))
            if await cur.fetchone() is None:
                break
            code = gen_code()
        await db.execute(
            "INSERT INTO scripts (code, title, content, created_by, created_at) VALUES (?, ?, ?, ?, ?)",
            (code, title, content, created_by, int(time.time())),
        )
        await db.commit()
    return code


async def get_script(code: str):
    async with aiosqlite.connect(DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        cur = await db.execute("SELECT * FROM scripts WHERE code = ?", (code,))
        row = await cur.fetchone()
        return dict(row) if row else None


async def list_scripts(limit: int = 20):
    async with aiosqlite.connect(DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        cur = await db.execute("SELECT * FROM scripts ORDER BY id DESC LIMIT ?", (limit,))
        return [dict(r) for r in await cur.fetchall()]


async def delete_script(code: str) -> bool:
    async with aiosqlite.connect(DB_PATH) as db:
        cur = await db.execute("DELETE FROM scripts WHERE code = ?", (code,))
        await db.commit()
        return cur.rowcount > 0


async def bump_clicks(code: str) -> None:
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("UPDATE scripts SET clicks = clicks + 1 WHERE code = ?", (code,))
        await db.commit()


async def bump_unlocks(code: str) -> None:
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("UPDATE scripts SET unlocks = unlocks + 1 WHERE code = ?", (code,))
        await db.commit()


async def save_pending(user_id: int, script_code: str, sponsors: list[dict]) -> None:
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute(
            "INSERT OR REPLACE INTO pending_checks (user_id, script_code, sponsors_json) VALUES (?, ?, ?)",
            (user_id, script_code, json.dumps(sponsors)),
        )
        await db.commit()


async def get_pending(user_id: int, script_code: str) -> list[dict]:
    async with aiosqlite.connect(DB_PATH) as db:
        cur = await db.execute(
            "SELECT sponsors_json FROM pending_checks WHERE user_id = ? AND script_code = ?",
            (user_id, script_code),
        )
        row = await cur.fetchone()
        return json.loads(row[0]) if row else []


async def clear_pending(user_id: int, script_code: str) -> None:
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute(
            "DELETE FROM pending_checks WHERE user_id = ? AND script_code = ?",
            (user_id, script_code),
        )
        await db.commit()


async def add_admin(user_id: int, username: str | None) -> None:
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute(
            "INSERT OR REPLACE INTO admins (user_id, username) VALUES (?, ?)", (user_id, username)
        )
        await db.commit()


async def remove_admin(user_id: int) -> bool:
    async with aiosqlite.connect(DB_PATH) as db:
        cur = await db.execute("DELETE FROM admins WHERE user_id = ?", (user_id,))
        await db.commit()
        return cur.rowcount > 0


async def find_user_by_username(username: str):
    username = username.lstrip("@")
    async with aiosqlite.connect(DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        cur = await db.execute("SELECT * FROM users WHERE username = ? COLLATE NOCASE", (username,))
        row = await cur.fetchone()
        return dict(row) if row else None


async def is_admin(user_id: int) -> bool:
    if user_id in ADMIN_IDS:
        return True
    async with aiosqlite.connect(DB_PATH) as db:
        cur = await db.execute("SELECT 1 FROM admins WHERE user_id = ?", (user_id,))
        return await cur.fetchone() is not None


async def get_stats() -> dict:
    now = int(time.time())
    day_ago = now - 86400
    week_ago = now - 7 * 86400
    async with aiosqlite.connect(DB_PATH) as db:
        total = (await (await db.execute("SELECT COUNT(*) FROM users")).fetchone())[0]
        last_24h = (await (await db.execute(
            "SELECT COUNT(*) FROM users WHERE first_seen >= ?", (day_ago,)
        )).fetchone())[0]
        last_7d = (await (await db.execute(
            "SELECT COUNT(*) FROM users WHERE first_seen >= ?", (week_ago,)
        )).fetchone())[0]
    return {"total": total, "last_24h": last_24h, "last_7d": last_7d}


async def all_active_user_ids() -> list[int]:
    async with aiosqlite.connect(DB_PATH) as db:
        cur = await db.execute("SELECT user_id FROM users WHERE is_blocked = 0")
        return [r[0] for r in await cur.fetchall()]


# ======================================================================
#  TRAFSLY API
# ======================================================================

class TrafslyError(Exception):
    pass


_HEADERS = {"Auth": TRAFSLY_TOKEN, "Content-Type": "application/json"}
_TIMEOUT = httpx.Timeout(10.0)


async def trafsly_get_sponsors(user_id: int, first_name=None, username=None, language_code=None, is_premium=False):
    payload = {"user_id": user_id, "max_sponsors": 5, "is_premium": is_premium, "action": "subscribe"}
    if first_name:
        payload["first_name"] = first_name
    if username:
        payload["username"] = username
    if language_code:
        payload["language_code"] = language_code
    async with httpx.AsyncClient(timeout=_TIMEOUT) as client:
        try:
            resp = await client.post(f"{TRAFSLY_BASE_URL}/api/v1/get-sponsors", headers=_HEADERS, json=payload)
        except httpx.HTTPError as e:
            raise TrafslyError(f"network error: {e}") from e
    if resp.status_code != 200:
        raise TrafslyError(f"HTTP {resp.status_code}: {resp.text}")
    data = resp.json()
    logger.info("Trafsly raw response for %s: %s", user_id, data)
    return data.get("sponsors", [])


async def trafsly_confirm(user_id: int, ads_id):
    async with httpx.AsyncClient(timeout=_TIMEOUT) as client:
        try:
            resp = await client.post(
                f"{TRAFSLY_BASE_URL}/api/v1/confirm-subscription",
                headers=_HEADERS, json={"user_id": user_id, "ads_id": ads_id},
            )
        except httpx.HTTPError as e:
            raise TrafslyError(f"network error: {e}") from e
    if resp.status_code != 200:
        raise TrafslyError(f"HTTP {resp.status_code}: {resp.text}")
    data = resp.json()
    return {"subscribed": bool(data.get("subscribed", False))}


async def trafsly_balance():
    async with httpx.AsyncClient(timeout=_TIMEOUT) as client:
        resp = await client.post(f"{TRAFSLY_BASE_URL}/api/v1/get-balance", headers=_HEADERS)
    if resp.status_code != 200:
        raise TrafslyError(f"HTTP {resp.status_code}: {resp.text}")
    return resp.json()


# ======================================================================
#  КЛАВИАТУРЫ
# ======================================================================

def kb_admin_main() -> InlineKeyboardMarkup:
    kb = InlineKeyboardBuilder()
    kb.button(text="➕ Создать скрипт", callback_data="admin:create")
    kb.button(text="📋 Мои скрипты", callback_data="admin:list")
    kb.button(text="💰 Баланс Trafsly", callback_data="admin:balance")
    kb.button(text="📊 Статистика", callback_data="admin:stats")
    kb.adjust(1)
    return kb.as_markup()


def kb_scripts_list(scripts: list[dict]) -> InlineKeyboardMarkup:
    kb = InlineKeyboardBuilder()
    for s in scripts:
        kb.button(text=f"📄 {s['title']}", callback_data=f"admin:view:{s['code']}")
    kb.button(text="⬅️ Назад", callback_data="admin:menu")
    kb.adjust(1)
    return kb.as_markup()


def kb_script_view(code: str) -> InlineKeyboardMarkup:
    kb = InlineKeyboardBuilder()
    kb.button(text="🗑 Удалить", callback_data=f"admin:delete:{code}")
    kb.button(text="⬅️ К списку", callback_data="admin:list")
    kb.adjust(1)
    return kb.as_markup()


def short_link_for(code: str) -> str:
    return f"https://t.me/{BOT_USERNAME}?start={code}"


def kb_sponsors(sponsors: list[dict], script_code: str) -> InlineKeyboardMarkup:
    kb = InlineKeyboardBuilder()
    for sp in sponsors:
        title = sp.get("title") or sp.get("name") or "Канал"
        link = sp.get("link") or sp.get("url")
        if not link:
            continue
        kb.button(text=f"📢 {title}", url=link)
    kb.button(text="✅ Я подписался", callback_data=f"check:{script_code}")
    kb.adjust(1)
    return kb.as_markup()


def kb_channel() -> InlineKeyboardMarkup:
    kb = InlineKeyboardBuilder()
    kb.button(text="📣 Наш канал", url=CHANNEL_URL)
    kb.adjust(1)
    return kb.as_markup()


# ======================================================================
#  FSM
# ======================================================================

class CreateScript(StatesGroup):
    waiting_title = State()
    waiting_content = State()


class Broadcast(StatesGroup):
    waiting_message = State()


# ======================================================================
#  MIDDLEWARE — считает каждого, кто хоть раз написал боту
# ======================================================================

class TrackUsersMiddleware(BaseMiddleware):
    async def __call__(self, handler, event: Update, data):
        user = None
        if event.message:
            user = event.message.from_user
        elif event.callback_query:
            user = event.callback_query.from_user
        if user and not user.is_bot:
            await upsert_user(user.id, user.username)
        return await handler(event, data)


# ======================================================================
#  ХЕНДЛЕРЫ — ПОЛЬЗОВАТЕЛИ
# ======================================================================

router = Router()


async def send_script(target, script: dict) -> None:
    await bump_unlocks(script["code"])
    text = f"✅ Готово! Вот твой скрипт «{script['title']}»:\n\n<code>{script['content']}</code>"
    if len(text) > 4000:
        text = text[:4000] + "…</code>\n\n⚠️ Скрипт обрезан, он слишком длинный для сообщения."
    await target.answer(text, parse_mode="HTML", reply_markup=kb_channel())


async def request_sponsors_and_show(message: Message, script_code: str) -> bool:
    user = message.from_user
    try:
        sponsors = await trafsly_get_sponsors(
            user_id=user.id, first_name=user.first_name, username=user.username,
            language_code=user.language_code, is_premium=bool(getattr(user, "is_premium", False)),
        )
    except TrafslyError as e:
        logger.warning("get_sponsors failed, letting user through: %s", e)
        return True
    if not sponsors:
        return True
    await save_pending(user.id, script_code, sponsors)
    await message.answer(
        "Чтобы получить скрипт, подпишись на спонсоров ниже, а затем нажми «Я подписался»:",
        reply_markup=kb_sponsors(sponsors, script_code),
    )
    return False


@router.message(CommandStart())
async def cmd_start(message: Message, command: CommandObject) -> None:
    payload = (command.args or "").strip()
    if not payload:
        await message.answer(
            "Привет! Я бот со скриптами для Roblox 👋\nЧтобы получить скрипт, перейди по ссылке, которую тебе прислали.",
            reply_markup=kb_channel(),
        )
        return
    script = await get_script(payload)
    if not script:
        await message.answer("Эта ссылка недействительна или скрипт был удалён 😕")
        return
    await bump_clicks(payload)
    can_proceed = await request_sponsors_and_show(message, payload)
    if can_proceed:
        await send_script(message, script)


@router.callback_query(F.data.startswith("check:"))
async def cb_check(call: CallbackQuery) -> None:
    script_code = call.data.split(":", 1)[1]
    user_id = call.from_user.id
    script = await get_script(script_code)
    if not script:
        await call.answer("Скрипт больше не доступен", show_alert=True)
        return
    pending = await get_pending(user_id, script_code)
    if not pending:
        can_proceed = await request_sponsors_and_show(call.message, script_code)
        await call.answer()
        if can_proceed:
            await send_script(call.message, script)
        return
    still_pending = []
    for sp in pending:
        ads_id = sp.get("ads_id")
        if ads_id is None:
            continue
        try:
            result = await trafsly_confirm(user_id, ads_id)
        except TrafslyError:
            still_pending.append(sp)
            continue
        if not result["subscribed"]:
            still_pending.append(sp)
    if still_pending:
        await save_pending(user_id, script_code, still_pending)
        await call.answer("Похоже, подписался не на всё. Проверь ещё раз ⤴️", show_alert=True)
        await call.message.edit_reply_markup(reply_markup=kb_sponsors(still_pending, script_code))
        return
    await clear_pending(user_id, script_code)
    await call.answer("Все подписки подтверждены ✅")
    await call.message.edit_text("✅ Все задания выполнены!")
    await send_script(call.message, script)


# ======================================================================
#  ХЕНДЛЕРЫ — АДМИНКА
# ======================================================================

@router.message(Command("admin"))
async def cmd_admin(message: Message) -> None:
    if not await is_admin(message.from_user.id):
        return
    await message.answer("🛠 Админ-панель\n\nВыбери действие:", reply_markup=kb_admin_main())


@router.message(Command("commands"))
async def cmd_commands(message: Message) -> None:
    if not await is_admin(message.from_user.id):
        return
    await message.answer(
        "📜 Команды администратора:\n\n"
        "/admin — открыть админ-панель\n"
        "/stats — статистика по боту\n"
        "/broadcast — сделать рассылку всем пользователям\n"
        "/adadmin <username> — назначить админа\n"
        "/deladmin <username> — снять админа\n"
        "/commands — этот список"
    )


@router.callback_query(F.data == "admin:menu")
async def cb_admin_menu(call: CallbackQuery, state: FSMContext) -> None:
    if not await is_admin(call.from_user.id):
        return await call.answer()
    await state.clear()
    await call.message.edit_text("🛠 Админ-панель\n\nВыбери действие:", reply_markup=kb_admin_main())
    await call.answer()


@router.callback_query(F.data == "admin:create")
async def cb_create_start(call: CallbackQuery, state: FSMContext) -> None:
    if not await is_admin(call.from_user.id):
        return await call.answer()
    await state.set_state(CreateScript.waiting_title)
    await call.message.edit_text("Введи название скрипта (для себя, юзеры его не увидят):")
    await call.answer()


@router.message(CreateScript.waiting_title)
async def create_title(message: Message, state: FSMContext) -> None:
    if not await is_admin(message.from_user.id):
        return
    title = (message.text or "").strip()
    if not title:
        await message.answer("Название не может быть пустым. Введи ещё раз:")
        return
    await state.update_data(title=title)
    await state.set_state(CreateScript.waiting_content)
    await message.answer("Теперь пришли сам Lua-скрипт текстом:")


@router.message(CreateScript.waiting_content)
async def create_content(message: Message, state: FSMContext) -> None:
    if not await is_admin(message.from_user.id):
        return
    content = message.text or message.caption
    if not content:
        await message.answer("Не вижу текста скрипта. Пришли его текстовым сообщением:")
        return
    data = await state.get_data()
    code = await create_script(title=data["title"], content=content, created_by=message.from_user.id)
    await state.clear()
    link = short_link_for(code)
    await message.answer(
        f"✅ Скрипт «{data['title']}» создан.\n\nКороткая ссылка:\n{link}",
        reply_markup=kb_admin_main(),
    )


@router.callback_query(F.data == "admin:list")
async def cb_list(call: CallbackQuery) -> None:
    if not await is_admin(call.from_user.id):
        return await call.answer()
    scripts = await list_scripts()
    if not scripts:
        await call.message.edit_text("Скриптов пока нет.", reply_markup=kb_admin_main())
        return await call.answer()
    await call.message.edit_text("📋 Твои скрипты:", reply_markup=kb_scripts_list(scripts))
    await call.answer()


@router.callback_query(F.data.startswith("admin:view:"))
async def cb_view(call: CallbackQuery) -> None:
    if not await is_admin(call.from_user.id):
        return await call.answer()
    code = call.data.split(":", 2)[2]
    script = await get_script(code)
    if not script:
        await call.answer("Не найден", show_alert=True)
        return
    link = short_link_for(code)
    preview = script["content"][:500]
    text = (
        f"📄 {script['title']}\nСсылка: {link}\n"
        f"Переходов: {script['clicks']} | Разблокировок: {script['unlocks']}\n\n"
        f"<code>{preview}</code>"
    )
    await call.message.edit_text(text, reply_markup=kb_script_view(code), parse_mode="HTML")
    await call.answer()


@router.callback_query(F.data.startswith("admin:delete:"))
async def cb_delete(call: CallbackQuery) -> None:
    if not await is_admin(call.from_user.id):
        return await call.answer()
    code = call.data.split(":", 2)[2]
    ok = await delete_script(code)
    await call.answer("Удалено" if ok else "Не найдено", show_alert=True)
    scripts = await list_scripts()
    if scripts:
        await call.message.edit_text("📋 Твои скрипты:", reply_markup=kb_scripts_list(scripts))
    else:
        await call.message.edit_text("Скриптов пока нет.", reply_markup=kb_admin_main())


@router.callback_query(F.data == "admin:balance")
async def cb_balance(call: CallbackQuery) -> None:
    if not await is_admin(call.from_user.id):
        return await call.answer()
    try:
        data = await trafsly_balance()
    except TrafslyError:
        await call.answer("Не удалось получить баланс", show_alert=True)
        return
    await call.message.edit_text(
        f"💰 Баланс: {data.get('balance', 0)} ₽\n⏳ В холде: {data.get('hold_balance', 0)} ₽",
        reply_markup=kb_admin_main(),
    )
    await call.answer()


@router.callback_query(F.data == "admin:stats")
async def cb_stats(call: CallbackQuery) -> None:
    if not await is_admin(call.from_user.id):
        return await call.answer()
    s = await get_stats()
    await call.message.edit_text(
        f"📊 Статистика\n\n"
        f"Юзеров в боте: {s['total']}\n"
        f"За 24 часа: {s['last_24h']}\n"
        f"За 7 дней: {s['last_7d']}",
        reply_markup=kb_admin_main(),
    )
    await call.answer()


@router.message(Command("stats"))
async def cmd_stats(message: Message) -> None:
    if not await is_admin(message.from_user.id):
        return
    s = await get_stats()
    await message.answer(
        f"📊 Статистика\n\nЮзеров в боте: {s['total']}\nЗа 24 часа: {s['last_24h']}\nЗа 7 дней: {s['last_7d']}"
    )


# ---------- Управление админами ----------

@router.message(Command("adadmin"))
async def cmd_adadmin(message: Message, command: CommandObject) -> None:
    if not await is_admin(message.from_user.id):
        return
    if not command.args:
        await message.answer("Использование: /adadmin username (без @)")
        return
    username = command.args.strip()
    user = await find_user_by_username(username)
    if not user:
        await message.answer(
            f"Не нашёл @{username} среди тех, кто писал боту. "
            f"Пусть этот человек сначала откроет бота и нажмёт /start, потом попробуй снова."
        )
        return
    await add_admin(user["user_id"], user["username"])
    await message.answer(f"✅ @{username} теперь администратор.")


@router.message(Command("deladmin"))
async def cmd_deladmin(message: Message, command: CommandObject) -> None:
    if not await is_admin(message.from_user.id):
        return
    if not command.args:
        await message.answer("Использование: /deladmin username (без @)")
        return
    username = command.args.strip()
    user = await find_user_by_username(username)
    if not user:
        await message.answer(f"Не нашёл пользователя @{username}.")
        return
    ok = await remove_admin(user["user_id"])
    await message.answer(f"✅ @{username} больше не администратор." if ok else f"@{username} не был администратором.")


# ---------- Рассылка ----------

@router.message(Command("broadcast"))
async def cmd_broadcast(message: Message, state: FSMContext) -> None:
    if not await is_admin(message.from_user.id):
        return
    await state.set_state(Broadcast.waiting_message)
    await message.answer("Пришли сообщение, которое нужно разослать всем пользователям бота:")


@router.message(Broadcast.waiting_message)
async def do_broadcast(message: Message, state: FSMContext) -> None:
    if not await is_admin(message.from_user.id):
        return
    await state.clear()
    user_ids = await all_active_user_ids()
    status_msg = await message.answer(f"⏳ Рассылаю на {len(user_ids)} пользователей...")

    success = blocked = deactivated = errors = 0
    for uid in user_ids:
        try:
            await message.copy_to(chat_id=uid)
            success += 1
        except Exception as e:
            err_text = str(e).lower()
            if "blocked" in err_text or "forbidden" in err_text:
                blocked += 1
                await mark_blocked(uid, True)
            elif "deactivated" in err_text or "not found" in err_text:
                deactivated += 1
                await mark_blocked(uid, True)
            else:
                errors += 1
        await asyncio.sleep(0.05)  # чтобы не упереться в лимиты Telegram

    await status_msg.edit_text(
        f"Рассылка завершена:\n\n"
        f"Успешно: {success}\n"
        f"Заблокировали бота: {blocked}\n"
        f"Удалённые аккаунты: {deactivated}\n"
        f"Ошибок: {errors}"
    )


# ======================================================================
#  ТОЧКА ВХОДА
# ======================================================================

async def main() -> None:
    await init_db()
    bot = Bot(token=BOT_TOKEN, default=DefaultBotProperties(parse_mode=ParseMode.HTML))
    dp = Dispatcher(storage=MemoryStorage())
    dp.update.outer_middleware(TrackUsersMiddleware())
    dp.include_router(router)

    logger.info("Бот запускается...")
    await bot.delete_webhook(drop_pending_updates=True)
    await dp.start_polling(bot)


if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        logger.info("Бот остановлен")
