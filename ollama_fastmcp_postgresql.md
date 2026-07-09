# Clasificación local de tiquetes con Ollama, FastMCP y PostgreSQL

> **Entorno de referencia:** Windows 10/11, WSL2 con Ubuntu, GPU NVIDIA, Python 3 y PostgreSQL  
> **Modelo utilizado:** `qwen3:8b`  
> **Última revisión:** 5 de julio de 2026

---

## Objetivo

Esta guía implementa una solución local para clasificar tiquetes de soporte técnico. El catálogo de servicios se almacena en PostgreSQL, se publica mediante un servidor FastMCP y se entrega a un modelo de lenguaje ejecutado localmente con Ollama.

La solución final tiene esta arquitectura:

```text
Texto del tiquete
       |
       v
clasificar_tiquete_mcp.py
       |
       +---- HTTP/MCP ----> FastMCP ----> PostgreSQL
       |                    localhost:8000
       |
       +---- HTTP/JSON ---> Ollama -----> qwen3:8b
                            localhost:11434
```

El flujo es deliberadamente determinista:

1. El programa consulta a PostgreSQL, por medio de MCP, cuáles servicios están activos.
2. Entrega al modelo únicamente ese catálogo permitido.
3. Solicita una respuesta con un esquema JSON estricto.
4. Valida que el identificador y el nombre devueltos por el modelo existan realmente en PostgreSQL.

> **Nota terminológica:** el protocolo se denomina **MCP**, *Model Context Protocol*. 

---

# Parte I. Requerimientos de hardware y selección del modelo

## 1. CPU, RAM, GPU y VRAM

Un modelo puede ejecutarse solamente con CPU y RAM, pero la inferencia suele ser mucho más lenta. Una GPU compatible permite acelerar gran parte o la totalidad de las capas del modelo.

Los recursos importantes son:

- **CPU:** ejecuta la aplicación, tokenización y las capas que no quepan en GPU.
- **RAM:** almacena el sistema operativo, la aplicación y cualquier parte del modelo descargada desde la GPU.
- **GPU:** acelera las operaciones matriciales de inferencia.
- **VRAM:** memoria propia de la GPU en la que se cargan pesos, caché de contexto y estructuras de ejecución.
- **Disco:** almacena los archivos descargados de los modelos.

El tamaño publicado por Ollama corresponde principalmente al archivo cuantizado del modelo. **La VRAM real necesaria es mayor**, porque también deben almacenarse la caché KV, el contexto y otros búferes de ejecución.

Por esa razón, un modelo cuyo archivo mide exactamente 8 GB no cabe cómodamente en una GPU de 8 GB.

## 2. Cuantización

La cuantización reduce la precisión numérica de los pesos para disminuir memoria y almacenamiento. Una variante como `Q4_K_M` usa aproximadamente cuatro bits por peso, con ajustes adicionales para conservar calidad.

Para estaciones de trabajo y computadoras personales, las variantes Q4 suelen proporcionar un buen equilibrio entre:

- calidad;
- velocidad;
- consumo de VRAM;
- tamaño en disco.

## 3. Modelos apropiados para una GPU de 8 GB

La siguiente tabla usa los tamaños publicados por Ollama para las variantes indicadas. La columna de VRAM es una recomendación práctica aproximada, no una garantía exacta.

| Modelo              | Tamaño publicado | VRAM práctica | Uso recomendado                                                                    |
| ------------------- | ---------------: | ------------: | ---------------------------------------------------------------------------------- |
| `qwen3:4b-instruct` |           2.5 GB |        4–6 GB | Clasificación rápida, extracción, equipos con pocos recursos                       |
| `gemma3:4b`         |           3.3 GB |        5–7 GB | Texto, clasificación y tareas con imágenes                                         |
| `llama3.1:8b`       |           4.9 GB |       7–10 GB | Chat general, resumen, extracción y asistentes                                     |
| `qwen3:8b`          |           5.2 GB |       8–10 GB | Clasificación multilingüe, JSON y seguimiento de instrucciones                     |
| `deepseek-r1:8b`    |           5.2 GB |       8–10 GB | Razonamiento local; más lento y generalmente innecesario para clasificación simple |
| `gemma3:12b`        |           8.1 GB |      12–16 GB | Mayor capacidad y visión, pero no es apropiado para una GPU de 8 GB                |

### Recomendación para este laboratorio

Se utilizará:

```text
qwen3:8b
```

Razones:

- buena capacidad multilingüe, incluido español;
- seguimiento consistente de instrucciones;
- soporte de salida estructurada;
- tamaño manejable en una GPU NVIDIA de 8 GB;
- suficiente capacidad para una clasificación basada en descripciones y ejemplos.

