# Agente Financiero Conversacional para Emprendedoras

> **Hackatón Women in Tech 2026 — CIX BCP / UTEC Ventures**  
> **Reto 2:** Crecimiento financiero para emprendedoras  
> **Canal:** WhatsApp mediante **Twilio Sandbox for WhatsApp**  
> **Estado:** Diseño y guía de implementación del MVP; no implica funcionalidades ya desplegadas.

## 1. Resumen ejecutivo

Asistente financiero conversacional por WhatsApp que convierte mensajes cotidianos —en texto y, opcionalmente, audio— en registros estructurados de actividad económica. Permite **diferenciar el dinero del negocio del dinero personal**, consultar movimientos e indicadores de flujo de efectivo y elaborar un **historial financiero autodeclarado**.

La experiencia se propone como futura extensión del ecosistema **Telecrédito BCP**, pero el **MVP no se integra realmente con el BCP, Yape ni Plin**: estas conexiones requieren acceso y autorización explícitos. El historial generado no equivale a un reporte bancario verificado ni a un historial crediticio oficial.

**Propuesta de valor:** ayudar a microemprendedoras a comprender cuánto ingresan, cuánto gastan, qué retiran para el hogar y cuánto efectivo han registrado como disponible, sin exigir conocimientos contables ni instalar otra aplicación.

## 2. Contexto, usuaria y objetivos

**Problema:** en pequeños negocios, las ventas y los gastos familiares pueden compartir la misma caja o billetera. Esto dificulta identificar recursos para proveedores, registrar resultados y justificar la actividad del emprendimiento.

**Usuaria objetivo:** microemprendedora peruana que vende por redes sociales o presencialmente, cobra por efectivo/Yape/Plin y maneja registros informales.

### Objetivos del MVP

1. Recibir y responder **mensajes reales** mediante Twilio Sandbox y teléfonos vinculados.
2. Interpretar texto en español y extraer operaciones mediante un LLM con salida estructurada.
3. Validar importes, categorías y confirmaciones antes de persistir.
4. Distinguir ventas cobradas, gastos del negocio, retiros personales y aportes propios.
5. Consultar indicadores y movimientos desde PostgreSQL, no desde cifras inventadas por el LLM.
6. Evitar duplicados por reintentos de webhooks y permitir correcciones auditadas.

### Fuera de alcance inicial

- Sincronización real con Telecrédito BCP, Yape, Plin ni cuentas bancarias.
- Ejecución de transferencias, bloqueo real de dinero, otorgamiento de créditos o scoring.
- Conciliación bancaria y contabilidad tributaria completa.
- Aplicación web compleja, notificaciones masivas o plantillas de producción.
- Fiados y audios como funcionalidades obligatorias (son mejoras P1).

## 3. Reglas de negocio

| ID | Regla |
|---|---|
| RN-01 | Solo registrar información financiera después del consentimiento de la usuaria. |
| RN-02 | Una **venta cobrada** aumenta el ingreso de caja registrado. |
| RN-03 | Un **gasto operativo** disminuye caja y se muestra separado de los retiros personales. |
| RN-04 | Un **retiro personal** disminuye caja del negocio, pero **no es un gasto operativo** ni una pérdida contable. |
| RN-05 | Un **aporte personal** aumenta los fondos disponibles, pero **no es una venta**. |
| RN-06 | Una venta **fiada** (P1) genera cuenta por cobrar; no aumenta caja hasta registrar su cobro. |
| RN-07 | Ante un monto o tipo de operación ambiguo, solicitar aclaración antes de guardar. |
| RN-08 | Confirmar operaciones cuando la interpretación sea incierta; nunca inferir un monto omitido. |
| RN-09 | Cada `MessageSid` entrante se procesa como máximo una vez a nivel de efecto financiero; implementar protección adicional contra dobles confirmaciones. |
| RN-10 | Las correcciones crean una reversión auditada y, si corresponde, un nuevo movimiento; no eliminar silenciosamente el historial. |
| RN-11 | Utilizar `Decimal`/`NUMERIC` para dinero; fechas relativas según `America/Lima`. |
| RN-12 | El LLM solo interpreta y redacta; **las reglas financieras y cálculos son determinísticos**. |
| RN-13 | Diferenciar «flujo neto registrado», «utilidad» y «saldo bancario». No presentar uno como el otro. |
| RN-14 | Registrar el origen del dato (`USER_DECLARED`); nunca etiquetarlo como verificado sin evidencia autorizada. |

