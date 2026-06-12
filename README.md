# 🚀 Dashboard Inteligente de Ingeniería: Orquestación con Model Context Protocol (MCP)

**Proyecto extraclase para la asignatura de Ingeniería de Software II** **Escuela Tecnológica Instituto Técnico Central (ETITC)** **Autores:** Andrés Felipe Gerena Contreras  - Fabián Estevan Suarez - Laura Camila Mosquera

---

## 1. Descripción del Proyecto
Este proyecto demuestra la implementación y orquestación de una arquitectura multi-servidor basada en el **Model Context Protocol (MCP)**. El objetivo principal es resolver el problema de aislamiento de los Modelos de Lenguaje Grande (LLMs), permitiéndoles interactuar de forma estandarizada, segura y local con sistemas empresariales externos (control de versiones y bases de datos relacionales) sin necesidad de desarrollar APIs de integración ad-hoc.

Para este entregable se adoptó la **Opción B (Orquestación)**, configurando Antigravity IDE como cliente/host y conectándolo simultáneamente a repositorios en GitHub y una base de datos PostgreSQL alojada en Supabase.

---

## 2. Conceptos Clave y Arquitectura
El ecosistema se fundamenta en los siguientes conceptos de ingeniería:

* **Model Context Protocol (MCP):** Estándar abierto que define un protocolo de comunicación bidireccional mediante JSON-RPC, unificando cómo la IA accede a herramientas (Tools) y recursos.
* **Host / Cliente (Antigravity IDE):** El entorno de desarrollo que actúa como "cerebro". Inyecta las herramientas disponibles al LLM y maneja la comunicación con los servidores locales.
* **Comunicación vía `stdio`:** Los servidores MCP se ejecutan como procesos hijos en memoria local (usando `npx`). La comunicación entre el IDE y los servidores no viaja por internet, sino a través de entradas y salidas estándar (`stdio`), garantizando cero latencia y máxima seguridad de los tokens.
* **Safety by Design (Seguridad desde el Diseño):** El servidor MCP de PostgreSQL está restringido exclusivamente a consultas de lectura (`SELECT`). Esta decisión arquitectónica evita mutaciones accidentales o maliciosas en la base de datos por parte de un agente autónomo.
* **Human-in-the-Loop (Humanos en el Bucle - HitL):** Patrón de diseño implementado para operaciones críticas (como `INSERT` o `UPDATE`), donde la IA extrae, transforma y prepara los scripts SQL, pero requiere la validación y ejecución manual del ingeniero para persistir los datos de forma segura.

---

## 3. Diagrama de la Arquitectura

```text
[ LLM / Agente IA ] <--> [ Antigravity IDE (Host) ]
                                |
                                | (JSON-RPC sobre stdio)
                                |
        +-----------------------+-----------------------+
        |                                               |
[ @modelcontextprotocol/server-github ]   [ @modelcontextprotocol/server-postgres ]
        |                                               |
        | (API REST / HTTPS)                            | (Conexión TCP / URI)
        v                                               v
  [ GitHub ]                                     [ Supabase (PostgreSQL) ]
(Repositorios e Issues)                         (Tabla: github_issues_tracker)
```


##  4. Requisitos y Preparación del Entorno

Para replicar esta arquitectura de forma local, se requiere:

- Node.js instalado (para ejecutar comandos `npx`)
- Antigravity IDE o cualquier cliente compatible con MCP (Claude Desktop, Cursor)
- Token de Acceso Personal (PAT) de GitHub con alcance `repo`
- Base de Datos PostgreSQL (en la nube con Supabase)

### 📄 Estructura de la tabla en PostgreSQL

```sql
CREATE TABLE github_issues_tracker (
    id SERIAL PRIMARY KEY,
    issue_number INT NOT NULL,
    titulo VARCHAR(255) NOT NULL,
    descripcion_corta TEXT,
    prioridad VARCHAR(50),
    estado VARCHAR(50)
);
```



---

## 5. Configuración del Cliente

* El ecosistema se orquesta centralizando las configuraciones en el archivo mcp_config.json del cliente (ubicado habitualmente en %USERPROFILE%\.gemini\config\mcp_config.json en Windows o en el panel de configuración de servidores MCP de Antigravity).

### JSON

```
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_TU_TOKEN_DE_GITHUB_AQUI"
      }
    },
    "postgres": {
      "command": "npx",
      "args": [
        "-y", 
        "@modelcontextprotocol/server-postgres", 
        "postgresql://postgres.[tu_id]:[tu_contraseña]@aws-0-[region][.pooler.supabase.com:6543/postgres](https://.pooler.supabase.com:6543/postgres)"
      ]
    }
  }
}
```
---

---

## 6. Flujo de Ejecución y Caso de Uso (MVP)
El escenario de demostración aborda un flujo ETL (Extracción, Transformación y Carga) asistido por IA de extremo a extremo:

### Fase 1: Extracción y Transformación (Análisis de Issues)

Prompt utilizado:

"Actúa como un ingeniero de datos. Tienes dos tareas: 1) Utiliza la herramienta de Postgres para consultar la estructura y columnas exactas de la tabla 'github_issues_tracker' en la base de datos. 2) Utiliza la herramienta de GitHub para leer los últimos issues abiertos en el repositorio 'andres-gerena-itc/demo-mcp-issues'. Finalmente, cruza ambas fuentes de información y genérame el código SQL exacto con los comandos INSERT INTO preparados y listos para que yo los ejecute de forma segura, asignando a cada issue una prioridad basada en su contenido."

Resultado: El agente orquesta peticiones a GitHub, evalúa la gravedad de los problemas con NLP (Procesamiento de Lenguaje Natural) y genera sentencias SQL estructuradas. El intento de inserción directa es rechazado correctamente por las políticas de seguridad nativas del servidor MCP.

### Fase 2: Ejecución (Human-in-the-Loop)

El ingeniero copia el script generado (INSERT INTO...) y lo ejecuta manualmente en la consola SQL de Supabase, garantizando la integridad de los datos de producción y cumpliendo las restricciones de seguridad.


### Fase 3: Auditoría (Reporte Automatizado de Lectura)

Prompt utilizado:

"Actúa como un auditor de bases de datos. Utiliza tu herramienta de Postgres para ejecutar una consulta SQL que extraiga todos los registros actuales de la tabla 'github_issues_tracker'. Una vez que obtengas los datos, preséntamelos en una tabla limpia de Markdown y hazme un breve resumen indicando cuántos issues tenemos en estado crítico (Prioridad ALTA)."

## 7. Conclusiones
El escenario de demostración aborda un flujo ETL (Extracción, Transformación y Carga) asistido por IA de extremo a extremo:

1. Eficiencia en Integración: MCP elimina la necesidad de desarrollar, probar y mantener microservicios o backends intermedios dedicados exclusivamente a comunicar herramientas independientes.
2. Modularidad Estricta: El desacoplamiento es total. Un cambio en el proveedor de base de datos o de repositorios se gestiona modificando una línea del archivo de configuración JSON, sin alterar la lógica de las herramientas ni del agente de IA.
3. Seguridad Robusta: La restricción de solo lectura en el servidor de Postgres valida que el estándar abierto MCP prioriza el control operativo y la protección de infraestructuras críticas por encima de la autonomía ciega de los LLMs.
---