Para equipos con 4 o 6 GB de VRAM puede utilizarse:

```text
qwen3:4b-instruct
```

El cambio se realiza posteriormente en la variable `OLLAMA_MODEL`.

## 4. Modelos con requerimientos altos

### DeepSeek-R1

Ollama publica varias variantes de DeepSeek-R1:

| Modelo | Tamaño publicado aproximado |
|---|---:|
| `deepseek-r1:8b` | 5.2 GB |
| `deepseek-r1:14b` | 9.0 GB |
| `deepseek-r1:32b` | 20 GB |
| `deepseek-r1:70b` | 43 GB |
| `deepseek-r1:671b` | 404 GB |

En una GPU de 8 GB solamente son razonables las variantes pequeñas. Las variantes de 14B en adelante requieren descarga parcial a RAM o GPUs con más VRAM. Aunque Ollama puede repartir el modelo entre GPU y CPU, el rendimiento disminuye considerablemente.

DeepSeek-R1 está orientado a razonamiento extenso. Para clasificar un tiquete entre categorías conocidas, ese razonamiento normalmente agrega latencia sin aportar una mejora proporcional.

### DeepSeek-V3

DeepSeek-V3 es un modelo MoE de cientos de miles de millones de parámetros. Su ejecución local completa requiere infraestructura de servidor con gran cantidad de memoria y, normalmente, múltiples GPU. No es un modelo apropiado para una estación de trabajo con 8 GB de VRAM.

### GLM

Algunos ejemplos actuales ilustran la diferencia entre modelos de estación de trabajo y modelos de infraestructura:

- `glm-4.7-flash` tiene una variante Q4 publicada de aproximadamente **19 GB**. Para ejecutarlo completamente en GPU se requiere una tarjeta de alrededor de 24 GB o más, considerando la memoria adicional de ejecución.
- GLM-5 y GLM-5.2 aparecen con aproximadamente **756B parámetros**.
- En Ollama, `glm-5.2` se ofrece actualmente como modelo **cloud**, no como una descarga local adecuada para una GPU de 8 GB.

Estos modelos están orientados a programación avanzada, agentes, razonamiento prolongado y contextos muy grandes. No son necesarios para la clasificación propuesta en este laboratorio.

## 5. Casos de uso de modelos locales

Un modelo local puede utilizarse para:

- clasificación de tiquetes, documentos, correos o solicitudes;
- extracción de campos estructurados;
- resumen de documentos internos;
- generación de borradores;
- anonimización o normalización de texto;
- asistentes internos conectados a herramientas mediante MCP;
- consultas sobre catálogos o bases de datos;
- generación y explicación de código;
- análisis de contenido sin enviar datos a un proveedor externo.

Los modelos pequeños de 4B a 8B son especialmente útiles cuando la tarea está bien definida y el sistema proporciona:

- categorías explícitas;
- instrucciones claras;
- ejemplos representativos;
- validación de la salida;
- acceso controlado a datos externos.

---

# Parte II. Instalación de Ollama y prueba del modelo

## 6. Requisitos previos en Windows y WSL2

Se requiere:

- Windows 10 21H2 o posterior, o Windows 11;
- WSL2;
- una distribución Linux basada en `glibc`, como Ubuntu;
- controlador NVIDIA para Windows compatible con CUDA en WSL;
- conectividad HTTPS hacia el registro de Ollama.

En PowerShell puede verificarse WSL:

```powershell
wsl --status
wsl --update
```

Dentro de Ubuntu/WSL, compruebe la GPU:

```bash
nvidia-smi
```

Debe aparecer la GPU NVIDIA y su memoria disponible.

> En WSL se utiliza el controlador NVIDIA instalado en Windows. No se debe instalar un segundo controlador Linux del kernel dentro de la distribución WSL.

## 7. Instalar Ollama

Dentro de Ubuntu/WSL:

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Inicie y habilite el servicio:

```bash
sudo systemctl enable --now ollama
```

Compruebe su estado:

```bash
sudo systemctl status ollama
```

Debe aparecer:

```text
Active: active (running)
```

Verifique la API local:

```bash
curl http://127.0.0.1:11434/api/tags
```

## 8. Verificar detección de la GPU

Revise los mensajes del servicio:

```bash
sudo journalctl -u ollama --no-pager | tail -n 50
```

Busque información semejante a:

```text
library=CUDA
compute=7.5
total_vram="8.0 GiB"
```

La capacidad de cómputo y la VRAM exactas dependen de la GPU.

