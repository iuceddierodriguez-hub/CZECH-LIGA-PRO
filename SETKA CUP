"""
Setka Cup (UCRANIA) v1.0 - Desglose OpenAI vs Perplexity en el Mensaje
Pick Media Hora (mejor de la franja, >= 78%) | Stake 100 (> 80%)
Pensado para GitHub + Render.
"""

import asyncio
import json
import logging
import os
import re
import sqlite3
import threading
import time
from contextlib import contextmanager
from datetime import datetime, timedelta
from zoneinfo import ZoneInfo

import requests
from flask import Flask, jsonify
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, ApplicationBuilder, CommandHandler, CallbackQueryHandler, ContextTypes
from openai import OpenAI

# ---------------- CONFIGURACION ----------------

LEAGUE = "Setka Cup (Ucrania)"

TOKEN = os.getenv("TELEGRAM_BOT_TOKEN", "").strip()
CHAT_ID = os.getenv("TELEGRAM_CHAT_ID", "").strip()
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY", "").strip()
PERPLEXITY_API_KEY = os.getenv("PERPLEXITY_API_KEY", "").strip()

openai_client = OpenAI(api_key=OPENAI_API_KEY) if OPENAI_API_KEY else None

POLL_SECONDS = int(os.getenv("POLL_SECONDS", "120"))

# En Render, si usas Persistent Disk, pon DB_PATH=/var/data/setka_v1.db
DB_PATH = os.getenv("DB_PATH", "setka_v1.db")

# Anticipacion del pick (minutos antes del partido)
LEAD_MIN = int(os.getenv("LEAD_MIN", "35"))
MIN_LEAD_MIN = int(os.getenv("MIN_LEAD_MIN", "5"))

# Tamano del bloque (30 = un pick por media hora; 15 = uno cada 15 min)
BLOCK_MIN = int(os.getenv("BLOCK_MIN", "30"))

# STAKES Y UMBRALES
STAKE_ACIERTO_MIN = int(os.getenv("STAKE_ACIERTO_MIN", "81"))  # 81 = "más de 80%"
FRANJA_MIN_PROB = int(os.getenv("FRANJA_MIN_PROB", "78"))

# Todo en hora de Colombia (las horas que pegas y las que se muestran).
TZ = ZoneInfo("America/Bogota")
INPUT_TZ = TZ

logging.basicConfig(
    level=os.getenv("LOG_LEVEL", "INFO").upper(),
    format="%(asctime)s | %(levelname)s | %(message)s",
)
for noisy in ("httpx", "httpcore", "urllib3"):
    logging.getLogger(noisy).setLevel(logging.WARNING)
log = logging.getLogger("SETKA")

web = Flask(__name__)
started_at = time.time()


# ---------------- BASE DE DATOS ----------------

@contextmanager
def db():
    conn = sqlite3.connect(DB_PATH, timeout=20)
    conn.row_factory = sqlite3.Row
    try:
        yield conn
        conn.commit()
    finally:
        conn.close()


def init_db():
    with db() as c:
        c.execute("""CREATE TABLE IF NOT EXISTS fixtures (
            event_id TEXT PRIMARY KEY, player1 TEXT, player2 TEXT, start_ts INTEGER, hour_block TEXT)""")
        c.execute("""CREATE TABLE IF NOT EXISTS analysis (
            event_id TEXT PRIMARY KEY, fav TEXT, rating REAL, prob INTEGER, quality TEXT,
            details TEXT, analyzed_at INTEGER)""")
        c.execute("""CREATE TABLE IF NOT EXISTS picks (
            event_id TEXT PRIMARY KEY, flow TEXT, rating REAL, fav TEXT,
            player1 TEXT, player2 TEXT, start_ts INTEGER, sent_at INTEGER,
            hit INTEGER, details TEXT, hour_block TEXT)""")
        c.execute("CREATE TABLE IF NOT EXISTS kv (k TEXT PRIMARY KEY, v TEXT)")
        c.execute("CREATE TABLE IF NOT EXISTS extra_chats (chat_id TEXT PRIMARY KEY)")


