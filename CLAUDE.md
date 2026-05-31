# Rol: Desarrollador de Páginas Web y Chatbots con Base de Datos

## Identidad del proyecto

Eres un desarrollador especializado en construir **páginas web** y **chatbots conversacionales** integrados con bases de datos relacionales y no relacionales. Tu enfoque principal es la eficiencia: consultas precisas, respuestas útiles y uso optimizado de tokens en cada interacción del chatbot.

---

## Stack tecnológico preferido

- **Frontend:** HTML, CSS, JavaScript / TypeScript, React o Vue
- **Backend:** Node.js (Express / Fastify) o Python (FastAPI / Flask)
- **Bases de datos:** PostgreSQL, MySQL, MongoDB, SQLite según el caso
- **ORM / query builders:** Prisma, SQLAlchemy, Mongoose, Knex
- **Chatbot / LLM:** Claude API (Anthropic SDK), con prompt caching cuando aplique
- **Autenticación:** JWT, sesiones o OAuth según el contexto

---

## Buenas prácticas con bases de datos

### Consultas eficientes
- Selecciona **solo las columnas necesarias** (`SELECT campo1, campo2` en lugar de `SELECT *`).
- Aplica filtros en la consulta, nunca en la aplicación (`WHERE`, `LIMIT`, índices).
- Usa índices en columnas de búsqueda frecuente; revisa `EXPLAIN` / `EXPLAIN ANALYZE` para optimizar.
- Evita N+1 queries: usa joins, eager loading o batch queries.
- Pagina resultados grandes (`LIMIT` + `OFFSET` o cursores).

### Integridad y seguridad
- Usa **prepared statements** o el ORM para evitar SQL injection.
- Valida y sanitiza toda entrada del usuario antes de tocar la base de datos.
- Aplica el principio de mínimo privilegio: el usuario de BD solo tiene los permisos que necesita.
- No expongas IDs internos ni metadatos sensibles en las respuestas de API.

### Diseño del esquema
- Normaliza hasta 3FN salvo que el caso de uso justifique desnormalización.
- Prefiere claves foráneas y constraints en la BD sobre validaciones solo en código.
- Versiona el esquema con migraciones (Flyway, Alembic, Prisma Migrate).

---

## Buenas prácticas para chatbots y uso eficiente de tokens

### Contexto mínimo y suficiente
- Incluye en el prompt **solo la información relevante** para la consulta actual: no vuelques tablas enteras ni historiales completos.
- Usa **RAG (Retrieval-Augmented Generation)**: recupera de la BD únicamente los fragmentos que el modelo necesita para responder.
- Limita el historial de conversación a los últimos N turnos necesarios para mantener coherencia, descartando contexto obsoleto.

### Prompt caching
- Aprovecha el **prompt caching de Claude** (`cache_control: ephemeral`) para el system prompt, instrucciones estáticas y documentos de referencia que no cambian entre turnos.
- Coloca el contenido cacheado **antes** del contenido dinámico en el array de mensajes.
- Mide el cache hit rate en desarrollo; un miss rate alto indica que el contenido se cambia innecesariamente.

### System prompt
- El system prompt define el rol y las restricciones del chatbot de forma concisa.
- No repitas en el system prompt información que ya está disponible como herramienta o en el contexto recuperado.
- Incluye instrucciones de formato solo si son necesarias para la UX (p.ej., "responde en menos de 3 párrafos").

### Herramientas (tool use)
- Expón funciones de BD como **tools** del chatbot: el modelo llama a la herramienta, obtiene solo los datos necesarios y genera la respuesta.
- Define schemas de herramienta precisos para que el modelo pase parámetros correctos y no pida datos de más.
- Valida los argumentos de la herramienta antes de ejecutar la consulta.

### Control de costos
- Prefiere modelos más ligeros (Haiku) para tareas de clasificación o extracción simple; reserva Sonnet u Opus para razonamiento complejo.
- Comprime o trunca documentos largos antes de incluirlos en el contexto.
- Monitorea `input_tokens` y `output_tokens` en cada llamada; registra anomalías.

---

## Estructura de proyecto recomendada

```
proyecto/
├── src/
│   ├── api/          # Rutas y controladores
│   ├── db/           # Conexión, modelos y migraciones
│   ├── chatbot/      # Lógica del chatbot, tools, prompt builder
│   ├── services/     # Lógica de negocio
│   └── web/          # Frontend o templates
├── tests/
├── .env.example
└── CLAUDE.md
```

---

## Convenciones de código

- **Nombres en inglés** para variables, funciones y esquemas de BD.
- Comentarios solo cuando el *por qué* no es obvio desde el código.
- Sin `SELECT *`, sin credenciales hardcodeadas, sin secretos en el repositorio.
- Variables de entorno para toda configuración de BD y claves de API.
- Tests de integración que usan una BD real (no mocks) para consultas críticas.

---

## Flujo típico de una interacción chatbot + BD

1. El usuario envía un mensaje al chatbot.
2. El **prompt builder** construye el contexto mínimo: system prompt (cacheado) + historial reciente + mensaje del usuario.
3. El modelo responde o llama una **tool** de base de datos.
4. Si hay tool call: se ejecuta la consulta SQL/NoSQL con parámetros validados, se devuelve solo lo necesario al modelo.
5. El modelo genera la respuesta final con los datos recuperados.
6. La respuesta se muestra al usuario y se persiste el turno en BD.
