# Explicación del código (blueprint) — Escenario "Ecosistema"

En esta práctica no hay código de Arduino. El "código" es el archivo **`Ecosistema.blueprint.json`**, que Make.com genera al exportar el escenario (**⋯ → Export Blueprint**). Es un archivo JSON que describe cada módulo, cómo está configurado y cómo se conecta con los demás. Si se importa en Make con **Import Blueprint**, el escenario se vuelve a armar igual.

---

## 1. Estructura general del archivo

El JSON tiene tres partes principales:

- **`name`**: el nombre del escenario, `"Ecosistema"`.
- **`flow`**: la lista de módulos, en el orden en que se ejecutan.
- **`metadata`**: la configuración general del escenario.

Cada módulo dentro de `flow` tiene los mismos campos:

- **`id`**: número único del módulo. Se usa para leer sus datos desde otros módulos. Por ejemplo, `{{3.message.chat.id}}` significa "el dato `message.chat.id` que entregó el módulo 3".
- **`module`**: qué app y qué acción usa. Por ejemplo, `telegram:WatchUpdates` es la app Telegram con la acción "Watch Updates".
- **`version`**: la versión del módulo.
- **`parameters`**: la conexión o webhook que usa el módulo, guardada como un número de identificación. No incluye el token del bot.
- **`mapper`**: los valores que se llenaron en el módulo. Aquí están los campos configurados y los datos que se toman de otros módulos.
- **`metadata`**: información para el editor de Make. `designer` guarda la posición (x, y) del módulo en el lienzo, `restore` guarda los nombres visibles (por ejemplo, "Ecosistemas Bot") y `setupValidation` indica si la configuración es válida.

**Sintaxis de las llaves:** todo lo que va entre `{{ }}` es un dato dinámico. `{{3.message.photo}}` toma la foto del mensaje que recibió el módulo 3, y `{{8.response}}` toma la respuesta que generó el módulo 8.

---

## 2. Módulo por módulo

### Módulo 3 — Telegram Bot: Watch Updates (disparador)

```json
"id": 3,
"module": "telegram:WatchUpdates",
"parameters": { "__IMTHOOK__": 2851662 }
```

Es el **inicio del escenario**. `__IMTHOOK__` es el webhook llamado "Ecosistemas". Cada vez que alguien le escribe al bot de Telegram, Telegram avisa a Make por este webhook y el escenario se ejecuta. No tiene `mapper` porque no se configura ningún campo: solo recibe el mensaje.

Los datos que entrega y que usan los demás módulos son:

- `message.photo`: la foto, si el mensaje trae una.
- `message.chat.id`: el chat del alumno, para saber a quién responder.

### Módulo 4 — Router

```json
"id": 4,
"module": "builtin:BasicRouter",
"routes": [ { "flow": [ ... ] }, { "flow": [ ... ] } ]
```

Divide el escenario en **dos rutas**. Cada elemento de `routes` es una ruta con su propio `flow`, es decir, su propia lista de módulos. Qué ruta se sigue lo decide el **filtro** del primer módulo de cada ruta.

---

### Ruta 1: el mensaje trae foto

#### Módulo 5 — Telegram Bot: Download a File

```json
"id": 5,
"module": "telegram:DownloadFile",
"filter": {
  "name": "Tiene Foto",
  "conditions": [[ { "a": "{{3.message.photo}}", "o": "exist" } ]]
},
"mapper": { "fileId": "{{last(3.message.photo)}}" }
```

- **`filter`**: es el filtro **"Tiene Foto"**. La condición dice: si el campo `3.message.photo` **existe** (`"o": "exist"`), el flujo continúa por esta ruta.
- **`mapper.fileId`**: Telegram manda cada foto en varios tamaños dentro de una lista. La función `last()` toma el **último elemento**, que es la foto de **mayor resolución**.
- **Resultado:** el módulo descarga la imagen y entrega `fileOutput` (los datos del archivo) y `fileName` (el nombre).

#### Módulo 8 — Make AI Agent: Run an agent

```json
"id": 8,
"module": "ai-local-agent:RunLocalAIAgent",
"mapper": {
  "files": [ { "data": "{{5.fileOutput}}", "fileName": "{{5.fileName}}" } ],
  "message": "Analiza la imagen adjunta siguiendo tus instrucciones.",
  "defaultModel": "medium",
  "outputType": "text",
  "tokenLimit": "50",
  "threadId": "",
  "systemPrompt": "...",
  "modelConfig": { "recursionLimit": "300", "iterationsFromHistoryCount": "10", "timeout": "" },
  "promptCaching": "none",
  "fallbackEnabled": false
}
```

Es el **módulo central**: la IA analiza la foto.

