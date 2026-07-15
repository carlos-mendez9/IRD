# Guía de laboratorio: Clasificación de tiquetes con OpenRouter, FastMCP y PostgreSQL

**Curso/laboratorio:** Integración de modelos de IA mediante API, MCP y base de datos  
**Escenario:** Clasificar tiquetes institucionales usando un catálogo de servicios almacenado en PostgreSQL  
**Versión:** Julio 2026  
**Sistema de referencia:** Ubuntu en WSL2 o Linux nativo

---

## Objetivo general

Implementar una solución sencilla para clasificar tiquetes de soporte técnico usando:

1. **OpenRouter** como proveedor de modelos de IA mediante API.
2. **FastMCP** como capa MCP para consultar PostgreSQL.
3. **PostgreSQL/Supabase** como fuente del catálogo de servicios válidos.
4. **systemd** para dejar FastMCP ejecutándose en segundo plano.

La solución evita instalar un modelo local en la computadora. A diferencia de una implementación con Ollama, aquí la inferencia se realiza en un servicio externo mediante API.

---

# 1. Requerimientos y selección del modelo

## 1.1 Diferencia entre modelo local y modelo vía API

Existen dos formas comunes de usar modelos de lenguaje en una aplicación:

| Modalidad | Ejemplo | Requiere GPU local | Ventaja principal | Desventaja principal |
|---|---|---:|---|---|
| Modelo local | Ollama + Qwen, Llama, Gemma | Sí, si se busca buen rendimiento | Mayor privacidad y control local | Requiere VRAM, instalación y mantenimiento |
| Modelo remoto por API | OpenRouter | No | Fácil de probar, muchos modelos disponibles | El texto viaja a un tercero y puede haber límites o costos |

En este laboratorio se utiliza **OpenRouter**, por lo que la computadora del estudiante no necesita una GPU NVIDIA ni una cantidad alta de VRAM.

## 1.2 Requerimientos para esta práctica con OpenRouter

Requisitos mínimos:

- Linux, WSL2 o máquina virtual con acceso a Internet.
- Python 3.10 o superior.
- Acceso a PostgreSQL o Supabase.
- Cuenta en OpenRouter.
- Una API key de OpenRouter.
- Conexión HTTPS saliente hacia `https://openrouter.ai`.

No se requiere:

- CUDA.
- NVIDIA driver en Linux.
- Ollama.
- GPU local.
- Descarga de modelos de varios GB.

## 1.3 Modelos gratuitos en OpenRouter

OpenRouter ofrece modelos gratuitos marcados normalmente con sufijo `:free` y también un router llamado:

```text
openrouter/free
```

`openrouter/free` selecciona automáticamente entre modelos gratuitos disponibles y filtra modelos según las capacidades que solicite la petición, por ejemplo texto, visión, herramientas o salida estructurada.

Para este laboratorio hay dos opciones razonables:

| Modelo | Uso recomendado | Observación |
|---|---|---|
| `openrouter/free` | Pruebas iniciales y clases | Es la opción más simple; el modelo real puede variar |
| `openai/gpt-oss-20b:free` | Clasificación estructurada simple | Modelo gratuito reportado por OpenRouter con capacidades de razonamiento, herramientas y salida estructurada |

En esta guía se usará por defecto:

```text
openrouter/free
```

Si se desea mayor reproducibilidad, se puede cambiar por un modelo gratuito específico, por ejemplo:

```text
openai/gpt-oss-20b:free
```

> Importante: los modelos gratuitos pueden cambiar, agotarse temporalmente o tener límites. Para producción se recomienda evaluar modelos pagados, límites, latencia, privacidad y estabilidad.

## 1.4 Casos de uso de modelos vía API

Este enfoque es útil para:

- Clasificación de tiquetes.
- Extracción de campos desde texto libre.
- Generación de resúmenes.
- Normalización de solicitudes.
- Priorización inicial.
- Asistencia para mesas de servicio.
- Prototipos de integración con bases de datos.
- Laboratorios donde los estudiantes no tienen GPU.

No es la mejor opción cuando:

- Los datos no pueden salir de la institución.
- Se requiere funcionamiento sin Internet.
- Se necesita latencia muy estable.
- Se necesita control total del modelo.
- Se procesan datos sensibles sin un análisis formal de privacidad.

