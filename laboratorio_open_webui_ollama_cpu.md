# Laboratorio: IA generativa en el borde con Open WebUI y Ollama en una computadora sin GPU

**Modalidad:** laboratorio individual o en parejas  
**Duración estimada:** 90–120 minutos  
**Plataforma principal:** Docker Desktop o Docker Engine  
**Acelerador:** no requerido; la inferencia se realizará con CPU  
**Actualizado:** 15 de julio de 2026

---

## 1. Descripción

En este laboratorio se implementará una solución básica de inteligencia artificial generativa ejecutada localmente, también denominada **IA en el borde** (*edge AI*). La computadora descargará y ejecutará un modelo de lenguaje sin enviar cada consulta a un proveedor externo.

La solución utilizará los siguientes componentes:

- **Ollama:** motor encargado de descargar, administrar y ejecutar el modelo.
- **Open WebUI:** interfaz web similar a un servicio de chat.
- **Qwen2.5 0.5B:** modelo pequeño y multilingüe, adecuado para realizar pruebas en una computadora sin GPU.
- **Docker:** plataforma que encapsulará Open WebUI y Ollama en un único contenedor.

> Este laboratorio tiene fines educativos. Un modelo de 0.5 mil millones de parámetros permite probar la arquitectura y el flujo de inferencia, pero su calidad será considerablemente menor que la de modelos grandes ejecutados en servicios comerciales.

---

## 2. Objetivos de aprendizaje

Al finalizar el laboratorio, el estudiante podrá:

1. Explicar la función de Ollama y Open WebUI dentro de una solución local de IA.
2. Ejecutar un modelo de lenguaje utilizando únicamente CPU.
3. Descargar y administrar modelos con Ollama.
4. interactuar con el modelo desde una terminal y desde una interfaz web.
5. Observar el consumo de CPU y memoria durante una inferencia.
6. Identificar las limitaciones prácticas de los modelos pequeños ejecutados en el borde.

---

## 3. Arquitectura

```text
┌──────────────────────┐
│ Navegador del usuario│
│ http://localhost:3000│
└──────────┬───────────┘
           │ HTTP
           ▼
┌────────────────────────────────────┐
│ Contenedor Docker                  │
│                                    │
│  ┌──────────────┐   API local      │
│  │ Open WebUI   │ ───────────────┐ │
│  └──────────────┘                │ │
│                                  ▼ │
│                       ┌────────────┐│
│                       │ Ollama     ││
│                       └─────┬──────┘│
│                             │       │
│                       ┌─────▼──────┐│
│                       │ Qwen2.5    ││
│                       │ 0.5B       ││
│                       └────────────┘│
└────────────────────────────────────┘
             │
             ▼
       CPU y memoria RAM
```

En esta implementación, Open WebUI y Ollama se ejecutan dentro del mismo contenedor. Los datos se almacenan en volúmenes persistentes de Docker.

---

## 4. Requisitos

### 4.1 Requisitos mínimos sugeridos

- Procesador x86-64 o ARM64 relativamente reciente.
- 4 GB de memoria RAM como mínimo.
- 8 GB de RAM recomendados.
- Al menos 8 GB de espacio libre en disco.
- Windows 10/11, Linux o macOS.
- Docker Desktop o Docker Engine.
- Conexión a Internet únicamente para descargar las imágenes y el modelo.

> Después de descargar todos los componentes, la generación de respuestas puede realizarse sin conexión a Internet.

### 4.2 Verificar Docker

Ejecute:

```bash
docker version
```

También puede verificar el complemento de Docker Compose:

```bash
docker compose version
```

El laboratorio utiliza `docker run`, por lo que Docker Compose no es obligatorio.

### 4.3 Revisar los recursos del equipo

En Linux:

```bash
lscpu
free -h
df -h
```

En Windows PowerShell:

```powershell
Get-CimInstance Win32_Processor |
    Select-Object Name, NumberOfCores, NumberOfLogicalProcessors

Get-CimInstance Win32_ComputerSystem |
    Select-Object TotalPhysicalMemory
```

---

## 5. Selección del modelo

Se utilizará el modelo:

```text
qwen2.5:0.5b
```

La variante disponible en Ollama tiene aproximadamente:

- 494 millones de parámetros.
- 398 MB de tamaño.
- Cuantización Q4_K_M.
- Soporte multilingüe, incluido español.

Modelos alternativos para comparar:

| Modelo | Tamaño aproximado | Uso sugerido |
|---|---:|---|
| `smollm2:360m` | Cerca de 300 MB, según la cuantización | Equipo con muy pocos recursos; menor calidad |
| `qwen2.5:0.5b` | 398 MB | Opción principal del laboratorio |
| `qwen3:0.6b` | 523 MB | Alternativa más reciente; puede consumir más tiempo al razonar |