## 4. Requisitos

### Requisitos funcionales

| ID | Requisito | Prioridad |
|---|---|---|
| RF-01 | Iniciar conversación y obtener consentimiento. | P0 |
| RF-02 | Recibir mensajes por webhook de Twilio. | P0 |
| RF-03 | Registrar ventas cobradas. | P0 |
| RF-04 | Registrar gastos operativos. | P0 |
| RF-05 | Registrar retiros personales y aportes. | P0 |
| RF-06 | Consultar resumen por periodo y últimos movimientos. | P0 |
| RF-07 | Solicitar aclaraciones y confirmar registros ambiguos. | P0 |
| RF-08 | Revertir/corregir operaciones conservando auditoría. | P0 |
| RF-09 | Deduplicar mensajes ante reenvíos de webhooks. | P0 |
| RF-10 | Recibir notas de voz con transcripción y confirmación de montos. | P1 |
| RF-11 | Gestionar fiados y abonos sin contar doble las ventas. | P1 |
| RF-12 | Generar historial exportable identificando datos autodeclarados. | P1 |
| RF-13 | Integración autorizada con Telecrédito BCP mediante adaptador. | Futuro |

### Requisitos no funcionales

| ID | Requisito / criterio |
|---|---|
| RNF-01 Seguridad | Webhook HTTPS con verificación de `X-Twilio-Signature`; credenciales solo en entorno seguro. |
| RNF-02 Disponibilidad | Aceptar y persistir un mensaje antes de ejecutar procesos lentos; reintentos de worker. |
| RNF-03 Integridad | Restricciones SQL, transacciones atómicas, idempotencia y trazabilidad. |
| RNF-04 Rendimiento | Responder rápido al webhook; LLM y persistencia financiera en worker independiente. |
| RNF-05 Privacidad | Minimizar datos, consentimiento y mecanismos de eliminación; no pedir claves ni DNI. |
| RNF-06 Mantenibilidad | Separar adaptador Twilio, lógica financiera, LLM y repositorios. |
| RNF-07 Usabilidad | Respuestas breves en español claro; aclaraciones concretas; cifras con `S/`. |
| RNF-08 Portabilidad | Sustituir Twilio por canal alternativo sin reescribir el núcleo de negocio. |

## 5. Arquitectura técnica

**Estilo:** monolito modular con separación por capas, **inbox persistente** y procesamiento asíncrono. FastAPI recibe webhooks, un worker procesa el mensaje y el adaptador Twilio envía la respuesta.

```mermaid
flowchart TD
    U[Emprendedora en WhatsApp] --> T[Twilio Sandbox / Messaging API]
    T -->|POST form-urlencoded| W[FastAPI: /webhooks/twilio/whatsapp]
    W --> V[Validador X-Twilio-Signature]
    V --> I[(PostgreSQL: inbound_messages)]
    I --> K[Worker de mensajes]
    K --> C[Gestor de conversación]
    C --> L[LLM: extracción estructurada]
    L --> P[Pydantic: validación]
    P --> R[Motor de reglas financieras]
    R --> D[(PostgreSQL: transactions / receivables)]
    D --> Q[Consultas y reportes]
    Q --> S[Adaptador de salida Twilio SDK]
    S --> T
```

**Capas y módulos:**