## 1.5 Comparación con modelos locales grandes

Algunos modelos modernos, como GLM, DeepSeek, Qwen grande, Llama grande o modelos MoE de cientos de miles de millones de parámetros, pueden ser muy potentes, pero no son viables localmente en una estación de trabajo común.

| Familia/modelo | Uso típico | Requerimiento local aproximado | Comentario |
|---|---|---|---|
| Qwen 3 4B/8B cuantizado | Clasificación, chat, extracción | 4 GB a 8 GB VRAM | Viable en GPU de consumidor |
| Llama 3.x 8B cuantizado | Clasificación y chat general | 6 GB a 8 GB VRAM | Viable en GPU de 8 GB con cuidado |
| Gemma 3/4 pequeños | Clasificación simple, multilingüe | 4 GB a 8 GB VRAM | Buen punto de entrada |
| DeepSeek R1/V3 grandes | Razonamiento complejo, código | Decenas o cientos de GB VRAM | Normalmente se usa por API o servidores especializados |
| GLM grande | Razonamiento, agentes, código | Decenas o cientos de GB VRAM | Mejor vía API o infraestructura de servidor |
| Modelos MoE grandes | Agentes, investigación, código | Infraestructura especializada | Aunque activen pocos parámetros, almacenan muchos pesos |

Para clasificación de tiquetes no se necesita un modelo enorme. La clave está en:

1. Catálogo de clases claro.
2. Prompt restrictivo.
3. Salida JSON validada.
4. Validación posterior contra PostgreSQL.
5. Evaluación con ejemplos reales o sintéticos.

---

# 2. Pruebas iniciales con OpenRouter

## 2.1 Crear la clave API

1. Entrar a OpenRouter.
2. Crear una cuenta.
3. Crear una API key.
4. Guardarla localmente como variable de entorno.

Nunca se debe compartir una clave real en documentos, repositorios, chats o capturas de pantalla.


3. Crear la API key.

https://openrouter.ai/workspaces/default/keys

![[1.png|672]]


![[2.png]]


## 2.2 Configurar la variable de entorno

En WSL/Linux:

```bash
export OPENROUTER_API_KEY="sk-or-v1-REEMPLAZAR_CON_SU_CLAVE"
```

Para verificar:

```bash
echo "$OPENROUTER_API_KEY"
```

Para guardarla de forma persistente en el laboratorio, puede agregarse al archivo `.env` del proyecto, pero ese archivo no debe subirse a Git.

## 2.3 Prueba simple con `curl`

El siguiente comando es similar al ejemplo básico de OpenRouter, pero usa variable de entorno en lugar de escribir la clave en el comando.

```bash
curl https://openrouter.ai/api/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENROUTER_API_KEY" \
  -d '{
    "model": "openrouter/free",
    "messages": [
      {
        "role": "user",
        "content": "Explique en una oración qué es un tiquete de soporte técnico."
      }
    ]
  }'
```

Respuesta esperada: un objeto JSON con un arreglo `choices` y un mensaje generado por el modelo.

## 2.4 Prueba de clasificación directa con `curl`

```bash
curl https://openrouter.ai/api/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENROUTER_API_KEY" \
  -d '{
    "model": "openrouter/free",
    "messages": [
      {
        "role": "system",
        "content": "Eres un clasificador determinista de tiquetes de soporte técnico."
      },
      {
        "role": "user",
        "content": "Clasifica el siguiente tiquete en una sola categoría: HPC, SSO, Plataforma web, Balanceador, Base de datos, Redes u Otro. Devuelve únicamente la categoría. Tiquete: El trabajo enviado a Slurm permanece pendiente por una restricción de QoS."
      }
    ],
    "temperature": 0
  }'
```

Respuesta esperada:

```text
HPC
```

## 2.5 Prueba con salida JSON

```bash
curl https://openrouter.ai/api/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENROUTER_API_KEY" \
  -d '{
    "model": "openrouter/free",
    "messages": [
      {
        "role": "system",
        "content": "Eres un clasificador de tiquetes. Responde solo JSON válido."
      },
      {
        "role": "user",
        "content": "Clasifica el tiquete en uno de estos servicios: HPC, SSO, Plataforma web, Balanceador, Base de datos, Redes, Otro. Devuelve JSON con servicio y confianza. Tiquete: No puedo iniciar sesión mediante Keycloak en la aplicación institucional."
      }
    ],
    "temperature": 0
  }'
```

