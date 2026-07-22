# Laboratorio: RAG local con Ollama, PostgreSQL y pgvector

**Curso:** Inteligencia Artificial  
**Área de aplicación:** Telemática y gestión de servicios de TI  
**Modalidad:** Individual o parejas  
**Duración estimada:** 4 a 6 horas  
**Fecha de revisión:** 22 de julio de 2026

---

## 1. Descripción

En este laboratorio se construirá una aplicación de **generación aumentada por recuperación** (*Retrieval-Augmented Generation*, RAG) completamente local. La aplicación almacenará documentos técnicos en PostgreSQL, generará representaciones vectoriales mediante Ollama, recuperará los fragmentos más cercanos a una pregunta y enviará el contexto recuperado a un modelo de lenguaje para producir una respuesta con referencias.

El laboratorio reutiliza Ollama y los modelos Qwen trabajados anteriormente:

- **Modelo generativo principal:** `qwen3:4b-instruct`
- **Modelo generativo alternativo:** `qwen3:8b`
- **Modelo de embeddings:** `qwen3-embedding:0.6b`
- **Base de datos vectorial:** PostgreSQL con la extensión `pgvector`
- **Lenguaje de integración:** Python

> `qwen3:4b-instruct` y `qwen3:8b` generan las respuestas.  
> `qwen3-embedding:0.6b` no conversa: transforma textos y preguntas en vectores numéricos.

---

## 2. Objetivos

Al finalizar el laboratorio, el estudiante podrá:

1. Explicar la diferencia entre un modelo generativo y un modelo de embeddings.
2. Instalar y habilitar la extensión `pgvector` en PostgreSQL.
3. Consumir las API locales de generación y embeddings de Ollama.
4. Dividir documentos en fragmentos y almacenarlos como vectores.
5. Ejecutar búsquedas semánticas mediante distancia coseno.
6. Construir un flujo RAG con respuestas fundamentadas en documentos.
7. Comparar una respuesta directa del LLM con una respuesta aumentada mediante RAG.
8. Identificar limitaciones relacionadas con calidad, rendimiento y alucinaciones.

---

## 3. Arquitectura

```text
Documentos Markdown
        |
        v
Fragmentación del texto
        |
        v
Ollama: qwen3-embedding:0.6b
        |
        v
Vectores de 1024 dimensiones
        |
        v
PostgreSQL + pgvector
        |
        |  búsqueda por similitud coseno
        v
Tres fragmentos más relevantes
        |
        v
Ollama: qwen3:4b-instruct
        |
        v
Respuesta fundamentada + fuentes
```

![[rag1.png]]

---

## 4. Requisitos

### Requisitos mínimos sugeridos

- Windows 11 o Windows 10 compatible con WSL 2.
- Procesador de 64 bits.
- 8 GB de RAM como mínimo; 16 GB facilitan la ejecución.
- Aproximadamente 10 GB libres para programas, modelos y datos.
- Python 3.10 o superior.
- Acceso a Internet durante la instalación y descarga de modelos.
- No se requiere GPU.

### Modelos utilizados

| Modelo | Uso | Tamaño aproximado de descarga |
|---|---|---:|
| `qwen3-embedding:0.6b` | Embeddings | 639 MB |
| `qwen3:4b-instruct` | Generación principal | 2.5 GB |
| `qwen3:8b` | Comparación opcional | 5.2 GB |

La opción recomendada para computadoras sin GPU es `qwen3:4b-instruct`. El modelo de 8B puede producir mejores respuestas en algunos casos, pero consumirá más RAM y será más lento en CPU.

---

## 5. Selección del entorno

Se ofrecen dos rutas:

- **Ruta A: Windows nativo**
  - Ollama para Windows.
  - PostgreSQL para Windows.
  - `pgvector` compilado para la instalación de PostgreSQL de Windows.
  - Python para Windows.

- **Ruta B: WSL 2 con Ubuntu**
  - Ollama para Linux instalado dentro de WSL.
  - PostgreSQL y `pgvector` dentro de WSL.
  - Python dentro de WSL.

> **Recomendación:** complete todo el laboratorio en un solo entorno. No instale PostgreSQL en Windows y ejecute la aplicación en WSL, o viceversa, salvo que comprenda la configuración de red, autenticación y puertos entre ambos sistemas.

---

# PARTE I. INSTALACIÓN EN WINDOWS NATIVO

## 6. Instalar PostgreSQL en Windows

Si PostgreSQL ya está instalado, identifique su versión y continúe con la sección 7.

1. Descargue el instalador oficial para Windows.
2. Instale PostgreSQL y conserve seleccionados:
   - PostgreSQL Server.
   - Command Line Tools.
   - pgAdmin 4, opcional.
3. Defina una contraseña para el usuario administrativo `postgres`.
4. Mantenga el puerto predeterminado `5432`, salvo que ya esté ocupado.
5. Finalice la instalación.

Compruebe la versión desde PowerShell. Ajuste `17` a la versión instalada:

```powershell
$env:Path += ";C:\Program Files\PostgreSQL\17\bin"
psql --version
pg_config --version
```

Ejemplo de resultado:

```text
psql (PostgreSQL) 17.x
```

---

## 7. Instalar pgvector en PostgreSQL para Windows

La instalación nativa se realiza compilando `pgvector` con las herramientas de C++ de Microsoft.

