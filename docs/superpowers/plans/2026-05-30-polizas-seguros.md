# Sistema de Gestión de Pólizas de Seguros con Chatbot — Plan de Implementación

> **Para workers agenticos:** SUB-SKILL REQUERIDO: Usa superpowers:subagent-driven-development (recomendado) o superpowers:executing-plans para implementar este plan tarea por tarea. Los pasos usan sintaxis de checkbox (`- [ ]`) para seguimiento.

**Goal:** Construir un sistema local que extrae datos de PDFs de pólizas de seguros, los almacena en SQLite, y los expone a través de un chatbot web donde los clientes consultan sus pólizas y reciben PDFs por correo.

**Architecture:** FastAPI en el backend con SQLite (3 tablas: clientes, polizas, coberturas). Los PDFs se guardan en `storage/<carpeta_cliente>/`. El chatbot usa Claude API con tool use para consultar solo los datos necesarios en cada pregunta, minimizando tokens. El frontend es HTML/JS vanilla con tres páginas: login, chat y admin.

**Tech Stack:** Python 3.11+, FastAPI, SQLite (sqlite3 stdlib), pdfplumber, anthropic SDK, PyJWT, python-dotenv, smtplib (stdlib), HTML/JS vanilla.

---

## Mapa de archivos

| Archivo | Responsabilidad |
|---------|----------------|
| `backend/db.py` | Conexión SQLite, init_db(), todas las funciones CRUD |
| `backend/auth.py` | Crear/decodificar JWT, dependencia FastAPI para rutas protegidas |
| `backend/extractor.py` | Extraer texto PDF con pdfplumber, llamar a Claude para parsear campos |
| `backend/email_service.py` | Enviar correo con PDF adjunto via smtplib/Gmail |
| `backend/chatbot.py` | Definición de tools, dispatcher, función chat() con agentic loop |
| `backend/main.py` | FastAPI app, todos los endpoints, CORS |
| `frontend/index.html` | Página de login por nombre |
| `frontend/chat.html` | Interfaz del chatbot |
| `frontend/admin.html` | Panel de carga de PDFs y gestión de clientes |
| `tests/conftest.py` | Fixtures compartidas (BD temporal) |
| `tests/test_db.py` | Tests de funciones CRUD |
| `tests/test_auth.py` | Tests de JWT |
| `tests/test_extractor.py` | Tests de normalización y extracción (mock Claude) |
| `tests/test_chatbot_tools.py` | Tests del dispatcher de tools |
| `tests/test_email.py` | Tests del envío de correo (mock smtplib) |

---

## Task 1: Setup del proyecto

**Files:**
- Create: `polizas-chatbot/requirements.txt`
- Create: `polizas-chatbot/.env.example`
- Create: `polizas-chatbot/.gitignore`
- Create: `polizas-chatbot/backend/__init__.py`
- Create: `polizas-chatbot/tests/__init__.py`

- [ ] **Step 1: Crear estructura de carpetas**

```bash
mkdir -p polizas-chatbot/backend/storage
mkdir -p polizas-chatbot/frontend
mkdir -p polizas-chatbot/tests
touch polizas-chatbot/backend/__init__.py
touch polizas-chatbot/tests/__init__.py
cd polizas-chatbot
```

- [ ] **Step 2: Crear requirements.txt**

```
fastapi==0.115.0
uvicorn==0.32.0
pdfplumber==0.11.4
anthropic==0.40.0
PyJWT==2.10.1
python-dotenv==1.0.1
python-multipart==0.0.12
pytest==8.3.0
pytest-mock==3.14.0
```

- [ ] **Step 3: Crear .env.example**

```
ANTHROPIC_API_KEY=sk-ant-...
ADMIN_PASSWORD=cambia_esto
GMAIL_USER=tu_cuenta@gmail.com
GMAIL_APP_PASSWORD=xxxx_xxxx_xxxx_xxxx
JWT_SECRET=cambia_esto_por_algo_largo_y_aleatorio
```

- [ ] **Step 4: Crear .gitignore**

```
.env
__pycache__/
*.pyc
*.db
backend/storage/
.pytest_cache/
```

- [ ] **Step 5: Instalar dependencias**

```bash
python -m venv venv
# Windows:
venv\Scripts\activate
# Mac/Linux:
source venv/bin/activate

pip install -r requirements.txt
```

Esperado: instalación sin errores.

- [ ] **Step 6: Crear .env desde el ejemplo**

```bash
cp .env.example .env
# Editar .env con valores reales antes de continuar
```

- [ ] **Step 7: Commit inicial**

```bash
git init
git add requirements.txt .env.example .gitignore backend/__init__.py tests/__init__.py
git commit -m "chore: project setup"
```

---

## Task 2: Capa de base de datos

**Files:**
- Create: `backend/db.py`
- Create: `tests/conftest.py`
- Create: `tests/test_db.py`

- [ ] **Step 1: Escribir los tests primero**

Crear `tests/conftest.py`:

```python
import pytest
from pathlib import Path
import backend.db as db_module


@pytest.fixture
def tmp_db(monkeypatch, tmp_path):
    db_file = tmp_path / "test.db"
    monkeypatch.setattr(db_module, "DB_PATH", db_file)
    db_module.init_db()
    return db_file
```

Crear `tests/test_db.py`:

```python
import pytest
from backend.db import (
    init_db, create_cliente, get_cliente_by_nombre,
    create_poliza, get_polizas_by_cliente, get_detalle_poliza,
    create_coberturas, get_coberturas_by_poliza
)


def test_init_db_creates_tables(tmp_db):
    import sqlite3
    conn = sqlite3.connect(tmp_db)
    tables = {r[0] for r in conn.execute("SELECT name FROM sqlite_master WHERE type='table'").fetchall()}
    conn.close()
    assert {"clientes", "polizas", "coberturas"}.issubset(tables)


def test_create_and_get_cliente(tmp_db):
    cliente_id = create_cliente("Juan García", "juan@mail.com", "C001", "RFC123", "Calle 1")
    cliente = get_cliente_by_nombre("Juan García")
    assert cliente is not None
    assert cliente["id"] == cliente_id
    assert cliente["email"] == "juan@mail.com"


def test_get_cliente_case_insensitive(tmp_db):
    create_cliente("María López", "maria@mail.com", None, None, None)
    assert get_cliente_by_nombre("maría lópez") is not None
    assert get_cliente_by_nombre("MARÍA LÓPEZ") is not None


def test_get_cliente_not_found(tmp_db):
    assert get_cliente_by_nombre("Nadie Aqui") is None


def test_create_poliza_and_get_detalle(tmp_db):
    cliente_id = create_cliente("Ana Torres", "ana@mail.com", None, None, None)
    data = {
        "numero_poliza": "GNP-001",
        "aseguradora": "GNP",
        "tipo_seguro": "auto",
        "plan_producto": "Plan A",
        "contratante": "Ana Torres",
        "asegurado": "Ana Torres",
        "agente": "Pedro Ruiz",
        "clave_agente": "AG001",
        "fecha_inicio_vigencia": "2025-01-01",
        "fecha_termino_vigencia": "2026-01-01",
        "fecha_expedicion": "2024-12-15",
        "fecha_vencimiento_pago": "2025-01-15",
        "prima_neta": 5000.0,
        "derecho_poliza": 200.0,
        "iva": 851.2,
        "importe_total": 6051.2,
        "moneda": "MXN",
        "forma_pago": "anual",
        "registro_condusef_cnsf": "123-ABC",
        "archivo_pdf": "storage/ana_torres/GNP-001.pdf",
    }
    poliza_id = create_poliza(cliente_id, data)
    detalle = get_detalle_poliza(poliza_id, cliente_id)
    assert detalle is not None
    assert detalle["numero_poliza"] == "GNP-001"
    assert detalle["prima_neta"] == 5000.0


def test_get_detalle_poliza_wrong_cliente(tmp_db):
    c1 = create_cliente("Cliente Uno", "c1@mail.com", None, None, None)
    c2 = create_cliente("Cliente Dos", "c2@mail.com", None, None, None)
    poliza_id = create_poliza(c1, {"numero_poliza": "P-001", "aseguradora": "X",
        "tipo_seguro": None, "plan_producto": None, "contratante": None,
        "asegurado": None, "agente": None, "clave_agente": None,
        "fecha_inicio_vigencia": None, "fecha_termino_vigencia": None,
        "fecha_expedicion": None, "fecha_vencimiento_pago": None,
        "prima_neta": None, "derecho_poliza": None, "iva": None,
        "importe_total": None, "moneda": None, "forma_pago": None,
        "registro_condusef_cnsf": None, "archivo_pdf": None})
    assert get_detalle_poliza(poliza_id, c2) is None


def test_get_polizas_by_cliente_filter_tipo(tmp_db):
    cliente_id = create_cliente("Luis Mora", "luis@mail.com", None, None, None)
    base = {"aseguradora": "X", "plan_producto": None, "contratante": None,
            "asegurado": None, "agente": None, "clave_agente": None,
            "fecha_inicio_vigencia": None, "fecha_termino_vigencia": None,
            "fecha_expedicion": None, "fecha_vencimiento_pago": None,
            "prima_neta": None, "derecho_poliza": None, "iva": None,
            "importe_total": None, "moneda": None, "forma_pago": None,
            "registro_condusef_cnsf": None, "archivo_pdf": None}
    create_poliza(cliente_id, {**base, "numero_poliza": "A-001", "tipo_seguro": "auto"})
    create_poliza(cliente_id, {**base, "numero_poliza": "V-001", "tipo_seguro": "vida"})
    autos = get_polizas_by_cliente(cliente_id, tipo_seguro="auto")
    assert len(autos) == 1
    assert autos[0]["numero_poliza"] == "A-001"


def test_create_coberturas(tmp_db):
    cliente_id = create_cliente("Rosa Gil", "rosa@mail.com", None, None, None)
    poliza_id = create_poliza(cliente_id, {"numero_poliza": "P-COB", "aseguradora": "AXA",
        "tipo_seguro": None, "plan_producto": None, "contratante": None,
        "asegurado": None, "agente": None, "clave_agente": None,
        "fecha_inicio_vigencia": None, "fecha_termino_vigencia": None,
        "fecha_expedicion": None, "fecha_vencimiento_pago": None,
        "prima_neta": None, "derecho_poliza": None, "iva": None,
        "importe_total": None, "moneda": None, "forma_pago": None,
        "registro_condusef_cnsf": None, "archivo_pdf": None})
    coberturas = [
        {"nombre_cobertura": "Daños materiales", "suma_asegurada": 200000.0,
         "deducible": 5000.0, "coaseguro": 0.0, "prima_cobertura": 1500.0},
        {"nombre_cobertura": "Robo total", "suma_asegurada": 200000.0,
         "deducible": 10000.0, "coaseguro": 0.0, "prima_cobertura": 800.0},
    ]
    create_coberturas(poliza_id, coberturas)
    result = get_coberturas_by_poliza(poliza_id)
    assert len(result) == 2
    assert result[0]["nombre_cobertura"] == "Daños materiales"
```

