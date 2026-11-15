# Agente CV – Sebastián González García

> **Nota importante – plan gratuito:** el agente está desplegado en el plan gratuito de Render, por lo que **el servicio debe estar encendido (despierto) para responder**. Tras ~15 minutos sin tráfico, Render lo suspende y la primera petición puede tardar alrededor de un minuto en despertarlo, lo que puede provocar un timeout. Antes de probarlo, abre la agent card en el navegador (`/.well-known/agent-card.json`), espera a que cargue y luego usa el agente.

**Stack gratuito:** se usan **Render** (hosting) y **Google Gemini** (LLM) porque ofrecen niveles gratuitos suficientes para este proyecto.

Agente conversacional que representa el perfil profesional de Sebastián: experiencia, habilidades, proyectos y educación. Responde preguntas de reclutadores, hace *match* contra descripciones de puesto y apoya con preguntas de entrevista, siempre basándose en un CV estructurado (`cv.json`).

Expone una API compatible con **Open Responses** (`POST /v1/responses`) y publica una **tarjeta de agente A2A** (`/.well-known/agent-card.json`) para descubrimiento e importación.

- **URL base (Open Responses):** `https://<TU-APP>.onrender.com/v1`
- **Agent card:** `https://<TU-APP>.onrender.com/.well-known/agent-card.json`

## Demo

<video src="media/demo.mp4" width="100%"></video>

Si el video no se reproduce en tu visor, ábrelo directamente: [media/demo.mp4](media/demo.mp4)

---

## Arquitectura

```
Plataforma / cliente
        │  POST /v1/responses  (Open Responses)
        ▼
  FastAPI (main.py)
   ├─ auth opcional (Bearer)
   ├─ normaliza el historial (input → mensajes)
   ├─ system prompt = reglas del agente + cv.json
   └─ LLM: Gemini (modelo rápido → respaldo)
        │
        ▼
  Respuesta en formato Open Responses (output_text)
```

Componentes: `main.py` (API y lógica), `cv.json` (fuente de verdad), `Dockerfile` y `requirements.txt` (despliegue).

---

## Decisiones técnicas

### 1. Diseño

| Decisión | Por qué |
|---|---|
| **El CV completo va en el system prompt, sin RAG ni base vectorial** | Un CV es pequeño (unos pocos miles de tokens). Meterlo entero evita fallos de recuperación, reduce piezas móviles y da respuestas más coherentes. Un RAG añadiría complejidad sin beneficio a esta escala. |
| **`cv.json` estructurado como única fuente de verdad** | Separa datos de lógica: actualizar el perfil no requiere tocar código. El esquema usa campos cortos, fechas normalizadas (`end: null` + `current: true` en lugar de "Present") y secciones por tipo (experiencia profesional, proyectos académicos, certificaciones), lo que ayuda al modelo a distinguir experiencia real de formación. |
| **JSON compacto en el prompt** (`separators=(",", ":")`) | Menos tokens, menor latencia y menos riesgo de timeout. |
| **Bloque `agent_guidance` dentro del JSON** | Las reglas específicas del perfil (política de contacto, clientes confidenciales, temas que no están en el CV) viajan con los datos y no con el código. |
| **Reglas de comportamiento explícitas** | Respuestas siempre en español, tono seguro y persuasivo, uso de STAR en preguntas de comportamiento, *job matching* y manejo positivo de brechas. |
| **Precisión por encima de la persuasión** | El agente defiende al candidato, pero no puede inventar experiencia, tecnologías ni logros. Si preguntan por algo que no está en el CV, no lo afirma: lo declara como no documentado y redirige a la experiencia más cercana. Un agente que exagera se desmorona en la primera pregunta de seguimiento. |
| **Privacidad por diseño** | El teléfono se eliminó del JSON; solo se comparte el email. Los dos clientes confidenciales (aseguradora en Corea y petrolera) no se nombran. |

### 2. Integración