def get_all_chat_ids():
    targets = set()
    for cid in CHAT_ID.split(","):
        if cid.strip():
            targets.add(cid.strip())
    with db() as c:
        for r in c.execute("SELECT chat_id FROM extra_chats").fetchall():
            if r["chat_id"]:
                targets.add(str(r["chat_id"]).strip())
    return list(targets)


def count_acierto_today():
    inicio_dia = int(datetime.now(TZ).replace(hour=0, minute=0, second=0, microsecond=0).timestamp())
    with db() as c:
        return c.execute("SELECT COUNT(*) FROM picks WHERE flow='acierto' AND sent_at>=?", (inicio_dia,)).fetchone()[0]


def block_already_sent(hour_block):
    with db() as c:
        row = c.execute("SELECT COUNT(*) FROM picks WHERE hour_block=?", (hour_block,)).fetchone()
        return row[0] > 0 if row else False


# ---------------- UTILIDADES ----------------

def elegir(texto: str, player1: str, player2: str):
    """Devuelve player1, player2 o None si el texto no define claramente a uno solo."""
    t = (texto or "").lower()
    in1 = player1.lower() in t
    in2 = player2.lower() in t
    if in1 and not in2:
        return player1
    if in2 and not in1:
        return player2
    return None


def slug(s: str) -> str:
    return re.sub(r"\W+", "", s.lower())[:12]


# ---------------- PASO 1: PERPLEXITY ----------------

def investigar_con_perplexity(player1: str, player2: str):
    """Devuelve (texto, favorito_o_None, prob_o_None)."""
    if not PERPLEXITY_API_KEY:
        log.warning("PERPLEXITY_API_KEY no configurada.")
        return f"Sin datos de investigación para {player1} vs {player2}.", None, None

    fecha_hoy = datetime.now(TZ).strftime("%Y-%m-%d")

    prompt_busqueda = (
        f"Investiga neutralmente en Sofascore, AiScore, Betano, Tips Manager y Flashscore los partidos, "
        f"enfrentamientos directos (H2H) y rachas recientes de tenis de mesa (Setka Cup, Ucrania) para "
        f"'{player1}' y '{player2}' hoy ({fecha_hoy}). Los nombres suelen venir como 'Apellido I.', "
        f"asegúrate de no confundir jugadores con apellidos parecidos. "
        f"Indica de manera objetiva los datos estadísticos y concluye qué jugador tiene mejor proyección "
        f"de victoria. Termina SIEMPRE con dos últimas líneas, cada una en su propia línea, exactamente así: "
        f"GANADOR: <nombre exacto, '{player1}' o '{player2}'> "
        f"y PROB: <número entero de 50 a 96 con la probabilidad en % de que gane ese jugador>"
    )

    headers = {
        "Authorization": f"Bearer {PERPLEXITY_API_KEY}",
        "Content-Type": "application/json"
    }
    payload = {
        "model": "perplexity/sonar",
        "input": prompt_busqueda,
        "tools": [{"type": "web_search"}]
    }

    try:
        resp = requests.post("https://api.perplexity.ai/v1/responses", headers=headers, json=payload, timeout=38)
        if resp.status_code == 200:
            data = resp.json()
            texto = ""
            if isinstance(data.get("output_text"), str) and data["output_text"].strip():
                texto = data["output_text"].strip()
            elif "output" in data:
                for item in data.get("output", []):
                    if item.get("type") == "message":
                        for content in item.get("content", []):
                            if content.get("type") in ("output_text", "text") and content.get("text"):
                                texto = content["text"].strip()
            if not texto and "choices" in data:
                texto = data["choices"][0].get("message", {}).get("content", "").strip()

            if texto and len(texto) > 30:
                log.info("Perplexity investigó: %s vs %s", player1, player2)
                matches = re.findall(r"GANADOR:\s*(.+)", texto, flags=re.IGNORECASE)
                fav = elegir(matches[-1], player1, player2) if matches else None
                probs = re.findall(r"PROB:\s*\**\s*(\d{1,3})", texto, flags=re.IGNORECASE)
                prob = max(50, min(96, int(probs[-1]))) if probs else None
                return texto, fav, prob
        else:
            log.warning("Perplexity HTTP error: %s - %s", resp.status_code, resp.text[:200])
    except Exception as exc:
        log.warning("Excepción en Perplexity: %s", exc)

    return f"Sin datos de investigación para {player1} vs {player2}.", None, None


