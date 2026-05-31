# Sistema de Gestión de Pólizas de Seguros con Chatbot

**Fecha:** 2026-05-30  
**Estado:** Aprobado  

---

## Resumen

Sistema local (con miras a nube) que permite cargar pólizas de seguros en PDF, extraer sus datos automáticamente, organizarlos en una base de datos SQLite y exponerlos a través de un chatbot web. Los clientes acceden por nombre para consultar sus pólizas y solicitar el envío de documentos a su correo electrónico.

---

## Stack tecnológico

| Capa | Tecnología |
|------|-----------|
| Backend | Python + FastAPI |
| Base de datos | SQLite (migrable a PostgreSQL) |
| Extracción PDF | pdfplumber |
| LLM / Chatbot | Claude API (Anthropic) con tool use |
| Frontend | HTML + JavaScript vanilla |
| Envío de correo | smtplib (Gmail) |
| Autenticación | JWT (8h), login por nombre |

---

## Arquitectura general

```
┌─────────────────────────────────────────────────────┐
│                   FRONTEND (HTML/JS)                 │
│  • Página de login por nombre                        │
│  • Interfaz de chat                                  │
│  • Panel admin: subir PDFs de pólizas                │
└────────────────────┬────────────────────────────────┘
                     │ HTTP / REST
┌────────────────────▼────────────────────────────────┐
│                  BACKEND (FastAPI)                   │
│                                                      │
│  /upload-pdf   → extrae datos → guarda en BD         │
│  /chat         → Claude API con tool use             │
│  /send-policy  → envía PDF por correo                │
│  /auth         → autenticación por nombre            │
└───────┬──────────────────────┬───────────────────────┘
        │                      │
┌───────▼──────┐    ┌──────────▼──────────┐
│  SQLite DB   │    │  Sistema de archivos │
│              │    │                      │
│  clientes    │    │  storage/            │
│  pólizas     │    │    Juan García/      │
│  coberturas  │    │      poliza_001.pdf  │
└──────────────┘    │    Empresa SA/       │
                    │      poliza_002.pdf  │
                    └─────────────────────┘
```

---

## Base de datos

### Tabla `clientes`

| Campo | Tipo | Notas |
|-------|------|-------|
| id | INTEGER PK | autoincrement |
| nombre | TEXT | nombre completo real |
| email | TEXT | para envío de pólizas |
| codigo_cliente | TEXT | código interno |
| rfc | TEXT | RFC del contratante |
| domicilio | TEXT | domicilio del contratante |
| created_at | DATETIME | |

### Tabla `polizas`

| Campo | Tipo | Notas |
|-------|------|-------|
| id | INTEGER PK | autoincrement |
| cliente_id | INTEGER FK | → clientes.id |
| numero_poliza | TEXT | |
| aseguradora | TEXT | |
| tipo_seguro | TEXT | auto, vida, GMM, etc. |
| plan_producto | TEXT | |
| contratante | TEXT | nombre en la póliza |
| asegurado | TEXT | |
| agente | TEXT | |
| clave_agente | TEXT | |
| fecha_inicio_vigencia | DATE | |
| fecha_termino_vigencia | DATE | |
| fecha_expedicion | DATE | |
| fecha_vencimiento_pago | DATE | |
| prima_neta | REAL | |
| derecho_poliza | REAL | gastos por expedición |
| iva | REAL | |
| importe_total | REAL | |
| moneda | TEXT | MXN, USD |
| forma_pago | TEXT | anual, semestral, etc. |
| registro_condusef_cnsf | TEXT | |
| archivo_pdf | TEXT | ruta relativa en storage/ |
| created_at | DATETIME | |

### Tabla `coberturas`

Una póliza puede tener múltiples coberturas.

| Campo | Tipo | Notas |
|-------|------|-------|
| id | INTEGER PK | autoincrement |
| poliza_id | INTEGER FK | → polizas.id |
| nombre_cobertura | TEXT | |
| suma_asegurada | REAL | |
| deducible | REAL | |
| coaseguro | REAL | porcentaje |
| prima_cobertura | REAL | |

---

## Organización de archivos en disco

Los PDFs se almacenan en `backend/storage/` bajo una carpeta con el nombre del cliente normalizado (sin acentos, sin caracteres especiales, espacios reemplazados por guiones bajos). El nombre de carpeta se genera una sola vez al crear el cliente y no cambia aunque se edite el nombre.