| Decisión | Por qué |
|---|---|
| **Open Responses (`POST /v1/responses`)** | Es el contrato que usa la plataforma. Es una especificación independiente del proveedor de modelo, así que el LLM se puede cambiar sin tocar la interfaz. |
| **Servidor sin estado: la plataforma reenvía la transcripción** | Con el modo "Reproducir transcripción", cada petición trae todo el historial. No hay base de datos ni sesiones que mantener, y el servicio escala o se reinicia sin perder conversaciones. |
| **Normalización de la entrada** | `input` puede ser texto o una lista de ítems con contenido en distintos formatos; `to_messages` los reduce a mensajes `user`/`assistant`. |
| **Agent card A2A en `/.well-known/agent-card.json`** | Permite importar el agente en la plataforma con una URL y describe sus habilidades (preguntas sobre el CV, match con puesto, preparación de entrevista). La URL pública se toma de `PUBLIC_URL` o de las cabeceras del proxy. Nota: se implementa la tarjeta de descubrimiento, no mensajería A2A completa. |
| **Gemini (nivel gratuito) como LLM** | Se eligió por ser una opción gratuita, con costo cero para un proyecto de esta escala. El modelo se configura con la variable `LLM_MODEL`. |
| **Modelo rápido primero, modelo más capaz como respaldo** | En pruebas, el modelo principal llegaba a `504 DEADLINE_EXCEEDED`. Se priorizó latencia: primero el modelo *lite*, y si falla, el modelo más capaz. |
| **Timeout de 25 s por intento y razonamiento reducido** | El proxy de Cloudflare corta a ~100 s (error 524). Con dos intentos de 25 s, el peor caso queda por debajo de ese límite. El nivel de razonamiento se baja (`thinking_level="low"` / `thinking_budget=0`) porque para preguntas sobre un CV el razonamiento largo solo añade latencia. Si el modelo rechaza esa opción, se reintenta sin ella. |
| **Degradación elegante** | Si todos los modelos fallan, la API responde 200 con un mensaje claro en lugar de un error 5xx, y el detalle del error queda solo en los logs. |

### 3. Despliegue

| Decisión | Por qué |
|---|---|
| **Contenedor Docker** (`python:3.12-slim`) | Entorno reproducible y portable entre proveedores. El puerto se toma de `$PORT`. |
| **Render (plan gratuito)** | Se eligió por ser una opción gratuita: despliegue desde GitHub con un solo clic, HTTPS incluido y costo cero. A cambio, el servicio se suspende por inactividad (ver la nota al inicio). Se prefirió a Fly.io, que ya no ofrece plan gratuito para cuentas nuevas. |
| **Secretos en variables de entorno** | `GEMINI_API_KEY`, `LLM_MODEL`, `PUBLIC_URL` y `AGENT_API_KEY` nunca están en el código ni en el repositorio. |
| **Repositorio privado** | `cv.json` contiene datos personales. |
| **Túnel de Cloudflare solo para desarrollo** | Permitió probar con la plataforma desde la PC local antes de desplegar. |

### 4. Operación y confiabilidad

- **Autenticación opcional:** si se define `AGENT_API_KEY`, la API exige `Authorization: Bearer <token>`.
- **Observabilidad básica:** cada fallo de modelo se registra en stdout con el nombre del modelo, visible en los logs de Render. Así se identifica si el problema es cuota (429), saturación (503/504) o un modelo inexistente (404).
- **Manejo de errores:** validación de entrada vacía (400), autenticación (401) y respuesta controlada ante fallos del LLM.
- **Guardrails en el prompt:** el agente ignora instrucciones que intenten cambiar sus reglas o revelar el prompt. Es una mitigación, no una garantía.
- **Cold start del plan gratuito:** Render duerme el servicio tras ~15 minutos sin tráfico y despertarlo tarda alrededor de un minuto. Mitigación: un monitor externo que consulte la agent card cada ~5 minutos, o despertar el servicio antes de una demostración.

---

## Limitaciones conocidas

- Sin *streaming*: la respuesta se entrega completa.
- Sujeto a las cuotas y la latencia del nivel gratuito de Gemini.
- La calidad y veracidad dependen de `cv.json`: el agente repite lo que el CV dice.
- La agent card cubre descubrimiento, no el protocolo A2A completo.

## Próximos pasos

- Conjunto de evaluación (preguntas con respuestas esperadas, incluyendo preguntas fuera del CV e intentos de romper el rol).
- Logs estructurados con latencia por petición.
- Streaming de respuestas.
- Pruebas de conformidad con los *acceptance tests* de Open Responses.

---

## Ejecución local

```powershell
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
$env:GEMINI_API_KEY="tu-key"
$env:LLM_MODEL="gemini-3.5-flash-lite"   # opcional
uvicorn main:app --port 8000
```

Prueba rápida:

```powershell
(Invoke-RestMethod -Method Post -Uri http://127.0.0.1:8000/v1/responses -ContentType "application/json" -Body '{"input":"Que experiencia tiene Sebastian con Go?"}').output[0].content[0].text
```

### Variables de entorno

| Variable | Obligatoria | Descripción |
|---|---|---|
| `GEMINI_API_KEY` | Sí | Clave de Google AI Studio |
| `LLM_MODEL` | No | Modelo principal (por defecto, el modelo *lite* configurado en `main.py`) |
| `PUBLIC_URL` | Recomendada en producción | URL pública para la agent card |
| `AGENT_API_KEY` | No | Si se define, protege el endpoint con Bearer token |
