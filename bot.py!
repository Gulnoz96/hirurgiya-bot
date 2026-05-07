import os
import json
import sqlite3
import logging
import requests
from datetime import datetime, timedelta
from telegram import Update, ReplyKeyboardMarkup, KeyboardButton
from telegram.ext import (
    Application, CommandHandler, MessageHandler,
    filters, ContextTypes, ConversationHandler
)

# ==================== CONFIGURATION ====================
BOT_TOKEN = "8697948223:AAFatHivHocpqAG8CFbasTYYpj6q4iYqgyw"
GROQ_API_KEY = "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"
GROQ_URL = "https://api.groq.com/openai/v1/chat/completions"
GROQ_MODEL = "llama-3.3-70b-versatile"

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# ==================== DATABASE ====================
def init_db():
    conn = sqlite3.connect("hirurgiya.db")
    c = conn.cursor()
    c.execute("""
        CREATE TABLE IF NOT EXISTS users (
            user_id INTEGER PRIMARY KEY,
            username TEXT,
            first_name TEXT,
            total_questions INTEGER DEFAULT 0,
            correct_answers INTEGER DEFAULT 0,
            last_active TEXT
        )
    """)
    c.execute("""
        CREATE TABLE IF NOT EXISTS results (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER,
            topic TEXT,
            correct INTEGER,
            date TEXT
        )
    """)
    conn.commit()
    conn.close()

def get_user(user_id, username, first_name):
    conn = sqlite3.connect("hirurgiya.db")
    c = conn.cursor()
    c.execute("INSERT OR IGNORE INTO users (user_id, username, first_name) VALUES (?, ?, ?)",
              (user_id, username, first_name))
    conn.commit()
    conn.close()

def save_result(user_id, topic, correct):
    conn = sqlite3.connect("hirurgiya.db")
    c = conn.cursor()
    c.execute("UPDATE users SET total_questions = total_questions + 1, last_active = ? WHERE user_id = ?",
              (datetime.now().isoformat(), user_id))
    if correct:
        c.execute("UPDATE users SET correct_answers = correct_answers + 1 WHERE user_id = ?", (user_id,))
    c.execute("INSERT INTO results (user_id, topic, correct, date) VALUES (?, ?, ?, ?)",
              (user_id, topic, 1 if correct else 0, datetime.now().isoformat()))
    conn.commit()
    conn.close()

def get_score(user_id):
    conn = sqlite3.connect("hirurgiya.db")
    c = conn.cursor()
    c.execute("SELECT total_questions, correct_answers, first_name FROM users WHERE user_id = ?", (user_id,))
    row = c.fetchone()
    
    # Get weak topics
    c.execute("""
        SELECT topic, COUNT(*) as total, SUM(correct) as correct_count
        FROM results WHERE user_id = ?
        GROUP BY topic
        ORDER BY (CAST(SUM(correct) AS FLOAT)/COUNT(*)) ASC
        LIMIT 3
    """, (user_id,))
    weak = c.fetchall()
    conn.close()
    return row, weak

def get_all_scores():
    conn = sqlite3.connect("hirurgiya.db")
    c = conn.cursor()
    c.execute("""
        SELECT first_name, username, total_questions, correct_answers
        FROM users WHERE total_questions > 0
        ORDER BY (CAST(correct_answers AS FLOAT)/total_questions) DESC
        LIMIT 20
    """)
    rows = c.fetchall()
    conn.close()
    return rows

# ==================== GROQ AI ====================
def ask_groq(system_prompt, user_message, max_tokens=2000):
    headers = {
        "Authorization": f"Bearer {GROQ_API_KEY}",
        "Content-Type": "application/json"
    }
    data = {
        "model": GROQ_MODEL,
        "messages": [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_message}
        ],
        "max_tokens": max_tokens,
        "temperature": 0.7
    }
    try:
        response = requests.post(GROQ_URL, headers=headers, json=data, timeout=30)
        result = response.json()
        return result["choices"][0]["message"]["content"]
    except Exception as e:
        logger.error(f"Groq error: {e}")
        return "Хато рух дод. Лутфан дубора кӯшиш кунед. / Произошла ошибка. Попробуйте снова."