Respuesta esperada aproximada:

```json
{
  "servicio": "SSO",
  "confianza": 0.98
}
```

Esta prueba no garantiza validación estricta. La validación formal se hará desde Python con Pydantic y, opcionalmente, con `response_format` de tipo `json_schema`.

---

# 3. Implementación con FastMCP y PostgreSQL

## 3.1 Arquitectura

La solución tendrá este flujo:

```text
Texto del tiquete
      │
      ▼
clasificar_tiquete_openrouter_mcp.py
      │
      ├── MCP HTTP → crm_mcp_server.py → PostgreSQL/Supabase
      │                              obtiene servicios válidos
      │
      └── HTTPS → OpenRouter → modelo gratuito
                         clasifica el tiquete
```

El servidor MCP expone funciones de consulta a PostgreSQL. El clasificador consulta el catálogo de servicios mediante MCP y luego envía al modelo solamente los servicios válidos.

## 3.2 Crear carpeta del proyecto

```bash
mkdir -p ~/clasificador_tiquetes_openrouter
cd ~/clasificador_tiquetes_openrouter

python3 -m venv .venv
source .venv/bin/activate

pip install --upgrade pip
pip install fastmcp "psycopg[binary]" httpx pydantic python-dotenv
```

## 3.3 Variables de entorno

Crear archivo `.env`:

```bash
nano .env
```

Contenido:

```dotenv
DATABASE_URL=postgresql://usuario:clave@servidor:5432/postgres?sslmode=require

OPENROUTER_API_KEY=sk-or-v1-REEMPLAZAR_CON_SU_CLAVE
OPENROUTER_MODEL=openrouter/free
OPENROUTER_BASE_URL=https://openrouter.ai/api/v1

MCP_HOST=127.0.0.1
MCP_PORT=8000
MCP_SERVER_URL=http://127.0.0.1:8000/mcp
```

Proteger el archivo:

```bash
chmod 600 .env
```

> En un repositorio real, `.env` debe agregarse a `.gitignore`.

## 3.4 Tabla de servicios en PostgreSQL

Crear tabla:

```sql
CREATE TABLE IF NOT EXISTS public.servicio (
    id BIGSERIAL PRIMARY KEY,
    nombre VARCHAR(150) NOT NULL UNIQUE,
    descripcion TEXT,
    activo BOOLEAN NOT NULL DEFAULT TRUE
);
```

Insertar datos de ejemplo:

```sql
INSERT INTO public.servicio (nombre, descripcion)
VALUES
    ('HPC', 'Clúster, Slurm, colas, nodos, GPU, software científico y almacenamiento HPC'),
    ('SSO', 'Keycloak, SAML, OpenID Connect, LDAP, autenticación y atributos'),
    ('Plataforma web', 'Drupal, WordPress, Joomla, Apache, PHP-FPM, Varnish y hospedaje web'),
    ('Balanceador', 'HAProxy, Traefik, balanceo de carga, VIP, certificados y proxy inverso'),
    ('Base de datos', 'PostgreSQL, MariaDB, consultas SQL, conexiones y respaldos'),
    ('Redes', 'DNS, VPN, firewall, conectividad, puertos y enrutamiento'),
    ('Otro', 'Solicitudes que no corresponden a los servicios anteriores')
ON CONFLICT (nombre) DO UPDATE
SET descripcion = EXCLUDED.descripcion,
    activo = TRUE;
```

## 3.5 Servidor MCP: `crm_mcp_server.py`

Crear archivo:

```bash
nano crm_mcp_server.py
```

Contenido:

```python
import os
from typing import Any

import psycopg
from dotenv import load_dotenv
from fastmcp import FastMCP
from psycopg.rows import dict_row


load_dotenv()

DATABASE_URL = os.environ["DATABASE_URL"]
MCP_HOST = os.getenv("MCP_HOST", "127.0.0.1")
MCP_PORT = int(os.getenv("MCP_PORT", "8000"))


mcp = FastMCP(
    name="CRM PostgreSQL",
    instructions=(
        "Servidor MCP de solo lectura para consultar los servicios "
        "utilizados en la clasificación de tiquetes."
    ),
)


def get_connection() -> psycopg.Connection:
    """
    Crea una conexión nueva a PostgreSQL.
    """

    return psycopg.connect(
        DATABASE_URL,
        row_factory=dict_row,
        connect_timeout=10,
    )


@mcp.tool
def listar_servicios() -> list[dict[str, Any]]:
    """
    Obtiene los servicios activos que pueden asignarse a un tiquete.
    """

    query = """
        SELECT
            id,
            nombre,
            COALESCE(descripcion, '') AS descripcion
        FROM public.servicio
        WHERE activo = TRUE
        ORDER BY nombre
    """

    with get_connection() as conn:
        with conn.cursor() as cursor:
            cursor.execute(query)
            return list(cursor.fetchall())


@mcp.tool
def obtener_servicio_por_id(servicio_id: int) -> dict[str, Any] | None:
    """
    Busca un servicio activo por su identificador.
    """

    query = """
        SELECT
            id,
            nombre,
            COALESCE(descripcion, '') AS descripcion
        FROM public.servicio
        WHERE id = %s
          AND activo = TRUE
    """

    with get_connection() as conn:
        with conn.cursor() as cursor:
            cursor.execute(query, (servicio_id,))
            return cursor.fetchone()


@mcp.tool
def buscar_servicios(texto: str) -> list[dict[str, Any]]:
    """
    Busca servicios por nombre o descripción.
    """

    texto = texto.strip()

    if not texto:
        return []

    patron = f"%{texto}%"

    query = """
        SELECT
            id,
            nombre,
            COALESCE(descripcion, '') AS descripcion
        FROM public.servicio
        WHERE activo = TRUE
          AND (
              nombre ILIKE %s
              OR descripcion ILIKE %s
          )
        ORDER BY nombre
        LIMIT 20
    """

    with get_connection() as conn:
        with conn.cursor() as cursor:
            cursor.execute(query, (patron, patron))
            return list(cursor.fetchall())


if __name__ == "__main__":
    mcp.run(
        transport="http",
        host=MCP_HOST,
        port=MCP_PORT,
    )
```

## 3.6 Probar el servidor MCP manualmente

En una terminal:

```bash
cd ~/clasificador_tiquetes_openrouter
source .venv/bin/activate
python crm_mcp_server.py
```

Debe quedar escuchando en:

```text
http://127.0.0.1:8000/mcp
```

En otra terminal:

```bash
ss -lntp | grep 8000
```

Debe verse algo semejante a:

```text
LISTEN ... 127.0.0.1:8000
```

## 3.7 Cliente de prueba MCP: `probar_mcp.py`

Crear archivo:

```bash
nano probar_mcp.py
```

Contenido:

```python
import asyncio
import os

from dotenv import load_dotenv
from fastmcp import Client


load_dotenv()

MCP_SERVER_URL = os.getenv("MCP_SERVER_URL", "http://127.0.0.1:8000/mcp")


async def main() -> None:
    async with Client(MCP_SERVER_URL) as client:
        await client.ping()

        herramientas = await client.list_tools()
        print("Conexión MCP correcta")
        print("Herramientas disponibles:")

        for herramienta in herramientas:
            print(f"- {herramienta.name}")

        resultado = await client.call_tool("listar_servicios", {})
        print("\nServicios devueltos por PostgreSQL:")
        print(resultado)


if __name__ == "__main__":
    asyncio.run(main())
```

Ejecutar:

```bash
python probar_mcp.py
```

Resultado esperado:

```text
Conexión MCP correcta
Herramientas disponibles:
- listar_servicios
- obtener_servicio_por_id
- buscar_servicios
```

## 3.8 Clasificador con OpenRouter: `clasificar_tiquete_openrouter_mcp.py`

Crear archivo:

```bash
nano clasificar_tiquete_openrouter_mcp.py
```

Contenido:

```python
import argparse
import asyncio
import json
import os
from typing import Any

import httpx
from dotenv import load_dotenv
from fastmcp import Client
from pydantic import BaseModel, Field, ValidationError


load_dotenv()

MCP_SERVER_URL = os.getenv("MCP_SERVER_URL", "http://127.0.0.1:8000/mcp")
OPENROUTER_API_KEY = os.environ["OPENROUTER_API_KEY"]
OPENROUTER_BASE_URL = os.getenv("OPENROUTER_BASE_URL", "https://openrouter.ai/api/v1")
OPENROUTER_MODEL = os.getenv("OPENROUTER_MODEL", "openrouter/free")


class Clasificacion(BaseModel):
    servicio_id: int
    servicio: str
    confianza: float = Field(ge=0.0, le=1.0)
    justificacion: str


def normalizar_resultado_mcp(resultado: Any) -> list[dict[str, Any]]:
    """
    Extrae datos estructurados devueltos por FastMCP.
    """

    if hasattr(resultado, "data") and resultado.data is not None:
        datos = resultado.data
    elif hasattr(resultado, "structured_content"):
        datos = resultado.structured_content
    else:
        raise RuntimeError("El resultado MCP no contiene datos estructurados reconocibles.")

    if isinstance(datos, list):
        return datos

    if isinstance(datos, dict):
        for clave in ("result", "servicios", "data"):
            if isinstance(datos.get(clave), list):
                return datos[clave]

    raise RuntimeError(f"Formato inesperado al leer servicios: {type(datos).__name__}")


async def consultar_servicios_mcp() -> list[dict[str, Any]]:
    """
    Consulta el catálogo de servicios mediante MCP HTTP.
    """

    async with Client(MCP_SERVER_URL) as client:
        resultado = await client.call_tool("listar_servicios", {})
        servicios = normalizar_resultado_mcp(resultado)

    if not servicios:
        raise RuntimeError("PostgreSQL no devolvió servicios activos.")

    return servicios


def construir_prompt(texto_tiquete: str, servicios: list[dict[str, Any]]) -> str:
    catalogo = "\n".join(
        (
            f"- ID {servicio['id']}: {servicio['nombre']}. "
            f"{servicio.get('descripcion', '')}"
        )
        for servicio in servicios
    )

    return f"""
Clasifica el tiquete en exactamente uno de los servicios permitidos.

SERVICIOS PERMITIDOS:
{catalogo}

REGLAS:
1. Debes escoger únicamente un servicio de la lista.
2. servicio_id y servicio deben corresponder al mismo registro.
3. No inventes identificadores ni nombres.
4. confianza debe estar entre 0 y 1.
5. La justificación debe ser breve y basarse en el texto del tiquete.
6. No incluyas texto fuera del objeto JSON.

TIQUETE:
{texto_tiquete}
""".strip()


async def clasificar_con_openrouter(
    texto_tiquete: str,
    servicios: list[dict[str, Any]],
) -> Clasificacion:
    """
    Envía el tiquete a OpenRouter y solicita salida JSON.
    """

    prompt = construir_prompt(texto_tiquete, servicios)

    schema = Clasificacion.model_json_schema()

    payload = {
        "model": OPENROUTER_MODEL,
        "messages": [
            {
                "role": "system",
                "content": (
                    "Eres un clasificador determinista de tiquetes "
                    "institucionales de tecnología. Responde solo JSON válido."
                ),
            },
            {
                "role": "user",
                "content": prompt,
            },
        ],
        "temperature": 0,
        "response_format": {
            "type": "json_schema",
            "json_schema": {
                "name": "clasificacion_tiquete",
                "strict": True,
                "schema": schema,
            },
        },
    }

    headers = {
        "Authorization": f"Bearer {OPENROUTER_API_KEY}",
        "Content-Type": "application/json",
        # Opcionales, pero recomendados por OpenRouter para identificación de la app.
        "HTTP-Referer": "http://localhost",
        "X-OpenRouter-Title": "Laboratorio clasificador de tiquetes",
    }

    timeout = httpx.Timeout(
        connect=10.0,
        read=180.0,
        write=30.0,
        pool=10.0,
    )

    async with httpx.AsyncClient(timeout=timeout) as client:
        response = await client.post(
            f"{OPENROUTER_BASE_URL}/chat/completions",
            headers=headers,
            json=payload,
        )
        response.raise_for_status()
        datos = response.json()

    contenido = datos["choices"][0]["message"]["content"]

    try:
        return Clasificacion.model_validate_json(contenido)
    except ValidationError as error:
        raise RuntimeError(f"OpenRouter devolvió JSON inválido: {contenido}") from error


def validar_clasificacion(
    clasificacion: Clasificacion,
    servicios: list[dict[str, Any]],
) -> Clasificacion:
    """
    Verifica que el servicio devuelto exista en PostgreSQL.
    """

    servicios_por_id = {int(servicio["id"]): servicio for servicio in servicios}
    servicio_bd = servicios_por_id.get(clasificacion.servicio_id)

    if servicio_bd is None:
        raise RuntimeError(f"El modelo devolvió un ID inexistente: {clasificacion.servicio_id}")

    nombre_bd = str(servicio_bd["nombre"])

    if clasificacion.servicio.strip().casefold() != nombre_bd.casefold():
        raise RuntimeError(
            "El nombre del servicio no corresponde al ID devuelto: "
            f"ID={clasificacion.servicio_id}, "
            f"modelo={clasificacion.servicio!r}, "
            f"base_de_datos={nombre_bd!r}"
        )

    clasificacion.servicio = nombre_bd
    return clasificacion


async def ejecutar(texto_tiquete: str) -> Clasificacion:
    servicios = await consultar_servicios_mcp()
    clasificacion = await clasificar_con_openrouter(texto_tiquete, servicios)
    return validar_clasificacion(clasificacion, servicios)


def main() -> None:
    parser = argparse.ArgumentParser(
        description="Clasifica un tiquete usando OpenRouter, MCP y PostgreSQL."
    )
    parser.add_argument("tiquete", help="Texto descriptivo del tiquete")
    args = parser.parse_args()

    try:
        resultado = asyncio.run(ejecutar(args.tiquete))
        print(json.dumps(resultado.model_dump(), ensure_ascii=False, indent=2))
    except httpx.HTTPStatusError as error:
        raise SystemExit(
            f"OpenRouter respondió con error HTTP {error.response.status_code}: "
            f"{error.response.text}"
        ) from error
    except httpx.ConnectError as error:
        raise SystemExit("No fue posible conectarse con OpenRouter.") from error
    except Exception as error:
        raise SystemExit(f"Error: {error}") from error


if __name__ == "__main__":
    main()
```