- [ ] **Step 2: Ejecutar tests para verificar que fallan**

```bash
pytest tests/test_db.py -v
```

Esperado: todos los tests FAIL con `ModuleNotFoundError` o `ImportError`.

- [ ] **Step 3: Implementar backend/db.py**

```python
import sqlite3
import unicodedata
import re
from contextlib import contextmanager
from pathlib import Path

DB_PATH = Path(__file__).parent / "polizas.db"
STORAGE_PATH = Path(__file__).parent / "storage"


def normalize_folder_name(nombre: str) -> str:
    nfkd = unicodedata.normalize("NFKD", nombre)
    ascii_str = nfkd.encode("ascii", "ignore").decode("ascii")
    return re.sub(r"[^a-zA-Z0-9]+", "_", ascii_str.lower()).strip("_")


@contextmanager
def get_conn():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA foreign_keys = ON")
    try:
        yield conn
        conn.commit()
    finally:
        conn.close()


def init_db() -> None:
    with get_conn() as conn:
        conn.executescript("""
            CREATE TABLE IF NOT EXISTS clientes (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                nombre TEXT NOT NULL,
                email TEXT,
                codigo_cliente TEXT,
                rfc TEXT,
                domicilio TEXT,
                carpeta TEXT NOT NULL,
                created_at DATETIME DEFAULT CURRENT_TIMESTAMP
            );
            CREATE TABLE IF NOT EXISTS polizas (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                cliente_id INTEGER NOT NULL REFERENCES clientes(id),
                numero_poliza TEXT,
                aseguradora TEXT,
                tipo_seguro TEXT,
                plan_producto TEXT,
                contratante TEXT,
                asegurado TEXT,
                agente TEXT,
                clave_agente TEXT,
                fecha_inicio_vigencia TEXT,
                fecha_termino_vigencia TEXT,
                fecha_expedicion TEXT,
                fecha_vencimiento_pago TEXT,
                prima_neta REAL,
                derecho_poliza REAL,
                iva REAL,
                importe_total REAL,
                moneda TEXT,
                forma_pago TEXT,
                registro_condusef_cnsf TEXT,
                archivo_pdf TEXT,
                created_at DATETIME DEFAULT CURRENT_TIMESTAMP
            );
            CREATE TABLE IF NOT EXISTS coberturas (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                poliza_id INTEGER NOT NULL REFERENCES polizas(id),
                nombre_cobertura TEXT,
                suma_asegurada REAL,
                deducible REAL,
                coaseguro REAL,
                prima_cobertura REAL
            );
        """)


def _row_to_dict(row) -> dict | None:
    if row is None:
        return None
    return dict(row)


def create_cliente(nombre: str, email: str | None, codigo_cliente: str | None,
                   rfc: str | None, domicilio: str | None) -> int:
    carpeta = normalize_folder_name(nombre)
    with get_conn() as conn:
        cur = conn.execute(
            "INSERT INTO clientes (nombre, email, codigo_cliente, rfc, domicilio, carpeta) "
            "VALUES (?, ?, ?, ?, ?, ?)",
            (nombre, email, codigo_cliente, rfc, domicilio, carpeta)
        )
        return cur.lastrowid


def get_cliente_by_nombre(nombre: str) -> dict | None:
    with get_conn() as conn:
        row = conn.execute(
            "SELECT * FROM clientes WHERE LOWER(nombre) = LOWER(?)", (nombre,)
        ).fetchone()
        return _row_to_dict(row)


def get_cliente_by_id(cliente_id: int) -> dict | None:
    with get_conn() as conn:
        row = conn.execute("SELECT * FROM clientes WHERE id = ?", (cliente_id,)).fetchone()
        return _row_to_dict(row)


def update_cliente_email(cliente_id: int, email: str) -> None:
    with get_conn() as conn:
        conn.execute("UPDATE clientes SET email = ? WHERE id = ?", (email, cliente_id))


def list_clientes() -> list[dict]:
    with get_conn() as conn:
        rows = conn.execute("SELECT * FROM clientes ORDER BY nombre").fetchall()
        return [dict(r) for r in rows]


def create_poliza(cliente_id: int, data: dict) -> int:
    with get_conn() as conn:
        cur = conn.execute(
            """INSERT INTO polizas
               (cliente_id, numero_poliza, aseguradora, tipo_seguro, plan_producto,
                contratante, asegurado, agente, clave_agente,
                fecha_inicio_vigencia, fecha_termino_vigencia, fecha_expedicion,
                fecha_vencimiento_pago, prima_neta, derecho_poliza, iva,
                importe_total, moneda, forma_pago, registro_condusef_cnsf, archivo_pdf)
               VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)""",
            (cliente_id, data.get("numero_poliza"), data.get("aseguradora"),
             data.get("tipo_seguro"), data.get("plan_producto"),
             data.get("contratante"), data.get("asegurado"),
             data.get("agente"), data.get("clave_agente"),
             data.get("fecha_inicio_vigencia"), data.get("fecha_termino_vigencia"),
             data.get("fecha_expedicion"), data.get("fecha_vencimiento_pago"),
             data.get("prima_neta"), data.get("derecho_poliza"), data.get("iva"),
             data.get("importe_total"), data.get("moneda"), data.get("forma_pago"),
             data.get("registro_condusef_cnsf"), data.get("archivo_pdf"))
        )
        return cur.lastrowid


def get_polizas_by_cliente(cliente_id: int, tipo_seguro: str | None = None) -> list[dict]:
    with get_conn() as conn:
        if tipo_seguro:
            rows = conn.execute(
                "SELECT id, numero_poliza, aseguradora, tipo_seguro, "
                "fecha_inicio_vigencia, fecha_termino_vigencia, fecha_vencimiento_pago "
                "FROM polizas WHERE cliente_id = ? AND LOWER(tipo_seguro) = LOWER(?)",
                (cliente_id, tipo_seguro)
            ).fetchall()
        else:
            rows = conn.execute(
                "SELECT id, numero_poliza, aseguradora, tipo_seguro, "
                "fecha_inicio_vigencia, fecha_termino_vigencia, fecha_vencimiento_pago "
                "FROM polizas WHERE cliente_id = ?",
                (cliente_id,)
            ).fetchall()
        return [dict(r) for r in rows]


def get_detalle_poliza(poliza_id: int, cliente_id: int) -> dict | None:
    with get_conn() as conn:
        row = conn.execute(
            "SELECT * FROM polizas WHERE id = ? AND cliente_id = ?",
            (poliza_id, cliente_id)
        ).fetchone()
        return _row_to_dict(row)


def create_coberturas(poliza_id: int, coberturas: list[dict]) -> None:
    with get_conn() as conn:
        conn.executemany(
            "INSERT INTO coberturas (poliza_id, nombre_cobertura, suma_asegurada, "
            "deducible, coaseguro, prima_cobertura) VALUES (?,?,?,?,?,?)",
            [(poliza_id, c.get("nombre_cobertura"), c.get("suma_asegurada"),
              c.get("deducible"), c.get("coaseguro"), c.get("prima_cobertura"))
             for c in coberturas]
        )


def get_coberturas_by_poliza(poliza_id: int) -> list[dict]:
    with get_conn() as conn:
        rows = conn.execute(
            "SELECT * FROM coberturas WHERE poliza_id = ?", (poliza_id,)
        ).fetchall()
        return [dict(r) for r in rows]
```