## 9. Descargar el modelo recomendado

```bash
ollama pull qwen3:8b
```

Compruebe los modelos instalados:

```bash
ollama list
```

## 10. Prueba interactiva de clasificación

Ejecute:

```bash
ollama run qwen3:8b
```

Use este mensaje:

```text
Clasifica el siguiente tiquete en exactamente una categoría:
HPC, SSO, Plataforma web, Balanceador, Base de datos, Redes u Otro.

Responde únicamente con el nombre de la categoría.

Tiquete:
El trabajo enviado a Slurm permanece pendiente por una restricción de QoS.
```

La salida esperada es:

```text
HPC
```

Otro ejemplo:

```text
Clasifica el siguiente tiquete en exactamente una categoría:
HPC, SSO, Plataforma web, Balanceador, Base de datos, Redes u Otro.

Responde únicamente con el nombre de la categoría.

Tiquete:
La aplicación no recibe el atributo de usuario después de autenticarse con Keycloak.
```

Salida esperada:

```text
SSO
```

Finalice la sesión interactiva con:

```text
/bye
```

## 11. Verificar el uso de GPU

Mientras el modelo está cargado, abra otra terminal y ejecute:

```bash
watch -n 1 nvidia-smi
```

También puede consultar cómo Ollama distribuyó el modelo:

```bash
ollama ps
```

En una GPU de 8 GB se espera que `qwen3:8b` se ejecute completamente o mayoritariamente en GPU cuando el contexto es moderado.

Para esta solución se utilizará un contexto de 4096 tokens. Los tiquetes suelen ser cortos y no necesitan contextos de 32K, 128K o superiores.

## 12. Problemas de DNS al descargar modelos

Si aparece un error como:

```text
lookup registry.ollama.ai ... i/o timeout
```

verifique primero:

```bash
ping -c 3 1.1.1.1
getent hosts registry.ollama.ai
curl -I https://registry.ollama.ai
```

Si la resolución vuelve a funcionar, repita la descarga antes de modificar permanentemente `/etc/resolv.conf`:

```bash
ollama pull qwen3:8b
```

---

# Parte III. Implementación con FastMCP y PostgreSQL

## 13. Responsabilidad de cada componente

### PostgreSQL

Mantiene el catálogo autorizado de servicios.

### FastMCP

Expone funciones Python como herramientas MCP. En este laboratorio publica herramientas de solo lectura para consultar PostgreSQL.

### Cliente de clasificación

Ejecuta dos operaciones:

1. consulta los servicios permitidos al servidor MCP;
2. solicita a Ollama la clasificación del tiquete.

### Ollama

Ejecuta `qwen3:8b` y devuelve un objeto JSON validado mediante un esquema.

## 14. Crear el proyecto Python

```bash
mkdir -p ~/clasificador_tiquetes
cd ~/clasificador_tiquetes

python3 -m venv .venv
source .venv/bin/activate

python -m pip install --upgrade pip
pip install fastmcp "psycopg[binary]" httpx pydantic python-dotenv
```

Opcionalmente, registre las versiones instaladas:

```bash
pip freeze > requirements-lock.txt
```

Cada vez que abra una nueva terminal:

```bash
cd ~/clasificador_tiquetes
source .venv/bin/activate
```

## 15. Crear la tabla en PostgreSQL

Ejecute el siguiente SQL en PostgreSQL o Supabase:

```sql
CREATE TABLE IF NOT EXISTS public.servicio (
    id BIGSERIAL PRIMARY KEY,
    nombre VARCHAR(150) NOT NULL UNIQUE,
    descripcion TEXT NOT NULL DEFAULT '',
    activo BOOLEAN NOT NULL DEFAULT TRUE
);

INSERT INTO public.servicio (nombre, descripcion)
VALUES
    (
        'HPC',
        'Clúster, Slurm, colas, nodos, GPU, QoS, software científico y almacenamiento HPC.'
    ),
    (
        'SSO',
        'Keycloak, SAML, OpenID Connect, LDAP, autenticación, autorización y atributos.'
    ),
    (
        'Plataforma web',
        'Drupal, WordPress, Joomla, Apache, Nginx, PHP-FPM, Varnish y hospedaje web.'
    ),
    (
        'Balanceador',
        'HAProxy, Traefik, proxy inverso, VIP, certificados TLS y distribución de tráfico.'
    ),
    (
        'Base de datos',
        'PostgreSQL, MariaDB, SQL, conexiones, rendimiento, respaldos y restauración.'
    ),
    (
        'Redes',
        'DNS, VPN, firewall, conectividad, puertos, enrutamiento y direccionamiento.'
    ),
    (
        'Otro',
        'Solicitudes que no corresponden claramente a los demás servicios.'
    )
ON CONFLICT (nombre) DO UPDATE
SET descripcion = EXCLUDED.descripcion,
    activo = TRUE;
```