### 7.1. Instalar las herramientas necesarias

Instale:

1. **Git para Windows**.
2. **Visual Studio Build Tools 2022** o una edición de Visual Studio.
3. En el instalador de Visual Studio seleccione la carga de trabajo:

```text
Desktop development with C++
```

Asegúrese de incluir:

- MSVC para x64.
- Windows SDK.
- Herramientas de compilación de C++.

Git también puede instalarse desde PowerShell:

```powershell
winget install --id Git.Git -e
```

Cierre y vuelva a abrir las terminales después de instalar Git.

### 7.2. Compilar pgvector

1. Busque en el menú Inicio:

```text
x64 Native Tools Command Prompt for VS 2022
```

2. Ejecútelo mediante **Run as administrator / Ejecutar como administrador**.
3. Compruebe que `nmake` y `cl` estén disponibles:

```bat
nmake /?
cl
```

4. Defina `PGROOT` con la ruta real de PostgreSQL. El siguiente ejemplo usa PostgreSQL 17:

```bat
set "PGROOT=C:\Program Files\PostgreSQL\17"
```

5. Descargue y compile `pgvector`:

```bat
cd %TEMP%
git clone --branch v0.8.5 --depth 1 https://github.com/pgvector/pgvector.git
cd pgvector
nmake /F Makefile.win
nmake /F Makefile.win install
```

La instalación debe copiar, entre otros archivos:

```text
%PGROOT%\lib\vector.dll
%PGROOT%\share\extension\vector.control
%PGROOT%\share\extension\vector--*.sql
```

Verifique:

```bat
dir "%PGROOT%\lib\vector.dll"
dir "%PGROOT%\share\extension\vector.control"
```

> Si tiene más de una versión de PostgreSQL instalada, `PGROOT` debe apuntar exactamente a la versión del servidor donde se habilitará la extensión.

### 7.3. Crear la base y habilitar la extensión

Abra PowerShell o **SQL Shell (psql)**. Ajuste la ruta a su versión:

```powershell
$env:Path += ";C:\Program Files\PostgreSQL\17\bin"
```

Cree el usuario y la base de datos:

```powershell
psql -U postgres -c "CREATE USER rag_user WITH PASSWORD 'rag_password';"
psql -U postgres -c "CREATE DATABASE rag_lab OWNER rag_user;"
```

Habilite la extensión en la base:

```powershell
psql -U postgres -d rag_lab -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

Verifique:

```powershell
psql -U postgres -d rag_lab -c "SELECT extname, extversion FROM pg_extension WHERE extname = 'vector';"
```

Debe aparecer una fila con el nombre `vector`.

> Las credenciales anteriores son exclusivamente para el laboratorio. No deben reutilizarse en ambientes reales.

---

## 8. Instalar Ollama en Windows

### Método mediante PowerShell

Abra PowerShell y ejecute:

```powershell
irm https://ollama.com/install.ps1 | iex
```

También puede utilizar el instalador gráfico `OllamaSetup.exe` disponible en el sitio oficial de Ollama.

Compruebe la instalación:

```powershell
ollama --version
```

Ollama se ejecuta normalmente como aplicación en segundo plano y publica su API local en:

```text
http://localhost:11434
```

Compruebe el servicio:

```powershell
Invoke-RestMethod http://localhost:11434/api/version
```

### Descargar los modelos

```powershell
ollama pull qwen3:4b-instruct
ollama pull qwen3-embedding:0.6b
```

Modelo alternativo:

```powershell
ollama pull qwen3:8b
```

Liste los modelos:

```powershell
ollama list
```

### Probar el modelo generativo

```powershell
ollama run qwen3:4b-instruct "Responda únicamente: Ollama funciona correctamente."
```

### Probar el modelo de embeddings

```powershell
$body = @{
    model = "qwen3-embedding:0.6b"
    input = "Prueba de búsqueda semántica"
} | ConvertTo-Json

$respuesta = Invoke-RestMethod `
    -Method Post `
    -Uri "http://localhost:11434/api/embed" `
    -ContentType "application/json" `
    -Body $body

$respuesta.embeddings[0].Count
```

El resultado esperado es:

```text
1024
```

---

# PARTE II. INSTALACIÓN EN WSL 2

## 9. Instalar WSL 2 y Ubuntu

Si WSL ya está instalado, continúe con la sección 10.

Abra PowerShell como administrador:

```powershell
wsl --install -d Ubuntu
```

Reinicie Windows cuando se solicite. Después abra Ubuntu y cree el usuario y contraseña de Linux.

Compruebe desde PowerShell:

```powershell
wsl --list --verbose
```

La distribución debería utilizar WSL 2:

```text
NAME      STATE    VERSION
Ubuntu    Running  2
```

Actualice WSL:

```powershell
wsl --update
```

---

## 10. Instalar PostgreSQL y pgvector en WSL

Todos los comandos de esta sección se ejecutan en la terminal de Ubuntu/WSL.

### 10.1. Instalar PostgreSQL y herramientas de compilación

```bash
sudo apt update
sudo apt install -y \
  postgresql \
  postgresql-contrib \
  postgresql-server-dev-all \
  build-essential \
  git \
  curl \
  python3 \
  python3-venv \
  python3-pip
