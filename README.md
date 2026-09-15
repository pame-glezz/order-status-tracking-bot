# Seguimiento de pedidos con IA — WhatsApp + Make + Claude

Automatización para e-commerce que atiende consultas de seguimiento de pedidos por WhatsApp, clasifica automáticamente la gravedad del caso con IA, y escala a un humano solo cuando realmente hace falta — con aprobación humana antes de responder al cliente en los casos delicados.

Este es el proyecto #2 de mi portafolio de automatización para e-commerce, después de un sistema de clasificación de preguntas de compradores con Human-in-the-loop (Make + Airtable + Claude + Slack + Gmail).

---

## 🧩 El problema que resuelve

En cualquier tienda online, "¿dónde está mi pedido?" representa entre el 30% y 50% de los tickets de soporte. Es información que ya existe en el sistema, pero que un agente humano tiene que buscar y comunicar una y otra vez — el peor uso posible de tiempo humano especializado.

**Sin esta automatización:**
- Un agente gasta minutos por consulta repitiendo la misma información.
- Los casos que sí importan (retrasos graves, extravíos, daños) se ahogan en el mismo canal que las consultas simples.
- El cliente espera horas por una respuesta que podría ser instantánea.

**Con esta automatización:** el sistema resuelve solo las consultas simples, y escala únicamente los casos que de verdad requieren criterio humano — con todo el contexto ya armado para que el humano decida rápido.

---

## 🗺️ Diagrama de flujo

```
Cliente escribe por WhatsApp
        │
        ▼
┌───────────────────┐
│  Detecta número    │   Busca un patrón "PED-XXX"
│  de pedido en el    │   en el texto del mensaje
│  mensaje (Text      │
│  Parser)            │
└───────────────────┘
        │
        ├── No lo encuentra ──► Responde pidiendo el número de pedido (fin)
        │
        └── Sí lo encuentra
                │
                ▼
        ┌───────────────┐
        │  Busca el       │   Google Sheets, columna
        │  pedido         │   numero_pedido
        └───────────────┘
                │
                ▼
        ┌───────────────┐
        │  Claude         │   Clasifica: intención,
        │  clasifica      │   urgencia, ¿requiere humano?,
        │                 │   respuesta sugerida
        └───────────────┘
                │
                ▼
        ┌───────────────┐
        │  Limpieza de    │   Extrae el JSON puro de
        │  JSON           │   la respuesta de Claude
        └───────────────┘
                │
                ▼
          ┌───────────┐
          │  Router     │  ¿requiere_humano?
          └───────────┘
                │
     ┌──────────┴──────────┐
     ▼                      ▼
  false                   true
     │                      │
     ▼                      ▼
Notifica por            Avisa a un humano por
Telegram con la          Telegram con el contexto
respuesta sugerida       completo del caso
(⚠️ escalamiento)
                              │
                              ▼
                    Humano responde en Telegram
                    (formato: "teléfono: mensaje")
                              │
                              ▼
                    Se reenvía al cliente por
                    WhatsApp (Escenario 2)
```

⚠️ **Estado actual de construcción:** en esta versión, **ambas ramas del Router notifican a Telegram** — una con la etiqueta "Respuesta automática" (caso normal, con la `respuesta_sugerida` ya redactada por Claude) y otra con "⚠️ Caso requiere atención" (caso grave, con todo el contexto). El envío directo al cliente por WhatsApp para el caso normal (usando Twilio) fue la intención original de diseño, pero **no está reconectado en la versión que se dejó funcionando** — queda como el siguiente paso pendiente antes de considerar el Escenario 1 completo end-to-end.

---

## ⚙️ Por qué el flujo pide el número de pedido en 2 turnos

La primera versión de este proyecto intentaba identificar al cliente automáticamente por su número de teléfono, cruzándolo contra una base de datos de pedidos. Se abandonó ese diseño por una razón técnica real (documentada en la sección de resiliencia más abajo), y se reemplazó por un flujo más simple y —de hecho— más realista: la mayoría de los sistemas de soporte reales piden explícitamente el número de orden como primer paso, en vez de asumir que pueden identificar al cliente solo por su teléfono.