Compruebe:

```sql
SELECT id, nombre, descripcion, activo
FROM public.servicio
ORDER BY id;
```

### Recomendación de seguridad

La cuenta usada por MCP debería tener únicamente permiso de lectura sobre las tablas necesarias. No utilice una cuenta administradora de PostgreSQL en una aplicación de producción.

## 16. Crear el archivo `.env`

```bash
nano ~/clasificador_tiquetes/.env
```

Contenido:

```dotenv
DATABASE_URL=postgresql://usuario:contrasena@servidor:5432/postgres?sslmode=require

MCP_HOST=127.0.0.1
MCP_PORT=8000
MCP_SERVER_URL=http://127.0.0.1:8000/mcp

OLLAMA_URL=http://127.0.0.1:11434
OLLAMA_MODEL=qwen3:8b
```

Proteja el archivo:

```bash
chmod 600 ~/clasificador_tiquetes/.env
```

Consideraciones:

- Sustituya usuario, contraseña, servidor, puerto y base de datos.
- Para una base local sin TLS puede retirar `sslmode=require` si la configuración lo permite.
- En una conexión remota, mantenga TLS.
- Si la contraseña contiene caracteres especiales, codifíquelos correctamente dentro de la URL.
- No publique `.env` en Git.

Cree un `.gitignore`:

```bash
cat > ~/clasificador_tiquetes/.gitignore <<'EOF_GITIGNORE'
.venv/
.env
__pycache__/
*.pyc
EOF_GITIGNORE
```

## 17. Servidor FastMCP: `crm_mcp_server.py`

Cree el archivo:

```bash
nano ~/clasificador_tiquetes/crm_mcp_server.py
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

DATABASE_URL = os.getenv("DATABASE_URL")
MCP_HOST = os.getenv("MCP_HOST", "127.0.0.1")
MCP_PORT = int(os.getenv("MCP_PORT", "8000"))

if not DATABASE_URL:
    raise RuntimeError("La variable DATABASE_URL no está configurada.")


mcp = FastMCP(
    name="CRM PostgreSQL",
    instructions=(
        "Servidor MCP de solo lectura para consultar el catálogo "
        "de servicios utilizado en la clasificación de tiquetes."
    ),
)


def get_connection() -> psycopg.Connection:
    """Crea una conexión a PostgreSQL con resultados tipo diccionario."""
    return psycopg.connect(
        DATABASE_URL,
        row_factory=dict_row,
        connect_timeout=10,
    )


@mcp.tool
def listar_servicios() -> list[dict[str, Any]]:
    """Devuelve todos los servicios activos disponibles para clasificación."""
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
    """Busca un servicio activo por su identificador."""
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
    """Busca servicios activos por nombre o descripción."""
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

### Por qué se usa HTTP

FastMCP usa STDIO de forma predeterminada. Con STDIO, el cliente inicia un nuevo proceso del servidor para cada sesión.

Con:

```python
mcp.run(transport="http", host="127.0.0.1", port=8000)
```

el servidor queda residente y atiende varios clientes mediante Streamable HTTP en:

```text
http://127.0.0.1:8000/mcp
```

Se utiliza `127.0.0.1` para no exponer el servidor MCP a otros equipos de la red.

## 18. Cliente de clasificación: `clasificar_tiquete_mcp.py`

Cree el archivo:

```bash
nano ~/clasificador_tiquetes/clasificar_tiquete_mcp.py
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

MCP_SERVER_URL = os.getenv(
    "MCP_SERVER_URL",
    "http://127.0.0.1:8000/mcp",
)
OLLAMA_URL = os.getenv(
    "OLLAMA_URL",
    "http://127.0.0.1:11434",
)
OLLAMA_MODEL = os.getenv("OLLAMA_MODEL", "qwen3:8b")


class Clasificacion(BaseModel):
    servicio_id: int
    servicio: str
    confianza: float = Field(ge=0.0, le=1.0)
    justificacion: str = Field(min_length=1, max_length=300)


async def consultar_servicios_mcp() -> list[dict[str, Any]]:
    """Obtiene el catálogo actual de servicios mediante MCP."""
    async with Client(MCP_SERVER_URL) as client:
        resultado = await client.call_tool("listar_servicios", {})

    servicios = resultado.data

    if not isinstance(servicios, list) or not servicios:
        raise RuntimeError(
            "El servidor MCP no devolvió una lista de servicios activos."
        )

    return servicios