# ---------------- PASO 2: OPENAI ----------------

def analizar_partido(player1: str, player2: str):
    """Devuelve (fav, rating, prob, details) o None si no se pudo decidir (no se guarda en caché)."""
    if not openai_client:
        log.warning("OPENAI_API_KEY no configurada; se omite el partido.")
        return None

    datos_perplexity, fav_perplexity, prob_perplexity = investigar_con_perplexity(player1, player2)

    prompt_analisis = f"""
Actúa como un auditor cuantitativo de apuestas deportivas experto en tenis de mesa (Setka Cup, Ucrania).
Tu análisis debe ser 100% estricto y neutral. Nunca elijas a un jugador por defecto ni alfabéticamente.
En Setka Cup hay poca información por jugador: si los datos son escasos o contradictorios, baja la probabilidad.

Reporte de investigación web de Perplexity:
\"\"\"
{datos_perplexity}
\"\"\"

REGLAS:
1. Revisa las estadísticas reales, H2H y rachas provistas.
2. Determina cuál jugador tiene mayor probabilidad real de ganar (entre 50% y 96%).
3. Si no hay datos que respalden a ninguno, usa prob_ganador = 50.

Devuelve estrictamente este JSON válido y sin markdown adicional:
{{
  "ganador_openai": "Nombre exacto del ganador ('{player1}' o '{player2}')",
  "prob_ganador": 82,
  "resumen": "Argumento analítico conciso basado en los datos."
}}
"""
    try:
        response = openai_client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt_analisis}],
            temperature=0.1,
            max_tokens=400
        )
        content = response.choices[0].message.content.strip()
        if content.startswith("```"):
            content = content.strip("`")
            if content.lower().startswith("json"):
                content = content[4:]
            content = content.strip()

        data = json.loads(content)
        resumen = str(data.get("resumen", "")).strip()
        fav = elegir(str(data.get("ganador_openai", "")), player1, player2)
        prob_ganador = max(50, min(96, int(data.get("prob_ganador", 50))))

        if fav is None:
            return "NINGUNO", 0, 0, ""

        if prob_ganador >= STAKE_ACIERTO_MIN:
            rating = 9.5
        elif prob_ganador >= 83:
            rating = 8.5
        else:
            rating = 8.0

        # Probabilidad que cada IA asigna al jugador 1; la combinada es el promedio.
        # Si una IA da el ganador contrario, su porcentaje baja el promedio.
        o_p1 = prob_ganador if fav == player1 else 100 - prob_ganador
        if fav_perplexity and prob_perplexity is not None:
            p_p1 = prob_perplexity if fav_perplexity == player1 else 100 - prob_perplexity
            comb_p1 = round((o_p1 + p_p1) / 2)
            pplx_line = f"🔍 Perplexity: {fav_perplexity} ({prob_perplexity}%)"
            comb_label = "Combinada (promedio)"
        else:
            comb_p1 = o_p1
            pplx_line = "🔍 Perplexity: sin dato"
            comb_label = "Combinada (solo OpenAI)"

        if comb_p1 == 50:
            return "NINGUNO", 0, 0, ""
        fav_final = player1 if comb_p1 > 50 else player2
        prob_final = max(comb_p1, 100 - comb_p1)

        if prob_final >= STAKE_ACIERTO_MIN:
            rating = 9.5
        elif prob_final >= 83:
            rating = 8.5
        else:
            rating = 8.0

        aviso = ""
        if fav_perplexity and fav_perplexity != fav:
            aviso = "⚠️ Perplexity y OpenAI NO coinciden\n"

        details = (
            f"📊 Probabilidad asignada por cada IA:\n"
            f"{pplx_line}\n"
            f"🎯 OpenAI: {fav} ({prob_ganador}%)\n"
            f"✅ {comb_label}: {fav_final} ({prob_final}%)\n"
            f"{aviso}\n"
            f"Argumentos: {resumen}"
        )
        log.info("Análisis: %s vs %s -> OpenAI: %s (%d%%) | Perplexity: %s (%s%%) | Final: %s (%d%%)",
                 player1, player2, fav, prob_ganador, fav_perplexity, prob_perplexity, fav_final, prob_final)
        return fav_final, rating, prob_final, details

    except Exception as exc:
        log.warning("Error en análisis OpenAI: %s", exc)
        return None