```

Compruebe:

```bash
psql --version
pg_config --version
```

Inicie PostgreSQL:

```bash
sudo systemctl enable --now postgresql
```

Si su distribución de WSL no tiene `systemd` habilitado:

```bash
sudo service postgresql start
```

Compruebe:

```bash
sudo -u postgres psql -c "SELECT version();"
```

### 10.2. Compilar e instalar pgvector

```bash
cd /tmp
git clone --branch v0.8.5 --depth 1 https://github.com/pgvector/pgvector.git
cd pgvector
make
sudo make install
```

Verifique que PostgreSQL pueda encontrar la extensión:

```bash
pg_config --sharedir
find "$(pg_config --sharedir)/extension" -name "vector.control"
```

Si existen varias versiones de PostgreSQL, compile para la versión correcta. Ejemplo para PostgreSQL 16:

```bash
sudo apt install -y postgresql-server-dev-16

cd /tmp/pgvector
make clean
make PG_CONFIG=/usr/lib/postgresql/16/bin/pg_config
sudo make install PG_CONFIG=/usr/lib/postgresql/16/bin/pg_config
```

### 10.3. Crear el usuario, la base y la extensión

```bash
sudo -u postgres psql -c \
  "CREATE USER rag_user WITH PASSWORD 'rag_password';"

sudo -u postgres createdb \
  --owner=rag_user \
  rag_lab

sudo -u postgres psql \
  -d rag_lab \
  -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

Verifique:

```bash
sudo -u postgres psql \
  -d rag_lab \
  -c "SELECT extname, extversion FROM pg_extension WHERE extname = 'vector';"
```

---

## 11. Instalar Ollama dentro de WSL

En Ubuntu/WSL:

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Compruebe:

```bash
ollama --version
```

Si la instalación creó el servicio:

```bash
sudo systemctl enable --now ollama
sudo systemctl status ollama --no-pager
```

Si `systemd` no está disponible, ejecute Ollama manualmente en una terminal:

```bash
ollama serve
```

Mantenga esa terminal abierta y utilice otra terminal para continuar.

Compruebe la API:

```bash
curl -s http://localhost:11434/api/version
```

### Descargar los modelos

```bash
ollama pull qwen3:4b-instruct
ollama pull qwen3-embedding:0.6b
```

Modelo alternativo:

```bash
ollama pull qwen3:8b
```

Compruebe:

```bash
ollama list
```

### Probar el modelo generativo

```bash
ollama run qwen3:4b-instruct \
  "Responda únicamente: Ollama funciona correctamente."
```

### Probar el modelo de embeddings

```bash
curl -s http://localhost:11434/api/embed \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3-embedding:0.6b",
    "input": "Prueba de búsqueda semántica"
  }' |
python3 -c \
  'import json,sys; print(len(json.load(sys.stdin)["embeddings"][0]))'
```

Resultado esperado:

```text
1024
```

---

# PARTE III. CONSTRUCCIÓN DE LA APLICACIÓN RAG

## 12. Crear el proyecto

### En Windows PowerShell

```powershell
mkdir lab-rag-local
cd lab-rag-local

mkdir documentos
mkdir src
```

### En WSL

```bash
mkdir -p ~/lab-rag-local/{documentos,src}
cd ~/lab-rag-local
```

La estructura final será:

```text
lab-rag-local/
├── .env
├── requirements.txt
├── documentos/
│   ├── balanceador_waf.md
│   ├── firma_digital.md
│   ├── hospedaje_web.md
│   ├── hpc.md
│   └── sso.md
└── src/
    ├── cargar_documentos.py
    └── consultar_rag.py
```

---

## 13. Crear documentos de prueba

Los documentos son ficticios y se utilizan únicamente con fines académicos.

### `documentos/hospedaje_web.md`

```markdown
# Procedimiento básico para hospedaje web

Un error HTTP 503 indica que el servicio no se encuentra disponible temporalmente. Antes de reiniciar componentes se debe comprobar si el servidor web responde, si el proceso PHP-FPM está activo, si existe espacio libre en disco y si la base de datos acepta conexiones. También debe revisarse si el error afecta un único sitio o toda la plataforma.

Cuando el sitio muestra errores después de una actualización, se debe revisar el registro de errores de la aplicación y del servidor web. No se recomienda restaurar una copia ni modificar archivos antes de conservar la evidencia necesaria. Si se sospecha una vulneración, el sitio debe aislarse, conservar los registros y solicitar una copia limpia al responsable.
```

### `documentos/hpc.md`

```markdown
# Atención de solicitudes e incidentes de HPC

Cuando un trabajo permanece en estado pendiente, se debe consultar la razón indicada por el planificador. Entre las causas comunes se encuentran falta de recursos, límites de calidad de servicio, dependencia de otro trabajo o solicitud de una partición no disponible. Los comandos de diagnóstico pueden incluir squeue, sinfo y scontrol show job.

Para solicitar acceso al clúster, la persona debe indicar el proyecto de investigación, los recursos aproximados, el software requerido y si utilizará CPU o GPU. No se deben ejecutar procesos intensivos directamente en el nodo de acceso. Los cálculos deben enviarse mediante el planificador de trabajos.
```

### `documentos/sso.md`