Los tamaños pueden cambiar según la variante y cuantización seleccionadas.

---

## 6. Implementación

### Paso 1. Crear los volúmenes persistentes

Ejecute:

```bash
docker volume create ollama
docker volume create open-webui
```

Compruebe su creación:

```bash
docker volume ls
```

Los volúmenes almacenarán:

- `ollama`: archivos del modelo descargado.
- `open-webui`: usuarios, configuración y conversaciones de Open WebUI.

---

### Paso 2. Crear el contenedor

Ejecute el siguiente comando:

```bash
docker run -d \
  --name open-webui \
  --restart unless-stopped \
  -p 127.0.0.1:3000:8080 \
  -v ollama:/root/.ollama \
  -v open-webui:/app/backend/data \
  ghcr.io/open-webui/open-webui:ollama
```

En Windows PowerShell puede utilizar el comando en una sola línea:

```powershell
docker run -d --name open-webui --restart unless-stopped -p 127.0.0.1:3000:8080 -v ollama:/root/.ollama -v open-webui:/app/backend/data ghcr.io/open-webui/open-webui:ollama
```

La imagen `open-webui:ollama` incluye tanto Open WebUI como Ollama.

La publicación `127.0.0.1:3000:8080` permite acceder al servicio únicamente desde la computadora local. No se debe eliminar `127.0.0.1` en este laboratorio, ya que no se pretende publicar la interfaz en la red.

---

### Paso 3. Verificar el contenedor

Ejecute:

```bash
docker ps
```

Debe aparecer un contenedor llamado `open-webui`.

Revise los mensajes iniciales:

```bash
docker logs --tail 100 open-webui
```

Para seguir los mensajes en tiempo real:

```bash
docker logs -f open-webui
```

Presione `Ctrl+C` para dejar de observar los registros. Esto no detiene el contenedor.

---

### Paso 4. Descargar el modelo

Ejecute:

```bash
docker exec -it open-webui ollama pull qwen2.5:0.5b
```

La descarga puede tardar algunos minutos, según la velocidad de Internet y del disco.

Compruebe los modelos instalados:

```bash
docker exec open-webui ollama list
```

Resultado esperado similar a:

```text
NAME              ID              SIZE
qwen2.5:0.5b      ...             398 MB
```

---

### Paso 5. Probar el modelo desde la terminal

Inicie una conversación:

```bash
docker exec -it open-webui ollama run qwen2.5:0.5b
```

Pruebe estas consultas:

```text
Responde en español. Explica en tres oraciones qué es un sistema operativo.
```

```text
Convierte esta información en JSON: servidor web, estado activo, puerto 443.
```

```text
Clasifica el siguiente reporte como incidente o solicitud:
"No puedo ingresar al sistema desde esta mañana".
```

Para finalizar la conversación, presione `Ctrl+D` o `Ctrl+C`.

También puede realizar una consulta no interactiva:

```bash
docker exec open-webui ollama run qwen2.5:0.5b \
  "Explica brevemente qué significa ejecutar un modelo en el borde."
```

En PowerShell:

```powershell
docker exec open-webui ollama run qwen2.5:0.5b "Explica brevemente qué significa ejecutar un modelo en el borde."
```

---

### Paso 6. Acceder a Open WebUI

Abra en el navegador:

```text
http://localhost:3000
```

Realice lo siguiente:

1. Complete el registro local inicial.
2. Inicie una conversación nueva.
3. Seleccione `qwen2.5:0.5b` en la lista de modelos.
4. Envíe una consulta sencilla.
5. Compruebe que la respuesta se genera localmente.

Si el modelo no aparece inmediatamente:

1. Actualice la página.
2. Compruebe el modelo con:

   ```bash
   docker exec open-webui ollama list
   ```

3. Reinicie el contenedor:

   ```bash
   docker restart open-webui
   ```

---

## 7. Pruebas funcionales

### Prueba 1. Seguimiento de instrucciones

Utilice:

```text
Responde únicamente con tres viñetas. Indica tres ventajas de ejecutar un modelo local.
```

Evalúe:

- ¿Respetó el número de elementos?
- ¿Agregó información no solicitada?
- ¿La respuesta fue coherente?

### Prueba 2. Resumen

Utilice:

```text
Resume en una oración:

Docker permite empaquetar una aplicación con sus dependencias dentro de
contenedores. Esto facilita que la aplicación se ejecute de manera consistente
en distintos equipos y sistemas operativos.
```

Evalúe si el resumen conserva la idea principal.

### Prueba 3. Salida estructurada

Utilice:

```text
Devuelve exclusivamente JSON válido con los campos servicio, estado y prioridad.

Servicio: correo institucional
Situación: los usuarios no pueden enviar mensajes
Prioridad: alta
```

Compruebe:

- ¿La respuesta es JSON válido?
- ¿Agregó texto antes o después del objeto?
- ¿Los valores corresponden al enunciado?