| Capa | Módulos | Responsabilidad |
|---|---|---|
| Entrada / infraestructura | `api.twilio_webhook`, `integrations.twilio` | Recibir formulario, verificar firma, guardar inbox, enviar mensajes. |
| Aplicación | `conversation`, `inbox_worker`, `commands` | Orquestar consentimiento, intenciones, aclaraciones y reintentos. |
| Dominio | `transactions`, `financial_rules`, `financial_history` | Clasificación financiera, invariantes, cálculos y casos de uso. |
| IA (adaptador externo) | `nlp` | Convertir lenguaje natural a un contrato tipado sin tocar la BD. |
| Persistencia | `repositories`, `db`, `migrations` | SQLAlchemy, PostgreSQL, auditoría y consultas parametrizadas. |
| Integración futura | `integrations.telecredito` | Puerto/adaptador opcional, **mock** hasta contar con API autorizada. |

**Stack:** Python, FastAPI, Twilio Python SDK, Pydantic, SQLAlchemy, Alembic, Supabase PostgreSQL, LLM con salida JSON estructurada, `pytest` y `httpx`. Para el MVP no se requieren microservicios, Kubernetes ni una cola distribuida; para producción conviene una cola robusta y observabilidad.

### Secuencia de un registro

```mermaid
sequenceDiagram
    participant U as Usuaria
    participant T as Twilio
    participant A as FastAPI
    participant DB as PostgreSQL
    participant W as Worker
    participant IA as LLM
    U->>T: "Vendí S/150 por Yape"
    T->>A: POST /webhooks/twilio/whatsapp
    A->>A: Validar X-Twilio-Signature
    A->>DB: INSERT inbox (MessageSid único)
    A-->>T: HTTP 200 (TwiML vacío)
    W->>DB: Reclamar mensaje pendiente
    W->>IA: Extraer intención y monto
    IA-->>W: SALE, PEN 150, Yape
    W->>DB: Validar + crear venta (transacción SQL)
    W->>T: Enviar confirmación mediante Messaging API
    T-->>U: "Registré tu venta de S/150"
```

> **Nota operativa:** el `POST` de Twilio para mensajes entrantes normalmente llega como `application/x-www-form-urlencoded`. Puede incluir `MessageSid`, `From`, `To`, `Body`, `NumMedia` y `MediaUrl0` (si hay archivos). **No** es un evento JSON de Meta y **no** requiere verificación `GET` con `hub.challenge`.

## 6. Contratos de comunicación con Twilio

### Webhooks

| Método | Ruta | Función |
|---|---|---|
| `POST` | `/webhooks/twilio/whatsapp` | Entrada de mensajes; firma validada, persistencia, respuesta inmediata. |
| `POST` | `/webhooks/twilio/status` | Opcional: eventos de entrega de mensajes salientes; no registrarlos como ventas. |
| `GET` | `/health` | Salud del servidor. |
| `GET` | `/api/v1/users/{id}/summary` | Resumen protegido y autorizado; no público. |
| `GET` | `/api/v1/users/{id}/transactions` | Lista de operaciones con controles de acceso. |
| `POST` | `/api/v1/transactions/{id}/reverse` | Reversión autenticada y auditada. |

**Ejemplo ilustrativo de formulario entrante** (no JSON; valores ficticios):

```text
POST /webhooks/twilio/whatsapp
Content-Type: application/x-www-form-urlencoded
X-Twilio-Signature: FIRMA_GENERADA_POR_TWILIO

MessageSid=SMxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
From=whatsapp%3A%2B51999999999
To=whatsapp%3A%2B14155238886
Body=Vendi+150+soles+por+Yape
NumMedia=0
```

Twilio crea la firma a partir de la **URL pública exacta** del webhook y los parámetros. Validar mediante `twilio.request_validator.RequestValidator` y el **Twilio Auth Token**, usando `X-Twilio-Signature`. No utilizar `X-Hub-Signature-256`, `META_APP_SECRET` ni verificación `hub.challenge` (pertenecen al flujo de Meta). Si hay ngrok o proxy inverso, usar una URL pública configurada explícitamente para validación.