# ==================== SYSTEM PROMPTS ====================
SYSTEM_PROMPT = """Ты — профессиональный ассистент кафедры Общей Хирургии ТГМУ (ДДТТ) имени Абуали ибн Сино, Таджикистан.
Ты помогаешь студентам 3-го курса медицинского факультета.

ВАЖНЫЕ ПРАВИЛА:
1. Отвечай на том языке, на котором написан вопрос (тоҷикӣ или русский)
2. Используй профессиональную медицинскую терминологию
3. Будь точным и структурированным как учебник

ТЕМЫ КУРСА (VI семестр):
1. Нарушение кровообращения: тромбозы, эмболия, некрозы
2. Нарушение кровообращения: гангрены, свищи, язвы, пролежни
3. Травма: закрытые повреждения мягких тканей, Краш-синдром
4. Переломы: классификация, лечение, иммобилизация
5. Вывихи: классификация, гипсовые повязки
6. Ожоги: классификация, площадь, ожоговая болезнь
7. Лечение ожогов. Электротравма и отморожение
8. Сепсис: Sepsis-3, qSOFA, SOFA критерии
9. Опухоли: TNM классификация, диагностика, лечение
10. Острая специфическая инфекция: анаэробная, клостридиальная, сибиреязвенная, карбункул, дифтерия раны
11. Острая специфическая инфекция: Столбняк (Куззоз), бешенство
12. Хроническая специфическая инфекция: сифилис костей, лепра
13. Хроническая специфическая инфекция: актиномикоз, туберкулёз костей и суставов
14. Диагностика и лечение актиномикоза, туберкулёза костей и суставов
15. Хирургическая операция: техника, пред- и послеоперационный период
16. Курация больных. Оформление истории болезни
17. Основы пластической и реконструктивной хирургии
"""

TEST_SYSTEM = SYSTEM_PROMPT + """
ДЛЯ ТЕСТОВ:
- Создавай клинические ситуационные вопросы (не просто теорию)
- Вопрос должен описывать реального пациента с симптомами
- 4 варианта ответа (A, B, C, D)
- Только один правильный ответ
- После ответа студента объясни почему правильный ответ правильный, а остальные неправильные

Формат вопроса:
ВОПРОС [N] из [Total]
━━━━━━━━━━━━━━━━━━━━

[Клиническая ситуация]

A) [вариант]
B) [вариант]  
C) [вариант]
D) [вариант]

Введите ответ: A, B, C или D
"""

# ==================== KEYBOARDS ====================
def main_keyboard():
    keyboard = [
        [KeyboardButton("❓ Задать вопрос / Савол додан")],
        [KeyboardButton("📋 План урока / Нақшаи дарс")],
        [KeyboardButton("📝 Тест / Санҷиш")],
        [KeyboardButton("🏥 Клинический случай / Холати клиникӣ")],
        [KeyboardButton("📊 Мои результаты / Натиҷаҳои ман")],
        [KeyboardButton("ℹ️ Помощь / Ёрдам")]
    ]
    return ReplyKeyboardMarkup(keyboard, resize_keyboard=True)

# ==================== USER SESSIONS ====================
user_sessions = {}

# ==================== HANDLERS ====================
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    get_user(user.id, user.username or "", user.first_name or "")
    
    text = """🏥 *Хуш омадед ба Ассистенти Кафедраи Ҷарроҳии Умумии ДДТТ!*
🏥 *Добро пожаловать в Ассистент Кафедры Общей Хирургии ТГМУ!*

━━━━━━━━━━━━━━━━━━━━
🎓 *Донишгоҳи Давлатии Тиббии Тоҷикистон*
🎓 *Таджикский Государственный Медицинский Университет*
*им. Абуали ибн Сино*
━━━━━━━━━━━━━━━━━━━━

📚 *Фармонҳо / Команды:*

❓ `/ask` — Савол аз ҷарроҳӣ / Вопрос по хирургии
📋 `/lesson` — Нақшаи дарс / План занятия  
📝 `/test` — Санҷиш / Тест
🏥 `/cases` — Холати клиникӣ / Клинический случай
📊 `/score` — Натиҷаҳо / Результаты

*Барои оғоз тугмаро пахш кунед / Нажмите кнопку для начала* 👇"""
    
    await update.message.reply_text(text, parse_mode="Markdown", reply_markup=main_keyboard())

async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await start(update, context)

async def ask_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_sessions[update.effective_user.id] = {"mode": "ask"}
    await update.message.reply_text(
        "❓ *Саволи худро нависед / Напишите ваш вопрос:*\n\n"
        "Мисол / Пример:\n"
        "• Признаки газовой гангрены?\n"
        "• Куззоз чист?\n"
        "• Критерии Sepsis-3?",
        parse_mode="Markdown"
    )