- [ ] **Step 4: Ejecutar tests y verificar que pasan**

```bash
pytest tests/test_db.py -v
```

Esperado: todos los tests PASS.

- [ ] **Step 5: Commit**

```bash
git add backend/db.py tests/conftest.py tests/test_db.py
git commit -m "feat: database layer with SQLite CRUD functions"
```

---

## Task 3: Módulo de autenticación

**Files:**
- Create: `backend/auth.py`
- Create: `tests/test_auth.py`

- [ ] **Step 1: Escribir los tests**

Crear `tests/test_auth.py`:

```python
import time
import pytest
import jwt as pyjwt
from backend.auth import create_token, decode_token


def test_create_and_decode_token(monkeypatch):
    monkeypatch.setenv("JWT_SECRET", "test-secret")
    token = create_token(cliente_id=42, nombre="Juan García")
    payload = decode_token(token)
    assert payload["sub"] == "42"
    assert payload["nombre"] == "Juan García"


def test_expired_token_raises(monkeypatch):
    monkeypatch.setenv("JWT_SECRET", "test-secret")
    import backend.auth as auth_module
    monkeypatch.setattr(auth_module, "EXPIRY_HOURS", 0)
    token = create_token(cliente_id=1, nombre="Test")
    time.sleep(1)
    with pytest.raises(pyjwt.ExpiredSignatureError):
        decode_token(token)


def test_invalid_token_raises():
    with pytest.raises(pyjwt.InvalidTokenError):
        decode_token("token.invalido.aqui")
```

- [ ] **Step 2: Ejecutar tests para verificar que fallan**

```bash
pytest tests/test_auth.py -v
```

Esperado: FAIL con `ModuleNotFoundError`.

- [ ] **Step 3: Implementar backend/auth.py**

```python
import os
from datetime import datetime, timedelta, timezone

import jwt

SECRET = os.getenv("JWT_SECRET", "dev-secret-insecuro")
ALGORITHM = "HS256"
EXPIRY_HOURS = 8


def create_token(cliente_id: int, nombre: str) -> str:
    payload = {
        "sub": str(cliente_id),
        "nombre": nombre,
        "exp": datetime.now(timezone.utc) + timedelta(hours=EXPIRY_HOURS),
    }
    return jwt.encode(payload, SECRET, algorithm=ALGORITHM)


def decode_token(token: str) -> dict:
    return jwt.decode(token, SECRET, algorithms=[ALGORITHM])
```

- [ ] **Step 4: Ejecutar tests y verificar que pasan**

```bash
pytest tests/test_auth.py -v
```

Esperado: todos PASS.

- [ ] **Step 5: Commit**

```bash
git add backend/auth.py tests/test_auth.py
git commit -m "feat: JWT authentication module"
```

---

## Task 4: Extractor de PDFs

**Files:**
- Create: `backend/extractor.py`
- Create: `tests/test_extractor.py`

- [ ] **Step 1: Escribir los tests**

Crear `tests/test_extractor.py`:

```python
import json
import pytest
from unittest.mock import MagicMock, patch
from backend.db import normalize_folder_name
from backend.extractor import extract_fields_from_text


def test_normalize_folder_name_basic():
    assert normalize_folder_name("Juan García Pérez") == "juan_garcia_perez"


def test_normalize_folder_name_empresa():
    assert normalize_folder_name("Empresa Constructora, S.A.") == "empresa_constructora_s_a"


def test_normalize_folder_name_special_chars():
    assert normalize_folder_name("  Ñoño & Cía  ") == "nono_cia"


def test_extract_fields_from_text_calls_claude(monkeypatch):
    expected = {
        "numero_poliza": "GNP-001",
        "aseguradora": "GNP",
        "tipo_seguro": "auto",
        "plan_producto": "Flotilla",
        "contratante": "Juan García",
        "asegurado": "Juan García",
        "rfc_contratante": "GAJU800101ABC",
        "codigo_cliente": "C001",
        "domicilio_contratante": "Calle 1 #2, CDMX",
        "agente": "Pedro Ruiz",
        "clave_agente": "AG001",
        "fecha_inicio_vigencia": "2025-01-01",
        "fecha_termino_vigencia": "2026-01-01",
        "fecha_expedicion": "2024-12-15",
        "fecha_vencimiento_pago": "2025-01-15",
        "prima_neta": 5000.0,
        "derecho_poliza": 200.0,
        "iva": 851.2,
        "importe_total": 6051.2,
        "moneda": "MXN",
        "forma_pago": "anual",
        "registro_condusef_cnsf": "123-ABC",
        "coberturas": [
            {"nombre_cobertura": "Daños materiales", "suma_asegurada": 200000.0,
             "deducible": 5000.0, "coaseguro": 0.0, "prima_cobertura": 1500.0}
        ]
    }
    mock_text_block = MagicMock()
    mock_text_block.text = json.dumps(expected)
    mock_response = MagicMock()
    mock_response.content = [mock_text_block]

    with patch("backend.extractor.anthropic.Anthropic") as mock_anthropic:
        mock_client = MagicMock()
        mock_anthropic.return_value = mock_client
        mock_client.messages.create.return_value = mock_response

        result = extract_fields_from_text("texto de póliza de prueba")

    assert result["numero_poliza"] == "GNP-001"
    assert result["prima_neta"] == 5000.0
    assert len(result["coberturas"]) == 1
```

- [ ] **Step 2: Ejecutar tests para verificar que fallan**

```bash
pytest tests/test_extractor.py -v
```

Esperado: FAIL con `ModuleNotFoundError`.

- [ ] **Step 3: Implementar backend/extractor.py**

```python
import json
import os
import re
from pathlib import Path

import anthropic
import pdfplumber

EXTRACTION_PROMPT = """Extrae los siguientes campos del texto de esta póliza de seguro y devuélvelos ÚNICAMENTE como JSON válido, sin texto adicional ni marcadores markdown.

Campos a extraer (usa null si no se encuentra):
- numero_poliza (string)
- aseguradora (string)
- tipo_seguro (string: auto, vida, GMM, responsabilidad, etc.)
- plan_producto (string)
- contratante (string, nombre completo)
- asegurado (string, nombre completo)
- rfc_contratante (string)
- codigo_cliente (string)
- domicilio_contratante (string)
- agente (string)
- clave_agente (string)
- fecha_inicio_vigencia (string YYYY-MM-DD)
- fecha_termino_vigencia (string YYYY-MM-DD)
- fecha_expedicion (string YYYY-MM-DD)
- fecha_vencimiento_pago (string YYYY-MM-DD)
- prima_neta (number)
- derecho_poliza (number)
- iva (number)
- importe_total (number)
- moneda (string: MXN o USD)
- forma_pago (string: anual, semestral, trimestral, mensual)
- registro_condusef_cnsf (string)
- coberturas (array de objetos con: nombre_cobertura, suma_asegurada, deducible, coaseguro, prima_cobertura)

Texto de la póliza:
{text}"""


def extract_text_from_pdf(pdf_path: str) -> str:
    text_parts = []
    with pdfplumber.open(pdf_path) as pdf:
        for page in pdf.pages:
            page_text = page.extract_text()
            if page_text:
                text_parts.append(page_text)
    return "\n".join(text_parts)


def extract_fields_from_text(text: str) -> dict:
    client = anthropic.Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=2048,
        messages=[
            {
                "role": "user",
                "content": EXTRACTION_PROMPT.format(text=text[:8000])
            }
        ]
    )
    raw = response.content[0].text.strip()
    # Remove markdown code blocks if present
    raw = re.sub(r"^```(?:json)?\n?", "", raw)
    raw = re.sub(r"\n?```$", "", raw)
    return json.loads(raw)