**Turno 1:** el cliente pregunta sin dar contexto ("¿cómo va mi pedido?") → el sistema responde pidiendo el número de pedido.

**Turno 2:** el cliente escribe su número de pedido → el sistema lo busca, lo clasifica, y responde o escala según corresponda.

Cada mensaje se evalúa de forma independiente (Make no necesita "recordar" la conversación): un Text Parser revisa si el texto entrante contiene un patrón `PED-\d+`, y esa detección es el diferenciador entre ambos turnos.

---

## 🧰 Stack y por qué se eligió cada pieza

| Función | Herramienta | Por qué |
|---|---|---|
| Canal del cliente | WhatsApp (Twilio Sandbox) | Canal real de e-commerce; el sandbox permite probar sin aprobación de WhatsApp Business |
| Base de datos de pedidos | Google Sheets | Simple de inspeccionar y editar a mano durante pruebas; suficiente para el alcance de un prototipo |
| Clasificación de intención | Claude Haiku 4.5 | Tarea de clasificación + redacción corta: no requiere el razonamiento de un modelo más grande, y es notablemente más barato para iterar sin restricción |
| Canal del humano (HITL) | Telegram Bot | Gratis sin límites de prueba, configuración en minutos, y funciona como una "bandeja de soporte" simulada |

**Nota de diseño honesta:** en un e-commerce real, el canal del agente humano normalmente sería una plataforma de atención dedicada (Zendesk, Freshdesk, Gorgias, o el WhatsApp Business Inbox del equipo), no Telegram personal. Se usó Telegram aquí específicamente por ser gratuito y rápido de configurar para un proyecto de portafolio — la lógica de negocio (recibir el aviso, decidir, responder) es la misma que se usaría con cualquier herramienta profesional.

---

## 🗂️ Modelo de datos

Hoja de cálculo `pedidos_ecommerce`:

| Campo | Tipo | Ejemplo |
|---|---|---|
| numero_pedido | Texto | PED-001 |
| telefono_cliente | Texto | +5215585532763 |
| estado | Texto | retrasado / a tiempo |
| fecha_estimada | Texto | 2 de septiembre |
| dias_retraso | Número | 10 |

⚠️ **Limitación consciente:** el sistema busca por `numero_pedido` exacto (patrón `PED-\d+` en el texto). No maneja variantes de escritura más allá de eso (ej. "cero cero uno" en vez de "001"), ni el caso de un cliente con múltiples pedidos activos simultáneos preguntando de forma ambigua — son extensiones razonables para una v2, no defectos del alcance actual.

---

## 🧠 Clasificación con IA

Claude recibe el mensaje del cliente junto con el estado real del pedido, y responde en JSON con 4 campos: `intencion`, `urgencia`, `requiere_humano`, y `respuesta_sugerida`. La regla de escalamiento: se marca `requiere_humano: true` si el cliente expresa frustración fuerte, si el retraso supera 5 días, o si menciona daño o extravío — en cualquier otro caso, Claude redacta directamente la respuesta al cliente.

Esto es lo que hace que la IA sea necesaria (no un bot de reglas fijas): interpretar lenguaje libre y variable ("llevo una semana esperando y nadie me dice nada" vs. "¿ya casi llega?") es justo el tipo de ambigüedad que un árbol de decisiones no maneja bien.

---

## 🛡️ Manejo de errores y resiliencia (lecciones reales del proceso)

Esta sección documenta problemas reales encontrados durante la construcción, no hipotéticos — es, en mi opinión, la parte más valiosa del proyecto para demostrar criterio técnico.

**1. Funciones de texto que no se evaluaban dentro de campos de mapeo.**
Al intentar limpiar el prefijo `whatsapp:` de un número de teléfono con la función `replace()`, escrita a mano dentro de un campo de Make, la función nunca se ejecutaba — se guardaba como texto literal en vez de evaluarse, sin importar el separador de argumentos usado ni el módulo de destino (se probó en un filtro de Data Store, un filtro de Google Sheets, un campo de Telegram, y un módulo dedicado "Set variable"). En vez de seguir depurando ese comportamiento puntual, se rediseñó el flujo para no depender de esa transformación de texto en absoluto: comparar el número de pedido (texto simple, sin prefijos) en vez del teléfono. Lección: cuando una pieza técnica específica se resiste después de varios intentos razonables de diagnóstico, a veces la solución más eficiente es rediseñar para evitar esa dependencia, no insistir en resolverla a toda costa.

