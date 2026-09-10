FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && apt-get install -y \
    wget \
    curl \
    gnupg \
    libglib2.0-0 \
    libnss3 \
    libnspr4 \
    libdbus-1-3 \
    libatk1.0-0 \
    libatk-bridge2.0-0 \
    libcups2 \
    libexpat1 \
    libxcb1 \
    libxkbcommon0 \
    libatspi2.0-0 \
    libx11-6 \
    libxcomposite1 \
    libxdamage1 \
    libxext6 \
    libxfixes3 \
    libxrandr2 \
    libgbm1 \
    libpango-1.0-0 \
    libcairo2 \
    libasound2 \
    && rm -rf /var/lib/apt/lists/*

RUN pip install --no-cache-dir playwright==1.52.0 paho-mqtt==2.1.0

RUN playwright install chromium

RUN cat > /app/scraper.py << 'SCRAPER_EOF'
from playwright.sync_api import sync_playwright
import paho.mqtt.client as mqtt
import json
import re
import time
import traceback
from datetime import datetime

print("CEPM MONITOR MEDIDORES INICIADO", flush=True)


def log(msg):
    now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    print(f"[{now}] {msg}", flush=True)


with open("/data/options.json") as f:
    options = json.load(f)

CEPM_USER = options["cepm_user"]
CEPM_PASS = options["cepm_pass"]
CUENTA_PREPAGO = options.get("cuenta_prepago", "9062280")
CUENTA_POSTPAGO = options.get("cuenta_postpago", "9019302")
POLL_INTERVAL_MIN = options.get("poll_interval_min", 15)

MQTT_HOST = options["mqtt_host"]
MQTT_PORT = options["mqtt_port"]
MQTT_USER = options["mqtt_user"]
MQTT_PASS = options["mqtt_pass"]

LOGIN_URL = "https://oficina.cepm.com.do/Account/Login"

DEBUG_SCREENSHOT = "/share/cepm_prepago_debug.png"
DEBUG_HTML = "/share/cepm_prepago_debug.html"

log("Configuracion cargada")
log(f"Usuario CEPM: {CEPM_USER}")
log(f"Longitud de la contrasena: {len(CEPM_PASS) if CEPM_PASS else 0}")
log(f"Cuenta prepago: {CUENTA_PREPAGO} / Cuenta postpago: {CUENTA_POSTPAGO}")
log(f"Intervalo de sondeo: {POLL_INTERVAL_MIN} min")

client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)

if MQTT_USER:
    client.username_pw_set(MQTT_USER, MQTT_PASS)

client.connect(MQTT_HOST, MQTT_PORT, 60)

DISCOVERY_PREFIX = "homeassistant"
BASE_TOPIC = "cepm_medidores"


def publish_discovery():

    sensors = {
        # --- Prepago: balance (kWh hasta el corte) + consumo ---
        "prepago_balance_rd": {"name": "CEPM Prepago Balance RD", "unit": "RD$"},
        "prepago_balance_kwh": {
            "name": "CEPM Prepago Balance kWh Restante",
            "unit": "kWh",
        },
        "prepago_fecha_balance": {"name": "CEPM Prepago Fecha Balance", "unit": ""},
        "prepago_lectura_acumulada": {
            "name": "CEPM Prepago Lectura Acumulada",
            "unit": "kWh",
            "device_class": "energy",
            "state_class": "total_increasing",
        },
        "prepago_consumo_ultimo_intervalo": {
            "name": "CEPM Prepago Consumo Ultimo Intervalo 15min",
            "unit": "kWh",
        },
        "prepago_consumo_periodo": {
            "name": "CEPM Prepago Consumo Periodo Consultado",
            "unit": "kWh",
        },
        "prepago_promedio_diario": {
            "name": "CEPM Prepago Promedio Diario",
            "unit": "kWh",
        },
        "prepago_proyeccion_30d": {
            "name": "CEPM Prepago Proyeccion 30 Dias",
            "unit": "kWh",
        },

        # --- Postpago: solo consumo/lectura (no aplica corte por saldo) ---
        "postpago_lectura_acumulada": {
            "name": "CEPM Postpago Lectura Acumulada",
            "unit": "kWh",
            "device_class": "energy",
            "state_class": "total_increasing",
        },
        "postpago_consumo_ultimo_intervalo": {
            "name": "CEPM Postpago Consumo Ultimo Intervalo 15min",
            "unit": "kWh",
        },
        "postpago_consumo_periodo": {
            "name": "CEPM Postpago Consumo Periodo Consultado",
            "unit": "kWh",
        },
        "postpago_promedio_diario": {
            "name": "CEPM Postpago Promedio Diario",
            "unit": "kWh",
        },
        "postpago_proyeccion_30d": {
            "name": "CEPM Postpago Proyeccion 30 Dias",
            "unit": "kWh",
        },

        "login_status": {"name": "CEPM Login Status", "unit": ""},
    }

    for key, sensor in sensors.items():

        topic = f"{DISCOVERY_PREFIX}/sensor/cepm_medidores_{key}/config"

        payload = {
            "name": sensor["name"],
            "state_topic": f"{BASE_TOPIC}/state",
            "value_template": f"{{{{ value_json.{key} }}}}",
            "unique_id": f"cepm_medidores_{key}"
        }

        if sensor["unit"]:
            payload["unit_of_measurement"] = sensor["unit"]

        if "device_class" in sensor:
            payload["device_class"] = sensor["device_class"]

        if "state_class" in sensor:
            payload["state_class"] = sensor["state_class"]

        client.publish(topic, json.dumps(payload), retain=True)

        log(f"Discovery MQTT enviado: {key}")


def parse_number(raw):
    if raw is None:
        return None
    try:
        return float(raw.replace(",", ""))
    except ValueError:
        return None


def guardar_diagnostico(page, html):
    try:
        page.screenshot(path=DEBUG_SCREENSHOT, full_page=True)
        log(f"Captura de diagnostico guardada en {DEBUG_SCREENSHOT}")
    except Exception:
        log("No se pudo guardar la captura de pantalla de diagnostico")

    try:
        with open(DEBUG_HTML, "w", encoding="utf-8") as f:
            f.write(html)
        log(f"HTML de diagnostico guardado en {DEBUG_HTML}")
    except Exception:
        log("No se pudo guardar el HTML de diagnostico")


def buscar_mensaje_error(html):
    patrones = [
        r'<[^>]*class="[^"]*(?:validation-summary-errors|text-danger|alert-danger|field-validation-error)[^"]*"[^>]*>(.*?)</',
        r'(usuario o contrase\u00f1a incorrect[oa]s?)',
        r'(credenciales? inv\u00e1lid[oa]s?)',
        r'(cuenta bloqueada)',
    ]
    for patron in patrones:
        m = re.search(patron, html, re.IGNORECASE | re.DOTALL)
        if m:
            texto = re.sub(r'<[^>]+>', '', m.group(1)).strip()
            if texto:
                return texto
    return None


def extract_balance_cuenta(html, cuenta):
    """Extrae balance RD$, balance kWh restante (si existe la anotacion,
    tipico de cuentas prepago) y fecha de esa lectura, anclado a la fila
    de la cuenta indicada en la pagina de inicio."""
    idx = html.find(cuenta)
    if idx == -1:
        log(f"No se encontro la cuenta {cuenta} en la pagina de inicio")
        return None, None, None

    window = html[idx:idx + 1500]

    balance_rd = None
    m = re.search(r'<td[^>]*>\s*([\d,]+\.\d{2})\s*</td>', window)
    if m:
        balance_rd = parse_number(m.group(1))

    balance_kwh = None
    m2 = re.search(r'Balance\s*=\s*([\d.]+)\s*kWh', window)
    if m2:
        balance_kwh = float(m2.group(1))

    fecha = None
    m3 = re.search(
        r'(\d{1,2}/[A-Za-z]+\.\s*\d{2}:\d{2}\s*[ap]\.\s*m\.)',
        window,
        re.IGNORECASE
    )
    if m3:
        fecha = m3.group(1)

    return balance_rd, balance_kwh, fecha


def extract_lectura_acumulada(html):
    pattern = (
        r'<td class="nivel2-col-90">[\d-]+<br\s*/>\s*([\d,\.]+)\s*</td>\s*'
        r'<td class="nivel2-col-90"[^>]*>[\d-]+\s*<br\s*/>\s*([\d,\.]+)\s*</td>\s*'
        r'<td class="nivel2-col-80">\s*([\d.]+)\s*</td>\s*'
        r'<td class="nivel2-col-80">([\d.]+)</td>\s*'
        r'<td class="nivel2-col-80">([\d.]+)</td>'
    )
    m = re.search(pattern, html, re.DOTALL)

    if not m:
        log("No se encontro la tabla de lectura acumulada")
        return None, None, None, None

    lectura_ini = parse_number(m.group(1))
    lectura_fin = parse_number(m.group(2))
    consumo_periodo = float(m.group(3))
    promedio_diario = float(m.group(4))
    proyeccion_30d = float(m.group(5))

    log(f"Lectura inicial: {lectura_ini} / Lectura final: {lectura_fin}")

    return lectura_fin, consumo_periodo, promedio_diario, proyeccion_30d