**Mensajes salientes:** utilizar `twilio.rest.Client(...).messages.create(from_="whatsapp:+...", to="whatsapp:+...", body="...")`. Las respuestas libres funcionan dentro de la **ventana de atención de 24 horas** después del último mensaje de la persona; fuera de ella se requieren plantillas aprobadas según el tipo de mensaje y condiciones de la cuenta.

### Contrato de extracción IA

```python
from decimal import Decimal
from enum import Enum
from pydantic import BaseModel, Field

class Intent(str, Enum):
    REGISTER = "REGISTER"
    QUERY = "QUERY"
    CORRECT = "CORRECT"
    UNKNOWN = "UNKNOWN"

class TransactionType(str, Enum):
    SALE = "SALE"
    BUSINESS_EXPENSE = "BUSINESS_EXPENSE"
    PERSONAL_WITHDRAWAL = "PERSONAL_WITHDRAWAL"
    CAPITAL_CONTRIBUTION = "CAPITAL_CONTRIBUTION"
    CREDIT_SALE = "CREDIT_SALE"  # P1: se guarda en receivables
    CREDIT_COLLECTION = "CREDIT_COLLECTION"  # P1

class ExtractionResult(BaseModel):
    intent: Intent
    transaction_type: TransactionType | None = None
    amount: Decimal | None = Field(default=None, gt=0)
    currency: str = "PEN"
    payment_method: str | None = None
    description: str | None = None
    occurred_at: str | None = None
    missing_fields: list[str] = Field(default_factory=list)
```

Restricciones P0: **una operación por mensaje**, importe positivo, moneda `PEN` por defecto, fechas interpretadas en Lima, aclarar montos y categorías ambiguas y confirmar antes de persistir cuando haga falta. El LLM no ejecuta SQL.

## 7. Modelo de datos PostgreSQL

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phone_number TEXT UNIQUE NOT NULL,
    business_name TEXT,
    business_category TEXT,
    consent_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE inbound_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    provider TEXT NOT NULL DEFAULT 'TWILIO',
    provider_message_id TEXT NOT NULL,
    user_id UUID REFERENCES users(id),
    message_type TEXT NOT NULL DEFAULT 'TEXT',
    message_text TEXT,
    processing_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK (processing_status IN ('PENDING','PROCESSING','COMPLETED','FAILED')),
    attempts INTEGER NOT NULL DEFAULT 0,
    received_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at TIMESTAMPTZ,
    UNIQUE (provider, provider_message_id)
);

CREATE TABLE transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    source_message_id UUID UNIQUE REFERENCES inbound_messages(id),
    transaction_type TEXT NOT NULL CHECK (
        transaction_type IN ('SALE','BUSINESS_EXPENSE',
        'PERSONAL_WITHDRAWAL','CAPITAL_CONTRIBUTION','CREDIT_COLLECTION')
    ),
    amount NUMERIC(14,2) NOT NULL CHECK (amount > 0),
    currency CHAR(3) NOT NULL DEFAULT 'PEN',
    payment_method TEXT,
    category TEXT,
    description TEXT,
    source TEXT NOT NULL DEFAULT 'USER_DECLARED',
    occurred_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    reversed_at TIMESTAMPTZ,
    reversal_reason TEXT
);

-- P1: fiados sin efecto de caja hasta el cobro
CREATE TABLE receivables (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    source_message_id UUID UNIQUE REFERENCES inbound_messages(id),
    customer_alias TEXT,
    original_amount NUMERIC(14,2) NOT NULL CHECK (original_amount > 0),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE receivable_payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    receivable_id UUID NOT NULL REFERENCES receivables(id),
    transaction_id UUID NOT NULL UNIQUE REFERENCES transactions(id),
    amount NUMERIC(14,2) NOT NULL CHECK (amount > 0)
);