**2. JSON de Claude envuelto en marcadores de código.**
El modelo ocasionalmente devolvía el JSON envuelto en ` ```json ... ``` ` a pesar de instrucciones explícitas en el prompt pidiendo JSON puro. Solución: un módulo de Text Parser (`(\{[\s\S]*\})`, con paréntesis de captura) entre Claude y el parser de JSON, que extrae solo el bloque `{...}` sin importar qué texto lo rodee — resiliente independientemente de si el modelo decide envolver la respuesta o no.

**3. Comparación de booleanos con el operador equivocado.**
El Router que decide entre respuesta automática y escalamiento comparaba el campo `requiere_humano` (booleano) usando un operador de texto contra el valor `"False"` con mayúscula — nunca coincidía con el valor real (`false`, minúsculas, tipo lógico) que regresa el parser de JSON. Corregido usando el operador específico para booleanos ("Boolean operators: Equal to") contra `true`/`false` en minúsculas.

**4. Límite de cuota en cuentas de prueba.**
Twilio Sandbox (plan trial) limita a 5 mensajes salientes por día. Al encontrar el error `RateLimitError` (código 63038), se confirmó revisando el log de ejecución que el mensaje se había armado correctamente (destinatario, cuerpo, remitente) — el fallo era de cuota, no de configuración. Distinguir esto evitó "arreglar" algo que no estaba roto.

---

## 💰 Comparativo de costos (Claude Haiku vs. Sonnet)

Para esta tarea (clasificar un mensaje corto + redactar una respuesta breve), se eligió **Claude Haiku 4.5** por ser significativamente más económico y rápido que un modelo de la familia Sonnet, sin sacrificar calidad perceptible en una tarea de esta complejidad.

⚠️ Los precios exactos por token de cada modelo cambian con cierta frecuencia — antes de publicar una cifra específica de costo por conversación en este README, verifica el precio vigente en [docs.claude.com/en/docs/about-claude/models](https://docs.claude.com/en/docs/about-claude/models) y complétalo aquí con el cálculo real (tokens de entrada + salida por ejecución × número de mensajes esperados al mes).

---

## 🏗️ Estructura del escenario en Make

**Escenario 1 — Recepción y clasificación:**
Webhook (Twilio) → Text Parser (detecta número de pedido) → Router → [sin pedido: responde pidiendo el número] / [con pedido: Google Sheets → Claude → Text Parser (limpieza JSON) → Parse JSON → Router → Telegram (respuesta automática, caso normal) / Telegram (aviso de escalamiento, caso grave)]

⚠️ Pendiente: reconectar la rama de "caso normal" a Twilio para que el cliente reciba la respuesta directo por WhatsApp, en vez de que ambas rutas notifiquen a Telegram como está ahora.

**Escenario 2 — Aprobación humana:**
Telegram (Watch Updates) → Text Parser (separa teléfono y mensaje) → Twilio (reenvía la respuesta aprobada al cliente)

---

## ✅ Habilidades demostradas

- Diseño de automatización multi-escenario con patrón de dos escenarios para Human-in-the-loop real
- Integración de IA generativa (Claude) para clasificación de intención y redacción contextual
- Prompt engineering para forzar salida estructurada (JSON) de forma confiable
- Diagnóstico técnico basado en evidencia (revisión de raw input/output en vez de solo la interfaz visual)
- Rediseño de arquitectura ante una limitación técnica, en vez de insistir en un enfoque que no escalaba
- Manejo de errores y resiliencia ante límites de cuota de terceros (Twilio) y comportamiento inconsistente de modelos de IA
- Modelado de datos simple y documentación honesta de limitaciones de alcance