```markdown
# Integración y diagnóstico del SSO

Una integración OIDC requiere un identificador de cliente, un secreto cuando corresponda, una URI de redirección registrada y la URL de descubrimiento del proveedor. El valor redirect_uri enviado por la aplicación debe coincidir exactamente con una URI autorizada, incluyendo protocolo, dominio, puerto y ruta.

El error invalid_redirect_uri suele indicar una diferencia entre la URI enviada y la configurada. También se deben revisar la sincronización de hora, la validez de los certificados, el flujo OIDC utilizado y los registros tanto de la aplicación como del proveedor de identidad.
```

### `documentos/balanceador_waf.md`

```markdown
# Diagnóstico de balanceador y WAF

Los errores HTTP 502 y 503 publicados por un balanceador pueden originarse en un backend no saludable, un puerto incorrecto, un tiempo de espera agotado o la ausencia de servidores disponibles. Se debe comprobar el estado del monitor, la conectividad desde el balanceador hacia el backend y la respuesta directa del servicio.

Cuando el WAF bloquea una solicitud, se debe registrar la fecha, la hora, la dirección solicitada, el método HTTP, el código de respuesta y el identificador de correlación del evento. Una exclusión debe limitarse a la regla y ruta estrictamente necesarias. No se recomienda deshabilitar toda la política de seguridad para resolver un caso particular.
```

### `documentos/firma_digital.md`

```markdown
# Diagnóstico básico de firma digital

Antes de probar una firma digital se debe comprobar que el certificado se encuentre vigente, que el dispositivo criptográfico sea reconocido, que los controladores estén instalados y que la aplicación pueda acceder al proveedor criptográfico. También se debe verificar la fecha y hora del equipo.

Si la firma falla, se debe registrar el mensaje exacto, el sistema operativo, la versión de Java o de la aplicación, el tipo de certificado y el paso donde ocurre el error. No se debe solicitar ni copiar la clave privada de la persona usuaria. Las pruebas deben utilizar certificados y ambientes destinados para simulación cuando estén disponibles.
```

---

## 14. Configurar Python

### Windows

```powershell
py -m venv .venv
Set-ExecutionPolicy -Scope Process Bypass
.\.venv\Scripts\Activate.ps1
```

### WSL

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Cree `requirements.txt`:

```text
numpy
pgvector
psycopg[binary]
python-dotenv
requests
```

Instale:

```bash
python -m pip install --upgrade pip
pip install -r requirements.txt
```

---

## 15. Configurar variables de entorno

Cree el archivo `.env` en la raíz:

```dotenv
DATABASE_URL=postgresql://rag_user:rag_password@localhost:5432/rag_lab
OLLAMA_URL=http://localhost:11434
CHAT_MODEL=qwen3:4b-instruct
EMBED_MODEL=qwen3-embedding:0.6b
TOP_K=3
```

Para utilizar el modelo alternativo:

```dotenv
CHAT_MODEL=qwen3:8b
```

> No publique archivos `.env` con contraseñas reales en repositorios Git.

---

## 16. Crear el programa de carga

Cree `src/cargar_documentos.py`:

```python
from __future__ import annotations

import hashlib
import os
from pathlib import Path
from typing import Iterable

import numpy as np
import psycopg
import requests
from dotenv import load_dotenv
from pgvector.psycopg import register_vector


BASE_DIR = Path(__file__).resolve().parent.parent
DOCUMENTOS_DIR = BASE_DIR / "documentos"

load_dotenv(BASE_DIR / ".env")

DATABASE_URL = os.getenv("DATABASE_URL")
OLLAMA_URL = os.getenv("OLLAMA_URL", "http://localhost:11434").rstrip("/")
EMBED_MODEL = os.getenv("EMBED_MODEL", "qwen3-embedding:0.6b")

DIMENSION_EMBEDDING = 1024
TAMANO_FRAGMENTO = 180
SOLAPAMIENTO = 30
TAMANO_LOTE = 8


def validar_configuracion() -> None:
    if not DATABASE_URL:
        raise RuntimeError("Falta DATABASE_URL en el archivo .env")

    if not DOCUMENTOS_DIR.exists():
        raise RuntimeError(
            f"No existe el directorio de documentos: {DOCUMENTOS_DIR}"
        )


def fragmentar(
    texto: str,
    max_palabras: int = TAMANO_FRAGMENTO,
    solapamiento: int = SOLAPAMIENTO,
) -> list[str]:
    palabras = texto.split()
    if not palabras:
        return []

    fragmentos: list[str] = []
    inicio = 0

    while inicio < len(palabras):
        fin = min(inicio + max_palabras, len(palabras))
        fragmento = " ".join(palabras[inicio:fin]).strip()

        if fragmento:
            fragmentos.append(fragmento)

        if fin >= len(palabras):
            break

        inicio = fin - solapamiento

    return fragmentos


def dividir_lotes(
    elementos: list[str],
    tamano: int,
) -> Iterable[list[str]]:
    for inicio in range(0, len(elementos), tamano):
        yield elementos[inicio:inicio + tamano]


def generar_embeddings(textos: list[str]) -> list[np.ndarray]:
    vectores: list[np.ndarray] = []

    for lote in dividir_lotes(textos, TAMANO_LOTE):
        respuesta = requests.post(
            f"{OLLAMA_URL}/api/embed",
            json={
                "model": EMBED_MODEL,
                "input": lote,
            },
            timeout=300,
        )
        respuesta.raise_for_status()

        datos = respuesta.json()
        embeddings = datos.get("embeddings")

        if not embeddings or len(embeddings) != len(lote):
            raise RuntimeError(
                "Ollama no devolvió la cantidad esperada de embeddings"
            )

        for embedding in embeddings:
            if len(embedding) != DIMENSION_EMBEDDING:
                raise RuntimeError(
                    "Dimensión inesperada: "
                    f"{len(embedding)}; se esperaban "
                    f"{DIMENSION_EMBEDDING}"
                )

            vectores.append(
                np.asarray(embedding, dtype=np.float32)
            )

    return vectores


def preparar_registros() -> list[dict]:
    registros: list[dict] = []

    archivos = sorted(DOCUMENTOS_DIR.glob("*.md"))
    if not archivos:
        raise RuntimeError(
            f"No se encontraron archivos Markdown en {DOCUMENTOS_DIR}"
        )

    for archivo in archivos:
        texto = archivo.read_text(encoding="utf-8")
        fragmentos = fragmentar(texto)

        for numero, contenido in enumerate(fragmentos, start=1):
            contenido_hash = hashlib.sha256(
                f"{archivo.name}:{numero}:{contenido}".encode("utf-8")
            ).hexdigest()

            registros.append(
                {
                    "fuente": archivo.name,
                    "fragmento_nro": numero,
                    "contenido": contenido,
                    "contenido_hash": contenido_hash,
                }
            )

    return registros


def crear_esquema(cursor: psycopg.Cursor) -> None:
    cursor.execute(
        """
        CREATE TABLE IF NOT EXISTS documento_fragmento (
            id BIGSERIAL PRIMARY KEY,
            fuente TEXT NOT NULL,
            fragmento_nro INTEGER NOT NULL,
            contenido TEXT NOT NULL,
            contenido_hash CHAR(64) NOT NULL UNIQUE,
            embedding VECTOR(1024) NOT NULL,
            creado_en TIMESTAMPTZ NOT NULL DEFAULT now()
        )
        """
    )


def main() -> None:
    validar_configuracion()
    registros = preparar_registros()
    textos = [registro["contenido"] for registro in registros]

    print(f"Documentos por indexar: {len(list(DOCUMENTOS_DIR.glob('*.md')))}")
    print(f"Fragmentos generados: {len(registros)}")
    print(f"Modelo de embeddings: {EMBED_MODEL}")

    embeddings = generar_embeddings(textos)

    with psycopg.connect(DATABASE_URL) as conexion:
        register_vector(conexion)

        with conexion.cursor() as cursor:
            crear_esquema(cursor)

            # Para que cada ejecución refleje exactamente los documentos
            # actuales del directorio.
            cursor.execute(
                "TRUNCATE TABLE documento_fragmento RESTART IDENTITY"
            )

            for registro, embedding in zip(
                registros,
                embeddings,
                strict=True,
            ):
                cursor.execute(
                    """
                    INSERT INTO documento_fragmento (
                        fuente,
                        fragmento_nro,
                        contenido,
                        contenido_hash,
                        embedding
                    )
                    VALUES (%s, %s, %s, %s, %s)
                    """,
                    (
                        registro["fuente"],
                        registro["fragmento_nro"],
                        registro["contenido"],
                        registro["contenido_hash"],
                        embedding,
                    ),
                )

    print("Carga finalizada correctamente.")


if __name__ == "__main__":
    main()
```

---

## 17. Cargar los documentos

Desde la raíz del proyecto, con el ambiente virtual activo:

### Windows

```powershell
python .\src\cargar_documentos.py
```

### WSL

```bash
python src/cargar_documentos.py
```

Resultado aproximado:

```text
Documentos por indexar: 5
Fragmentos generados: 5
Modelo de embeddings: qwen3-embedding:0.6b
Carga finalizada correctamente.
```

Compruebe en PostgreSQL:

### Windows

```powershell
psql -U rag_user -d rag_lab -c "SELECT id, fuente, fragmento_nro, vector_dims(embedding) AS dimensiones FROM documento_fragmento ORDER BY id;"
```

### WSL

```bash
psql -h localhost -U rag_user -d rag_lab \
  -c "SELECT id, fuente, fragmento_nro, vector_dims(embedding) AS dimensiones FROM documento_fragmento ORDER BY id;"
```

Cuando se solicite la contraseña:

```text
rag_password
```

---

## 18. Crear el programa de consulta RAG

Cree `src/consultar_rag.py`:

```python
from __future__ import annotations

import os
import sys
from pathlib import Path

import numpy as np
import psycopg
import requests
from dotenv import load_dotenv
from pgvector.psycopg import register_vector


BASE_DIR = Path(__file__).resolve().parent.parent
load_dotenv(BASE_DIR / ".env")

DATABASE_URL = os.getenv("DATABASE_URL")
OLLAMA_URL = os.getenv("OLLAMA_URL", "http://localhost:11434").rstrip("/")
CHAT_MODEL = os.getenv("CHAT_MODEL", "qwen3:4b-instruct")
EMBED_MODEL = os.getenv("EMBED_MODEL", "qwen3-embedding:0.6b")
TOP_K = int(os.getenv("TOP_K", "3"))

INSTRUCCION_BUSQUEDA = (
    "Instruct: Given a question in Spanish about information "
    "technology services, retrieve passages that answer the question.\n"
    "Query: "
)


def validar_configuracion() -> None:
    if not DATABASE_URL:
        raise RuntimeError("Falta DATABASE_URL en el archivo .env")


def generar_embedding_pregunta(pregunta: str) -> np.ndarray:
    texto_busqueda = f"{INSTRUCCION_BUSQUEDA}{pregunta}"

    respuesta = requests.post(
        f"{OLLAMA_URL}/api/embed",
        json={
            "model": EMBED_MODEL,
            "input": texto_busqueda,
        },
        timeout=300,
    )
    respuesta.raise_for_status()

    datos = respuesta.json()
    embeddings = datos.get("embeddings")

    if not embeddings:
        raise RuntimeError("Ollama no devolvió el embedding")

    return np.asarray(embeddings[0], dtype=np.float32)


def recuperar_fragmentos(
    pregunta: str,
) -> list[dict]:
    embedding = generar_embedding_pregunta(pregunta)

    with psycopg.connect(DATABASE_URL) as conexion:
        register_vector(conexion)

        with conexion.cursor() as cursor:
            cursor.execute(
                """
                SELECT
                    fuente,
                    fragmento_nro,
                    contenido,
                    1 - (embedding <=> %s) AS similitud
                FROM documento_fragmento
                ORDER BY embedding <=> %s
                LIMIT %s
                """,
                (embedding, embedding, TOP_K),
            )

            filas = cursor.fetchall()

    return [
        {
            "fuente": fila[0],
            "fragmento_nro": fila[1],
            "contenido": fila[2],
            "similitud": float(fila[3]),
        }
        for fila in filas
    ]


def construir_contexto(fragmentos: list[dict]) -> str:
    bloques: list[str] = []

    for indice, fragmento in enumerate(fragmentos, start=1):
        etiqueta = (
            f"Fuente {indice}: {fragmento['fuente']}"
            f"#fragmento-{fragmento['fragmento_nro']}"
        )

        bloques.append(
            f"[{etiqueta}]\n{fragmento['contenido']}"
        )

    return "\n\n".join(bloques)


def generar_respuesta(
    pregunta: str,
    fragmentos: list[dict],
) -> str:
    contexto = construir_contexto(fragmentos)

    mensaje_sistema = (
        "Usted es un asistente de soporte técnico. "
        "Responda únicamente con información contenida en el contexto. "
        "Si el contexto no permite responder, indique exactamente: "
        "\"No hay información suficiente en los documentos.\" "
        "Cite las fuentes mediante [Fuente 1], [Fuente 2], etc. "
        "No invente procedimientos, comandos, direcciones ni políticas. "
        "Responda en español de manera clara y breve."
    )

    mensaje_usuario = (
        f"CONTEXTO:\n{contexto}\n\n"
        f"PREGUNTA:\n{pregunta}"
    )

    respuesta = requests.post(
        f"{OLLAMA_URL}/api/chat",
        json={
            "model": CHAT_MODEL,
            "messages": [
                {
                    "role": "system",
                    "content": mensaje_sistema,
                },
                {
                    "role": "user",
                    "content": mensaje_usuario,
                },
            ],
            "stream": False,
            "options": {
                "temperature": 0.1,
                "num_ctx": 4096,
            },
        },
        timeout=600,
    )
    respuesta.raise_for_status()

    datos = respuesta.json()
    return datos["message"]["content"].strip()


def obtener_pregunta() -> str:
    if len(sys.argv) > 1:
        return " ".join(sys.argv[1:]).strip()

    return input("Pregunta: ").strip()


def main() -> None:
    validar_configuracion()
    pregunta = obtener_pregunta()

    if not pregunta:
        raise RuntimeError("La pregunta no puede estar vacía")

    fragmentos = recuperar_fragmentos(pregunta)

    print("\nFragmentos recuperados:")
    for indice, fragmento in enumerate(fragmentos, start=1):
        print(
            f"{indice}. {fragmento['fuente']} "
            f"(similitud={fragmento['similitud']:.4f})"
        )

    respuesta = generar_respuesta(pregunta, fragmentos)

    print("\nRespuesta RAG:")
    print(respuesta)


if __name__ == "__main__":
    main()
```

---

## 19. Ejecutar consultas

### Windows

```powershell
python .\src\consultar_rag.py "¿Qué se debe revisar cuando un sitio presenta un error HTTP 503?"
```

### WSL

```bash
python src/consultar_rag.py \
  "¿Qué se debe revisar cuando un sitio presenta un error HTTP 503?"
```

Ejemplo de salida:

```text
Fragmentos recuperados:
1. hospedaje_web.md (similitud=...)
2. balanceador_waf.md (similitud=...)
3. sso.md (similitud=...)

Respuesta RAG:
Se debe comprobar la disponibilidad del servidor web, PHP-FPM,
el espacio en disco y la conexión con la base de datos [Fuente 1].
Si el error es publicado por el balanceador, también debe revisarse
el monitor y la conectividad con el backend [Fuente 2].
```

---

## 20. Pruebas sugeridas

Ejecute las siguientes preguntas:

1. ¿Qué debo revisar cuando un sitio devuelve HTTP 503?
2. ¿Qué significa el error `invalid_redirect_uri`?
3. ¿Cómo se diagnostica un trabajo pendiente en HPC?
4. ¿Qué evidencia debe conservarse cuando el WAF bloquea una solicitud?
5. ¿Qué debe verificarse antes de probar una firma digital?
6. ¿Cómo cambio la contraseña de una red inalámbrica?