CREATE INDEX idx_transactions_user_date ON transactions(user_id, occurred_at DESC);
CREATE INDEX idx_inbox_status ON inbound_messages(processing_status, received_at);
```

**Invariantes:** los cobros y aportes aumentan caja; gastos y retiros la reducen; los retiros personales no se suman a costos operativos; fiados no son caja; los abonos disminuyen la cuenta por cobrar. Implementar en una transacción SQL el control de que los abonos acumulados no superen lo adeudado. Las reversiones deben recalcular los indicadores sin borrar la historia.

### Consulta de flujo de caja neto registrado

```sql
SELECT
    user_id,
    COALESCE(SUM(amount) FILTER (WHERE transaction_type = 'SALE'), 0) AS sales_collected,
    COALESCE(SUM(amount) FILTER (WHERE transaction_type = 'BUSINESS_EXPENSE'), 0) AS business_expenses,
    COALESCE(SUM(amount) FILTER (WHERE transaction_type = 'PERSONAL_WITHDRAWAL'), 0) AS personal_withdrawals,
    COALESCE(SUM(amount) FILTER (WHERE transaction_type = 'CREDIT_COLLECTION'), 0) AS credit_collections,
    COALESCE(SUM(CASE
        WHEN transaction_type IN ('SALE','CAPITAL_CONTRIBUTION','CREDIT_COLLECTION') THEN amount
        ELSE -amount
    END), 0) AS net_recorded_cash_flow
FROM transactions
WHERE user_id = :user_id
  AND reversed_at IS NULL
  AND occurred_at >= :start_at AND occurred_at < :end_at
GROUP BY user_id;
```

`net_recorded_cash_flow` es el **flujo neto de las operaciones registradas**, no utilidad, saldo bancario ni saldo total real. El inicio de caja, operaciones no reportadas e inventario no se infieren.

**Historial:** ventas cobradas, gastos operativos, retiros personales, aportes, flujo neto, conteo de registros y días activos; fiados y abonos en P1. Siempre etiquetar como `UNVERIFIED` si los datos son autodeclarados.

## 8. Organización del repositorio

```text
cix-bcp-financial-agent/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── core/
│   │   ├── config.py
│   │   └── security.py
│   ├── api/
│   │   ├── twilio_webhook.py
│   │   └── financial_history.py
│   ├── schemas/
│   │   ├── extraction.py
│   │   └── transactions.py
│   ├── services/
│   │   ├── inbox_worker.py
│   │   ├── conversation_service.py
│   │   ├── llm_service.py
│   │   ├── transaction_service.py
│   │   └── history_service.py
│   ├── integrations/
│   │   ├── twilio_client.py
│   │   └── telecredito_mock.py
│   ├── repositories/
│   │   ├── users.py
│   │   ├── inbound_messages.py
│   │   └── transactions.py
│   └── db/
│       └── session.py
├── migrations/
├── tests/
│   ├── test_twilio_webhook.py
│   ├── test_extraction.py
│   ├── test_transactions.py
│   └── test_financial_history.py
├── .env.example
├── .gitignore
├── requirements.txt
└── README.md
```

## 9. Instalación local (MVP)

**Windows PowerShell:**

```powershell
mkdir cix-bcp-financial-agent
cd cix-bcp-financial-agent
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install fastapi "uvicorn[standard]" sqlalchemy "psycopg[binary]" pydantic-settings python-multipart twilio httpx pytest alembic
```

**Linux/macOS:**

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install fastapi 'uvicorn[standard]' sqlalchemy 'psycopg[binary]' pydantic-settings python-multipart twilio httpx pytest alembic
```

### Variables de entorno

`.env.example` (valores de muestra; jamás subir el `.env` real):