def extract_from_pdf(pdf_path: str) -> dict:
    text = extract_text_from_pdf(pdf_path)
    return extract_fields_from_text(text)
```

- [ ] **Step 4: Ejecutar tests y verificar que pasan**

```bash
pytest tests/test_extractor.py -v
```

Esperado: todos PASS.

- [ ] **Step 5: Commit**

```bash
git add backend/extractor.py tests/test_extractor.py
git commit -m "feat: PDF extractor with pdfplumber and Claude parsing"
```

---

## Task 5: Servicio de correo electrónico

**Files:**
- Create: `backend/email_service.py`
- Create: `tests/test_email.py`

- [ ] **Step 1: Escribir los tests**

Crear `tests/test_email.py`:

```python
import pytest
from unittest.mock import patch, MagicMock
from backend.email_service import send_policy_email


def test_send_policy_email_calls_smtp(tmp_path, monkeypatch):
    monkeypatch.setenv("GMAIL_USER", "test@gmail.com")
    monkeypatch.setenv("GMAIL_APP_PASSWORD", "app-password")

    pdf_file = tmp_path / "poliza.pdf"
    pdf_file.write_bytes(b"%PDF-1.4 fake pdf content")

    with patch("backend.email_service.smtplib.SMTP_SSL") as mock_smtp_cls:
        mock_smtp = MagicMock()
        mock_smtp_cls.return_value.__enter__.return_value = mock_smtp

        send_policy_email(
            to_email="cliente@mail.com",
            pdf_path=str(pdf_file),
            numero_poliza="GNP-001"
        )

        mock_smtp.login.assert_called_once_with("test@gmail.com", "app-password")
        mock_smtp.send_message.assert_called_once()


def test_send_policy_email_pdf_not_found():
    with pytest.raises(FileNotFoundError):
        send_policy_email("a@b.com", "/no/existe/poliza.pdf", "X-001")
```

- [ ] **Step 2: Ejecutar tests para verificar que fallan**

```bash
pytest tests/test_email.py -v
```

Esperado: FAIL con `ModuleNotFoundError`.

- [ ] **Step 3: Implementar backend/email_service.py**

```python
import os
import smtplib
from email.mime.application import MIMEApplication
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from pathlib import Path


def send_policy_email(to_email: str, pdf_path: str, numero_poliza: str) -> None:
    pdf_file = Path(pdf_path)
    if not pdf_file.exists():
        raise FileNotFoundError(f"PDF no encontrado: {pdf_path}")

    gmail_user = os.getenv("GMAIL_USER")
    gmail_password = os.getenv("GMAIL_APP_PASSWORD")

    msg = MIMEMultipart()
    msg["From"] = gmail_user
    msg["To"] = to_email
    msg["Subject"] = f"Tu póliza {numero_poliza}"

    body = (
        f"Estimado cliente,\n\n"
        f"Adjuntamos tu póliza {numero_poliza} tal como solicitaste.\n\n"
        f"Si tienes alguna pregunta, puedes consultarla en nuestro chat.\n\n"
        f"Saludos."
    )
    msg.attach(MIMEText(body, "plain", "utf-8"))

    with open(pdf_path, "rb") as f:
        pdf_attachment = MIMEApplication(f.read(), _subtype="pdf")
        pdf_attachment.add_header(
            "Content-Disposition", "attachment", filename=pdf_file.name
        )
        msg.attach(pdf_attachment)

    with smtplib.SMTP_SSL("smtp.gmail.com", 465) as smtp:
        smtp.login(gmail_user, gmail_password)
        smtp.send_message(msg)
```

- [ ] **Step 4: Ejecutar tests y verificar que pasan**

```bash
pytest tests/test_email.py -v
```

Esperado: todos PASS.

- [ ] **Step 5: Commit**

```bash
git add backend/email_service.py tests/test_email.py
git commit -m "feat: email service for sending policy PDFs"
```

---

## Task 6: Chatbot con Tool Use

**Files:**
- Create: `backend/chatbot.py`
- Create: `tests/test_chatbot_tools.py`

- [ ] **Step 1: Escribir los tests**

Crear `tests/test_chatbot_tools.py`:

```python
import pytest
from unittest.mock import patch, MagicMock
from backend.chatbot import dispatch_tool


def test_dispatch_get_polizas(tmp_db, monkeypatch):
    import backend.db as db
    cliente_id = db.create_cliente("Test User", "t@mail.com", None, None, None)
    base = {"aseguradora": "GNP", "plan_producto": None, "contratante": None,
            "asegurado": None, "agente": None, "clave_agente": None,
            "fecha_inicio_vigencia": "2025-01-01", "fecha_termino_vigencia": "2026-01-01",
            "fecha_expedicion": None, "fecha_vencimiento_pago": None,
            "prima_neta": None, "derecho_poliza": None, "iva": None,
            "importe_total": None, "moneda": "MXN", "forma_pago": None,
            "registro_condusef_cnsf": None, "archivo_pdf": None}
    db.create_poliza(cliente_id, {**base, "numero_poliza": "P-001", "tipo_seguro": "auto"})

    result = dispatch_tool(
        tool_name="get_polizas_cliente",
        tool_input={},
        cliente_id=cliente_id,
        cliente_email="t@mail.com"
    )
    assert len(result) == 1
    assert result[0]["numero_poliza"] == "P-001"


def test_dispatch_get_detalle(tmp_db):
    import backend.db as db
    cliente_id = db.create_cliente("Test User2", "t2@mail.com", None, None, None)
    poliza_id = db.create_poliza(cliente_id, {
        "numero_poliza": "AXA-001", "aseguradora": "AXA", "tipo_seguro": "vida",
        "plan_producto": None, "contratante": None, "asegurado": None,
        "agente": None, "clave_agente": None,
        "fecha_inicio_vigencia": None, "fecha_termino_vigencia": None,
        "fecha_expedicion": None, "fecha_vencimiento_pago": None,
        "prima_neta": 3000.0, "derecho_poliza": None, "iva": None,
        "importe_total": None, "moneda": None, "forma_pago": None,
        "registro_condusef_cnsf": None, "archivo_pdf": None
    })
    result = dispatch_tool("get_detalle_poliza", {"poliza_id": poliza_id},
                           cliente_id, "t2@mail.com")
    assert result["prima_neta"] == 3000.0
    assert result["aseguradora"] == "AXA"


def test_dispatch_get_detalle_wrong_client(tmp_db):
    import backend.db as db
    c1 = db.create_cliente("Owner", "o@mail.com", None, None, None)
    c2 = db.create_cliente("Other", "other@mail.com", None, None, None)
    poliza_id = db.create_poliza(c1, {
        "numero_poliza": "X-001", "aseguradora": "X", "tipo_seguro": None,
        "plan_producto": None, "contratante": None, "asegurado": None,
        "agente": None, "clave_agente": None, "fecha_inicio_vigencia": None,
        "fecha_termino_vigencia": None, "fecha_expedicion": None,
        "fecha_vencimiento_pago": None, "prima_neta": None, "derecho_poliza": None,
        "iva": None, "importe_total": None, "moneda": None, "forma_pago": None,
        "registro_condusef_cnsf": None, "archivo_pdf": None
    })
    result = dispatch_tool("get_detalle_poliza", {"poliza_id": poliza_id},
                           c2, "other@mail.com")
    assert result is None