async def clasificar_con_ollama(
    texto_tiquete: str,
    servicios: list[dict[str, Any]],
) -> Clasificacion:
    """Solicita a Ollama una clasificación con salida JSON estructurada."""
    catalogo = "\n".join(
        (
            f"- ID {servicio['id']}: {servicio['nombre']}. "
            f"{servicio.get('descripcion', '')}"
        )
        for servicio in servicios
    )

    prompt = f"""
Clasifica el tiquete en exactamente uno de los servicios permitidos.

SERVICIOS PERMITIDOS:
{catalogo}

REGLAS:
1. Escoge únicamente un servicio de la lista.
2. servicio_id y servicio deben corresponder al mismo registro.
3. No inventes identificadores ni categorías.
4. confianza debe ser un número entre 0 y 1.
5. La justificación debe ser breve y referirse al texto del tiquete.
6. Devuelve exclusivamente el objeto JSON solicitado.

TIQUETE:
{texto_tiquete}
""".strip()

    payload = {
        "model": OLLAMA_MODEL,
        "messages": [
            {
                "role": "system",
                "content": (
                    "Eres un clasificador determinista de tiquetes "
                    "institucionales de tecnología."
                ),
            },
            {
                "role": "user",
                "content": prompt,
            },
        ],
        "format": Clasificacion.model_json_schema(),
        "stream": False,
        "think": False,
        "keep_alive": "10m",
        "options": {
            "temperature": 0,
            "num_ctx": 4096,
            "seed": 42,
        },
    }

    timeout = httpx.Timeout(
        connect=10.0,
        read=180.0,
        write=30.0,
        pool=10.0,
    )

    async with httpx.AsyncClient(timeout=timeout) as client:
        response = await client.post(
            f"{OLLAMA_URL}/api/chat",
            json=payload,
        )
        response.raise_for_status()
        datos = response.json()

    contenido = datos["message"]["content"]

    try:
        return Clasificacion.model_validate_json(contenido)
    except ValidationError as error:
        raise RuntimeError(
            f"Ollama devolvió una clasificación inválida: {contenido}"
        ) from error


def validar_clasificacion(
    clasificacion: Clasificacion,
    servicios: list[dict[str, Any]],
) -> Clasificacion:
    """Valida la respuesta del modelo contra el catálogo de PostgreSQL."""
    servicios_por_id = {
        int(servicio["id"]): servicio
        for servicio in servicios
    }

    servicio_bd = servicios_por_id.get(clasificacion.servicio_id)

    if servicio_bd is None:
        raise RuntimeError(
            f"El modelo devolvió un ID inexistente: "
            f"{clasificacion.servicio_id}"
        )

    nombre_bd = str(servicio_bd["nombre"])

    if clasificacion.servicio.strip().casefold() != nombre_bd.casefold():
        raise RuntimeError(
            "El nombre del servicio no corresponde al ID devuelto. "
            f"Modelo={clasificacion.servicio!r}; "
            f"PostgreSQL={nombre_bd!r}."
        )

    clasificacion.servicio = nombre_bd
    return clasificacion


async def ejecutar(texto_tiquete: str) -> Clasificacion:
    servicios = await consultar_servicios_mcp()
    clasificacion = await clasificar_con_ollama(
        texto_tiquete=texto_tiquete,
        servicios=servicios,
    )
    return validar_clasificacion(clasificacion, servicios)


def main() -> None:
    parser = argparse.ArgumentParser(
        description=(
            "Clasifica un tiquete usando FastMCP, PostgreSQL y Ollama."
        )
    )
    parser.add_argument(
        "tiquete",
        help="Texto descriptivo del tiquete",
    )
    args = parser.parse_args()

    try:
        resultado = asyncio.run(ejecutar(args.tiquete))
        print(
            json.dumps(
                resultado.model_dump(),
                ensure_ascii=False,
                indent=2,
            )
        )
    except httpx.ConnectError as error:
        raise SystemExit(
            "No fue posible conectarse con Ollama. "
            "Verifique el servicio ollama."
        ) from error
    except httpx.HTTPStatusError as error:
        raise SystemExit(
            f"Ollama respondió con HTTP "
            f"{error.response.status_code}: "
            f"{error.response.text}"
        ) from error
    except Exception as error:
        raise SystemExit(f"Error: {error}") from error


if __name__ == "__main__":
    main()