### Prueba 4. Conocimiento y alucinaciones

Utilice una pregunta para la cual pueda comprobar fácilmente la respuesta:

```text
¿Cuál es la función principal del kernel de un sistema operativo?
```

Luego pregunte:

```text
Indica qué partes de tu respuesta podrían ser imprecisas.
```

Discuta por qué una respuesta bien redactada no garantiza que la información sea correcta.

---

## 8. Medición básica de rendimiento

### 8.1 Observar el consumo de recursos

Abra una segunda terminal y ejecute:

```bash
docker stats open-webui
```

Mientras el comando está activo, envíe una consulta desde Open WebUI.

Registre:

- Porcentaje máximo de CPU.
- Memoria utilizada.
- Tiempo aproximado hasta que apareció el primer texto.
- Tiempo total de generación.

Presione `Ctrl+C` para finalizar `docker stats`.

### 8.2 Consultar el modelo cargado

Ejecute:

```bash
docker exec open-webui ollama ps
```

El comando permite observar si el modelo está cargado y qué recursos utiliza.

### 8.3 Medir una consulta desde Linux

```bash
time docker exec open-webui ollama run qwen2.5:0.5b \
  "Enumera cinco funciones de un sistema operativo."
```

En Windows PowerShell:

```powershell
Measure-Command {
    docker exec open-webui ollama run qwen2.5:0.5b `
        "Enumera cinco funciones de un sistema operativo."
}
```

> La primera consulta puede ser más lenta porque Ollama debe cargar el modelo en memoria. Las consultas posteriores pueden ser más rápidas mientras el modelo permanezca cargado.

---

## 9. Comparación opcional con un modelo todavía más pequeño

Descargue SmolLM2:

```bash
docker exec -it open-webui ollama pull smollm2:360m
```

Compruebe los modelos:

```bash
docker exec open-webui ollama list
```

Repita en Open WebUI las pruebas de seguimiento de instrucciones, resumen y JSON utilizando `smollm2:360m`.

Complete la siguiente tabla:

| Criterio | `qwen2.5:0.5b` | `smollm2:360m` |
|---|---|---|
| Tiempo de respuesta | | |
| Uso máximo de memoria | | |
| Calidad en español | | |
| Seguimiento de instrucciones | | |
| JSON válido | | |

Finalmente, indique cuál utilizaría en un equipo de recursos limitados y justifique su respuesta.

---

## 10. Persistencia

Reinicie el contenedor:

```bash
docker restart open-webui
```

Después del reinicio:

1. Acceda nuevamente a `http://localhost:3000`.
2. Compruebe que el modelo continúa disponible.
3. Compruebe que las conversaciones anteriores siguen registradas.

Esto demuestra que los datos no se almacenan únicamente en la capa temporal del contenedor, sino en los volúmenes `ollama` y `open-webui`.

---

## 11. Comandos de administración

### Detener el servicio

```bash
docker stop open-webui
```

### Iniciar el servicio

```bash
docker start open-webui
```

### Reiniciar el servicio

```bash
docker restart open-webui
```

### Ver registros

```bash
docker logs --tail 200 open-webui
```

### Listar modelos

```bash
docker exec open-webui ollama list
```

### Eliminar un modelo

```bash
docker exec open-webui ollama rm smollm2:360m
```

### Eliminar únicamente el contenedor

Los modelos y conversaciones se conservan en los volúmenes:

```bash
docker rm -f open-webui
```

### Eliminar completamente el laboratorio

> **Advertencia:** estos comandos eliminan los modelos, cuentas, configuración y conversaciones.

```bash
docker rm -f open-webui
docker volume rm ollama
docker volume rm open-webui
```

---

## 12. Solución de problemas

### Problema: el puerto 3000 ya está ocupado

Utilice otro puerto local, por ejemplo 3001:

```bash
docker rm -f open-webui
```

```bash
docker run -d \
  --name open-webui \
  --restart unless-stopped \
  -p 127.0.0.1:3001:8080 \
  -v ollama:/root/.ollama \
  -v open-webui:/app/backend/data \
  ghcr.io/open-webui/open-webui:ollama
```

Luego acceda a:

```text
http://localhost:3001
```

### Problema: el contenedor se detiene

Revise:

```bash
docker ps -a
docker logs --tail 200 open-webui
```

Posibles causas:

- Memoria RAM insuficiente.
- Poco espacio disponible en disco.
- Docker no inició correctamente.
- La descarga de la imagen quedó incompleta.

### Problema: la computadora se vuelve muy lenta

1. Cierre aplicaciones que consuman mucha memoria.
2. Evite ejecutar varios modelos simultáneamente.
3. Utilice `smollm2:360m`.
4. Envíe consultas cortas.
5. No cargue documentos grandes en esta práctica.
6. Limite el contexto a aproximadamente 2048 tokens desde los parámetros avanzados de Open WebUI, cuando esa opción esté disponible.