def test_dispatch_enviar_email(tmp_db, tmp_path, monkeypatch):
    import backend.db as db
    from pathlib import Path
    cliente_id = db.create_cliente("Email User", "eu@mail.com", None, None, None)
    pdf_file = tmp_path / "poliza.pdf"
    pdf_file.write_bytes(b"%PDF fake")
    poliza_id = db.create_poliza(cliente_id, {
        "numero_poliza": "E-001", "aseguradora": "Z", "tipo_seguro": None,
        "plan_producto": None, "contratante": None, "asegurado": None,
        "agente": None, "clave_agente": None, "fecha_inicio_vigencia": None,
        "fecha_termino_vigencia": None, "fecha_expedicion": None,
        "fecha_vencimiento_pago": None, "prima_neta": None, "derecho_poliza": None,
        "iva": None, "importe_total": None, "moneda": None, "forma_pago": None,
        "registro_condusef_cnsf": None, "archivo_pdf": str(pdf_file)
    })
    with patch("backend.chatbot.send_policy_email") as mock_send:
        result = dispatch_tool("enviar_poliza_por_email", {"poliza_id": poliza_id},
                               cliente_id, "eu@mail.com")
        mock_send.assert_called_once()
    assert result["enviado"] is True
```

- [ ] **Step 2: Ejecutar tests para verificar que fallan**

```bash
pytest tests/test_chatbot_tools.py -v
```

Esperado: FAIL con `ModuleNotFoundError`.

- [ ] **Step 3: Implementar backend/chatbot.py**

```python
import json
import os
from pathlib import Path

import anthropic

from backend.db import (
    get_polizas_by_cliente,
    get_detalle_poliza,
    get_coberturas_by_poliza,
)
from backend.email_service import send_policy_email

SYSTEM_PROMPT = """Eres un asistente de seguros. Ayudas a los clientes a consultar
información sobre sus pólizas de seguros contratadas.

Reglas:
- Solo puedes compartir información de las pólizas del cliente autenticado.
- Usa las herramientas disponibles para consultar la base de datos. No inventes información.
- Cuando el cliente pida su póliza por correo, usa la herramienta enviar_poliza_por_email.
- Si un cliente pregunta por una póliza que no existe en sus datos, díselo claramente.
- Responde en español, de forma clara y concisa.
- Usa formatos legibles: escribe fechas como "1 de enero de 2025" y montos como "$5,000.00 MXN"."""

TOOLS = [
    {
        "name": "get_polizas_cliente",
        "description": (
            "Obtiene la lista de pólizas del cliente. Devuelve número de póliza, "
            "aseguradora, tipo de seguro y fechas de vigencia."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "tipo_seguro": {
                    "type": "string",
                    "description": "Filtrar por tipo (auto, vida, GMM, etc.). Omitir para ver todas."
                }
            },
            "required": []
        }
    },
    {
        "name": "get_detalle_poliza",
        "description": "Obtiene todos los campos de una póliza específica por su ID.",
        "input_schema": {
            "type": "object",
            "properties": {
                "poliza_id": {"type": "integer", "description": "ID de la póliza"}
            },
            "required": ["poliza_id"]
        }
    },
    {
        "name": "get_coberturas_poliza",
        "description": "Obtiene las coberturas detalladas de una póliza específica.",
        "input_schema": {
            "type": "object",
            "properties": {
                "poliza_id": {"type": "integer", "description": "ID de la póliza"}
            },
            "required": ["poliza_id"]
        }
    },
    {
        "name": "enviar_poliza_por_email",
        "description": "Envía el PDF de la póliza al correo electrónico registrado del cliente.",
        "input_schema": {
            "type": "object",
            "properties": {
                "poliza_id": {"type": "integer", "description": "ID de la póliza a enviar"}
            },
            "required": ["poliza_id"]
        }
    }
]

MAX_HISTORY_MESSAGES = 20


def dispatch_tool(tool_name: str, tool_input: dict,
                  cliente_id: int, cliente_email: str) -> dict | list | None:
    if tool_name == "get_polizas_cliente":
        return get_polizas_by_cliente(
            cliente_id, tipo_seguro=tool_input.get("tipo_seguro")
        )
    if tool_name == "get_detalle_poliza":
        return get_detalle_poliza(tool_input["poliza_id"], cliente_id)
    if tool_name == "get_coberturas_poliza":
        return get_coberturas_by_poliza(tool_input["poliza_id"])
    if tool_name == "enviar_poliza_por_email":
        detalle = get_detalle_poliza(tool_input["poliza_id"], cliente_id)
        if detalle is None:
            return {"enviado": False, "error": "Póliza no encontrada"}
        if not detalle.get("archivo_pdf"):
            return {"enviado": False, "error": "El PDF de esta póliza no está disponible"}
        send_policy_email(
            to_email=cliente_email,
            pdf_path=detalle["archivo_pdf"],
            numero_poliza=detalle["numero_poliza"]
        )
        return {"enviado": True, "numero_poliza": detalle["numero_poliza"]}
    return {"error": f"Herramienta desconocida: {tool_name}"}


def chat(cliente_id: int, cliente_email: str, messages: list[dict]) -> str:
    limited = messages[-MAX_HISTORY_MESSAGES:] if len(messages) > MAX_HISTORY_MESSAGES else messages
    client = anthropic.Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))

    response = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=1024,
        system=[{"type": "text", "text": SYSTEM_PROMPT,
                 "cache_control": {"type": "ephemeral"}}],
        messages=limited,
        tools=TOOLS
    )

    while response.stop_reason == "tool_use":
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = dispatch_tool(block.name, block.input, cliente_id, cliente_email)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": json.dumps(result, ensure_ascii=False, default=str)
                })

        limited = limited + [
            {"role": "assistant", "content": response.content},
            {"role": "user", "content": tool_results}
        ]

        response = client.messages.create(
            model="claude-haiku-4-5-20251001",
            max_tokens=1024,
            system=[{"type": "text", "text": SYSTEM_PROMPT,
                     "cache_control": {"type": "ephemeral"}}],
            messages=limited,
            tools=TOOLS
        )

    for block in response.content:
        if hasattr(block, "text"):
            return block.text
    return "No pude generar una respuesta."
```

- [ ] **Step 4: Ejecutar tests y verificar que pasan**

```bash
pytest tests/test_chatbot_tools.py -v
```

Esperado: todos PASS.

- [ ] **Step 5: Ejecutar toda la suite de tests**

```bash
pytest tests/ -v
```

Esperado: todos los tests PASS.

- [ ] **Step 6: Commit**

```bash
git add backend/chatbot.py tests/test_chatbot_tools.py
git commit -m "feat: chatbot with Claude tool use and history management"
```

---

## Task 7: API FastAPI (main.py)

**Files:**
- Create: `backend/main.py`

- [ ] **Step 1: Implementar backend/main.py**

No hay tests de integración para la API en este paso (se prueban manualmente en el siguiente task). Crear `backend/main.py`:

```python
import os
import shutil
from pathlib import Path

from dotenv import load_dotenv
from fastapi import Depends, FastAPI, File, Form, HTTPException, UploadFile
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
from pydantic import BaseModel

import jwt as pyjwt

load_dotenv()

from backend.auth import create_token, decode_token
from backend.chatbot import chat
from backend.db import (
    create_cliente,
    create_coberturas,
    create_poliza,
    get_cliente_by_id,
    get_cliente_by_nombre,
    init_db,
    list_clientes,
    normalize_folder_name,
    update_cliente_email,
)
from backend.extractor import extract_from_pdf

STORAGE_PATH = Path(__file__).parent / "storage"
STORAGE_PATH.mkdir(exist_ok=True)

app = FastAPI(title="Pólizas de Seguros API")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)


@app.on_event("startup")
def startup():
    init_db()


# ── Auth ──────────────────────────────────────────────────────────────────────

bearer = HTTPBearer(auto_error=False)


def get_current_cliente(
    credentials: HTTPAuthorizationCredentials = Depends(bearer),
) -> dict:
    if not credentials:
        raise HTTPException(status_code=401, detail="Token requerido")
    try:
        return decode_token(credentials.credentials)
    except pyjwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expirado")
    except pyjwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Token inválido")


def verify_admin(credentials: HTTPAuthorizationCredentials = Depends(bearer)) -> None:
    admin_password = os.getenv("ADMIN_PASSWORD", "")
    if not credentials or credentials.credentials != admin_password:
        raise HTTPException(status_code=403, detail="Acceso de administrador requerido")


class LoginRequest(BaseModel):
    nombre: str