async def lesson_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_sessions[update.effective_user.id] = {"mode": "lesson"}
    topics = """📋 *Мавзӯро нависед / Напишите тему:*

1. Тромбоз, эмболия, некроз
2. Гангрена, свищи, язвы, пролежни
3. Травма, Краш-синдром
4. Переломы
5. Вывихи
6. Ожоги
7. Электротравма, отморожение
8. Сепсис
9. Опухоли (TNM)
10. Анаэробная инфекция
11. Столбняк / Куззоз
12. Сифилис костей, лепра
13. Актиномикоз, туберкулёз костей
14. Диагностика актиномикоза
15. Хирургическая операция
16. История болезни
17. Пластическая хирургия

_Номер ё номи мавзӯъро нависед / Напишите номер или название_"""
    await update.message.reply_text(topics, parse_mode="Markdown")

async def test_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_sessions[update.effective_user.id] = {"mode": "test_setup"}
    await update.message.reply_text(
        "📝 *Тест / Санҷиш*\n\n"
        "Мавзӯ ва шумораи саволро нависед:\n"
        "Напишите тему и количество вопросов:\n\n"
        "Мисол / Пример:\n"
        "• `Сепсис, 5`\n"
        "• `Куззоз, 10`\n"
        "• `Переломы, 3`\n\n"
        "_(аз 1 то 20 савол / от 1 до 20 вопросов)_",
        parse_mode="Markdown"
    )

async def cases_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_sessions[update.effective_user.id] = {"mode": "cases"}
    await update.message.reply_text(
        "🏥 *Клинический случай / Холати клиникӣ*\n\n"
        "Мавзӯро нависед / Напишите тему:\n\n"
        "• Сепсис\n• Куззоз\n• Перитонит\n• Аппендицит\n"
        "• Переломы\n• Ожоги\n• Гангрена\n• Опухоли\n"
        "• Тромбоз\n• ва дигар мавзӯъҳо...",
        parse_mode="Markdown"
    )