## 3.9 Prueba rápida completa

En una terminal, mantener el servidor MCP activo:

```bash
cd ~/clasificador_tiquetes_openrouter
source .venv/bin/activate
python crm_mcp_server.py
```

En otra terminal:

```bash
cd ~/clasificador_tiquetes_openrouter
source .venv/bin/activate

python clasificar_tiquete_openrouter_mcp.py \
  "El trabajo enviado a Slurm permanece pendiente por límite de QoS"
```

Resultado esperado:

```json
{
  "servicio_id": 1,
  "servicio": "HPC",
  "confianza": 0.95,
  "justificacion": "El tiquete menciona Slurm y una restricción de QoS, elementos propios del clúster HPC."
}
```

Otro ejemplo:

```bash
python clasificar_tiquete_openrouter_mcp.py \
  "La aplicación no recibe el atributo ucrUserId después de autenticarse con Keycloak"
```

Resultado esperado:

```json
{
  "servicio_id": 2,
  "servicio": "SSO",
  "confianza": 0.98,
  "justificacion": "El problema corresponde a atributos de usuario enviados por Keycloak."
}
```

## 3.10 Si el modelo gratuito no soporta salida estructurada

Algunos modelos gratuitos pueden no soportar `response_format` con `json_schema`. Si aparece un error relacionado con `response_format`, hay tres alternativas:

### Alternativa A: cambiar a otro modelo gratuito

En `.env`:

```dotenv
OPENROUTER_MODEL=openai/gpt-oss-20b:free
```

Reintentar la clasificación.

### Alternativa B: usar `openrouter/free`

En `.env`:

```dotenv
OPENROUTER_MODEL=openrouter/free
```

El router puede seleccionar automáticamente un modelo que soporte las características solicitadas.

### Alternativa C: quitar `response_format` y validar con Pydantic

Para simplificar, se puede eliminar temporalmente el bloque:

```python
"response_format": {
    "type": "json_schema",
    "json_schema": {
        "name": "clasificacion_tiquete",
        "strict": True,
        "schema": schema,
    },
},
```

Y confiar en el prompt más la validación local. Esta alternativa es menos robusta, pero útil para depuración.

---

# 4. Instalación del servicio FastMCP con systemd

Hasta ahora el servidor MCP se ha ejecutado manualmente. Para evitar iniciarlo en cada prueba, se puede dejar ejecutándose como servicio local.

