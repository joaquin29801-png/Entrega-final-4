# 🤖 Checkpoint 4 – Integraciones Avanzadas con n8n

## 📌 Descripción del proyecto

Este proyecto corresponde al **Checkpoint 4 – Integraciones Avanzadas** y consiste en el desarrollo de un workflow automatizado en **n8n** para gestionar consultas recibidas por correo electrónico.

La automatización integra diferentes servicios y componentes:

- Gmail
- HubSpot CRM
- Slack
- Human-in-the-Loop (HITL)
- Webhooks
- Lógica condicional
- Preparación y transformación de datos

El objetivo principal es recibir automáticamente consultas por Gmail, procesar la información del mensaje, verificar el contacto en HubSpot, generar una respuesta propuesta y enviarla a Slack para que sea revisada por una persona antes de continuar con el proceso.

De esta manera se evita que una respuesta sea enviada automáticamente al cliente sin supervisión humana.

---

## 🎯 Objetivo

Construir una automatización capaz de integrar diferentes servicios externos dentro de un único workflow.

El flujo desarrollado permite:

1. Detectar nuevos correos electrónicos.
2. Evitar procesar respuestas automáticas.
3. Limpiar y normalizar los datos recibidos.
4. Preparar una respuesta propuesta.
5. Buscar al cliente en HubSpot mediante su correo electrónico.
6. Determinar si el contacto existe.
7. Crear o actualizar el contacto.
8. Preparar un borrador para revisión humana.
9. Notificar al equipo mediante Slack.
10. Detener temporalmente el workflow utilizando un nodo Wait.
11. Dejar preparado el proceso para continuar mediante webhook después de una intervención humana.

---

# 🏗️ Arquitectura del workflow

El flujo general implementado es:

Gmail Trigger
        ↓
IF - Anti Auto Reply
        ↓
Set - Limpiar Payload
        ↓
AI Agent - Respuesta Simulada
        ↓
HubSpot - Buscar Contacto
        ↓
IF - Contacto Existe?
       ↙ ↘
 EXISTE   NO EXISTE
    ↓         ↓
Actualizar   Crear contacto
contacto
       ↘     ↙
Preparar Gmail Draft - HITL
        ↓
Set - Payload Slack
        ↓
Slack - Send Message
        ↓
Wait
        ↓
Webhook / Continuación HITL

---

# 🔧 Explicación de los nodos

## 1. Gmail Trigger

El workflow comienza con un **Gmail Trigger**.

Este nodo monitorea la cuenta de Gmail conectada y detecta nuevos mensajes.

En la configuración utilizada, Gmail es consultado periódicamente para detectar nuevos correos.

Su función dentro del sistema es actuar como **disparador principal de la automatización**.

---

## 2. IF - Anti Auto Reply

Luego del Gmail Trigger se utiliza un nodo **IF** encargado de detectar respuestas automáticas.

Se verifican diferentes condiciones sobre el asunto y el remitente del correo.

Entre ellas:

- `Auto-reply`
- `Out of office`
- `Undeliverable`
- direcciones que contienen `no-reply@`

El objetivo es evitar que el sistema procese mensajes automáticos y genere ciclos innecesarios de automatización.

---

## 3. Set - Limpiar Payload

Una vez validado el correo, se utiliza un nodo **Set / Edit Fields** para normalizar los datos recibidos.

Se generan principalmente los siguientes campos:

### customer_email

Contiene el correo electrónico limpio del remitente.

```javascript
{{ ($json.From || '').match(/<([^>]+)>/)?.[1] || $json.From || '' }}