async def score_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    args = context.args
    
    if args and args[0].lower() == "all":
        # Teacher view
        scores = get_all_scores()
        if not scores:
            await update.message.reply_text("Ҳанӯз ягон донишҷӯ тест нагузаштааст.\nЕщё никто не проходил тесты.")
            return
        
        text = "📊 *Рейтинги донишҷӯён / Рейтинг студентов:*\n\n"
        for i, (name, username, total, correct) in enumerate(scores, 1):
            pct = round(correct/total*100) if total > 0 else 0
            bar = "🟢" if pct >= 70 else "🟡" if pct >= 50 else "🔴"
            text += f"{i}. {bar} *{name}* (@{username})\n"
            text += f"   {correct}/{total} ({pct}%)\n\n"
        await update.message.reply_text(text, parse_mode="Markdown")
        return
    
    # Personal score
    get_user(user.id, user.username or "", user.first_name or "")
    row, weak = get_score(user.id)
    
    if not row or row[0] == 0:
        await update.message.reply_text(
            "📊 Шумо ҳанӯз тест нагузаштаед.\nВы ещё не проходили тесты.\n\n"
            "Барои оғоз: /test\nДля начала: /test"
        )
        return
    
    total, correct, name = row
    pct = round(correct/total*100)
    bar = "🟢 Аъло!" if pct >= 80 else "🟡 Хуб" if pct >= 60 else "🔴 Такрор лозим"
    
    text = f"📊 *Натиҷаҳои шумо / Ваши результаты:*\n\n"
    text += f"👤 {name}\n"
    text += f"✅ Дуруст / Правильно: {correct}/{total}\n"
    text += f"📈 Фоиз / Процент: {pct}% {bar}\n\n"
    
    if weak:
        text += "⚠️ *Мавзӯъҳои заиф / Слабые темы:*\n"
        for topic, total_t, correct_t in weak:
            t_pct = round(correct_t/total_t*100) if total_t > 0 else 0
            text += f"• {topic}: {t_pct}%\n"
    
    text += f"\n_Барои рейтинги умумӣ: /score all_\n_Общий рейтинг: /score all_"
    await update.message.reply_text(text, parse_mode="Markdown")

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    text = update.message.text.strip()
    user_id = user.id
    
    get_user(user_id, user.username or "", user.first_name or "")
    
    # Handle keyboard buttons
    if "Задать вопрос" in text or "Савол додан" in text:
        await ask_command(update, context)
        return
    elif "План урока" in text or "Нақшаи дарс" in text:
        await lesson_command(update, context)
        return
    elif "Тест" in text or "Санҷиш" in text:
        await test_command(update, context)
        return
    elif "Клинический случай" in text or "Холати клиникӣ" in text:
        await cases_command(update, context)
        return
    elif "Мои результаты" in text or "Натиҷаҳои ман" in text:
        await score_command(update, context)
        return
    elif "Помощь" in text or "Ёрдам" in text:
        await start(update, context)
        return
    
    session = user_sessions.get(user_id, {})
    mode = session.get("mode", "ask")
    
    # TEST MODE - waiting for answer
    if mode == "test_answer":
        answer = text.upper().strip()
        if answer in ["A", "B", "C", "D"]:
            topic = session.get("topic", "Хирургия")
            question = session.get("current_question", "")
            correct_answer = session.get("correct_answer", "")
            q_num = session.get("q_num", 1)
            q_total = session.get("q_total", 5)
            
            # Check answer and explain
            is_correct = (answer == correct_answer)
            save_result(user_id, topic, is_correct)
            
            prompt = f"""Студент ответил {answer} на этот вопрос:

{question}

Правильный ответ: {correct_answer}
Ответ студента: {answer}

{'✅ ПРАВИЛЬНО! Объясни почему этот ответ правильный (2-3 предложения).' if is_correct else f'❌ НЕПРАВИЛЬНО! Объясни почему {answer} неверный и почему {correct_answer} правильный (2-3 предложения).'}

Потом спроси готов ли к следующему вопросу."""

            explanation = ask_groq(TEST_SYSTEM, prompt, 500)
            
            if q_num < q_total:
                user_sessions[user_id] = {
                    "mode": "test_next",
                    "topic": topic,
                    "q_num": q_num,
                    "q_total": q_total
                }
                await update.message.reply_text(
                    f"{'✅' if is_correct else '❌'} *Вопрос {q_num} из {q_total}*\n\n{explanation}\n\n"
                    f"_Напишите любое слово для следующего вопроса_",
                    parse_mode="Markdown"
                )
            else:
                user_sessions[user_id] = {"mode": "ask"}
                await update.message.reply_text(
                    f"{'✅' if is_correct else '❌'} *Вопрос {q_num} из {q_total}*\n\n{explanation}\n\n"
                    f"🏁 *Тест завершён! / Санҷиш тамом шуд!*\n"
                    f"Натиҷаҳо: /score",
                    parse_mode="Markdown"
                )
        else:
            await update.message.reply_text("Лутфан A, B, C ё D нависед / Пожалуйста напишите A, B, C или D")
        return
    
    # TEST NEXT QUESTION
    if mode == "test_next":
        topic = session.get("topic", "Хирургия")
        q_num = session.get("q_num", 1) + 1
        q_total = session.get("q_total", 5)
        await generate_test_question(update, context, topic, q_num, q_total)
        return
    
    # TEST SETUP
    if mode == "test_setup":
        parts = text.split(",")
        if len(parts) >= 2:
            topic = parts[0].strip()
            try:
                count = int(parts[1].strip())
                count = max(1, min(20, count))
            except:
                count = 5
        else:
            topic = text
            count = 5
        
        await update.message.reply_text(
            f"📝 Тайёр кардани {count} савол аз мавзӯи «{topic}»...\n"
            f"Готовлю {count} вопросов по теме «{topic}»...",
        )
        await generate_test_question(update, context, topic, 1, count)
        return
    
    # LESSON MODE
    if mode == "lesson":
        user_sessions[user_id] = {"mode": "ask"}
        await update.message.reply_text("⏳ Нақшаи дарс тайёр мешавад... / Готовлю план занятия...")
        
        prompt = f"""Барои мавзӯи «{text}» нақшаи машғулоти амалии 90-дақиқагӣ тайёр кун.

СОХТОР (8 қисм):
1. 🎯 МАВЗӮЪ ВА ҲАДАФҲО
   - Номи мавзӯъ
   - Ҳадафҳои таълимӣ (3-5 ҳадаф)

2. ⏱️ ХРОНОКАРТА (90 дақиқа)
   - Саломдиҳӣ ва санҷиши ҳузур (5 дақ)
   - Санҷиши дониши пешина (20 дақ)
   - Шарҳи мавзӯи нав (30 дақ)
   - Кори амалӣ (25 дақ)
   - Масъалаҳои клиникӣ (5 дақ)
   - Хулосабандӣ (5 дақ)

3. 📖 МАТНИ СУХАНРОНИИ МУАЛЛИМ (скрипт)
   Матни мукаммали шарҳи мавзӯъ

4. 🏥 МАСЪАЛАҲОИ КЛИНИКӢ (2 масъала)
   Масъала + Ҷавоби намунавӣ

5. ✅ САВОЛҲОИ ТЕСТ (10 савол бо 4 вариант)

6. 🧠 МНЕМОНИКАҲО
   Усулҳои ёдгирии осон

7. 📚 АДАБИЁТ
   - Петров С.В. Общая хирургия
   - Дигар манбаъҳо

8. 📝 ВАЗИФАИ ХОНАГӢ"""
        
        response = ask_groq(SYSTEM_PROMPT, prompt, 3000)
        
        # Split long messages
        if len(response) > 4000:
            parts = [response[i:i+4000] for i in range(0, len(response), 4000)]
            for part in parts:
                await update.message.reply_text(part)
        else:
            await update.message.reply_text(response)
        return
    
    # CASES MODE
    if mode == "cases":
        user_sessions[user_id] = {"mode": "ask"}
        await update.message.reply_text(f"🏥 Холати клиникӣ тайёр мешавад... / Готовлю клинический случай...")
        
        prompt = f"""Клинический случай по теме «{text}» для студентов 3 курса ТГМУ.

Формат:
🏥 КЛИНИЧЕСКИЙ СЛУЧАЙ / ХОЛАТИ КЛИНИКӢ

👤 ПАЦИЕНТ:
[Возраст, пол, профессия]

📋 ЖАЛОБЫ:
[Подробные жалобы]

🔍 ОСМОТР И АНАЛИЗЫ:
[Объективный статус, лабораторные данные]

❓ ВОПРОСЫ:
1. Поставьте предварительный диагноз
2. Назначьте дополнительные исследования
3. Составьте план лечения

✅ ЭТАЛОН ОТВЕТА:
[Полный разбор с обоснованием]

Потом задай студенту вопрос для самопроверки."""
        
        response = ask_groq(SYSTEM_PROMPT, prompt, 2000)
        await update.message.reply_text(response)
        return
    
    # DEFAULT - ASK MODE
    user_sessions[user_id] = {"mode": "ask"}
    await update.message.reply_text("⏳ Ҷавоб тайёр мешавад... / Готовлю ответ...")
    
    response = ask_groq(SYSTEM_PROMPT, text, 1500)
    await update.message.reply_text(response)