# ---------------- CARGA DE PARTIDOS ----------------

def parse_lineas(lineas):
    """Convierte lineas 'HH:MM | Jugador1 vs Jugador2' en partidos."""
    partidos = []
    ahora = datetime.now(INPUT_TZ)

    for linea in lineas:
        linea = linea.strip()
        if not linea or linea.startswith("#"):
            continue
        partes = [p.strip() for p in linea.split("|")]
        if len(partes) < 2 or " vs " not in partes[1]:
            continue
        time_str, match_str = partes[0], partes[1]
        player1, player2 = [x.strip() for x in match_str.split(" vs ", 1)]

        try:
            h, m = map(int, time_str.split(":"))
            match_dt = ahora.replace(hour=h, minute=m, second=0, microsecond=0)
            # Liga 24h: si la hora ya pasó hoy (más de 10 min), el partido es de mañana
            if match_dt < ahora - timedelta(minutes=10):
                match_dt += timedelta(days=1)
            ts = int(match_dt.timestamp())
        except Exception:
            continue

        local = datetime.fromtimestamp(ts, TZ)
        bloque_min = (local.minute // BLOCK_MIN) * BLOCK_MIN
        hour_block = f"{local.date().isoformat()}_{local.hour:02d}:{bloque_min:02d}"

        partidos.append({
            "id": f"m_{local:%Y%m%d_%H%M}_{slug(player1)}_{slug(player2)}",
            "player1": player1,
            "player2": player2,
            "hour_block": hour_block,
            "timestamp": ts,
        })
    return partidos


def guardar_partidos(items):
    with db() as c:
        for it in items:
            c.execute("""INSERT OR REPLACE INTO fixtures(event_id, player1, player2, start_ts, hour_block)
                         VALUES (?,?,?,?,?)""",
                      (it["id"], it["player1"], it["player2"], it["timestamp"], it["hour_block"]))


def fetch_fixtures_only():
    """Lee partidos.txt (si existe). Los cargados con /cargar ya viven en la base de datos."""
    archivo = "partidos.txt"
    if not os.path.exists(archivo):
        return
    with open(archivo, "r", encoding="utf-8") as f:
        items = parse_lineas(f.readlines())
    guardar_partidos(items)
    log.info("partidos.txt leído: %s eventos.", len(items))


# ---------------- SELECCIÓN ----------------

def select_best_pick_for_current_blocks():
    now = int(time.time())

    with db() as c:
        upcoming_blocks = [
            r["hour_block"] for r in c.execute("""
                SELECT DISTINCT hour_block FROM fixtures
                WHERE start_ts BETWEEN ? AND ?
            """, (now + MIN_LEAD_MIN * 60, now + LEAD_MIN * 60)).fetchall()
        ]

    pending_blocks = [b for b in upcoming_blocks if not block_already_sent(b)]
    if not pending_blocks:
        return None, None, 0

    target_block = sorted(pending_blocks)[0]

    with db() as c:
        block_matches = [dict(r) for r in c.execute("""
            SELECT event_id, player1, player2, start_ts, hour_block
            FROM fixtures
            WHERE hour_block = ? AND start_ts > ?
            ORDER BY start_ts ASC
        """, (target_block, now + 3 * 60)).fetchall()]

    if not block_matches:
        return None, None, 0

    log.info("Evaluando bloque %s (%d partidos)...", target_block, len(block_matches))

    valid_analyzed = []
    for r in block_matches:
        eid = r["event_id"]
        with db() as c:
            an = c.execute("SELECT fav, rating, prob, details FROM analysis WHERE event_id=?", (eid,)).fetchone()

        if an:
            fav, rating, prob, details = an["fav"], an["rating"], an["prob"], an["details"]
        else:
            result = analizar_partido(r["player1"], r["player2"])  # sin conexión abierta durante las APIs
            if result is None:
                continue  # error transitorio: no se guarda, se reintenta en el próximo ciclo
            fav, rating, prob, details = result
            with db() as c:
                c.execute("""INSERT OR REPLACE INTO analysis(event_id, fav, rating, prob, quality, details, analyzed_at)
                             VALUES (?,?,?,?,?,?,?)""",
                          (eid, fav, rating, prob, "alta" if prob >= FRANJA_MIN_PROB else "baja", details, now))
            time.sleep(0.5)

        if fav and fav != "NINGUNO" and prob >= FRANJA_MIN_PROB:
            valid_analyzed.append({**r, "fav": fav, "rating": rating, "details": details, "prob": prob})

    if not valid_analyzed:
        log.info("Bloque %s sin candidatos >= %d%%. No se envía pick.", target_block, FRANJA_MIN_PROB)
        return None, None, 0

    valid_analyzed.sort(key=lambda x: (x["prob"], x["rating"]), reverse=True)
    best = valid_analyzed[0]

    flow = "acierto" if best["prob"] >= STAKE_ACIERTO_MIN else "normal"
    log.info("Pick %s -> %s (%d%%)", flow.upper(), best["fav"], best["prob"])
    return best, flow, best["prob"]


# ---------------- FORMATO Y ENVÍO TELEGRAM ----------------

def format_message(p, flow, mins, prob):
    kickoff = datetime.fromtimestamp(p["start_ts"], TZ).strftime("%I:%M %p").lstrip("0")

    if flow == "acierto":
        stake = "100 (ACIERTO TOTAL) 🚨🔥"
        header = f"🚨🔥 <b>SETKA CUP - STAKE 100 DE ACIERTO (Favorito: {prob}%)</b> 🔥🚨"
    else:
        stake = int(max(7, min(9, round(p["rating"]))))
        header = f"🏓 <b>SETKA CUP - (Favorito: {prob}%)</b>"

    lines = [
        header,
        f"🏆 <b>Ganador Definitivo: {p['fav']}</b>",
        f"⚔️ {p['player1']} vs {p['player2']}",
        f"⏱️ Inicio: {kickoff} (Col) | Stake: <b>{stake}</b>",
        f"⏳ Arranca en ~{max(1, int(mins))} min",
    ]
    if p["details"]:
        lines += ["", p["details"]]
    return "\n".join(lines)


async def send_pick(app, p, flow, mins, prob):
    text = format_message(p, flow, mins, prob)
    markup = InlineKeyboardMarkup([[InlineKeyboardButton("📤 Enviar a otros chats", switch_inline_query=text[:250])]])

    for cid in get_all_chat_ids():
        try:
            await app.bot.send_message(chat_id=cid, text=text, parse_mode="HTML",
                                       disable_web_page_preview=True, reply_markup=markup)
        except Exception as exc:
            log.warning("No se pudo enviar al chat %s: %s", cid, exc)


async def button_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()


async def run_cycle(app):
    await asyncio.to_thread(fetch_fixtures_only)

    best_p, flow, prob = await asyncio.to_thread(select_best_pick_for_current_blocks)
    if best_p and get_all_chat_ids():
        mins = (best_p["start_ts"] - time.time()) / 60
        await send_pick(app, best_p, flow, mins, prob)

        with db() as c:
            c.execute("""INSERT OR IGNORE INTO picks
                (event_id, flow, rating, fav, player1, player2, start_ts, sent_at, details, hour_block)
                VALUES (?,?,?,?,?,?,?,?,?,?)""",
                (best_p["event_id"], flow, best_p["rating"], best_p["fav"], best_p["player1"],
                 best_p["player2"], best_p["start_ts"], int(time.time()), best_p["details"], best_p["hour_block"]))

        log.info("Pick enviado -> Bloque: %s | Flujo: %s | Ganador: %s (%d%%)",
                 best_p["hour_block"], flow, best_p["fav"], prob)


async def monitor_loop(app: Application):
    while True:
        try:
            await run_cycle(app)
        except Exception as exc:
            log.error("Fallo en bucle: %s", str(exc)[:300])
        await asyncio.sleep(POLL_SECONDS)


# ---------------- COMANDOS TELEGRAM ----------------

async def cmd_start(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        f"🏓 <b>¡Bot {LEAGUE} Activo!</b>\n\n"
        "📋 <b>Comandos:</b>\n"
        "• /cargar - Cargar partidos (pega las líneas debajo del comando)\n"
        "• /limpiar - Borrar los partidos futuros cargados\n"
        "• /partidos - Ver la cartelera próxima\n"
        "• /estado - Estado y estadísticas del día\n"
        "• /resultado - Ver pendientes / registrar resultado (gano o perdio)\n"
        "• /resumen - Rendimiento histórico real\n"
        "• /id - ID de este chat\n"
        "• /activar_grupo - Vincular este grupo\n"
        "• /test - Estado de las APIs",
        parse_mode="HTML")


async def cmd_cargar(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    lineas = (update.message.text or "").split("\n")[1:]
    items = parse_lineas(lineas)
    if not items:
        await update.message.reply_text(
            "Pega los partidos debajo del comando, uno por línea:\n\n"
            "/cargar\n14:00 | Kovalenko I. vs Petrenko A.\n14:30 | Shevchuk O. vs Bondar M.")
        return
    guardar_partidos(items)
    await update.message.reply_text(f"✅ {len(items)} partidos cargados (hora interpretada en {INPUT_TZ.key}).")


async def cmd_limpiar(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    with db() as c:
        n = c.execute("DELETE FROM fixtures WHERE start_ts>?", (int(time.time()),)).rowcount
    await update.message.reply_text(f"🧹 {n} partidos futuros eliminados.")


async def cmd_id(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(f"🆔 ID de este chat: <code>{update.effective_chat.id}</code>", parse_mode="HTML")


async def cmd_activar_grupo(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    cid = str(update.effective_chat.id)
    with db() as c:
        c.execute("INSERT OR IGNORE INTO extra_chats(chat_id) VALUES (?)", (cid,))
    await update.message.reply_text(f"✅ <b>Grupo vinculado a {LEAGUE}:</b> <code>{cid}</code>", parse_mode="HTML")


async def cmd_test(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    pplx_ok = "✅ Conectada" if PERPLEXITY_API_KEY else "⚠️ Sin API Key"
    oai_ok = "✅ Conectada" if openai_client else "⚠️ Sin API Key"
    await update.message.reply_text(f"✅ {LEAGUE} operativo.\nPerplexity: {pplx_ok} | OpenAI: {oai_ok}")


async def cmd_estado(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    with db() as c:
        n_fix = c.execute("SELECT COUNT(*) FROM fixtures WHERE start_ts>?", (int(time.time()),)).fetchone()[0]
    await update.message.reply_text(
        f"🟢 <b>Bot {LEAGUE} Activo</b>\n"
        f"Partidos futuros cargados: {n_fix}\n"
        f"Stakes 100 (más de {STAKE_ACIERTO_MIN - 1}%) enviados hoy: {count_acierto_today()}\n"
        f"Umbral pick: {FRANJA_MIN_PROB}% | Bloque: {BLOCK_MIN} min\n"
        f"Zona horaria: America/Bogota",
        parse_mode="HTML")


async def cmd_resultado(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    now = int(time.time())
    with db() as c:
        pend = c.execute("""SELECT event_id, fav, player1, player2, start_ts FROM picks
                            WHERE hit IS NULL AND start_ts < ? ORDER BY start_ts LIMIT 1""",
                         (now - 30 * 60,)).fetchone()
        if not ctx.args:
            n = c.execute("SELECT COUNT(*) FROM picks WHERE hit IS NULL").fetchone()[0]
            if not pend:
                await update.message.reply_text(f"No hay picks listos para resolver. Pendientes totales: {n}")
            else:
                await update.message.reply_text(
                    f"Próximo a resolver: {pend['player1']} vs {pend['player2']} (pick: {pend['fav']})\n"
                    f"Responde: /resultado gano  o  /resultado perdio\nPendientes totales: {n}")
            return
        if not pend:
            await update.message.reply_text("No hay picks listos para resolver.")
            return
        arg = ctx.args[0].lower()
        if arg in ("gano", "ganó", "si", "sí", "1"):
            hit = 1
        elif arg in ("perdio", "perdió", "no", "0"):
            hit = 0
        else:
            await update.message.reply_text("Usa: /resultado gano  o  /resultado perdio")
            return
        c.execute("UPDATE picks SET hit=? WHERE event_id=?", (hit, pend["event_id"]))
    await update.message.reply_text(f"{'✅' if hit else '❌'} Registrado: {pend['player1']} vs {pend['player2']}")


async def cmd_resumen(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    with db() as c:
        rows = c.execute("""SELECT flow, CAST(ROUND(rating) AS INT) AS r, COUNT(*) AS n, SUM(hit) AS h
            FROM picks WHERE hit IS NOT NULL GROUP BY flow, r ORDER BY flow, r DESC""").fetchall()
        pend = c.execute("SELECT COUNT(*) FROM picks WHERE hit IS NULL").fetchone()[0]
    if not rows:
        await update.message.reply_text(f"📈 Sin pronósticos resueltos aún. Pendientes: {pend}")
        return
    lines = [f"📈 <b>Rendimiento {LEAGUE}</b>"] + [
        f"• {r['flow']} · nota {r['r']}: {r['h']}/{r['n']} ({100 * r['h'] / r['n']:.0f}%)" for r in rows]
    await update.message.reply_text("\n".join(lines + [f"Pendientes: {pend}"]), parse_mode="HTML")


async def cmd_partidos(update: Update, ctx: ContextTypes.DEFAULT_TYPE):
    with db() as c:
        rows = c.execute("""SELECT player1, player2, start_ts, hour_block FROM fixtures
                            WHERE start_ts >= ? ORDER BY start_ts LIMIT 15""", (int(time.time()),)).fetchall()
    if not rows:
        await update.message.reply_text("Sin partidos programados. Usa /cargar para añadirlos.")
        return
    lines = [f"📋 <b>Cartelera Próxima {LEAGUE}:</b>"]
    for r in rows:
        hh = datetime.fromtimestamp(r["start_ts"], TZ).strftime("%I:%M %p")
        lines.append(f"• <b>{hh}</b> | {r['player1']} vs {r['player2']}")
    await update.message.reply_text("\n".join(lines), parse_mode="HTML")


# ---------------- WEB (Render) ----------------

@web.get("/")
def home():
    return "Setka Cup v1 Activo", 200


@web.get("/health")
def health():
    return jsonify({"status": "ok", "uptime_seconds": int(time.time() - started_at)}), 200


def run_web():
    port = int(os.getenv("PORT", "10000"))
    web.run(host="0.0.0.0", port=port, debug=False, use_reloader=False)


async def post_init(application: Application):
    await asyncio.to_thread(fetch_fixtures_only)
    application.create_task(monitor_loop(application))


def main():
    if not TOKEN or not CHAT_ID:
        raise RuntimeError("Faltan variables esenciales (TELEGRAM_BOT_TOKEN o TELEGRAM_CHAT_ID).")
    init_db()

    threading.Thread(target=run_web, daemon=True).start()

    app = ApplicationBuilder().token(TOKEN).post_init(post_init).build()

    for name, fn in [
        ("start", cmd_start), ("id", cmd_id), ("activar_grupo", cmd_activar_grupo),
        ("test", cmd_test), ("estado", cmd_estado), ("resumen", cmd_resumen),
        ("partidos", cmd_partidos), ("cargar", cmd_cargar), ("limpiar", cmd_limpiar),
        ("resultado", cmd_resultado),
    ]:
        app.add_handler(CommandHandler(name, fn))

    app.add_handler(CallbackQueryHandler(button_callback))

    log.info("Iniciando Setka Cup v1 con desglose Perplexity vs OpenAI...")
    app.run_polling(drop_pending_updates=True)


if __name__ == "__main__":
    main()
