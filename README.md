# 🌤️ Clima Automatizado — Bot de Telegram con n8n

> Proyecto de automatización meteorológica para el área metropolitana de Santander, Colombia. Envía reportes de clima en tiempo real a Telegram, guarda históricos en MySQL y Google Sheets, y usa IA (GPT) para mejorar la redacción del mensaje.

---

## 📋 Tabla de Contenidos

- [Descripción General](#-descripción-general)
- [Tecnologías Utilizadas](#-tecnologías-utilizadas)
- [Arquitectura del Flujo](#-arquitectura-del-flujo)
- [Explicación Nodo por Nodo](#-explicación-nodo-por-nodo)
  - [1. Schedule Trigger](#1-schedule-trigger)
  - [2. Code in JavaScript1 — Ciudades](#2-code-in-javascript1--ciudades)
  - [3. HTTP Request — Open-Meteo API](#3-http-request--open-meteo-api)
  - [4. Code in JavaScript — Procesamiento y Mensaje](#4-code-in-javascript--procesamiento-y-mensaje)
  - [5. Limit — Filtrar 1 Item](#5-limit--filtrar-1-item)
  - [6. Insert rows in a table — MySQL](#6-insert-rows-in-a-table--mysql)
  - [7. Append row in sheet — Google Sheets](#7-append-row-in-sheet--google-sheets)
  - [8. Message a model — OpenAI GPT](#8-message-a-model--openai-gpt)
  - [9. Send a text message — Telegram](#9-send-a-text-message--telegram)
- [Diagrama del Flujo](#-diagrama-del-flujo)
- [Instalación con Docker](#-instalación-con-docker)
- [Variables y Credenciales](#-variables-y-credenciales)
- [Estructura de la Base de Datos MySQL](#-estructura-de-la-base-de-datos-mysql)
- [Ejemplo de Mensaje en Telegram](#-ejemplo-de-mensaje-en-telegram)
- [Autor](#-autor)

---

## 📌 Descripción General

Este proyecto automatiza la consulta del clima actual de tres municipios del área metropolitana de **Bucaramanga, Santander** (Colombia):

- 🏙️ Girón
- 🏙️ Bucaramanga
- 🏙️ Floridablanca

Cada hora, el flujo:
1. Consulta la **API gratuita de Open-Meteo** para obtener temperatura y velocidad del viento.
2. Clasifica las ciudades según si superan los **25°C**.
3. Construye un **reporte formateado con emojis**.
4. Lo mejora con **Inteligencia Artificial (GPT)**.
5. Lo envía a un **chat de Telegram**.
6. Guarda el registro en **MySQL** y en **Google Sheets** como histórico.

---

## 🛠️ Tecnologías Utilizadas

| Tecnología | Uso |
|---|---|
| **n8n** | Motor de automatización (self-hosted con Docker) |
| **Docker** | Contenedor para ejecutar n8n |
| **Open-Meteo API** | API REST gratuita de clima (sin API key) |
| **JavaScript** | Lógica de procesamiento dentro de n8n |
| **OpenAI GPT-5-Nano** | Mejora la redacción del reporte meteorológico |
| **Telegram Bot API** | Envío del mensaje final al usuario |
| **MySQL** | Base de datos para guardar registros de clima |
| **Google Sheets** | Hoja de cálculo como histórico visual |

---

## 🗺️ Arquitectura del Flujo

```
[Schedule Trigger]
       │
       ▼
[Code JS1: Ciudades]  ──── Define Girón, Bucaramanga, Floridablanca
       │ (3 items)
       ▼
[HTTP Request]  ──── Consulta Open-Meteo por cada ciudad
       │ (3 items)
       ▼
[Code JS: Procesar]  ──── Construye registros + mensaje de texto
       │
       ├──► [Insert MySQL]       ──── Guarda temperatura y viento (3 filas)
       ├──► [Append Google Sheets] ── Guarda con fecha y alerta (3 filas)
       └──► [Limit: 1 item]
                 │
                 ▼
           [Message GPT] ──── Mejora la redacción del reporte
                 │
                 ▼
           [Send Telegram] ──── Envía el reporte al chat
```

---

## 🔍 Explicación Nodo por Nodo

### 1. Schedule Trigger

**¿Qué es?** Es el nodo de inicio del flujo. Actúa como un "despertador" que activa automáticamente el workflow según un horario definido.

**Configuración:**
- **Trigger Interval:** Hours (cada hora)
- **Hours Between Triggers:** 1
- **Trigger at Minute:** 0

**¿Para qué sirve?** Permite que el bot opere de forma completamente autónoma sin intervención humana. Cada hora en punto, n8n despierta el workflow y lo ejecuta desde el principio.

**Output que genera:**
```json
{
  "timestamp": "2026-03-05T11:59:51.702-05:00",
  "Readable date": "March 5th 2026, 11:59:51 am",
  "Day of week": "Thursday",
  "Timezone": "America/New_York (UTC-05:00)"
}
```

---

### 2. Code in JavaScript1 — Ciudades

**¿Qué es?** Un nodo de código JavaScript que define manualmente las ciudades que se van a monitorear, junto con sus coordenadas geográficas (latitud y longitud).

**Código utilizado:**
```javascript
return [
  {
    name: "Giron",
    lat: 7.0678,
    lon: -73.1698
  },
  {
    name: "Bucaramanga",
    lat: 7.125,
    lon: -73.1198
  },
  {
    name: "Floridablanca",
    lat: 7.0622,
    lon: -73.0864
  }
];
```

**¿Para qué sirve?** Genera una lista de 3 items (uno por ciudad). Cada item contiene el nombre de la ciudad y sus coordenadas exactas, que serán usadas en el siguiente nodo para consultar la API del clima.

**Output que genera:** 3 items con estructura `{ name, lat, lon }`.

> 💡 **Concepto clave para el aprendiz:** En n8n, cuando un nodo retorna un arreglo con 3 objetos, el siguiente nodo recibe 3 "items" y se ejecuta 3 veces (una por cada ciudad).

---

### 3. HTTP Request — Open-Meteo API

**¿Qué es?** Un nodo que realiza una solicitud HTTP de tipo GET a la API pública y gratuita de Open-Meteo para obtener el clima actual de cada ciudad.

**URL configurada:**
```
https://api.open-meteo.com/v1/forecast?latitude={{$json["lat"]}}&longitude={{$json["lon"]}}&current_weather=true
```

**¿Cómo funciona?** Los valores `{{$json["lat"]}}` y `{{$json["lon"]}}` son **expresiones dinámicas** de n8n. Por cada item que llega (cada ciudad), toma las coordenadas del item actual e inserta los valores reales en la URL antes de hacer la petición.

**¿Para qué sirve?** Conecta el flujo con la API de clima. La API devuelve temperatura actual, velocidad del viento, dirección del viento, y código meteorológico, sin necesidad de una API key.

**Output relevante por ciudad:**
```json
{
  "current_weather": {
    "temperature": 25.3,
    "windspeed": 3.1,
    "winddirection": 234,
    "weathercode": 95
  }
}
```

> 💡 **Concepto clave:** Este nodo se ejecuta 3 veces (una por cada ciudad), gracias a que recibe 3 items del nodo anterior.

---

### 4. Code in JavaScript — Procesamiento y Mensaje

**¿Qué es?** El nodo más importante del flujo. Procesa todos los datos climáticos, clasifica las ciudades por temperatura, construye el mensaje final para Telegram y prepara los datos para guardar en la base de datos.

**Código completo utilizado:**
```javascript
const items = $input.all();
const originalItems = $items("Code in JavaScript1");

let ciudadesCalor = [];
let ciudadesNormal = [];

// Construimos registros individuales (para MySQL)
const registros = items.map((item, index) => {
  const weather = item.json.current_weather;
  const ciudadOriginal = originalItems[index].json.name;

  const temperatura = weather.temperature;
  const viento = weather.windspeed;

  const registro = {
    json: {
      ciudad: ciudadOriginal,
      temperatura: temperatura,
      viento: viento
    }
  };

  // Clasificación por temperatura
  if (temperatura > 25) {
    ciudadesCalor.push(
      `🔥 ${ciudadOriginal}: ${temperatura}°C • viento ${viento} km/h`
    );
  } else {
    ciudadesNormal.push(
      `🌤️ ${ciudadOriginal}: ${temperatura}°C • viento ${viento} km/h`
    );
  }

  return registro;
});

// Construcción del mensaje final para Telegram
let mensajeFinal = `🌎 *Reporte del Clima - Santander*\n\n`;

if (ciudadesCalor.length > 0) {
  mensajeFinal += `🚨 *Temperaturas mayores a 25°C*\n`;
  mensajeFinal += ciudadesCalor.join("\n");
  mensajeFinal += `\n\n`;
}

if (ciudadesNormal.length > 0) {
  mensajeFinal += `✅ *Temperaturas 25°C o menores*\n`;
  mensajeFinal += ciudadesNormal.join("\n");
  mensajeFinal += `\n\n`;
}

if (ciudadesCalor.length > 0) {
  mensajeFinal += `⚠️ Recomendación: Mantente hidratado y evita exposición prolongada al sol.`;
} else {
  mensajeFinal += `Que tengas un excelente día 😊`;
}

// El mensaje se agrega SOLO al primer item (para Telegram)
registros[0].json.mensajeClima = mensajeFinal;

return registros;
```

**¿Qué hace paso a paso?**

| Paso | Descripción |
|---|---|
| `$input.all()` | Obtiene los 3 items con datos climáticos de la API |
| `$items("Code in JavaScript1")` | Recupera los nombres originales de las ciudades |
| `.map(...)` | Itera sobre cada ciudad y construye un objeto limpio con `ciudad`, `temperatura` y `viento` |
| `if (temperatura > 25)` | Clasifica la ciudad en "calor" o "normal" |
| Construcción del mensaje | Ensambla el texto con emojis y formato Markdown de Telegram |
| `registros[0].json.mensajeClima` | Agrega el mensaje completo solo al **primer item** para evitar enviarlo 3 veces a Telegram |

**Output que genera:** 3 items listos para MySQL/Sheets + el `mensajeClima` adjunto al primero.

---

### 5. Limit — Filtrar 1 Item

**¿Qué es?** Un nodo utilitario que filtra el flujo y deja pasar solo el **primer item** (Max Items: 1, Keep: First Items).

**¿Para qué sirve?** Como el mensaje de Telegram está adjunto únicamente al primer item (Girón), este nodo asegura que solo ese item llegue al nodo de GPT y al de Telegram. Sin este nodo, el mensaje se enviaría 3 veces (una por cada ciudad).

> 💡 **Concepto clave:** Este es un patrón común en n8n: se procesan N items en paralelo, pero al final se "colapsa" a 1 item para hacer una sola acción de notificación.

---

### 6. Insert rows in a table — MySQL

**¿Qué es?** Un nodo que inserta los datos de clima en una tabla de MySQL llamada `clima_registros`.

**Columnas mapeadas:**

| Columna MySQL | Valor n8n |
|---|---|
| `ciudad` | `{{$json.ciudad}}` |
| `temperatura` | `{{$json.temperatura}}` |
| `viento` | `{{$json.viento}}` |

**¿Para qué sirve?** Permite mantener un registro persistente y consultable de los datos climáticos. Al ejecutarse 3 veces (una por ciudad), inserta 3 filas por cada ejecución del workflow.

**Output:** `{ "success": true }`

> 💡 **Concepto para el aprendiz:** La tabla MySQL actúa como una base de datos estructurada. Luego puedes hacer consultas SQL como `SELECT * FROM clima_registros WHERE ciudad = 'Giron'` para analizar el histórico.

---

### 7. Append row in sheet — Google Sheets

**¿Qué es?** Un nodo que agrega una fila al final de una hoja de Google Sheets llamada `clima` en el documento `Histórico Climabot`.

**Columnas mapeadas:**

| Columna Sheets | Valor n8n |
|---|---|
| `Fecha` | `{{ $now }}` (fecha y hora actual) |
| `Ciudad` | `{{$json.ciudad}}` |
| `Temperatura` | `{{$json.temperatura}}` |
| `Viento` | `{{$json.viento}}` |
| `Alerta` | `{{ $json.temperatura > 25 ? "SI" : "NO" }}` |

**¿Para qué sirve?** Actúa como un histórico visual y accesible. La columna `Alerta` usa un **operador ternario** para mostrar "SI" si la temperatura supera los 25°C o "NO" en caso contrario. Es útil para revisar tendencias de temperatura sin necesidad de acceder a la base de datos.

---

### 8. Message a model — OpenAI GPT

**¿Qué es?** Un nodo que envía el `mensajeClima` generado por el código JavaScript al modelo **GPT-5-Nano** de OpenAI, junto con un prompt de sistema que le indica cómo debe tratar el texto.

**Prompt del sistema:**
```
Eres un asistente meteorológico profesional.

Toma el siguiente reporte y devuélvelo EXACTAMENTE con la misma 
estructura y clasificación.

NO lo resumas.
NO lo simplifiques.
NO elimines secciones.
Respeta los títulos y emojis.

Solo mejora ligeramente la redacción si es necesario, pero mantén 
la división entre:
- Temperaturas mayores a 25°C
- Temperaturas 25°C o menores

Aquí está el reporte:
{{$json.mensajeClima}}
```

**¿Para qué sirve?** Añade una capa de Inteligencia Artificial que puede pulir la redacción del mensaje antes de enviarlo. El prompt está diseñado con restricciones estrictas para que GPT no altere la estructura ni los datos, solo mejore el lenguaje si es necesario.

**Output relevante:** `output[0].content[0].text` con el mensaje mejorado.

---

### 9. Send a text message — Telegram

**¿Qué es?** El nodo final del flujo. Envía el mensaje procesado por GPT a un chat de Telegram usando la API del bot.

**Configuración:**
- **Chat ID:** `5792196708` (ID del usuario o grupo destino)
- **Text:** `{{$json["output"][0]["content"][0]["text"]}}` (el texto devuelto por GPT)

**¿Para qué sirve?** Es la "salida visible" del workflow: el usuario final recibe el reporte del clima directamente en su Telegram, con formato Markdown (negrita con `*texto*`), emojis y clasificación clara.

**Output confirmación:**
```json
{
  "ok": true,
  "result": {
    "message_id": 50,
    "from": { "first_name": "Climabot", "username": "Climabot2026_bot" },
    "chat": { "first_name": "Si se puede", "type": "private" }
  }
}
```

---

## 📊 Diagrama del Flujo

```
┌─────────────────┐
│  Schedule       │  → Se activa cada 1 hora
│  Trigger        │
└────────┬────────┘
         │ 1 item (timestamp)
         ▼
┌─────────────────┐
│  Code JS1       │  → Define 3 ciudades con lat/lon
│  (Ciudades)     │
└────────┬────────┘
         │ 3 items
         ▼
┌─────────────────┐
│  HTTP Request   │  → Consulta Open-Meteo (3 veces)
│  Open-Meteo API │
└────────┬────────┘
         │ 3 items (con datos climáticos)
         ▼
┌─────────────────┐
│  Code JS        │  → Clasifica, construye mensaje
│  (Procesamiento)│
└────────┬────────┘
         │ 3 items
    ┌────┴──────────────────┐
    │           │           │
    ▼           ▼           ▼
┌───────┐  ┌────────┐  ┌─────────┐
│ MySQL │  │Sheets  │  │ Limit   │
│Insert │  │Append  │  │(1 item) │
└───────┘  └────────┘  └────┬────┘
                             │ 1 item
                             ▼
                      ┌─────────────┐
                      │  OpenAI GPT │  → Mejora redacción
                      │  Message    │
                      └──────┬──────┘
                             │ 1 item
                             ▼
                      ┌─────────────┐
                      │  Telegram   │  → Envía reporte
                      │  Send Msg   │
                      └─────────────┘
```

---

## 🐳 Instalación con Docker

Este proyecto fue ejecutado con n8n en un contenedor Docker. A continuación los pasos para reproducirlo:

### Prerrequisitos

- Docker instalado en tu máquina
- Docker Compose (recomendado)

### Paso 1 — Ejecutar n8n desde la consola Linux

Si tienes Docker instalado, puedes levantar n8n directamente con el siguiente comando en la terminal:

```bash
docker run -it --rm --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```

**Explicación de cada parámetro:**

| Parámetro | Descripción |
|---|---|
| `docker run` | Crea y ejecuta un nuevo contenedor |
| `-it` | Modo interactivo, muestra los logs en la terminal |
| `--rm` | Elimina el contenedor automáticamente al detenerlo |
| `--name n8n` | Le asigna el nombre `n8n` al contenedor |
| `-p 5678:5678` | Expone el puerto 5678 del contenedor al puerto 5678 de tu máquina |
| `-v n8n_data:/home/node/.n8n` | Crea un volumen persistente para guardar los workflows y credenciales |
| `docker.n8n.io/n8nio/n8n` | Imagen oficial de n8n |

> ⚠️ **Nota:** Con `--rm` los datos del contenedor se eliminan al apagarlo, pero el volumen `n8n_data` **persiste**, por lo que tus workflows quedan guardados.

Después de ejecutar el comando, abre tu navegador en:
```
http://localhost:5678
```

---

### Paso 2 — Crear el archivo `docker-compose.yml`

```yaml
version: "3.8"

services:
  n8n:
    image: n8nio/n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=campus2023
      - N8N_BASIC_AUTH_PASSWORD=campus2023
      - GENERIC_TIMEZONE=America/Bogota
      - TZ=America/Bogota
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```

### Paso 3 — Iniciar el contenedor

```bash
docker-compose up -d
```

### Paso 4 — Acceder a n8n

Abre tu navegador en: `http://localhost:5678`

### Paso 5 — Importar el workflow

1. En n8n, ve al menú principal → **Workflows**
2. Haz clic en **Import from file**
3. Selecciona el archivo `automatizado.json`
4. Configura las credenciales (OpenAI, Telegram, MySQL, Google Sheets)

---

## 🔐 Variables y Credenciales

| Credencial | Descripción |
|---|---|
| **OpenAI API** | Clave de API de OpenAI para usar GPT-5-Nano |
| **Telegram API** | Token del bot de Telegram (obtenido desde @BotFather) |
| **MySQL** | Host, usuario, contraseña y base de datos MySQL |
| **Google Sheets OAuth2** | Cuenta de Google con acceso a la hoja de cálculo |

> ⚠️ **Importante:** Nunca compartas tus credenciales en el repositorio. Usa variables de entorno o el gestor de credenciales de n8n.

---

## 🗄️ Estructura de la Base de Datos MySQL

Para que el nodo de MySQL funcione correctamente, debes crear la tabla con el siguiente script SQL:

```sql
CREATE DATABASE IF NOT EXISTS clima_db;

USE clima_db;

CREATE TABLE clima_registros (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ciudad VARCHAR(100) NOT NULL,
    temperatura DECIMAL(5, 2) NOT NULL,
    viento DECIMAL(5, 2) NOT NULL,
    fecha_registro TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Descripción de las columnas:**

| Columna | Tipo | Descripción |
|---|---|---|
| `id` | INT AUTO_INCREMENT | Identificador único (llave primaria) |
| `ciudad` | VARCHAR(100) | Nombre de la ciudad |
| `temperatura` | DECIMAL(5,2) | Temperatura en grados Celsius |
| `viento` | DECIMAL(5,2) | Velocidad del viento en km/h |
| `fecha_registro` | TIMESTAMP | Fecha y hora de inserción automática |

---

## 📱 Ejemplo de Mensaje en Telegram

Así se ve el mensaje que recibe el usuario en Telegram:

```
🌎 Reporte del Clima - Santander

🚨 Temperaturas mayores a 25°C
🔥 Giron: 25.3°C • viento 3.1 km/h
🔥 Floridablanca: 25.4°C • viento 7.9 km/h

✅ Temperaturas 25°C o menores
🌤️ Bucaramanga: 24.8°C • viento 5.4 km/h

⚠️ Recomendación: Mantente hidratado y evita exposición prolongada al sol.
```

---

## 👨‍💻 Autor

Proyecto desarrollado como práctica de automatización con n8n, integrando APIs REST, bases de datos, hojas de cálculo e inteligencia artificial.

**Herramientas utilizadas:**
- n8n (self-hosted con Docker)
- API Open-Meteo (gratuita, sin registro)
- OpenAI GPT-5-Nano
- Telegram Bot API
- MySQL
- Google Sheets

---

