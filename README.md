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
````

### subject

Contiene el asunto del correo.

```javascript
{{ $json.Subject }}
```

### body_text

Contiene el contenido disponible del mensaje recibido.

```javascript
{{ $json.snippet }}
```

Este paso permite trabajar posteriormente con una estructura de datos más simple y controlada.

---

# 🤖 4. Respuesta simulada

El nodo:

`AI Agent - Respuesta Simulada`

se utiliza para simular la generación de una respuesta que posteriormente será revisada por una persona.

Para esta versión del proyecto se utiliza un nodo de transformación de datos en lugar de una llamada real a un modelo LLM.

La respuesta generada sigue una estructura similar a:

```text
Hola, gracias por contactarnos.

Recibimos tu consulta sobre: [ASUNTO].

El caso quedó preparado para revisión del equipo de soporte antes de responder.
```

Además, el nodo conserva:

* correo electrónico del cliente
* asunto
* cuerpo original
* respuesta propuesta

Esta implementación permite demostrar la arquitectura completa incluso sin depender de créditos o consumo de una API de inteligencia artificial.

---

# 🔎 5. HubSpot - Buscar Contacto

Después de preparar la respuesta, el workflow consulta **HubSpot CRM**.

El recurso utilizado es:

`Contact`

La operación utilizada es:

`Search`

La búsqueda se realiza utilizando el correo electrónico del cliente.

```javascript
{{ $json.customer_email }}
```

Además, se limita el resultado a un contacto.

El objetivo es determinar si la persona que envió el correo ya existe dentro del CRM.

---

# 🔀 6. IF - ¿Existe contacto?

Después de buscar el contacto en HubSpot se utiliza un nodo IF.

La condición verifica si HubSpot devolvió un identificador para el contacto.

Conceptualmente:

```javascript
{{ !!$json.id }}
```

Si existe un ID:

```text
TRUE → el contacto existe
```

Si no existe:

```text
FALSE → el contacto debe ser creado
```

Esto permite manejar automáticamente ambos escenarios.

---

# 👤 7. Crear o actualizar contacto

El workflow posee dos caminos provenientes del IF.

## Contacto existente

Si el contacto fue encontrado en HubSpot, se utiliza la información existente y se ejecuta la operación correspondiente sobre el contacto.

## Contacto nuevo

Si el contacto no existe, se utiliza:

`Contact → Create or Update`

para registrar al usuario utilizando su correo electrónico.

Esto permite que los clientes que escriben por primera vez queden registrados automáticamente en el CRM.

Luego ambos caminos vuelven a unirse.

---

# ✉️ 8. Preparar Gmail Draft - HITL

Una vez gestionado el contacto en HubSpot, el workflow prepara la información que será revisada por una persona.

Este nodo genera cuatro campos principales.

### draft_to

Destinatario del futuro correo.

```javascript
{{ $('AI Agent - Respuesta Simulada').item.json.customer_email }}
```

### draft_subject

Asunto de la respuesta.

```javascript
Re: {{ $('AI Agent - Respuesta Simulada').item.json.subject }}
```

### draft_body

Respuesta propuesta.

```javascript
{{ $('AI Agent - Respuesta Simulada').item.json.ai_response }}
```

### approval_status

Estado inicial:

```text
PENDIENTE_REVISION_HUMANA
```

Este estado representa el mecanismo **Human-in-the-Loop (HITL)** implementado en el proyecto.

---

# 🧹 9. Set - Payload Slack

Antes de enviar la notificación a Slack se genera un payload simplificado.

Los campos utilizados son:

```text
cliente
asunto
estado
respuesta_propuesta
```

Ejemplo:

```json
{
  "cliente": "cliente@email.com",
  "asunto": "Re: Consulta sobre mi pedido",
  "estado": "Borrador creado - pendiente de aprobación humana",
  "respuesta_propuesta": "Hola, gracias por contactarnos..."
}
```

Este nodo funciona como una **holgura de carga útil**, evitando enviar a Slack datos innecesarios provenientes de Gmail o HubSpot.

---

# 💬 10. Integración con Slack

Luego se utiliza Slack para informar al equipo que existe una nueva respuesta pendiente de revisión.