```
backend/storage/
  juan_garcia_perez/
    GNP-001_auto.pdf
    AXA-002_vida.pdf
  empresa_constructora_sa/
    MAPFRE-001_responsabilidad.pdf
```

---

## Flujo de carga de póliza

1. Admin accede a `/admin` (protegido por contraseña en `.env`)
2. Sube un PDF desde la interfaz
3. FastAPI recibe el archivo y lo pasa a `extractor.py`
4. `pdfplumber` extrae el texto crudo del PDF
5. Se envía ese texto a Claude con un prompt estructurado solicitando los campos en JSON
6. Los campos extraídos se insertan en `polizas` y `coberturas`
7. El PDF se mueve a `storage/<carpeta_cliente>/`
8. El admin ve el resultado: campos extraídos o campos con `null` que requieren revisión

---

## Chatbot con Tool Use

Claude no recibe la base de datos completa ni el historial completo. Dispone de tools específicas que consultan solo lo necesario:

### Tools disponibles

| Tool | Descripción |
|------|-------------|
| `get_polizas_cliente` | Lista de pólizas del cliente autenticado (campos básicos) |
| `get_detalle_poliza` | Todos los campos de una póliza específica por ID |
| `get_coberturas_poliza` | Coberturas de una póliza específica por ID |
| `enviar_poliza_por_email` | Envía el PDF de una póliza al correo registrado del cliente |

### Gestión de contexto

- El historial del chat se limita a los últimos 10 turnos
- El system prompt es estático y se cachea (`cache_control: ephemeral`)
- Claude solo recibe los datos que él mismo solicita vía tool calls
- Nunca se vuelca la tabla completa al contexto

### Ejemplo de flujo

```
Usuario: ¿Cuándo vence mi seguro de auto?

Claude → llama get_polizas_cliente(tipo="auto")
BD     → {id: 1, numero: "GNP-001", vencimiento: "2025-12-31"}
Claude → "Tu seguro de auto GNP-001 vence el 31 de diciembre de 2025."

Usuario: Mándame esa póliza por correo.
Claude → llama enviar_poliza_por_email(poliza_id=1)
Sistema → envía email con PDF adjunto
Claude → "Listo, te envié la póliza GNP-001 a tu correo registrado."
```

---

## Autenticación

- **Clientes:** login por nombre completo, sin contraseña (fase inicial). El sistema busca el nombre en la BD (búsqueda insensible a mayúsculas). Si existe, genera un JWT de 8 horas.
- **Admin:** contraseña fija configurada en `.env`, protege `/admin` y el endpoint `/upload-pdf`.
- **Mejora futura:** código de verificación enviado al correo del cliente para mayor seguridad.

---

## Envío de correo

- Librería: `smtplib` de Python estándar
- Proveedor: Gmail con App Password (configurado en `.env`)
- El PDF de la póliza se adjunta directamente al correo
- Las credenciales nunca van en el código fuente

---

## Frontend

Tres páginas HTML + JS vanilla (sin frameworks):

| Página | Ruta | Acceso |
|--------|------|--------|
| Login | `/` | Público |
| Chat | `/chat` | Cliente autenticado (JWT) |
| Admin | `/admin` | Contraseña de admin |

---

## Estructura de carpetas del proyecto

```
polizas-chatbot/
├── backend/
│   ├── main.py            # FastAPI: rutas y endpoints
│   ├── db.py              # Conexión SQLite, modelos, migraciones
│   ├── extractor.py       # Extracción de campos desde PDF
│   ├── chatbot.py         # Claude API + definición de tools
│   ├── email_service.py   # Envío de correos con adjunto
│   └── storage/           # PDFs organizados por cliente
├── frontend/
│   ├── index.html         # Login
│   ├── chat.html          # Interfaz del chatbot
│   └── admin.html         # Panel de carga de pólizas
├── .env.example           # Variables de entorno requeridas
└── requirements.txt       # Dependencias Python
```

---

## Variables de entorno requeridas (`.env`)

```
ANTHROPIC_API_KEY=
ADMIN_PASSWORD=
GMAIL_USER=
GMAIL_APP_PASSWORD=
JWT_SECRET=
```

---

## Consideraciones para despliegue en nube (futuro)

- SQLite → PostgreSQL: cambiar el driver en `db.py`, el resto del código no cambia
- `storage/` → bucket de almacenamiento (S3, GCS, etc.)
- Variables de entorno en el panel del proveedor (Railway, Render, Fly.io)
- Agregar HTTPS y autenticación más robusta (contraseña o verificación por email)