@app.post("/auth/login")
def login(req: LoginRequest):
    cliente = get_cliente_by_nombre(req.nombre.strip())
    if not cliente:
        raise HTTPException(status_code=404, detail="Cliente no encontrado")
    token = create_token(cliente_id=cliente["id"], nombre=cliente["nombre"])
    return {"token": token, "cliente": {
        "id": cliente["id"],
        "nombre": cliente["nombre"],
        "email": cliente["email"],
    }}


# ── Chat ──────────────────────────────────────────────────────────────────────

class ChatRequest(BaseModel):
    messages: list[dict]


@app.post("/chat")
def chat_endpoint(req: ChatRequest, current: dict = Depends(get_current_cliente)):
    cliente_id = int(current["sub"])
    cliente = get_cliente_by_id(cliente_id)
    if not cliente:
        raise HTTPException(status_code=404, detail="Cliente no encontrado")
    reply = chat(
        cliente_id=cliente_id,
        cliente_email=cliente["email"] or "",
        messages=req.messages,
    )
    return {"reply": reply}


# ── Admin ─────────────────────────────────────────────────────────────────────

class CreateClienteRequest(BaseModel):
    nombre: str
    email: str | None = None
    codigo_cliente: str | None = None
    rfc: str | None = None
    domicilio: str | None = None


class UpdateEmailRequest(BaseModel):
    email: str


@app.get("/admin/clientes", dependencies=[Depends(verify_admin)])
def get_clientes():
    return list_clientes()


@app.post("/admin/clientes", dependencies=[Depends(verify_admin)])
def post_cliente(req: CreateClienteRequest):
    existing = get_cliente_by_nombre(req.nombre)
    if existing:
        raise HTTPException(status_code=409, detail="Ya existe un cliente con ese nombre")
    cliente_id = create_cliente(req.nombre, req.email, req.codigo_cliente, req.rfc, req.domicilio)
    return {"id": cliente_id, "nombre": req.nombre}


@app.patch("/admin/clientes/{cliente_id}/email", dependencies=[Depends(verify_admin)])
def patch_email(cliente_id: int, req: UpdateEmailRequest):
    update_cliente_email(cliente_id, req.email)
    return {"ok": True}


@app.post("/admin/upload-pdf", dependencies=[Depends(verify_admin)])
async def upload_pdf(file: UploadFile = File(...)):
    if not file.filename.lower().endswith(".pdf"):
        raise HTTPException(status_code=400, detail="Solo se aceptan archivos PDF")

    tmp_path = STORAGE_PATH / f"_tmp_{file.filename}"
    try:
        with open(tmp_path, "wb") as f:
            f.write(await file.read())

        extracted = extract_from_pdf(str(tmp_path))

        # Find or create client from extracted data
        contratante_nombre = extracted.get("contratante")
        if not contratante_nombre:
            raise HTTPException(status_code=422, detail="No se pudo extraer el nombre del contratante")

        cliente = get_cliente_by_nombre(contratante_nombre)
        if not cliente:
            cliente_id = create_cliente(
                nombre=contratante_nombre,
                email=None,
                codigo_cliente=extracted.get("codigo_cliente"),
                rfc=extracted.get("rfc_contratante"),
                domicilio=extracted.get("domicilio_contratante"),
            )
            cliente = get_cliente_by_id(cliente_id)
        else:
            cliente_id = cliente["id"]

        # Move PDF to client folder
        carpeta = STORAGE_PATH / cliente["carpeta"]
        carpeta.mkdir(exist_ok=True)
        numero = extracted.get("numero_poliza", file.filename.replace(".pdf", ""))
        dest_name = f"{numero}.pdf".replace("/", "-").replace("\\", "-")
        dest_path = carpeta / dest_name
        shutil.move(str(tmp_path), str(dest_path))

        extracted["archivo_pdf"] = str(dest_path)
        coberturas = extracted.pop("coberturas", []) or []

        poliza_id = create_poliza(cliente_id, extracted)
        if coberturas:
            create_coberturas(poliza_id, coberturas)

        null_fields = [k for k, v in extracted.items() if v is None and k != "archivo_pdf"]

        return {
            "poliza_id": poliza_id,
            "cliente_id": cliente_id,
            "cliente_nombre": cliente["nombre"],
            "numero_poliza": extracted.get("numero_poliza"),
            "campos_nulos": null_fields,
            "coberturas_extraidas": len(coberturas),
        }

    except HTTPException:
        if tmp_path.exists():
            tmp_path.unlink()
        raise
    except Exception as e:
        if tmp_path.exists():
            tmp_path.unlink()
        raise HTTPException(status_code=500, detail=f"Error al procesar el PDF: {str(e)}")
```

- [ ] **Step 2: Iniciar el servidor**

```bash
uvicorn backend.main:app --reload --port 8000
```

Esperado: servidor corriendo en http://localhost:8000. Abrir http://localhost:8000/docs para ver la documentación interactiva.

- [ ] **Step 3: Commit**

```bash
git add backend/main.py
git commit -m "feat: FastAPI endpoints for auth, chat, and admin"
```

---

## Task 8: Frontend — Página de Login

**Files:**
- Create: `frontend/index.html`

- [ ] **Step 1: Crear frontend/index.html**

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Mis Pólizas — Iniciar sesión</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: system-ui, sans-serif;
      background: #f0f4f8;
      display: flex;
      align-items: center;
      justify-content: center;
      min-height: 100vh;
    }
    .card {
      background: #fff;
      border-radius: 12px;
      padding: 2.5rem;
      box-shadow: 0 4px 24px rgba(0,0,0,.08);
      width: 100%;
      max-width: 400px;
    }
    h1 { font-size: 1.5rem; color: #1a202c; margin-bottom: .5rem; }
    p { color: #718096; font-size: .9rem; margin-bottom: 1.5rem; }
    label { display: block; font-size: .875rem; font-weight: 600;
            color: #4a5568; margin-bottom: .4rem; }
    input {
      width: 100%;
      padding: .65rem .9rem;
      border: 1px solid #cbd5e0;
      border-radius: 8px;
      font-size: 1rem;
      margin-bottom: 1rem;
      outline: none;
      transition: border-color .2s;
    }
    input:focus { border-color: #4299e1; }
    button {
      width: 100%;
      padding: .75rem;
      background: #4299e1;
      color: #fff;
      border: none;
      border-radius: 8px;
      font-size: 1rem;
      font-weight: 600;
      cursor: pointer;
      transition: background .2s;
    }
    button:hover { background: #3182ce; }
    button:disabled { background: #a0aec0; cursor: not-allowed; }
    .error { color: #e53e3e; font-size: .875rem; margin-top: .5rem; display: none; }
  </style>
</head>
<body>
  <div class="card">
    <h1>Mis Pólizas</h1>
    <p>Ingresa tu nombre completo para consultar tus pólizas de seguro.</p>
    <form id="loginForm">
      <label for="nombre">Nombre completo</label>
      <input id="nombre" type="text" placeholder="Ej: Juan García Pérez"
             autocomplete="name" required />
      <button type="submit" id="btn">Entrar</button>
      <p class="error" id="error"></p>
    </form>
  </div>

  <script>
    const API = 'http://localhost:8000';

    document.getElementById('loginForm').addEventListener('submit', async (e) => {
      e.preventDefault();
      const nombre = document.getElementById('nombre').value.trim();
      const btn = document.getElementById('btn');
      const errorEl = document.getElementById('error');
      errorEl.style.display = 'none';
      btn.disabled = true;
      btn.textContent = 'Buscando...';

      try {
        const res = await fetch(`${API}/auth/login`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ nombre })
        });
        if (!res.ok) {
          const data = await res.json();
          throw new Error(data.detail || 'No se encontró el cliente');
        }
        const data = await res.json();
        sessionStorage.setItem('token', data.token);
        sessionStorage.setItem('cliente', JSON.stringify(data.cliente));
        window.location.href = 'chat.html';
      } catch (err) {
        errorEl.textContent = err.message;
        errorEl.style.display = 'block';
        btn.disabled = false;
        btn.textContent = 'Entrar';
      }
    });
  </script>
</body>
</html>
```

- [ ] **Step 2: Verificar manualmente**

Con el servidor corriendo (`uvicorn backend.main:app --reload`), abrir `frontend/index.html` directamente en el navegador (doble clic en el archivo o `file://...`).