```dotenv
DATABASE_URL=postgresql+psycopg://USER:PASSWORD@HOST:5432/DB
TWILIO_ACCOUNT_SID=AC_REPLACE_ME
TWILIO_AUTH_TOKEN=REPLACE_ME
TWILIO_WHATSAPP_FROM=whatsapp:+14155238886
TWILIO_WEBHOOK_PUBLIC_URL=https://YOUR_NGROK_DOMAIN.ngrok-free.app/webhooks/twilio/whatsapp
LLM_API_KEY=REPLACE_ME
```

`TWILIO_WHATSAPP_FROM` es **solo un ejemplo** común de Sandbox: usar exactamente el número mostrado en su propia consola. `TWILIO_WEBHOOK_PUBLIC_URL` debe ser exactamente la URL configurada en Twilio, incluido el esquema `https` y la ruta.

### `app/main.py`

```python
from fastapi import FastAPI
from app.api.twilio_webhook import router as twilio_router

app = FastAPI(title="CIX BCP Financial Agent", version="0.1.0")
app.include_router(twilio_router)

@app.get("/health")
def health():
    return {"status": "ok"}
```

### `app/api/twilio_webhook.py`

Ejemplo **mínimo de recepción segura**, que valida la firma y responde con TwiML vacío. Aún **no** registra datos; reemplazar el comentario con persistencia transaccional y deduplicación antes de activar el worker.

```python
import os
from fastapi import APIRouter, HTTPException, Request
from fastapi.responses import Response
from twilio.request_validator import RequestValidator
from twilio.twiml.messaging_response import MessagingResponse

router = APIRouter(prefix="/webhooks/twilio", tags=["Twilio"])

@router.post("/whatsapp")
async def incoming_whatsapp(request: Request):
    form = await request.form()
    values = dict(form)
    signature = request.headers.get("X-Twilio-Signature", "")
    public_url = os.environ["TWILIO_WEBHOOK_PUBLIC_URL"]
    auth_token = os.environ["TWILIO_AUTH_TOKEN"]

    if not RequestValidator(auth_token).validate(public_url, values, signature):
        raise HTTPException(status_code=403, detail="Invalid Twilio signature")

    message_sid = values.get("MessageSid")
    sender = values.get("From")
    body = values.get("Body", "")
    if not message_sid or not sender:
        raise HTTPException(status_code=400, detail="Missing message identifiers")

    # TODO: INSERT inbound_messages(provider='TWILIO', provider_message_id=message_sid,
    #       sender=sender, message_text=body) ON CONFLICT DO NOTHING; commit ANTES de 200.
    # TODO: el worker procesará el mensaje y enviará una respuesta mediante Twilio SDK.
    return Response(content=str(MessagingResponse()), media_type="application/xml")
```

Para completar la implementación deben **persistir también la identidad del remitente**, asociándola al registro `users`; nunca confiar en un `user_id` enviado por el LLM. En un webhook síncrono se puede responder con TwiML con texto; este diseño responde vacío y **envía después desde el worker** para evitar tiempos de espera del LLM.

### Envío desde el worker (`app/integrations/twilio_client.py`)

```python
import os
from twilio.rest import Client

def send_whatsapp(recipient: str, message: str) -> str:
    client = Client(os.environ["TWILIO_ACCOUNT_SID"], os.environ["TWILIO_AUTH_TOKEN"])
    sent = client.messages.create(
        from_=os.environ["TWILIO_WHATSAPP_FROM"],
        to=recipient,  # Ej.: whatsapp:+519XXXXXXXX
        body=message,
    )
    return sent.sid
```

**Advertencia:** el código anterior es una base de integración, no una aplicación completa. Faltan inbox SQL, worker, consentimiento, extracción LLM, manejo de estados, controles de reintentos de envío y lógica financiera.

**Ejecutar:**

```powershell
$env:TWILIO_AUTH_TOKEN="TU_TOKEN_REAL"
$env:TWILIO_WEBHOOK_PUBLIC_URL="https://TU_DOMINIO.ngrok-free.app/webhooks/twilio/whatsapp"
uvicorn app.main:app --reload --port 8000
```