```

## 19. Programa de prueba MCP: `probar_mcp.py`

Cree:

```bash
nano ~/clasificador_tiquetes/probar_mcp.py
```

Contenido:

```python
import asyncio
import os

from dotenv import load_dotenv
from fastmcp import Client


load_dotenv()

MCP_SERVER_URL = os.getenv(
    "MCP_SERVER_URL",
    "http://127.0.0.1:8000/mcp",
)


async def main() -> None:
    async with Client(MCP_SERVER_URL) as client:
        herramientas = await client.list_tools()
        resultado = await client.call_tool("listar_servicios", {})

    print("Conexión MCP correcta")
    print("Herramientas disponibles:")

    for herramienta in herramientas:
        print(f"- {herramienta.name}")

    print("\nServicios devueltos por PostgreSQL:")

    for servicio in resultado.data:
        print(f"- {servicio['id']}: {servicio['nombre']}")


if __name__ == "__main__":
    asyncio.run(main())
```

## 20. Prueba rápida en primer plano

### Terminal 1: comprobar Ollama

```bash
sudo systemctl status ollama
curl http://127.0.0.1:11434/api/tags
```

### Terminal 1: iniciar FastMCP manualmente

```bash
cd ~/clasificador_tiquetes
source .venv/bin/activate
python crm_mcp_server.py
```

El servidor debe indicar que escucha en una dirección semejante a:

```text
http://127.0.0.1:8000/mcp
```

### Terminal 2: probar MCP

```bash
cd ~/clasificador_tiquetes
source .venv/bin/activate
python probar_mcp.py
```

Salida esperada:

```text
Conexión MCP correcta
Herramientas disponibles:
- listar_servicios
- obtener_servicio_por_id
- buscar_servicios

Servicios devueltos por PostgreSQL:
- 1: HPC
- 2: SSO
...
```

Los identificadores dependen de los datos existentes en la base.

### Terminal 2: clasificar un tiquete

```bash
python clasificar_tiquete_mcp.py \
  "El trabajo enviado a Slurm permanece pendiente por una restricción de QoS"
```

Ejemplo de salida:

```json
{
  "servicio_id": 1,
  "servicio": "HPC",
  "confianza": 0.98,
  "justificacion": "El tiquete menciona Slurm y una restricción de QoS del clúster."
}
```

Otro ejemplo:

```bash
python clasificar_tiquete_mcp.py \
  "La aplicación no recibe el atributo de usuario después de autenticarse con Keycloak"
```

Ejemplo de salida:

```json
{
  "servicio_id": 2,
  "servicio": "SSO",
  "confianza": 0.99,
  "justificacion": "El problema se relaciona con atributos enviados por Keycloak."
}
```

Para detener la prueba manual del servidor MCP, use `Ctrl+C` en la Terminal 1.

## 21. Qué se valida en esta implementación

La solución no confía ciegamente en la respuesta del modelo. Aplica cuatro controles:

1. PostgreSQL define la lista autorizada.
2. Ollama recibe un esquema JSON de Pydantic.
3. Pydantic valida tipos y rango de confianza.
4. Python comprueba que el ID y el nombre correspondan al mismo registro de PostgreSQL.

Esta estrategia reduce categorías inventadas y respuestas difíciles de procesar.

## 22. Limitaciones del ejemplo

- Se abre una conexión PostgreSQL por cada llamada MCP.
- No existe autenticación en el endpoint MCP.
- El servidor escucha solo en `127.0.0.1`.
- La confianza es estimada por el modelo; no es una probabilidad calibrada estadísticamente.
- La calidad debe evaluarse con un conjunto de tiquetes históricos correctamente etiquetados.

Para mayor volumen se recomienda incorporar un pool de conexiones, métricas y una cola de procesamiento.

---

# Parte IV. Ejecutar FastMCP como servicio `systemd`

## 23. Objetivo del servicio

Durante la prueba rápida, `crm_mcp_server.py` se ejecuta en primer plano. El cliente puede conectarse rápidamente mientras ese proceso permanezca activo.

Para evitar iniciarlo manualmente, se registrará como servicio `systemd`.

`systemd` permitirá:

- iniciar FastMCP en segundo plano;
- iniciarlo al arrancar la distribución WSL;
- reiniciarlo si falla;
- consultar su estado;
- centralizar los logs;
- detenerlo de forma controlada.

## 24. Confirmar las rutas

```bash
whoami
realpath ~/clasificador_tiquetes
realpath ~/clasificador_tiquetes/.venv/bin/python
realpath ~/clasificador_tiquetes/crm_mcp_server.py
```

Supongamos el siguiente resultado:

```text
Usuario: cmendez
Proyecto: /home/cmendez/clasificador_tiquetes
Python: /home/cmendez/clasificador_tiquetes/.venv/bin/python
```

Cada estudiante debe sustituir `cmendez` por su usuario real.

## 25. Crear la unidad `systemd`

```bash
sudo nano /etc/systemd/system/crm-mcp.service
```

Contenido:

```ini
[Unit]
Description=Servidor FastMCP para clasificación de tiquetes
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=cmendez
Group=cmendez