- Verificar que el formulario se muestra correctamente
- Intentar con un nombre que no existe → debe mostrar error "No se encontró el cliente"
- Crear un cliente de prueba via la API docs (http://localhost:8000/docs → POST /admin/clientes, usar `ADMIN_PASSWORD` como Bearer token)
- Intentar el login con ese nombre → debe redirigir a `chat.html` (la página no existe aún, está bien por ahora)

- [ ] **Step 3: Commit**

```bash
git add frontend/index.html
git commit -m "feat: login page"
```

---

## Task 9: Frontend — Interfaz del Chatbot

**Files:**
- Create: `frontend/chat.html`

- [ ] **Step 1: Crear frontend/chat.html**

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Mis Pólizas — Chat</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: system-ui, sans-serif; background: #f0f4f8;
           display: flex; flex-direction: column; height: 100vh; }
    header {
      background: #fff;
      border-bottom: 1px solid #e2e8f0;
      padding: .75rem 1.5rem;
      display: flex;
      align-items: center;
      justify-content: space-between;
    }
    header h1 { font-size: 1.1rem; color: #1a202c; }
    header span { font-size: .85rem; color: #718096; }
    #logout { background: none; border: 1px solid #cbd5e0; border-radius: 6px;
              padding: .3rem .7rem; cursor: pointer; font-size: .8rem; color: #718096; }
    #messages {
      flex: 1;
      overflow-y: auto;
      padding: 1.5rem;
      display: flex;
      flex-direction: column;
      gap: .75rem;
    }
    .msg {
      max-width: 75%;
      padding: .75rem 1rem;
      border-radius: 12px;
      font-size: .9rem;
      line-height: 1.5;
      white-space: pre-wrap;
    }
    .msg.user { background: #4299e1; color: #fff; align-self: flex-end;
                border-bottom-right-radius: 3px; }
    .msg.bot  { background: #fff; color: #1a202c; align-self: flex-start;
                border-bottom-left-radius: 3px;
                box-shadow: 0 1px 4px rgba(0,0,0,.07); }
    .msg.bot.loading { color: #a0aec0; font-style: italic; }
    footer {
      background: #fff;
      border-top: 1px solid #e2e8f0;
      padding: 1rem 1.5rem;
      display: flex;
      gap: .75rem;
    }
    footer input {
      flex: 1;
      padding: .65rem 1rem;
      border: 1px solid #cbd5e0;
      border-radius: 8px;
      font-size: .95rem;
      outline: none;
    }
    footer input:focus { border-color: #4299e1; }
    footer button {
      background: #4299e1;
      color: #fff;
      border: none;
      border-radius: 8px;
      padding: .65rem 1.2rem;
      font-size: .95rem;
      font-weight: 600;
      cursor: pointer;
    }
    footer button:disabled { background: #a0aec0; cursor: not-allowed; }
  </style>
</head>
<body>
  <header>
    <h1>Mis Pólizas</h1>
    <div style="display:flex;align-items:center;gap:1rem">
      <span id="nombreCliente"></span>
      <button id="logout">Salir</button>
    </div>
  </header>

  <div id="messages"></div>

  <footer>
    <input id="input" type="text"
           placeholder="Pregunta sobre tus pólizas..." autocomplete="off" />
    <button id="send">Enviar</button>
  </footer>

  <script>
    const API = 'http://localhost:8000';
    const token = sessionStorage.getItem('token');
    const cliente = JSON.parse(sessionStorage.getItem('cliente') || 'null');

    if (!token || !cliente) {
      window.location.href = 'index.html';
    }

    document.getElementById('nombreCliente').textContent = cliente.nombre;
    document.getElementById('logout').addEventListener('click', () => {
      sessionStorage.clear();
      window.location.href = 'index.html';
    });

    const messages = [];
    const messagesEl = document.getElementById('messages');
    const inputEl = document.getElementById('input');
    const sendBtn = document.getElementById('send');

    function addMsg(role, text, loading = false) {
      const div = document.createElement('div');
      div.className = `msg ${role === 'user' ? 'user' : 'bot'}${loading ? ' loading' : ''}`;
      div.textContent = text;
      messagesEl.appendChild(div);
      messagesEl.scrollTop = messagesEl.scrollHeight;
      return div;
    }

    // Welcome message
    addMsg('bot', `Hola, ${cliente.nombre}. ¿En qué te puedo ayudar con tus pólizas?`);

    async function sendMessage() {
      const text = inputEl.value.trim();
      if (!text) return;
      inputEl.value = '';
      sendBtn.disabled = true;

      addMsg('user', text);
      messages.push({ role: 'user', content: text });

      const loadingEl = addMsg('bot', 'Consultando...', true);

      try {
        const res = await fetch(`${API}/chat`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${token}`
          },
          body: JSON.stringify({ messages })
        });

        if (res.status === 401) {
          sessionStorage.clear();
          window.location.href = 'index.html';
          return;
        }

        if (!res.ok) throw new Error('Error al conectar con el servidor');

        const data = await res.json();
        loadingEl.textContent = data.reply;
        loadingEl.classList.remove('loading');
        messages.push({ role: 'assistant', content: data.reply });
      } catch (err) {
        loadingEl.textContent = 'Ocurrió un error. Intenta de nuevo.';
        loadingEl.classList.remove('loading');
        messages.pop();
      } finally {
        sendBtn.disabled = false;
        inputEl.focus();
      }
    }

    sendBtn.addEventListener('click', sendMessage);
    inputEl.addEventListener('keydown', (e) => {
      if (e.key === 'Enter' && !e.shiftKey) sendMessage();
    });
  </script>
</body>
</html>
```

- [ ] **Step 2: Verificar manualmente el flujo completo**

1. Crear un cliente con email via `/admin/clientes` (usar ADMIN_PASSWORD como Bearer)
2. Crear una póliza para ese cliente via `/admin/upload-pdf` subiendo un PDF real
3. Hacer login desde `index.html`
4. En `chat.html`, preguntar: "¿Qué pólizas tengo?"
5. Verificar que el chatbot responde con los datos correctos sin inventar información
6. Preguntar: "Mándame esa póliza por correo"
7. Verificar que llega el correo

- [ ] **Step 3: Commit**

```bash
git add frontend/chat.html
git commit -m "feat: chat interface with JWT auth"
```

---

## Task 10: Frontend — Panel de Administración

**Files:**
- Create: `frontend/admin.html`

- [ ] **Step 1: Crear frontend/admin.html**

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Admin — Pólizas de Seguros</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: system-ui, sans-serif; background: #f0f4f8; }
    header { background: #1a202c; color: #fff; padding: 1rem 2rem; }
    header h1 { font-size: 1.1rem; }
    main { max-width: 800px; margin: 2rem auto; padding: 0 1rem; }
    .card { background: #fff; border-radius: 12px; padding: 1.5rem;
            box-shadow: 0 2px 12px rgba(0,0,0,.07); margin-bottom: 1.5rem; }
    h2 { font-size: 1rem; color: #2d3748; margin-bottom: 1rem; }
    label { display: block; font-size: .85rem; font-weight: 600;
            color: #4a5568; margin-bottom: .3rem; }
    input, select {
      width: 100%; padding: .6rem .9rem; border: 1px solid #cbd5e0;
      border-radius: 8px; font-size: .9rem; margin-bottom: .9rem; outline: none;
    }
    input:focus { border-color: #4299e1; }
    button {
      padding: .6rem 1.2rem; border: none; border-radius: 8px;
      font-size: .9rem; font-weight: 600; cursor: pointer;
    }
    .btn-primary { background: #4299e1; color: #fff; }
    .btn-primary:hover { background: #3182ce; }
    .btn-primary:disabled { background: #a0aec0; cursor: not-allowed; }
    .result {
      margin-top: 1rem; padding: 1rem; border-radius: 8px;
      font-size: .875rem; display: none;
    }
    .result.ok  { background: #f0fff4; border: 1px solid #9ae6b4; color: #276749; }
    .result.err { background: #fff5f5; border: 1px solid #feb2b2; color: #c53030; }
    table { width: 100%; border-collapse: collapse; font-size: .875rem; }
    th { text-align: left; padding: .5rem .75rem; background: #edf2f7;
         color: #4a5568; font-weight: 600; }
    td { padding: .5rem .75rem; border-bottom: 1px solid #e2e8f0; color: #2d3748; }
    .login-screen { max-width: 380px; margin: 4rem auto; }
    .null-field { color: #e53e3e; font-size: .8rem; }
  </style>
</head>
<body>

  <!-- Login -->
  <div id="loginScreen" class="login-screen">
    <div class="card">
      <h2>Acceso de Administrador</h2>
      <label>Contraseña</label>
      <input id="adminPass" type="password" placeholder="Contraseña de administrador" />
      <button class="btn-primary" onclick="doAdminLogin()">Entrar</button>
      <div id="loginErr" class="result err" style="margin-top:.5rem"></div>
    </div>
  </div>

  <!-- Panel principal -->
  <div id="adminPanel" style="display:none">
    <header><h1>Panel de Administración — Pólizas</h1></header>
    <main>

      <!-- Subir PDF -->
      <div class="card">
        <h2>Subir póliza (PDF)</h2>
        <p style="font-size:.85rem;color:#718096;margin-bottom:1rem">
          El sistema extraerá los datos automáticamente. Si el cliente no existe, se creará.
        </p>
        <label>Archivo PDF</label>
        <input type="file" id="pdfFile" accept=".pdf" />
        <button class="btn-primary" id="uploadBtn" onclick="uploadPdf()">Subir y extraer</button>
        <div id="uploadResult" class="result"></div>
      </div>

      <!-- Crear cliente manualmente -->
      <div class="card">
        <h2>Agregar cliente manualmente</h2>
        <label>Nombre completo *</label>
        <input id="cNombre" placeholder="Ej: Juan García Pérez" />
        <label>Correo electrónico</label>
        <input id="cEmail" type="email" placeholder="cliente@correo.com" />
        <label>Código de cliente</label>
        <input id="cCodigo" placeholder="C001" />
        <label>RFC</label>
        <input id="cRfc" placeholder="GAJU800101ABC" />
        <label>Domicilio</label>
        <input id="cDomicilio" placeholder="Calle 1 #2, CDMX" />
        <button class="btn-primary" onclick="createCliente()">Crear cliente</button>
        <div id="clienteResult" class="result"></div>
      </div>

      <!-- Lista de clientes -->
      <div class="card">
        <h2>Clientes registrados</h2>
        <button class="btn-primary" onclick="loadClientes()" style="margin-bottom:1rem">
          Actualizar lista
        </button>
        <div id="clientesList"></div>
      </div>

    </main>
  </div>

  <script>
    const API = 'http://localhost:8000';
    let adminToken = '';

    async function doAdminLogin() {
      const pass = document.getElementById('adminPass').value;
      const errEl = document.getElementById('loginErr');
      // Verify by calling a protected endpoint
      const res = await fetch(`${API}/admin/clientes`, {
        headers: { 'Authorization': `Bearer ${pass}` }
      });
      if (res.ok) {
        adminToken = pass;
        document.getElementById('loginScreen').style.display = 'none';
        document.getElementById('adminPanel').style.display = 'block';
        loadClientes();
      } else {
        errEl.textContent = 'Contraseña incorrecta';
        errEl.style.display = 'block';
      }
    }

    function showResult(elId, msg, ok) {
      const el = document.getElementById(elId);
      el.textContent = msg;
      el.className = `result ${ok ? 'ok' : 'err'}`;
      el.style.display = 'block';
    }

    async function uploadPdf() {
      const file = document.getElementById('pdfFile').files[0];
      if (!file) { showResult('uploadResult', 'Selecciona un archivo PDF', false); return; }
      const btn = document.getElementById('uploadBtn');
      btn.disabled = true; btn.textContent = 'Procesando...';

      const form = new FormData();
      form.append('file', file);

      try {
        const res = await fetch(`${API}/admin/upload-pdf`, {
          method: 'POST',
          headers: { 'Authorization': `Bearer ${adminToken}` },
          body: form
        });
        const data = await res.json();
        if (!res.ok) { showResult('uploadResult', data.detail, false); return; }

        let msg = `✓ Póliza "${data.numero_poliza}" cargada para ${data.cliente_nombre}. `;
        msg += `${data.coberturas_extraidas} cobertura(s) extraída(s). `;
        if (data.campos_nulos.length > 0) {
          msg += `Campos no encontrados: ${data.campos_nulos.join(', ')}.`;
        }
        showResult('uploadResult', msg, data.campos_nulos.length === 0);
        loadClientes();
      } catch {
        showResult('uploadResult', 'Error de conexión', false);
      } finally {
        btn.disabled = false; btn.textContent = 'Subir y extraer';
      }
    }

    async function createCliente() {
      const body = {
        nombre: document.getElementById('cNombre').value.trim(),
        email:  document.getElementById('cEmail').value.trim() || null,
        codigo_cliente: document.getElementById('cCodigo').value.trim() || null,
        rfc:    document.getElementById('cRfc').value.trim() || null,
        domicilio: document.getElementById('cDomicilio').value.trim() || null,
      };
      if (!body.nombre) { showResult('clienteResult', 'El nombre es requerido', false); return; }

      const res = await fetch(`${API}/admin/clientes`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json',
                   'Authorization': `Bearer ${adminToken}` },
        body: JSON.stringify(body)
      });
      const data = await res.json();
      if (res.ok) {
        showResult('clienteResult', `✓ Cliente "${data.nombre}" creado (ID: ${data.id})`, true);
        loadClientes();
      } else {
        showResult('clienteResult', data.detail, false);
      }
    }

    async function loadClientes() {
      const res = await fetch(`${API}/admin/clientes`, {
        headers: { 'Authorization': `Bearer ${adminToken}` }
      });
      const clientes = await res.json();
      const el = document.getElementById('clientesList');
      if (clientes.length === 0) {
        el.innerHTML = '<p style="color:#718096;font-size:.875rem">Sin clientes registrados.</p>';
        return;
      }
      el.innerHTML = `
        <table>
          <thead><tr>
            <th>Nombre</th><th>Email</th><th>RFC</th><th>Carpeta</th>
          </tr></thead>
          <tbody>
            ${clientes.map(c => `
              <tr>
                <td>${c.nombre}</td>
                <td>${c.email || '<span class="null-field">sin email</span>'}</td>
                <td>${c.rfc || '—'}</td>
                <td><code>${c.carpeta}</code></td>
              </tr>`).join('')}
          </tbody>
        </table>`;
    }
  </script>
</body>
</html>
```

- [ ] **Step 2: Verificar manualmente**

1. Abrir `frontend/admin.html` en el navegador
2. Ingresar la contraseña del admin (valor de `ADMIN_PASSWORD` en `.env`)
3. Crear un cliente manualmente con nombre y email
4. Subir un PDF de póliza real → verificar que aparecen los datos extraídos y los campos nulos
5. Verificar que la lista de clientes se actualiza

- [ ] **Step 3: Commit**

```bash
git add frontend/admin.html
git commit -m "feat: admin panel for uploading PDFs and managing clients"
```

---

## Task 11: Prueba de integración completa

- [ ] **Step 1: Ejecutar toda la suite de tests**

```bash
pytest tests/ -v
```

Esperado: todos los tests PASS.

- [ ] **Step 2: Arrancar el servidor**

```bash
uvicorn backend.main:app --reload --port 8000
```

- [ ] **Step 3: Flujo completo de prueba**

Ejecutar este flujo end-to-end:

1. Abrir `frontend/admin.html` → iniciar sesión con ADMIN_PASSWORD
2. Crear cliente: nombre="María Ejemplo", email="tu_correo@gmail.com"
3. Subir un PDF de póliza (cualquier póliza de seguro en PDF)
4. Verificar en la respuesta que se extrajo el número de póliza y la aseguradora
5. Abrir `frontend/index.html` → ingresar "María Ejemplo"
6. En `chat.html` preguntar: "¿Qué pólizas tengo?"
7. Verificar que el chatbot responde con la información correcta
8. Preguntar: "¿Cuánto pago de prima neta?"
9. Preguntar: "Mándame mi póliza por correo"
10. Verificar que llega el correo con el PDF adjunto

- [ ] **Step 4: Commit final**

```bash
git add .
git commit -m "feat: complete insurance policy management system with chatbot"
```

---

## Notas para producción (futuras)

- **SQLite → PostgreSQL:** cambiar `sqlite3` por `psycopg2` en `db.py`, las queries SQL no cambian.
- **storage/ → S3/GCS:** reemplazar operaciones de `Path` en `main.py` por llamadas al SDK del proveedor. `email_service.py` descarga el archivo antes de adjuntarlo.
- **Autenticación:** agregar contraseña o verificación por email al endpoint `/auth/login`.
- **CORS:** cambiar `allow_origins=["*"]` por el dominio real en producción.