La sexta pregunta no está cubierta por los documentos. Observe si el sistema se abstiene o si intenta construir una respuesta con contexto irrelevante.

---

## 21. Comparar sin RAG y con RAG

Primero consulte directamente al modelo, sin documentos:

```bash
ollama run qwen3:4b-instruct \
  "¿Qué se debe revisar cuando un sitio institucional presenta un error HTTP 503?"
```

Después ejecute la aplicación RAG:

```bash
python src/consultar_rag.py \
  "¿Qué se debe revisar cuando un sitio institucional presenta un error HTTP 503?"
```

Complete:

| Criterio | Sin RAG | Con RAG |
|---|---|---|
| ¿Respondió correctamente? | | |
| ¿Utilizó los procedimientos suministrados? | | |
| ¿Incluyó fuentes? | | |
| ¿Inventó información? | | |
| Tiempo aproximado | | |
| Observaciones | | |

---

## 22. Búsqueda vectorial ejecutada

La consulta principal utiliza el operador `<=>`, que representa distancia coseno:

```sql
SELECT
    fuente,
    contenido,
    1 - (embedding <=> :vector_pregunta) AS similitud
FROM documento_fragmento
ORDER BY embedding <=> :vector_pregunta
LIMIT 3;
```

Interpretación:

- Distancia coseno cercana a `0`: textos más similares.
- `1 - distancia`: puntuación de similitud más intuitiva.
- `ORDER BY ... LIMIT 3`: recupera los tres vecinos más cercanos.

Para este corpus pequeño se utiliza búsqueda exacta. Un índice aproximado no aporta una mejora significativa.

---

## 23. Ejercicio opcional: índice HNSW

Cuando existan cientos o miles de fragmentos, cree un índice HNSW:

```sql
CREATE INDEX documento_fragmento_embedding_hnsw
ON documento_fragmento
USING hnsw (embedding vector_cosine_ops);
```

Actualice estadísticas:

```sql
ANALYZE documento_fragmento;
```

Observe el plan:

```sql
EXPLAIN ANALYZE
SELECT id, fuente
FROM documento_fragmento
ORDER BY embedding <=> (
    SELECT embedding
    FROM documento_fragmento
    LIMIT 1
)
LIMIT 3;
```

Investigue:

1. Diferencia entre búsqueda exacta y aproximada.
2. Ventajas y costos de HNSW.
3. Razón por la que un índice no mejora necesariamente un corpus de cinco filas.

---

## 24. Ejercicio opcional: medir rendimiento

### Windows PowerShell

```powershell
Measure-Command {
    python .\src\consultar_rag.py `
      "¿Qué significa invalid_redirect_uri?"
}
```

### WSL

```bash
time python src/consultar_rag.py \
  "¿Qué significa invalid_redirect_uri?"