En esta práctica **solo FastMCP** queda como servicio. OpenRouter es externo y no requiere servicio local.

## 4.1 Verificar rutas

```bash
realpath ~/clasificador_tiquetes_openrouter
realpath ~/clasificador_tiquetes_openrouter/.venv/bin/python
```

Supongamos estas rutas:

```text
/home/cmendez/clasificador_tiquetes_openrouter
/home/cmendez/clasificador_tiquetes_openrouter/.venv/bin/python
```

Ajuste las rutas si el usuario o carpeta son diferentes.

## 4.2 Crear el servicio

```bash
sudo nano /etc/systemd/system/crm-mcp-openrouter.service
```

Contenido:

```ini
[Unit]
Description=Servidor FastMCP para clasificador de tiquetes con PostgreSQL
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=cmendez
Group=cmendez

WorkingDirectory=/home/cmendez/clasificador_tiquetes_openrouter
EnvironmentFile=/home/cmendez/clasificador_tiquetes_openrouter/.env

ExecStart=/home/cmendez/clasificador_tiquetes_openrouter/.venv/bin/python /home/cmendez/clasificador_tiquetes_openrouter/crm_mcp_server.py

Restart=on-failure
RestartSec=5

StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

Cambiar `User`, `Group` y las rutas según corresponda.

## 4.3 Iniciar y habilitar

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now crm-mcp-openrouter
```

Comprobar estado:

```bash
sudo systemctl status crm-mcp-openrouter
```

Ver logs:

```bash
sudo journalctl -u crm-mcp-openrouter -f
```

Verificar puerto:

```bash
ss -lntp | grep 8000
```

## 4.4 Probar con el servicio activo

Con FastMCP ya corriendo por `systemd`:

```bash
cd ~/clasificador_tiquetes_openrouter
source .venv/bin/activate

python probar_mcp.py
```

Luego:

```bash
python clasificar_tiquete_openrouter_mcp.py \
  "El sitio WordPress muestra error 500 después de actualizar PHP-FPM"
```

Resultado esperado:

```json
{
  "servicio_id": 3,
  "servicio": "Plataforma web",
  "confianza": 0.95,
  "justificacion": "El tiquete menciona WordPress, error 500 y PHP-FPM, componentes de hospedaje web."
}
```

## 4.5 Detener, reiniciar o deshabilitar el servicio

Detener temporalmente:

```bash
sudo systemctl stop crm-mcp-openrouter
```

Iniciar:

```bash
sudo systemctl start crm-mcp-openrouter
```

Reiniciar después de modificar `crm_mcp_server.py`:

```bash
sudo systemctl restart crm-mcp-openrouter
```

Deshabilitar inicio automático:

```bash
sudo systemctl disable crm-mcp-openrouter
```

Detener y deshabilitar:

```bash
sudo systemctl disable --now crm-mcp-openrouter
```

Eliminar completamente el servicio:

```bash
sudo systemctl disable --now crm-mcp-openrouter
sudo rm /etc/systemd/system/crm-mcp-openrouter.service
sudo systemctl daemon-reload
sudo systemctl reset-failed
```

---

# 5. Pruebas sugeridas para estudiantes

## 5.1 Casos de prueba

Ejecutar los siguientes ejemplos:

```bash
python clasificar_tiquete_openrouter_mcp.py \
  "El trabajo de Slurm quedó pendiente por límite de QoS"
```

```bash
python clasificar_tiquete_openrouter_mcp.py \
  "No se envía el atributo email desde Keycloak al proveedor de servicio SAML"
```

```bash
python clasificar_tiquete_openrouter_mcp.py \
  "El sitio Drupal devuelve 503 desde Varnish"
```

```bash
python clasificar_tiquete_openrouter_mcp.py \
  "Se venció el certificado instalado en el balanceador"
```

```bash
python clasificar_tiquete_openrouter_mcp.py \
  "No responde la consulta a PostgreSQL desde la aplicación"
```

```bash
python clasificar_tiquete_openrouter_mcp.py \
  "No hay resolución DNS para el dominio institucional"
```

## 5.2 Métricas mínimas

Crear una tabla con 30 a 50 tiquetes de prueba y medir:

- Categoría esperada.
- Categoría devuelta.
- Confianza.
- Tiempo de respuesta.
- Errores de JSON.
- Errores HTTP.
- Casos ambiguos.

