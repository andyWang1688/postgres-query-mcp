

# sql-query-mcp

[中文版](README-zh.md)

Un servidor MCP de propósito general que permite a la IA trabajar con múltiples bases de datos dentro de límites claros.

[![sql-query-mcp MCP server](https://glama.ai/mcp/servers/andyWang1688/sql-query-mcp/badges/card.svg)](https://glama.ai/mcp/servers/andyWang1688/sql-query-mcp)

## Soporte actual para bases de datos

| Base de datos | Estado | Disponibilidad actual |
| --- | --- | --- |
| PostgreSQL | Soportado | Disponible actualmente |
| MySQL | Soportado | Disponible actualmente |
| Hive | Soportado | Disponible actualmente |
| SQLite | Candidato | No soportado aún |
| SQL Server | Candidato | No soportado aún |
| ClickHouse | Candidato | No soportado aún |

## Valor del producto

`sql-query-mcp` ayuda a los clientes de IA a descubrir esquemas, obtener muestras de datos y analizar consultas de solo lectura a través de una interfaz MCP controlada.

Mantiene el manejo de conexiones, las reglas de espacio de nombres, la validación SQL y el registro de auditoría en el lado del servidor, por lo que puedes exponer un contexto útil de la base de datos a la IA sin exponer cadenas de conexión en texto plano ni simplificar conceptos específicos de cada motor.

## Lo que la IA puede hacer con este servidor

El conjunto actual de herramientas se centra en el descubrimiento de bases de datos, flujos de trabajo de consultas controlados, consultas de solo lectura asíncronas, exportaciones de resultados de consultas en lotes y una vía estrecha de importación de archivos locales. Puedes usarlo para ayudar a un asistente de IA a comprender la estructura antes de generar SQL, ejecutar una consulta acotada, iniciar una consulta de solo lectura de larga duración, exportar resultados de PostgreSQL o MySQL a un archivo local o importar un archivo CSV/XLSX preparado a una tabla existente.

MySQL y Hive son compatibles con `explain_query`. Hive utiliza `EXPLAIN` y `EXPLAIN ANALYZE` para `explain_query`.

| Herramienta | PostgreSQL | MySQL | Hive | Propósito |
| --- | --- | --- | --- | --- |
| `list_connections()` | Sí | Sí | Sí | Listar conexiones configuradas |
| `list_schemas(connection_id)` | Sí | No | No | Listar esquemas visibles de PostgreSQL |
| `list_databases(connection_id)` | No | Sí | Sí | Listar bases de datos visibles de MySQL o Hive |
| `list_tables(connection_id, schema?, database?)` | Sí | Sí | Sí | Listar tablas y vistas |
| `describe_table(connection_id, table_name, schema?, database?)` | Sí | Sí | Sí | Inspeccionar columnas, claves e índices |
| `run_select(connection_id, sql, limit?)` | Sí | Sí | Sí | Ejecutar consultas de solo lectura cortas y acotadas |
| `start_query(connection_id, sql, limit?)` | Sí | Sí | Sí | Iniciar consultas de solo lectura de larga duración |
| `get_query(query_id, offset?, limit?)` | Sí | Sí | Sí | Obtener el estado de la consulta asíncrona y los resultados paginados |
| `cancel_query(query_id)` | Sí | Sí | Sí | Cancelar consultas asíncronas en ejecución |
| `explain_query(connection_id, sql, analyze?)` | Sí | Sí | Sí | Inspeccionar planes de consulta |
| `get_table_sample(connection_id, table_name, schema?, database?, limit?)` | Sí | Sí | Sí | Obtener muestras pequeñas de tablas |
| `export_query_file(connection_id, sql, output_path, format?, limit?, export_all?, file_name?, overwrite?)` | Sí | Sí | No | Exportar resultados de consultas a archivos CSV/XLSX locales |
| `import_table_file(connection_id, table_name, file_path, schema?, database?, sheet_name?)` | Sí | Sí | Sí | Importar archivos CSV/XLSX locales |

Estas herramientas son útiles para tareas como listar espacios de nombres, inspeccionar definiciones de tablas, revisar índices, muestrear registros, ejecutar consultas de solo lectura cortas con `run_select`, ejecutar consultas de solo lectura largas con `start_query`, `get_query` y `cancel_query`, analizar consultas de solo lectura con `EXPLAIN` y exportar resultados de consultas de PostgreSQL o MySQL a archivos CSV/XLSX locales. También puedes importar archivos locales preparados. Para ver los detalles completos de las solicitudes y respuestas, consulta `docs/api-reference.md` (chinos).

## Cómo se establecen los límites

El límite del producto es intencionalmente estrecho hoy en día. PostgreSQL, MySQL y Hive están disponibles actualmente. Las herramientas de consulta permanecen de solo lectura, los resultados de consultas de PostgreSQL y MySQL pueden exportarse a archivos locales y la única vía de escritura en la base de datos es una importación controlada de archivos CSV/XLSX locales a tablas existentes.

El servicio mantiene esos límites explícitos de varias maneras.

- Las conexiones declaran `engine` explícitamente, por lo que el servidor nunca adivina a partir de `connection_id`.
- PostgreSQL utiliza `schema`, mientras que MySQL y Hive utilizan `database`, sin fusionar ambos en un único campo de espacio de nombres ambiguo.
- Los DSN reales permanecen en variables de entorno, mientras que los archivos de configuración almacenan solo los nombres de las variables de entorno.
- La ejecución de consultas pasa por la validación de `sqlglot` antes de llegar a la base de datos. Usa `run_select` para consultas de solo lectura cortas y acotadas, y usa `start_query`, `get_query` y `cancel_query` para consultas de solo lectura de larga duración.
- El servidor solo acepta `SELECT` y `WITH ... SELECT`, rechaza comentarios y entradas de múltiples declaraciones, y registra auditorías para cada llamada.
- `export_query_file` escribe archivos en la máquina del servidor MCP. Es síncrona, pero lee filas de la base de datos y escribe archivos CSV/XLSX en lotes. Las exportaciones grandes aún pueden alcanzar el tiempo de espera de la herramienta de tu cliente MCP. Para la salida XLSX, los valores UUID se escriben como texto y los valores datetime con zona horaria se escriben sin la zona horaria. La exportación de Hive aún no está soportada.
- `import_table_file` no acepta SQL sin procesar. Solo inserta columnas de archivos cuyos encabezados coincidan exactamente con las columnas de tablas existentes.
- `import_table_file` para Hive está destinado solo a archivos pequeños y rechaza archivos con más de 1000 filas de datos. Las importaciones de Hive escriben filas una por una, por lo que pueden ser lentas y alcanzar el tiempo de espera de la herramienta de tu cliente MCP. Para cargas masivas en Hive, usa `LOAD DATA` nativo de Hive, tablas externas o tu canal de ingestión de datos existente.

Para Hive, `explain_query` utiliza `EXPLAIN` y `EXPLAIN ANALYZE`.

## Inicio rápido

`sql-query-mcp` admite dos modos de configuración oficiales basados en PyPI. Ambos están destinados a un uso real, no solo a pruebas locales.

1. Elige cómo quieres que tu cliente MCP inicie el servidor.

Usa el modo de comando instalado si deseas un comando local simple tras una sola instalación.

```bash
pipx install sql-query-mcp
```

Usa el modo de lanzamiento administrado si deseas que la fuente del paquete se declare directamente en la configuración de tu cliente MCP.

```bash
pipx run --spec sql-query-mcp sql-query-mcp
```

Fija una versión con `pipx install 'sql-query-mcp==X.Y.Z'` o `pipx run --spec 'sql-query-mcp==X.Y.Z' sql-query-mcp`. Actualiza el modo de comando instalado con `pipx upgrade sql-query-mcp`.

2. Crea un archivo de configuración.

La configuración del servidor debe residir fuera del repositorio para que el mismo archivo funcione con cualquiera de los modos de inicio.

```bash
mkdir -p ~/.config/sql-query-mcp
```

Luego, guarda el JSON de ejemplo más adelante en esta sección como `~/.config/sql-query-mcp/connections.json`.

3. Registra el servidor en tu cliente MCP.

- Codex: `docs/codex-setup.md` (chinos)
- OpenCode: `docs/opencode-setup.md` (chinos)

El modo de comando instalado significa que tu cliente ejecuta `sql-query-mcp` directamente.
El modo de lanzamiento administrado significa que tu cliente inicia el servidor a través de `pipx run`.

En ambos modos, coloca `SQL_QUERY_MCP_CONFIG` y tus DSN reales de base de datos en el bloque de entorno del cliente MCP en lugar de exportarlos en tu shell.

El punto de entrada de consola es `sql-query-mcp`, que se mapea a `sql_query_mcp.app:main`.

El nombre de instalación de PyPI es `sql-query-mcp`, y la ruta de importación del paquete de Python es `sql_query_mcp`.

Para `pipx install` y `pipx run`, establece `SQL_QUERY_MCP_CONFIG` explícitamente en la ruta de tu archivo de configuración. La ruta predeterminada `config/connections.json` es principalmente para descargas del código fuente y desarrollo local.

La configuración de ejemplo se ve así.

```json
{
  "settings": {
    "default_limit": 200,
    "max_limit": 1000,
    "audit_log_path": "logs/audit.jsonl"
  },
  "connections": [
    {
      "connection_id": "crm_prod_main_ro",
      "engine": "postgres",
      "label": "CRM PostgreSQL production / Main / read-only",
      "env": "prod",
      "tenant": "main",
      "role": "ro",
      "dsn_env": "PG_CONN_CRM_PROD_MAIN_RO",
      "enabled": true,
      "default_schema": "public"
    },
    {
      "connection_id": "crm_mysql_prod_main_ro",
      "engine": "mysql",
      "label": "CRM MySQL production / Main / read-only",
      "env": "prod",
      "tenant": "main",
      "role": "ro",
      "dsn_env": "MYSQL_CONN_CRM_PROD_MAIN_RO",
      "enabled": true,
      "default_database": "crm"
    },
    {
      "connection_id": "warehouse_hive_prod_main_ro",
      "engine": "hive",
      "label": "Warehouse Hive production / Main / read-only",
      "env": "prod",
      "tenant": "main",
      "role": "ro",
      "dsn_env": "HIVE_CONN_WAREHOUSE_PROD_MAIN_RO",
      "enabled": true,
      "default_database": "default"
    }
  ]
}
```

Establece los DSN en el entorno del cliente MCP. Para Hive, usa un DSN de Hive como:

```bash
export HIVE_CONN_WAREHOUSE_PROD_MAIN_RO='hive://user:password@hive.example.com:10000/default?auth=CUSTOM'
```

## Documentación

Si deseas detalles de implementación, orientación de configuración o estructura interna, usa estos documentos como punto de partida.

- `docs/project-overview.md`: objetivos del proyecto, conceptos y estructura del código (chinos)
- `docs/api-reference.md`: referencia de herramientas MCP (chinos)
- `docs/codex-setup.md`: pasos de configuración de Codex (chinos)
- `docs/opencode-setup.md`: pasos de configuración de OpenCode (chinos)
- `docs/release-process.md`: flujo de trabajo de lanzamiento de PyPI y GitHub (chinos)
- `docs/git-workflow.md`: flujo de trabajo de colaboración del repositorio (chinos)

## Desarrollo

Si deseas modificar o verificar el proyecto localmente, usa esta ruta más corta. La instalación editable sigue siendo la vía de desarrollo, y el entorno local sigue requiriendo Python 3.10+.

```bash
python3.10 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -e .
PYTHONPATH=. python3 -m unittest discover -s tests
```

El punto de entrada principal es `sql_query_mcp/app.py`. Los módulos centrales incluyen:

- `sql_query_mcp/config.py`: carga y validación de configuración
- `sql_query_mcp/validator.py`: validación SQL de solo lectura
- `sql_query_mcp/introspection.py`: inspección de metadatos
- `sql_query_mcp/executor.py`: ejecución de consultas y límites
- `sql_query_mcp/adapters/`: adaptadores de PostgreSQL, MySQL y Hive

## Cómo contribuir

Si deseas contribuir o revisar el flujo de trabajo del repositorio, comienza con estas páginas.

- `CONTRIBUTING.md`
- `docs/roadmap.md`
- `docs/git-workflow.md` (chinos)

Ejecuta `PYTHONPATH=. python3 -m unittest discover -s tests` antes de enviar cambios.

## Licencia

Este proyecto se publica bajo la Licencia MIT. Consulta `LICENSE`.