WorkingDirectory=/home/cmendez/clasificador_tiquetes
ExecStart=/home/cmendez/clasificador_tiquetes/.venv/bin/python /home/cmendez/clasificador_tiquetes/crm_mcp_server.py

Restart=on-failure
RestartSec=5

NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full
UMask=0077

StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

Notas:

- `WorkingDirectory` permite que `python-dotenv` encuentre `.env`.
- `ExecStart` utiliza directamente el Python del entorno virtual.
- No se necesita ejecutar `source .venv/bin/activate` dentro de `systemd`.
- `Restart=on-failure` reinicia el proceso si termina por error.
- `127.0.0.1` mantiene MCP accesible solamente desde la misma instancia WSL.

## 26. Registrar e iniciar el servicio

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now crm-mcp
```

Compruebe:

```bash
sudo systemctl status crm-mcp
```

Verifique el puerto:

```bash
ss -lntp | grep 8000
```

Ejecute nuevamente la prueba:

```bash
cd ~/clasificador_tiquetes
source .venv/bin/activate
python probar_mcp.py
```

## 27. Consultar logs

Últimos mensajes:

```bash
sudo journalctl -u crm-mcp -n 100 --no-pager
```

Seguimiento en tiempo real:

```bash
sudo journalctl -u crm-mcp -f
```

Logs desde el arranque actual:

```bash
sudo journalctl -u crm-mcp -b --no-pager
```

## 28. Administrar el servicio

Detener:

```bash
sudo systemctl stop crm-mcp
```

Iniciar:

```bash
sudo systemctl start crm-mcp
```

Reiniciar después de modificar el código:

```bash
sudo systemctl restart crm-mcp
```

Consultar estado:

```bash
sudo systemctl status crm-mcp
```

Evitar que inicie automáticamente:

```bash
sudo systemctl disable crm-mcp
```

Detener y deshabilitar:

```bash
sudo systemctl disable --now crm-mcp
```

Volver a habilitar:

```bash
sudo systemctl enable --now crm-mcp
```

## 29. Eliminar el servicio

```bash
sudo systemctl disable --now crm-mcp
sudo rm /etc/systemd/system/crm-mcp.service
sudo systemctl daemon-reload
sudo systemctl reset-failed
```

## 30. Comportamiento en WSL

El servicio se inicia cuando arranca la distribución WSL con `systemd` habilitado.

El comando de Windows:

```powershell
wsl --shutdown
```

apaga la máquina virtual de WSL y, por lo tanto, detiene Ollama, FastMCP y los demás servicios Linux. Al abrir nuevamente la distribución, los servicios habilitados vuelven a iniciar.

## 31. Flujo de operación final

Compruebe los dos servicios:

```bash
sudo systemctl status ollama
sudo systemctl status crm-mcp
```

Luego ejecute solamente el cliente:

```bash
cd ~/clasificador_tiquetes
source .venv/bin/activate

python clasificar_tiquete_mcp.py \
  "No puedo iniciar sesión mediante Keycloak en la aplicación institucional"