async def generate_test_question(update, context, topic, q_num, q_total):
    user_id = update.effective_user.id
    
    prompt = f"""Савол {q_num} аз {q_total} аз мавзӯи «{topic}».
Вопрос {q_num} из {q_total} по теме «{topic}».

Клиникӣ масъала тайёр кун бо 4 ҷавоб (A, B, C, D).
Создай клиническую ситуацию с 4 вариантами ответа.

Формат точный:
САВОЛ {q_num} аз {q_total} / ВОПРОС {q_num} из {q_total}
━━━━━━━━━━━━━━━━━━━━

[Клиническая ситуация - пациент с симптомами]

A) [вариант]
B) [вариант]
C) [вариант]
D) [вариант]

ПРАВИЛЬНЫЙ_ОТВЕТ: [только одна буква A, B, C или D]

Введите ответ: A, B, C или D"""
    
    response = ask_groq(TEST_SYSTEM, prompt, 800)
    
    # Extract correct answer
    correct = "A"
    lines = response.split("\n")
    clean_lines = []
    for line in lines:
        if "ПРАВИЛЬНЫЙ_ОТВЕТ:" in line:
            try:
                correct = line.split(":")[1].strip()[0].upper()
            except:
                correct = "A"
        else:
            clean_lines.append(line)
    
    clean_response = "\n".join(clean_lines)
    
    user_sessions[user_id] = {
        "mode": "test_answer",
        "topic": topic,
        "q_num": q_num,
        "q_total": q_total,
        "current_question": clean_response,
        "correct_answer": correct
    }
    
    await update.message.reply_text(clean_response)

# ==================== MAIN ====================
def main():
    init_db()
    app = Application.builder().token(BOT_TOKEN).build()
    
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_command))
    app.add_handler(CommandHandler("ask", ask_command))
    app.add_handler(CommandHandler("lesson", lesson_command))
    app.add_handler(CommandHandler("test", test_command))
    app.add_handler(CommandHandler("cases", cases_command))
    app.add_handler(CommandHandler("score", score_command))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    
    logger.info("Бот запущен! / Bot started!")
    app.run_polling(drop_pending_updates=True)

if __name__ == "__main__":
    main()