En otra terminal ejecutar `ngrok http 8000`. Configurar la URL HTTPS obtenida en la consola de Twilio; si cambia el dominio, actualizar `TWILIO_WEBHOOK_PUBLIC_URL` y reiniciar el servidor. Para pruebas de firma no usar un token falso: las solicitudes reales requieren el Auth Token de Twilio.

## 10. Configurar Twilio Sandbox for WhatsApp

1. Crear una cuenta en [Twilio Console](https://console.twilio.com/) e ingresar a **Try WhatsApp / Try it out**, según la interfaz de la cuenta.
2. Activar el **entorno de pruebas de WhatsApp**. La cuenta de prueba puede mostrar el flujo **Try out WhatsApp**, mientras que la consola clásica presenta **Messaging → Try it out → Send a WhatsApp message / Sandbox settings**.
3. Desde el WhatsApp personal, enviar el mensaje `join <código>` **exacto** indicado por la consola al número Sandbox mostrado, o utilizar el QR disponible. Cada teléfono de prueba debe vincularse.
4. Verificar que Twilio confirme el ingreso al Sandbox. El número personal no se convierte en el número público del bot.
5. Ejecutar FastAPI localmente y exponerlo con un túnel HTTPS como `ngrok http 8000`.
6. En la sección **Receive a message**, elegir **Custom** y pegar la URL en **Webhook URL**, o en consola clásica definir **When a message comes in** con método **HTTP POST**. Guardar.
7. Copiar el **Account SID**, **Auth Token** y número habilitado para WhatsApp desde Twilio Console; configurar las variables de entorno.
8. Enviar `Hola` desde el teléfono vinculado. Revisar el webhook, los logs y la validez de la firma. El ejemplo base devuelve respuesta vacía: para ver una contestación se debe agregar TwiML de texto o implementar el worker y el envío con SDK.
9. Probar mensajes financieros, registrar una sola vez cada `MessageSid` y consultar el historial.
10. Para producción, registrar un **WhatsApp sender** propio con el proceso aplicable en Twilio/Meta. El Sandbox es exclusivo para pruebas y tiene límites; no permite asumir envíos libres o plantillas propias de producción.

**Documentación oficial:** [Twilio WhatsApp Quickstart](https://www.twilio.com/docs/whatsapp/quickstart) · [Twilio Sandbox](https://www.twilio.com/docs/whatsapp/sandbox) · [Validación de solicitudes](https://www.twilio.com/docs/usage/security) · [Recepción en Sandbox](https://help.twilio.com/articles/55716571409819-How-do-I-receive-WhatsApp-messages-while-testing).

## 11. Roadmap de la hackatón

| Fecha | Entregables |
|---|---|
| **6 nov. — Día 1** | Definir alcance; repositorio, FastAPI, PostgreSQL, configuración Sandbox, webhook con firma y `/health`; formulario de avance según bases. |
| **7 nov. — Día 2** | Inbox e idempotencia, worker, IA, registro de ventas/gastos/retiros, consultas y respuesta vía Twilio. |
| **8 nov. — Integración** | Pruebas end-to-end, correcciones, manejo de ambigüedad y demo; audios/fiados solo si P0 funciona. |
| **9 nov. — Demo Day** | Validación con usuarias, entrega y pitch según bases de la hackatón. |

### Reparto del equipo (4 integrantes)

| Rol | Responsabilidad principal |
|---|---|
| Backend / integraciones | Twilio webhook, seguridad, API, worker y envío. |
| IA / conversación | Extracción estructurada, prompts, manejo de ambigüedad. |
| Datos / finanzas | Esquema PostgreSQL, reglas de negocio, idempotencia, métricas. |
| Producto / UX | Investigación con usuarias, guion conversacional, pruebas y pitch. |

## 12. Pruebas y criterios de aceptación

| Entrada o situación | Resultado esperado |
|---|---|
| `Vendí 100 soles por Yape` | Registro `SALE` S/100. |
| `Compré 40 soles de pollo para vender` | `BUSINESS_EXPENSE` S/40. |
| `Saqué 20 soles para mi casa` | `PERSONAL_WITHDRAWAL` S/20; no es costo operativo. |
| `Puse 100 soles de mis ahorros en el negocio` | `CAPITAL_CONTRIBUTION` S/100; no es venta. |
| `Gasté 40` | Solicita si es gasto de negocio o personal antes de registrar. |
| Reenvío del mismo `MessageSid` | No duplica el efecto financiero. |
| Mensaje con firma inválida | HTTP 403; no guarda nada. |
| Reversión de una venta | Auditoría y recálculo de indicadores. |
| `¿Cómo va mi negocio esta semana?` | Resume transacciones SQL del periodo. |
| `Le fié 50 soles a Ana` (P1) | Cuenta por cobrar S/50; caja sin cambio. |
| `Ana pagó 20 del fiado` (P1) | Caja +S/20 y deuda pendiente S/30. |

**Criterio mínimo de MVP:** una usuaria vinculada al Sandbox envía una venta, un gasto y un retiro; los registros se guardan una sola vez en PostgreSQL y recibe un resumen financiero calculado desde la base de datos.

**Métricas a medir, no presumir:** exactitud de clasificación, registros correctos, p95 de respuesta, duplicados financieros (meta: cero), porcentaje de aclaraciones resueltas y comprensión de los resúmenes.

## 13. Seguridad, privacidad y límites

- Consentimiento antes de guardar datos y explicación comprensible de finalidad, retención y eliminación.
- Cumplir la legislación peruana aplicable en protección de datos personales antes de un piloto real.
- Proteger credenciales, evitar almacenar DNI, claves, tokens bancarios y medios innecesarios.
- Verificar `X-Twilio-Signature` con el SDK oficial y la URL pública exacta; usar HTTPS.
- Tratar `MessageSid` como identificador de transporte, no como autorización del usuario.
- Proteger los endpoints internos con autenticación y autorización por identidad/propietaria.
- Aplicar políticas de reintento y registro de errores sin duplicar transacciones ni mensajes salientes.
- No presentar datos declarados como movimientos verificados por BCP o Yape.
- No prometer crédito ni recomendar endeudamiento sin controles adicionales.
- No enviar mensajes financieros sensibles a un número sin verificar su titularidad y consentimiento; evitar detalles innecesarios en notificaciones.

## 14. Evolución posterior

1. Audios, cuentas por cobrar, recordatorios y metas de reposición.
2. Mejor calidad de datos, conciliación autorizada e historial con origen verificable.
3. Integración eventual con servicios financieros del BCP, solo mediante acuerdos y APIs oficialmente habilitadas.
4. Migración desde Sandbox a número de producción y plantillas autorizadas.
5. Cola de mensajes robusta, observabilidad, tests de carga y controles de acceso reforzados.

## 15. Referencias

- *Bases — Hackatón Women in Tech 2026* y *Reto 2 — Crecimiento financiero para emprendedoras* (documentos de investigación del equipo).
- [Twilio: Quickstart WhatsApp](https://www.twilio.com/docs/whatsapp/quickstart).
- [Twilio: Sandbox for WhatsApp](https://www.twilio.com/docs/whatsapp/sandbox).
- [Twilio: Validating requests](https://www.twilio.com/docs/usage/security).
- [Twilio Help: Receiving WhatsApp messages while testing](https://help.twilio.com/articles/55716571409819-How-do-I-receive-WhatsApp-messages-while-testing).

---

**Estado del proyecto:** README de análisis, diseño y guía de implementación. No indica que integraciones, endpoints financieros o pruebas end-to-end ya estén operativos.