def extract_intervalo_15min(html):
    blocks = re.findall(r'data\.addRows\(\[(.*?)\]\);', html, re.DOTALL)

    if not blocks:
        log("No se encontraron bloques addRows en la pagina Nivel2")
        return None

    bloque_min = blocks[0]

    entradas = re.findall(
        r'\[new Date\((\d+),\s*(\d+),\s*(\d+),\s*(\d+),\s*(\d+)\),\s*([\d.]+)\]',
        bloque_min
    )

    if not entradas:
        log("No se pudieron parsear entradas de 15 minutos")
        return None

    ultimo = entradas[-1]
    valor = float(ultimo[5])

    log(f"Ultimo intervalo de 15 min: {valor} kWh")

    return valor


def obtener_datos_consumo(page, cuenta):
    """Selecciona la cuenta indicada en la pagina Nivel2 (ya abierta) y
    extrae su lectura acumulada y consumo. Devuelve un dict."""

    log(f"Seleccionando cuenta {cuenta}")

    page.select_option("#Cuenta", cuenta)

    try:
        page.wait_for_load_state("networkidle", timeout=15000)
    except Exception:
        log("networkidle no se alcanzo a tiempo, continuando igual")

    page.wait_for_timeout(2000)

    try:
        html = page.content()
    except Exception:
        log("La pagina aun estaba navegando, esperando y reintentando")
        page.wait_for_timeout(3000)
        html = page.content()

    (
        lectura_fin,
        consumo_periodo,
        promedio_diario,
        proyeccion_30d,
    ) = extract_lectura_acumulada(html)

    consumo_ultimo_intervalo = extract_intervalo_15min(html)

    return {
        "lectura_acumulada": lectura_fin,
        "consumo_ultimo_intervalo": consumo_ultimo_intervalo,
        "consumo_periodo": consumo_periodo,
        "promedio_diario": promedio_diario,
        "proyeccion_30d": proyeccion_30d,
    }