```

En este punto:

- Ollama ya está residente en `127.0.0.1:11434`;
- FastMCP ya está residente en `127.0.0.1:8000/mcp`;
- el cliente no inicia un nuevo servidor MCP;
- el catálogo se obtiene de PostgreSQL en cada clasificación;
- el modelo permanece cargado durante el tiempo definido por `keep_alive`.

---

# Evaluación sugerida 

## 32. Construir un conjunto de prueba

Prepare al menos 100 tiquetes históricos con la categoría correcta. Evite utilizar los mismos ejemplos exactos incluidos en el prompt.

Campos recomendados:

```text
id_tiquete,texto,categoria_real
```

## 33. Métricas

Mida:

- exactitud global;
- precisión y exhaustividad por servicio;
- matriz de confusión;
- tiempo promedio por tiquete;
- porcentaje de respuestas rechazadas por validación;
- consumo de VRAM;
- diferencia entre `qwen3:4b-instruct` y `qwen3:8b`.

No seleccione el modelo solamente por una prueba manual. La decisión debe basarse en datos representativos.

## 34. Ejercicios de ampliación

1. Agregar ejemplos por servicio en PostgreSQL.
2. Crear una herramienta MCP que devuelva ejemplos históricos.
3. Registrar clasificaciones y correcciones humanas.
4. Implementar un umbral de confianza para revisión manual.
5. Comparar `qwen3:4b-instruct`, `qwen3:8b` y `deepseek-r1:8b`.
6. Añadir un pool de conexiones PostgreSQL.
7. Crear una API web para recibir tiquetes.
8. Incorporar autenticación si MCP se expone fuera de localhost.
9. Empaquetar la solución con contenedores.
10. Implementar pruebas automatizadas para las herramientas MCP.

---

# Solución de problemas

## FastMCP no inicia

```bash
sudo systemctl status crm-mcp
sudo journalctl -u crm-mcp -n 100 --no-pager
```

Compruebe:

- ruta del usuario;
- ruta del entorno virtual;
- permisos de `.env`;
- sintaxis de `DATABASE_URL`;
- acceso de red hacia PostgreSQL;
- disponibilidad del puerto 8000.

## El puerto 8000 ya está ocupado

```bash
ss -lntp | grep 8000
```

Cambie `MCP_PORT` en `.env`, por ejemplo:

```dotenv
MCP_PORT=8001
MCP_SERVER_URL=http://127.0.0.1:8001/mcp
```

Reinicie:

```bash
sudo systemctl restart crm-mcp
```

## PostgreSQL rechaza la conexión

Pruebe la cadena de conexión con `psql` si está instalado:

```bash
psql "$DATABASE_URL" -c \
  "SELECT id, nombre FROM public.servicio WHERE activo = TRUE;"
```

## Ollama no está disponible

```bash
sudo systemctl status ollama
sudo journalctl -u ollama -n 100 --no-pager
curl http://127.0.0.1:11434/api/tags
```

## El modelo no usa GPU

```bash
nvidia-smi
ollama ps
sudo journalctl -u ollama -n 100 --no-pager
```

## El modelo devuelve una clasificación incorrecta

Revise primero:

- calidad de las descripciones de los servicios;
- solapamiento entre categorías;
- ejemplos históricos;
- texto incompleto del tiquete;
- necesidad de una categoría de revisión manual.

La solución no debe forzar automáticamente una decisión de negocio cuando el tiquete sea ambiguo.

---

# Referencias

Consultadas el 5 de julio de 2026.

1. Ollama, instalación en Linux: <https://docs.ollama.com/linux>
2. Ollama, soporte de GPU: <https://docs.ollama.com/gpu>
3. Ollama, salidas estructuradas: <https://docs.ollama.com/capabilities/structured-outputs>
4. Ollama, API de generación y parámetros: <https://docs.ollama.com/api/generate>
5. Ollama, Qwen3 8B: <https://ollama.com/library/qwen3:8b>
6. Ollama, Qwen3 4B Instruct: <https://ollama.com/library/qwen3:4b-instruct>
7. Ollama, Llama 3.1 8B: <https://ollama.com/library/llama3.1:8b>
8. Ollama, Gemma 3: <https://ollama.com/library/gemma3>
9. Ollama, DeepSeek-R1 y tamaños: <https://ollama.com/library/deepseek-r1/tags>
10. Ollama, GLM-4.7-Flash: <https://ollama.com/library/glm-4.7-flash>
11. Ollama, GLM-5.2: <https://ollama.com/library/glm-5.2>
12. FastMCP, ejecución de servidores y transporte HTTP: <https://gofastmcp.com/deployment/running-server>
13. FastMCP, cliente: <https://gofastmcp.com/clients/client>
14. FastMCP, herramientas del cliente: <https://gofastmcp.com/clients/tools>
15. Microsoft, CUDA en WSL2: <https://learn.microsoft.com/en-us/windows/ai/directml/gpu-cuda-in-wsl>
16. Psycopg 3: <https://www.psycopg.org/psycopg3/>

---

# Resumen

La solución utiliza una separación clara de responsabilidades:

```text
PostgreSQL = fuente autorizada del catálogo
FastMCP   = interfaz de herramientas y datos
Ollama    = ejecución local del modelo
Qwen3 8B  = clasificación del lenguaje natural
Pydantic  = validación estructural
systemd   = administración de procesos residentes
```

Para una GPU NVIDIA de 8 GB, `qwen3:8b` constituye un punto de partida razonable. La calidad final no depende únicamente del modelo: también depende de un catálogo bien definido, instrucciones precisas, validación estricta y evaluación con datos históricos.