### Problema: el modelo no aparece en Open WebUI

Compruebe:

```bash
docker exec open-webui ollama list
docker exec open-webui ollama ps
docker logs --tail 200 open-webui
```

Luego reinicie:

```bash
docker restart open-webui
```

### Problema: Docker Desktop no inicia en Windows

Compruebe:

- Que la virtualización esté habilitada en el firmware del equipo.
- Que WSL 2 esté instalado y actualizado.
- Que Docker Desktop indique que el motor está en ejecución.

### Problema: la respuesta es incorrecta o incoherente

Esto no necesariamente representa una falla de instalación. Los modelos extremadamente pequeños:

- Poseen menor capacidad de razonamiento.
- Pierden instrucciones con mayor facilidad.
- Pueden producir información falsa.
- Tienen dificultades con conversaciones largas.
- No son equivalentes a modelos grandes alojados en la nube.

---

## 13. Consideraciones de seguridad

1. La interfaz se publica solamente en `127.0.0.1`.
2. No publique el puerto en Internet durante este laboratorio.
3. No utilice contraseñas reales, información personal ni datos institucionales sensibles.
4. No asuma que una respuesta generada es correcta.
5. Valide cualquier comando antes de ejecutarlo.
6. Las funciones externas de Open WebUI deben permanecer deshabilitadas salvo indicación del docente.
7. Para una implementación compartida o de producción se requieren controles adicionales: autenticación, TLS, respaldos, gestión de versiones, monitoreo y aislamiento de red.

---

## 14. Entregables

Cada estudiante o pareja debe entregar:

1. Captura de `docker ps` con el contenedor activo.
2. Salida de:

   ```bash
   docker exec open-webui ollama list
   ```

3. Captura de Open WebUI utilizando `qwen2.5:0.5b`.
4. Tabla con las mediciones de CPU, memoria y tiempo.
5. Resultado de la prueba de JSON.
6. Respuesta breve a las preguntas de análisis.

---

## 15. Preguntas de análisis

1. ¿Qué componente ejecuta realmente el modelo: Open WebUI u Ollama?
2. ¿Qué ventajas ofrece ejecutar el modelo localmente?
3. ¿Qué limitaciones se observaron al utilizar solamente CPU?
4. ¿Por qué el tamaño y la cuantización del modelo afectan el consumo de recursos?
5. ¿La privacidad local elimina todos los riesgos de seguridad? Justifique.
6. ¿Qué diferencia existe entre descargar una imagen Docker y descargar un modelo?
7. ¿Por qué se utilizaron volúmenes persistentes?
8. ¿Qué cambios serían necesarios para que varios usuarios accedan al servicio por red?
9. ¿En qué escenarios sería razonable utilizar un modelo tan pequeño?
10. ¿En qué escenarios sería preferible utilizar un modelo alojado en la nube?

---

## 16. Actividad de ampliación

Como ampliación, seleccione una tarea específica, por ejemplo:

- Clasificación de solicitudes de soporte.
- Resumen de mensajes.
- Extracción de campos en JSON.
- Generación de descripciones breves.
- Clasificación de sentimiento.

Diseñe tres instrucciones para esa tarea y evalúe:

- Exactitud.
- Consistencia.
- Tiempo de respuesta.
- Formato de salida.
- Errores o alucinaciones.

Concluya si `qwen2.5:0.5b` sería suficiente para automatizar la tarea o si únicamente sería adecuado para una demostración.

---

## 17. Referencias

- Open WebUI. **Quick Start with Docker**:  
  https://docs.openwebui.com/getting-started/quick-start/

- Open WebUI. **Conexión con Ollama**:  
  https://docs.openwebui.com/getting-started/quick-start/connect-a-provider/starting-with-ollama/

- Ollama. **Qwen2.5 0.5B**:  
  https://ollama.com/library/qwen2.5:0.5b

- Ollama. **Qwen3 0.6B**:  
  https://ollama.com/library/qwen3:0.6b

- Ollama. **SmolLM2**:  
  https://ollama.com/library/smollm2

- Ollama. **Documentación oficial**:  
  https://docs.ollama.com/

---

## 18. Conclusión

La práctica demuestra que es posible ejecutar un modelo generativo completamente local en una computadora sin GPU. Ollama proporciona el motor de inferencia y la administración de modelos, mientras Open WebUI proporciona una interfaz de usuario accesible desde el navegador.

El uso de un modelo pequeño reduce los requisitos de memoria y almacenamiento, pero también limita la precisión, el razonamiento y la calidad de las respuestas. Por esta razón, la selección de un modelo debe considerar el equilibrio entre recursos disponibles, latencia, privacidad y calidad requerida.