```

Compare:

- `qwen3:4b-instruct`
- `qwen3:8b`
- `TOP_K=1`
- `TOP_K=3`
- `TOP_K=5`

Registre:

| Modelo | Top-K | Tiempo | Calidad percibida | RAM aproximada |
|---|---:|---:|---|---:|
| Qwen3 4B | 1 | | | |
| Qwen3 4B | 3 | | | |
| Qwen3 8B | 3 | | | |

---

## 25. Preguntas de análisis

1. ¿Por qué no se utiliza `qwen3:4b-instruct` directamente para generar los vectores?
2. ¿Qué ocurriría si se cambia el modelo de embeddings después de indexar los documentos?
3. ¿Por qué el vector de la pregunta debe tener la misma dimensión que la columna de PostgreSQL?
4. ¿Qué diferencia existe entre búsqueda por palabras y búsqueda semántica?
5. ¿El uso de RAG elimina completamente las alucinaciones? Justifique.
6. ¿Cómo afecta el tamaño de los fragmentos a la recuperación?
7. ¿Por qué debe evaluarse una pregunta cuya respuesta no esté en los documentos?
8. ¿Qué información sensible no debería almacenarse ni enviarse al modelo?
9. ¿Qué ventajas ofrece ejecutar el laboratorio completamente local?
10. ¿Cuándo sería preferible utilizar un servicio administrado en la nube?

---

## 26. Producto por entregar

El estudiante debe entregar:

1. Código fuente del proyecto.
2. Archivo `README.md` con instrucciones de ejecución.
3. Evidencia de:
   - versión de PostgreSQL;
   - extensión `vector` habilitada;
   - modelos instalados en Ollama;
   - dimensión de los embeddings;
   - carga de documentos;
   - al menos seis consultas.
4. Tabla comparativa de respuestas con y sin RAG.
5. Tabla de rendimiento con al menos dos configuraciones.
6. Respuestas a las preguntas de análisis.
7. Una conclusión de entre 200 y 300 palabras.

No se debe entregar el directorio `.venv` ni los archivos descargados de los modelos.

---

## 27. Rúbrica sugerida

| Criterio | Valor |
|---|---:|
| Instalación funcional de PostgreSQL y pgvector | 15 |
| Instalación y uso de Ollama | 10 |
| Generación y almacenamiento de embeddings | 20 |
| Recuperación semántica correcta | 20 |
| Respuesta RAG con fuentes | 15 |
| Comparación y medición | 10 |
| Análisis y conclusiones | 10 |
| **Total** | **100** |

---

# PARTE IV. SOLUCIÓN DE PROBLEMAS

## 28. `nmake` no se reconoce en Windows

Causa probable:

- Se abrió una consola normal en lugar de la consola de Visual Studio.
- No se instaló la carga de trabajo de C++.

Solución:

1. Abra `x64 Native Tools Command Prompt for VS 2022`.
2. Compruebe `nmake /?`.
3. Modifique la instalación de Visual Studio y agregue `Desktop development with C++`.

---

## 29. `extension "vector" is not available`

Compruebe:

```sql
SELECT version();
```

En Windows, revise:

```bat
dir "%PGROOT%\share\extension\vector.control"
dir "%PGROOT%\lib\vector.dll"
```

En WSL:

```bash
find "$(pg_config --sharedir)/extension" -name "vector.control"
```

La causa habitual es haber compilado la extensión para una versión de PostgreSQL diferente de la que ejecuta el servidor.

---

## 30. Error `postgres.h: No such file or directory`

En WSL faltan los archivos de desarrollo:

```bash
sudo apt install -y postgresql-server-dev-all
```

Para una versión específica:

```bash
sudo apt install -y postgresql-server-dev-16
```

Después:

```bash
cd /tmp/pgvector
make clean
make
sudo make install
```

---

## 31. No responde `localhost:11434`

Compruebe Ollama.

### Windows

- Abra Ollama desde el menú Inicio.
- Revise que aparezca en el área de notificación.
- Ejecute:

```powershell
ollama serve
```

### WSL

```bash
sudo systemctl status ollama
sudo systemctl start ollama
```

Sin `systemd`:

```bash
ollama serve
```

---

## 32. `model not found`

Descargue exactamente los modelos configurados:

```bash
ollama pull qwen3:4b-instruct
ollama pull qwen3-embedding:0.6b
```

Compruebe:

```bash
ollama list
```

El valor de `CHAT_MODEL` y `EMBED_MODEL` en `.env` debe coincidir con el nombre mostrado por `ollama list`.

---

## 33. Error de dimensión del vector

El laboratorio espera:

```text
1024 dimensiones
```

Compruébelo mediante la API de embeddings. Si se cambia el modelo, deberá:

1. conocer la nueva dimensión;
2. eliminar o recrear la tabla;
3. cambiar `VECTOR(1024)` por la dimensión correcta;
4. volver a generar todos los embeddings.

No se deben mezclar embeddings producidos por modelos diferentes en la misma búsqueda.

---

## 34. Error de autenticación de PostgreSQL

Compruebe la conexión:

```bash
psql -h localhost \
  -U rag_user \
  -d rag_lab
```

Datos del laboratorio:

```text
Usuario: rag_user
Contraseña: rag_password
Base: rag_lab
Puerto: 5432
```

Si PostgreSQL está ejecutándose en otro puerto, modifique `DATABASE_URL`.

---

## 35. La respuesta utiliza una fuente irrelevante

Posibles causas:

- Hay muy pocos documentos.
- La pregunta está fuera del dominio.
- `TOP_K` es demasiado alto.
- Los fragmentos son demasiado grandes.
- El umbral de similitud no se controla.

Como ampliación, descarte fragmentos con similitud inferior a un umbral:

```python
fragmentos = [
    fragmento
    for fragmento in fragmentos
    if fragmento["similitud"] >= 0.45
]
```

El valor `0.45` es solo un punto de partida: debe validarse experimentalmente con un conjunto de preguntas.

---

# 36. Ampliaciones posibles

1. Leer documentos PDF.
2. Conservar encabezados y metadatos Markdown.
3. Crear una interfaz web con FastAPI o Streamlit.
4. Agregar búsqueda híbrida: texto completo + vectores.
5. Incorporar un reranker.
6. Evaluar precisión de recuperación con preguntas etiquetadas.
7. Publicar la búsqueda como una herramienta MCP.
8. Registrar métricas de latencia y consumo.
9. Aplicar controles contra *prompt injection* indirecta.
10. Integrar documentos técnicos reales previamente anonimizados y autorizados.

---

# 37. Referencias oficiales

- Ollama, instalación en Windows:  
  https://docs.ollama.com/windows

- Ollama, instalación en Linux:  
  https://docs.ollama.com/linux

- Ollama, API local:  
  https://docs.ollama.com/api/introduction

- Ollama, embeddings:  
  https://docs.ollama.com/capabilities/embeddings

- Modelo `qwen3:4b-instruct`:  
  https://ollama.com/library/qwen3:4b-instruct

- Modelo `qwen3:8b`:  
  https://ollama.com/library/qwen3:8b

- Modelo `qwen3-embedding:0.6b`:  
  https://ollama.com/library/qwen3-embedding:0.6b

- pgvector, instalación y uso:  
  https://github.com/pgvector/pgvector

- Adaptador pgvector para Python:  
  https://github.com/pgvector/pgvector-python

- PostgreSQL para Windows:  
  https://www.postgresql.org/download/windows/

- PostgreSQL para Ubuntu:  
  https://www.postgresql.org/download/linux/ubuntu/

- Instalación de WSL:  
  https://learn.microsoft.com/windows/wsl/install

- Herramientas de compilación de C++ de Microsoft:  
  https://learn.microsoft.com/cpp/build/building-on-the-command-line