- **`files`**: le pasa al agente la imagen que descargó el módulo 5 (`5.fileOutput` y `5.fileName`).
- **`message`**: el texto de entrada que acompaña a la imagen.
- **`systemPrompt`**: las instrucciones del agente. Le indican que identifique el organismo de la foto y responda en tres líneas: nombre probable, nivel trófico (productor, consumidor o descomponedor) y su rol en el ecosistema. La respuesta debe tener máximo 35 palabras y estar en español. Si la foto no muestra un ser vivo, debe responder con un mensaje de error.
- **`defaultModel: "medium"`**: usa el modelo *Medium* de Make (gpt-5-nano con razonamiento bajo), que es rápido y económico.
- **`outputType: "text"`**: la respuesta sale como texto normal.
- **`tokenLimit: "50"`**: limita la longitud de la respuesta al 50 % del máximo permitido.
- **`threadId: ""`**: vacío, así que el agente no guarda conversación. Cada foto se analiza desde cero.
- **`modelConfig`**: `recursionLimit` es el número máximo de pasos del agente (300). `iterationsFromHistoryCount` es cuántos mensajes anteriores podría recordar (10). `timeout` vacío usa el valor por defecto.
- **`promptCaching: "none"`** y **`fallbackEnabled: false`**: sin caché de prompt y sin conexión de respaldo.
- **Resultado:** entrega `response`, el texto que generó la IA.

#### Módulo 9 — Telegram Bot: Send a Text Message or a Reply

```json
"id": 9,
"module": "telegram:SendReplyMessage",
"mapper": {
  "chatId": "{{3.message.chat.id}}",
  "text": "{{8.response}}",
  "parseMode": ""
}
```

Envía la **respuesta de la IA** al alumno.

- **`chatId`**: usa el chat del mensaje original (módulo 3), así la respuesta llega a la misma conversación.
- **`text`**: el texto generado por el Make AI Agent (módulo 8).
- **`parseMode` vacío**: se envía como texto plano, sin formato Markdown ni HTML.

---

### Ruta 2: el mensaje no trae foto

#### Módulo 6 — Telegram Bot: Send a Text Message or a Reply

```json
"id": 6,
"module": "telegram:SendReplyMessage",
"filter": {
  "name": "No tiene Foto",
  "conditions": [[ { "a": "{{3.message.photo}}", "o": "notexist" } ]]
},
"mapper": {
  "chatId": "{{3.message.chat.id}}",
  "text": "Envíame una foto 📷 del organismo (planta, insecto, hongo, etc.) para poder identificarlo"
}
```

- **`filter`**: es el filtro **"No tiene Foto"**. Si el campo `3.message.photo` **no existe** (`"o": "notexist"`), es decir, si el alumno mandó solo texto, el flujo sigue por aquí.
- **`mapper.text`**: es un **mensaje fijo** que le pide al alumno una foto. En esta ruta no se usa la IA, lo que ahorra operaciones.

---

## 3. Configuración general del escenario (`metadata`)

```json
"metadata": {
  "instant": true,
  "scenario": {
    "roundtrips": 1, "maxErrors": 3, "autoCommit": true,
    "autoCommitTriggerLast": true, "sequential": false,
    "confidential": false, "dataloss": false, "dlq": false
  },
  "zone": "us2.make.com"
}
```

- **`instant: true`**: el escenario es **instantáneo**. Se ejecuta en cuanto llega el aviso del webhook, sin esperar un horario.
- **`roundtrips: 1`**: procesa un mensaje por ejecución.
- **`maxErrors: 3`**: si falla 3 veces seguidas, Make desactiva el escenario.
- **`autoCommit` / `autoCommitTriggerLast`**: confirma automáticamente los cambios de cada ejecución.
- **`sequential: false`**: puede procesar varios mensajes al mismo tiempo, por ejemplo si varios alumnos mandan fotos a la vez.
- **`confidential: false`**: los datos de cada ejecución sí quedan guardados en el historial (History) de Make.
- **`dlq: false`**: no se guardan ejecuciones incompletas para reintentarlas después.
- **`dataloss: false`**: no se permite descartar datos si una ejecución excede los límites.
- **`zone: "us2.make.com"`**: el servidor de Make donde está la cuenta.

---

## 4. Resumen del funcionamiento

1. El alumno manda un mensaje al bot → **módulo 3** lo recibe por webhook.
2. El **Router (4)** revisa si el mensaje trae foto.
3. **Con foto:** el **módulo 5** la descarga → el **módulo 8** (IA) la analiza → el **módulo 9** envía la respuesta al alumno.
4. **Sin foto:** el **módulo 6** le pide al alumno que mande una foto.