El nodo utiliza:

```text
Resource: Message
Operation: Send
Destination: Channel
Message Type: Simple Text Message
```

El mensaje enviado tiene una estructura similar a:

```text
🔔 NUEVA RESPUESTA PENDIENTE DE APROBACIÓN

Cliente: cliente@email.com

Asunto: Re: Consulta sobre mi pedido

Respuesta propuesta:
Hola, gracias por contactarnos...

Estado:
Borrador creado - pendiente de aprobación humana

⚠️ Se requiere revisión humana antes de enviar la respuesta al cliente.
```

De esta manera, el equipo recibe inmediatamente la información necesaria para revisar la respuesta.

---

# 🧑‍💻 11. Human-in-the-Loop (HITL)

Uno de los puntos principales de la arquitectura es evitar que el sistema envíe automáticamente una respuesta generada.

Por este motivo se implementa un mecanismo:

**Human-in-the-Loop (HITL)**.

El flujo funciona de la siguiente manera:

```text
Correo recibido
      ↓
Procesamiento automático
      ↓
Respuesta propuesta
      ↓
Notificación en Slack
      ↓
Revisión humana
      ↓
Continuación del workflow
```

La persona responsable puede revisar:

* cliente
* asunto
* respuesta propuesta
* estado del proceso

antes de permitir que la automatización continúe.

---

# ⏸️ 12. Wait / Webhook

Después de enviar la notificación a Slack se utiliza un nodo:

`Wait`

Configurado para:

```text
Resume: On Webhook Call
HTTP Method: GET
Response Code: 200
```

Este nodo pausa la ejecución del workflow.

n8n genera una URL de webhook específica durante la ejecución.

Cuando dicha URL recibe una llamada, la ejecución puede continuar.

Esto permite implementar procesos donde una automatización necesita esperar una decisión externa.

---

# 🔐 Seguridad y buenas prácticas

El workflow fue diseñado considerando diferentes buenas prácticas.

### Credenciales

Las conexiones con servicios externos utilizan el sistema de credenciales de n8n.

Se utilizan conexiones OAuth para:

* Gmail
* HubSpot
* Slack

Las claves y tokens no deben almacenarse manualmente dentro del workflow ni publicarse en GitHub.

### Filtrado de respuestas automáticas

El nodo Anti Auto Reply evita procesar correos automáticos y reduce el riesgo de loops.

### Human-in-the-Loop

Las respuestas no se envían directamente al cliente.

Primero requieren revisión humana.

### Minimización de datos

Antes de enviar información a Slack se genera un payload reducido que contiene únicamente los datos necesarios para la revisión.

---

# 🔄 Flujo completo

```text
┌─────────────────────┐
│    Gmail Trigger    │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Anti Auto Reply IF  │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│   Limpiar Payload   │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Respuesta Simulada  │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Buscar en HubSpot   │
└──────────┬──────────┘
           ↓
     ┌─────────────┐
     │ ¿Existe?    │
     └──────┬──────┘
        ↙       ↘
       Sí         No
       ↓           ↓
┌────────────┐ ┌────────────┐
│ Actualizar │ │   Crear    │
│ contacto   │ │  contacto  │
└──────┬─────┘ └─────┬──────┘
       └───────┬─────┘
               ↓
┌──────────────────────────┐
│ Preparar borrador - HITL │
└────────────┬─────────────┘
             ↓
┌──────────────────────────┐
│ Preparar Payload Slack   │
└────────────┬─────────────┘
             ↓
┌──────────────────────────┐
│   Notificación Slack     │
└────────────┬─────────────┘
             ↓
┌──────────────────────────┐
│      WAIT / WEBHOOK      │
│    Revisión humana       │
└──────────────────────────┘
```

---

# 🧪 Pruebas realizadas

Durante las pruebas se verificó el funcionamiento individual de los principales componentes.

## Gmail

Se recibió correctamente un correo mediante Gmail Trigger.

Se obtuvieron datos como:

* remitente
* asunto
* contenido del mensaje

## Limpieza del correo

Se extrajo correctamente el correo electrónico del remitente.

Ejemplo:

```text
"Nombre Usuario" <usuario@gmail.com>
```

se transforma en:

```text
usuario@gmail.com
```

## HubSpot

Se comprobó la búsqueda de contactos mediante correo electrónico.

El workflow puede determinar si el contacto ya existe y continuar por la rama correspondiente.

También se verificó la operación de creación/actualización del contacto.

## Slack

Se comprobó correctamente el envío de mensajes al canal configurado.

El mensaje incluye:

* cliente
* asunto
* respuesta propuesta
* estado
* advertencia de revisión humana

## Wait

El nodo Wait queda conectado después de Slack y configurado para esperar una llamada mediante webhook.

---

# 📊 Tecnologías utilizadas

| Tecnología | Función                               |
| ---------- | ------------------------------------- |
| n8n        | Orquestación y automatización         |
| Gmail      | Entrada de consultas                  |
| HubSpot    | Gestión de contactos CRM              |
| Slack      | Notificación al equipo                |
| Webhook    | Reanudación del workflow              |
| OAuth2     | Autenticación de integraciones        |
| JSON       | Intercambio y estructuración de datos |

---

# 📁 Estructura recomendada del repositorio

```text
Checkpoint-4-Integraciones-Avanzadas/
│
├── README.md
│
├── workflow/
│   └── Checkpoint-4-Integraciones-Avanzadas.json
│
├── evidencias/
│   ├── workflow-completo.png
│   ├── slack-notificacion.png
│   └── hubspot-contacto.png
│
└── .gitignore
```

---

# 🚀 Importar el workflow

Para importar el proyecto en n8n:

1. Descargar el archivo JSON del repositorio.
2. Abrir n8n.
3. Crear un nuevo workflow.
4. Seleccionar la opción para importar desde archivo.
5. Seleccionar:

```text
Checkpoint-4-Integraciones-Avanzadas.json
```

6. Configurar las credenciales correspondientes.
7. Verificar los canales, cuentas y parámetros antes de ejecutar.

---

# ⚙️ Configuración necesaria

Para ejecutar el proyecto en otra instancia de n8n será necesario configurar nuevamente las credenciales.

### Gmail

Configurar una cuenta mediante OAuth2.

### HubSpot

Configurar una cuenta de HubSpot mediante OAuth2.

### Slack

Configurar la aplicación o cuenta de Slack correspondiente y seleccionar el canal donde se enviarán las notificaciones.

> ⚠️ Las credenciales privadas, tokens y secretos no deben almacenarse en el repositorio.

---

# 🛡️ Consideraciones de seguridad

Nunca se deben publicar en GitHub:

```text
API Keys
Access Tokens
Refresh Tokens
Client Secrets
Passwords
Archivos .env
Credenciales OAuth
```

En caso de utilizar variables de entorno se recomienda incluir:

```gitignore
.env
.env.*
!.env.example
```

---

# 📈 Posibles mejoras futuras

El proyecto puede evolucionar incorporando:

* Un modelo LLM real para generar las respuestas.
* Clasificación automática de consultas.
* Priorización de tickets.
* Botones de aprobación/rechazo desde Slack.
* Webhooks separados para aprobar y rechazar.
* Creación automática de borradores reales en Gmail.
* Registro de auditoría de las decisiones humanas.
* Persistencia del estado de aprobación.
* Manejo de errores y reintentos.
* Alertas ante fallos de integraciones.
* Métricas de tiempo de respuesta.
* Integración con sistemas de tickets.

---

# ✅ Resultado final

El workflow implementa una arquitectura de integración entre:

**Gmail + n8n + HubSpot + Slack + HITL + Webhooks**

La solución permite automatizar gran parte del procesamiento de consultas sin eliminar el control humano sobre la respuesta final.

El sistema recibe el correo, normaliza la información, consulta el CRM, gestiona el contacto, prepara una respuesta, notifica al equipo y pausa la ejecución para permitir una revisión humana.

Esto demuestra cómo n8n puede utilizarse como plataforma de orquestación para conectar diferentes aplicaciones empresariales dentro de un proceso automatizado y controlado.

---

## 👨‍💻 Autor

**Joaquín Núñez**

Proyecto realizado como parte del:

**Checkpoint 4 – Integraciones Avanzadas**

Automatización e integración de sistemas con **n8n**.