def valores_vacios_consumo():
    return {
        "lectura_acumulada": None,
        "consumo_ultimo_intervalo": None,
        "consumo_periodo": None,
        "promedio_diario": None,
        "proyeccion_30d": None,
    }


def scrape():

    log("Iniciando navegador Playwright")

    resultado = {
        "prepago_balance_rd": None,
        "prepago_balance_kwh": None,
        "prepago_fecha_balance": None,
        "login_status": "desconocido",
    }
    resultado.update({f"prepago_{k}": v for k, v in valores_vacios_consumo().items()})
    resultado.update({f"postpago_{k}": v for k, v in valores_vacios_consumo().items()})

    with sync_playwright() as p:

        browser = p.chromium.launch(
            headless=True,
            args=["--no-sandbox"]
        )

        page = browser.new_page()

        log("Abriendo login")

        page.goto(LOGIN_URL)
        page.wait_for_timeout(3000)

        page.fill('input[type="text"]', CEPM_USER)
        page.fill('input[type="password"]', CEPM_PASS)

        log("Enviando login")

        page.wait_for_selector("text=Entrar")
        page.locator("text=Entrar").click()

        page.wait_for_timeout(6000)

        current_url = page.url
        log(f"URL actual tras login: {current_url}")

        if "Account/Login" in current_url:

            log("EL LOGIN NO REDIRIGIO fuera de la pagina de login")

            html_login = page.content()

            mensaje_error = buscar_mensaje_error(html_login)
            if mensaje_error:
                log(f"Mensaje de error detectado en la pagina: {mensaje_error}")
            else:
                log("No se detecto un mensaje de error visible en la pagina")

            guardar_diagnostico(page, html_login)

            browser.close()

            resultado["login_status"] = f"fallo: {mensaje_error or 'sin mensaje visible'}"
            resultado["prepago_fecha_balance"] = datetime.now().strftime("%d/%b. %I:%M %p")
            resultado["timestamp"] = int(time.time())
            return resultado

        resultado["login_status"] = "ok"
        log("Login exitoso")

        html_home = page.content()

        balance_rd, balance_kwh, fecha = extract_balance_cuenta(
            html_home, CUENTA_PREPAGO
        )

        if fecha is None:
            fecha = datetime.now().strftime("%d/%b. %I:%M %p")
            log("Fecha no encontrada en scraping, usando fecha actual")

        resultado["prepago_balance_rd"] = balance_rd
        resultado["prepago_balance_kwh"] = balance_kwh
        resultado["prepago_fecha_balance"] = fecha

        log(f"Balance prepago RD$: {balance_rd} / kWh restante: {balance_kwh}")

        # --- Pagina de consumo (Nivel2), primero prepago, luego postpago ---

        try:
            log("Abriendo pagina de consumo (Nivel2) via boton del menu")

            try:
                page.click('button[name="consumo"]')
                page.wait_for_timeout(3000)
            except Exception:
                log("No se encontro el boton del menu")
                raise

            log(f"URL tras navegar a consumo: {page.url}")

            page.wait_for_selector("#Cuenta", timeout=15000)

            datos_prepago = obtener_datos_consumo(page, CUENTA_PREPAGO)
            resultado.update({f"prepago_{k}": v for k, v in datos_prepago.items()})

            datos_postpago = obtener_datos_consumo(page, CUENTA_POSTPAGO)
            resultado.update({f"postpago_{k}": v for k, v in datos_postpago.items()})

        except Exception:
            log("ERROR obteniendo datos de consumo (Nivel2)")
            log(traceback.format_exc())
            try:
                guardar_diagnostico(page, page.content())
            except Exception:
                pass

        browser.close()

        resultado["timestamp"] = int(time.time())

        return resultado


publish_discovery()

log("Entrando en bucle principal")

while True:

    try:
        log("Ejecutando scrape")

        data = scrape()

        log(f"Datos obtenidos: {data}")

        client.publish(
            f"{BASE_TOPIC}/state",
            json.dumps(data),
            retain=True
        )

        log("MQTT publicado correctamente")

    except Exception:

        log("ERROR DETECTADO")
        log(traceback.format_exc())

    log(f"Esperando siguiente ejecucion ({POLL_INTERVAL_MIN} minutos)")
    time.sleep(POLL_INTERVAL_MIN * 60)
SCRAPER_EOF

CMD ["python3", "-u", "/app/scraper.py"]