## 5.3 Discusión

Preguntas para el grupo:

1. ¿Qué ocurre si el catálogo de PostgreSQL cambia?
2. ¿Qué ocurre si el modelo devuelve un ID inexistente?
3. ¿Por qué se valida nuevamente contra PostgreSQL?
4. ¿Qué riesgos existen al enviar tiquetes a un servicio externo?
5. ¿Qué diferencias habría con una solución local usando Ollama?
6. ¿Cuándo conviene usar un modelo gratuito y cuándo uno pagado?

---

# 6. Solución de problemas

## 6.1 Error: falta API key

Mensaje típico:

```text
KeyError: 'OPENROUTER_API_KEY'
```

Solución:

```bash
grep OPENROUTER_API_KEY .env
```

Verificar que `.env` exista y que el programa se ejecute desde la carpeta correcta.

## 6.2 Error HTTP 401

Causa probable:

- API key incorrecta.
- API key revocada.
- Variable mal copiada.

Solución:

1. Crear una nueva API key en OpenRouter.
2. Actualizar `.env`.
3. Reintentar.

## 6.3 Error de límite o cuota

Los modelos gratuitos pueden tener límites de uso. Posibles soluciones:

- Esperar y reintentar.
- Cambiar a otro modelo gratuito.
- Usar un modelo pagado de bajo costo.
- Reducir pruebas concurrentes.

## 6.4 Error con `response_format`

Causa probable:

- El modelo seleccionado no soporta salida estructurada estricta.

Soluciones:

- Cambiar `OPENROUTER_MODEL`.
- Usar `openrouter/free`.
- Quitar temporalmente `response_format` y validar localmente.

## 6.5 MCP no responde

Verificar servicio:

```bash
sudo systemctl status crm-mcp-openrouter
```

Ver logs:

```bash
sudo journalctl -u crm-mcp-openrouter -n 100
```

Ver puerto:

```bash
ss -lntp | grep 8000
```

## 6.6 PostgreSQL no responde

Probar conexión con `psql`:

```bash
psql "$DATABASE_URL" -c "SELECT id, nombre FROM public.servicio ORDER BY nombre;"
```

Si no funciona:

- Revisar usuario y clave.
- Revisar host y puerto.
- Revisar `sslmode=require` si es Supabase.
- Revisar firewall o VPN.

---

# 7. Seguridad y buenas prácticas

1. No compartir claves API.
2. No subir `.env` a Git.
3. Rotar claves expuestas.
4. Usar modelos gratuitos solo para pruebas y laboratorios.
5. No enviar datos personales o sensibles sin autorización.
6. Validar siempre la salida del modelo.
7. Registrar errores y resultados para evaluación.
8. Usar límites de gasto o cuota cuando estén disponibles.
9. Separar entorno de laboratorio y producción.
10. Revisar políticas de privacidad del proveedor antes de procesar datos reales.

---

# 8. Resumen

En esta práctica se implementó un clasificador de tiquetes con:

- OpenRouter como proveedor de inferencia por API.
- Un modelo gratuito o router gratuito.
- PostgreSQL como catálogo de servicios.
- FastMCP como interfaz de herramientas.
- Python como cliente de clasificación.
- systemd para dejar FastMCP en segundo plano.

La idea central es que el modelo no invente categorías: primero se consulta PostgreSQL, luego se envía el catálogo válido al modelo y finalmente se valida la respuesta contra la base de datos.

Este patrón puede extenderse a otros casos:

- Clasificación de incidentes.
- Priorización de solicitudes.
- Enrutamiento a equipos de soporte.
- Extracción de campos desde formularios.
- Validación semántica contra catálogos institucionales.

---

# 9. Referencias

- OpenRouter Quickstart: https://openrouter.ai/docs/quickstart
- OpenRouter Authentication: https://openrouter.ai/docs/api_reference/authentication
- OpenRouter Free Models: https://openrouter.ai/collections/free-models
- OpenRouter Free Models Router: https://openrouter.ai/openrouter/free
- OpenRouter Structured Outputs: https://openrouter.ai/docs/guides/features/structured-outputs
- OpenRouter Models API: https://openrouter.ai/docs/guides/overview/models
- FastMCP: https://gofastmcp.com/
- Psycopg 3: https://www.psycopg.org/psycopg3/
- systemd service files: https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html

