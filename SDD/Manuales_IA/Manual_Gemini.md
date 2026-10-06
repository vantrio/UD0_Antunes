[manual_de_despliegue_de_pila_de_ia_local.md](https://github.com/user-attachments/files/33092934/manual_de_despliegue_de_pila_de_ia_local.md)
---
title: "Manual de Instalación, Configuración y Operación: Pila de IA Local sobre Ubuntu Server"
author: "Administración de Sistemas / DevOps"
date: "2026-09-22"
version: "1.0"
category: "Infraestructura de IA & DevOps"
---

# Manual de Instalación, Configuración y Operación: Pila de IA Local en Docker (Ubuntu Server)

---

## 1. Prerrequisitos e Instalación Base

Esta sección guía paso a paso en la preparación de un servidor **Ubuntu Server 22.04 LTS / 24.04 LTS** con GPU NVIDIA (familia RTX 3050 / 4060 o superior), incluyendo la instalación de Docker, Docker Compose v2 y los controladores NVIDIA con el soporte **NVIDIA Container Toolkit (CUDA)**.

### 1.1 Actualización del Sistema y Paquetes Base
Ejecute como usuario con privilegios `sudo`:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git build-essential gnupg lsb-release ca-certificates software-properties-common
```

### 1.2 Instalación de Drivers NVIDIA y CUDA
1. Identifique y revise el modelo de su GPU:
   ```bash
   lspci | grep -i nvidia
   ```
2. Instale los controladores oficiales de NVIDIA recomendados:
   ```bash
   sudo ubuntu-drivers install
   ```
   *(Alternativa manual para controladores propietarios específicos):*
   ```bash
   sudo apt install -y nvidia-driver-535 nvidia-utils-535
   ```
3. Reinicie el sistema para cargar los módulos del kernel:
   ```bash
   sudo reboot
   ```
4. Verifique la correcta carga del controlador:
   ```bash
   nvidia-smi
   ```

### 1.3 Instalación del Motor Docker y Docker Compose
Instale la versión oficial de Docker Engine desde el repositorio de Docker Inc.

```bash
# Agregar clave GPG oficial de Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Configurar repositorio estable
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$UBUNTU_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalar Docker Engine y Compose plugin
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Agregar usuario actual al grupo docker (para evitar sudo en comandos de docker)
sudo usermod -aG docker $USER
newgrp docker
```

### 1.4 Instalación de NVIDIA Container Toolkit
Permite a los contenedores Docker acceder directamente a la GPU física mediante librerías CUDA.

```bash
# Configuración del repositorio oficial de NVIDIA Container Toolkit
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/experimental/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# Instalación del Toolkit
sudo apt update
sudo apt install -y nvidia-container-toolkit

# Configuración del runtime de Docker para usar GPU
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

### 1.5 Validación de Integración Docker + GPU
Verifique que los contenedores puedan ejecutar comandos en la GPU:

```bash
docker run --rm --gpus all nvidia/cuda:12.0.0-base-ubuntu22.04 nvidia-smi
```
*Si la salida muestra la tabla de la GPU equivalente a `nvidia-smi` en el host, la base está correctamente configurada.*

---

## 2. Estructura del Proyecto

Toda la pila de infraestructura residirá en el directorio `$HOME/proyecto`. Cree la estructura de carpetas necesaria para mapear los volúmenes persistentes y los ficheros de configuración modular.

### 2.1 Comando de Creación de Estructura
```bash
mkdir -p $HOME/proyecto/compose
mkdir -p $HOME/ollama
mkdir -p $HOME/openwebui
mkdir -p $HOME/hermes
mkdir -p $HOME/opencode
mkdir -p $HOME/comfyui
mkdir -p $HOME/yolo
mkdir -p $HOME/searxng
mkdir -p $HOME/rag
```

### 2.2 Árbol de Directorios Final
```text
$HOME/
├── proyecto/
│   ├── .env
│   ├── docker-compose.yml          # Fichero unificado o maestro de inclusión
│   └── compose/
│       ├── docker-ollama.yml
│       ├── docker-openwebui.yml
│       ├── docker-hermes-agent.yml
│       ├── docker-opencode.yml
│       ├── docker-comfyui.yml
│       ├── docker-yolo.yml
│       ├── docker-searxng.yml
│       └── docker-rag.yml
├── ollama/                         # Almacenamiento de modelos LLM
├── openwebui/                      # Datos, chats, usuarios de Open WebUI
├── hermes/                         # Configuración y memoria de Hermes Agent
├── opencode/                       # Proyectos y preferencias de IDE
├── comfyui/                        # Modelos, workflows y outputs generados
├── yolo/                           # Datasets y ejecuciones de visión
├── searxng/                        # Ajustes y caché del metabuscador
└── rag/                            # Documentos fuente y base vectorial RAG
```

---

## 3. Ficheros de Configuración Modular (`docker-<servicio>.yml`)

Navegue al directorio de configuraciones:
```bash
cd $HOME/proyecto/compose
```

### 3.1 `docker-ollama.yml`
```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  ollama:
    container_name: ollama
    image: ollama/ollama:latest
    restart: unless-stopped
    ports:
      - "${OLLAMA_PORT_HOST}:${OLLAMA_PORT_CONTAINER}"
    volumes:
      - ${PATH_OLLAMA_DATA}:/root/.ollama
    networks:
      - red-ia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

### 3.2 `docker-openwebui.yml`
```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  openwebui:
    container_name: openwebui
    image: ghcr.io/open-webui/open-webui:main
    restart: unless-stopped
    ports:
      - "${OPENWEBUI_PORT_HOST}:${OPENWEBUI_PORT_CONTAINER}"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - SEARXNG_QUERY_URL=http://searxng:8080/search?q=<query>
      - WEBUI_SECRET_KEY=${OPENWEBUI_SECRET_KEY}
    volumes:
      - ${PATH_OPENWEBUI_DATA}:/app/backend/data
    networks:
      - red-ia
    depends_on:
      - ollama
```

### 3.3 `docker-hermes-agent.yml`
```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  hermes-agent:
    container_name: hermes-agent
    image: python:3.11-slim
    restart: unless-stopped
    working_dir: /app
    command: >
      sh -c "pip install --no-cache-dir requests fastapi uvicorn &&
             python -c 'import time; print(\"Hermes Agent en ejecución...\"); time.sleep(86400)'"
    ports:
      - "${HERMES_PORT_HOST}:${HERMES_PORT_CONTAINER}"
    environment:
      - OLLAMA_HOST=http://ollama:11434
      - SEARXNG_HOST=http://searxng:8080
      - COMFYUI_HOST=http://comfyui:8188
    volumes:
      - ${PATH_HERMES_DATA}:/app/data
    networks:
      - red-ia
    depends_on:
      - ollama
      - searxng
```

### 3.4 `docker-opencode.yml`
```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  opencode:
    container_name: opencode
    image: lscr.io/linuxserver/code-server:latest
    restart: unless-stopped
    ports:
      - "${OPENCODE_PORT_HOST}:${OPENCODE_PORT_CONTAINER}"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Madrid
      - PASSWORD=${OPENCODE_PASSWORD}
      - DEFAULT_WORKSPACE=/config/workspace
    volumes:
      - ${PATH_OPENCODE_DATA}:/config
    networks:
      - red-ia
```

### 3.5 `docker-comfyui.yml`
```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  comfyui:
    container_name: comfyui
    image: yanwk/comfyui:boot
    restart: unless-stopped
    ports:
      - "${COMFYUI_PORT_HOST}:${COMFYUI_PORT_CONTAINER}"
    environment:
      - CLI_ARGS=--listen 0.0.0.0 --port 8188
    volumes:
      - ${PATH_COMFYUI_DATA}:/root/ComfyUI/output
    networks:
      - red-ia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

### 3.6 `docker-yolo.yml`
```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  yolo:
    container_name: yolo
    image: ultralytics/ultralytics:latest
    restart: unless-stopped
    ports:
      - "${YOLO_PORT_HOST}:${YOLO_PORT_CONTAINER}"
    environment:
      - CLI_ARGS=--host 0.0.0.0 --port 5000
    volumes:
      - ${PATH_YOLO_DATA}:/usr/src/app/datasets
    networks:
      - red-ia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

### 3.7 `docker-searxng.yml`
```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  searxng:
    container_name: searxng
    image: searxng/searxng:latest
    restart: unless-stopped
    ports:
      - "${SEARXNG_PORT_HOST}:${SEARXNG_PORT_CONTAINER}"
    environment:
      - SEARXNG_BASE_URL=http://localhost:8080/
    volumes:
      - ${PATH_SEARXNG_DATA}:/etc/searxng
    networks:
      - red-ia
```

### 3.8 `docker-rag.yml`
```yaml
version: '3.8'

networks:
  red-ia:
    external: true

services:
  rag:
    container_name: rag
    image: chromadb/chroma:latest
    restart: unless-stopped
    ports:
      - "${RAG_PORT_HOST}:${RAG_PORT_CONTAINER}"
    environment:
      - IS_PERSISTENT=TRUE
      - ANONYMIZED_TELEMETRY=FALSE
    volumes:
      - ${PATH_RAG_DATA}:/chroma/chroma
    networks:
      - red-ia
    depends_on:
      - ollama
```

---

## 4. Fichero de Entorno (`.env`)

Cree el archivo `$HOME/proyecto/.env` para parametrizar puertos, claves secretas y rutas en un solo punto centralizado:

```env
# ==========================================
# RUTAS DE VOLÚMENES PERSISTENTES
# ==========================================
PATH_OLLAMA_DATA=/home/usuario/ollama
PATH_OPENWEBUI_DATA=/home/usuario/openwebui
PATH_HERMES_DATA=/home/usuario/hermes
PATH_OPENCODE_DATA=/home/usuario/opencode
PATH_COMFYUI_DATA=/home/usuario/comfyui
PATH_YOLO_DATA=/home/usuario/yolo
PATH_SEARXNG_DATA=/home/usuario/searxng
PATH_RAG_DATA=/home/usuario/rag

# ==========================================
# MAPEADO DE PUERTOS (HOST:CONTAINER)
# ==========================================
# Ollama
OLLAMA_PORT_HOST=11434
OLLAMA_PORT_CONTAINER=11434

# Open WebUI
OPENWEBUI_PORT_HOST=3000
OPENWEBUI_PORT_CONTAINER=8080

# Hermes Agent
HERMES_PORT_HOST=8000
HERMES_PORT_CONTAINER=8000

# OpenCode IDE
OPENCODE_PORT_HOST=8443
OPENCODE_PORT_CONTAINER=8080

# ComfyUI
COMFYUI_PORT_HOST=8188
COMFYUI_PORT_CONTAINER=8188

# YOLO
YOLO_PORT_HOST=5000
YOLO_PORT_CONTAINER=5000

# SearXNG
SEARXNG_PORT_HOST=8080
SEARXNG_PORT_CONTAINER=8080

# RAG / ChromaDB
RAG_PORT_HOST=8001
RAG_PORT_CONTAINER=8000

# ==========================================
# SEGURIDAD Y CONFIGURACIONES VARIAS
# ==========================================
OPENWEBUI_SECRET_KEY=ClaveSecretaSuperSeguraIA2026
OPENCODE_PASSWORD=PasswordSysadmin2026!
```

> **Nota:** Reemplace `/home/usuario` por la ruta real absoluta de su usuario `$HOME` en el servidor (ejemplo: `/home/vantrio`).

---

## 5. Despliegue y Verificación

### 5.1 Creación de la Red Docker
Para asegurar la comunicación inter-contenedor mediante resolución DNS interna (`http://ollama:11434`, etc.), cree la red bridge previa al despliegue:

```bash
docker network create red-ia
```

### 5.2 Fichero Unificado Maestro (`$HOME/proyecto/docker-compose.yml`)
Para levantar todos los servicios con un solo comando, cree un archivo maestro que integre cada módulo:

```yaml
version: '3.8'

include:
  - compose/docker-ollama.yml
  - compose/docker-openwebui.yml
  - compose/docker-hermes-agent.yml
  - compose/docker-opencode.yml
  - compose/docker-comfyui.yml
  - compose/docker-yolo.yml
  - compose/docker-searxng.yml
  - compose/docker-rag.yml
```

### 5.3 Comandos de Arranque del Stack
Desde la carpeta `$HOME/proyecto`:

```bash
cd $HOME/proyecto

# Descargar imágenes y arrancar todos los servicios en segundo plano
docker compose up -d
```

### 5.4 Verificación del Estado de Servicios
1. **Comprobar contenedores activos:**
   ```bash
   docker compose ps
   ```
2. **Revisión de logs en tiempo real:**
   ```bash
   docker compose logs -f
   # O por servicio específico:
   docker logs -f ollama
   ```
3. **Verificación del uso de GPU en contenedores:**
   ```bash
   docker exec -it ollama nvidia-smi
   docker exec -it comfyui nvidia-smi
   ```

### 5.5 Descarga Inicial de Modelos en Ollama
Para comenzar a operar, descargue un modelo LLM ligero (ej. `llama3` o `mistral`):

```bash
docker exec -it ollama ollama run llama3
```

### 5.6 Tabla de Direccionamiento de Servicios
Acceda a los servicios desde el navegador web de su red local utilizando la IP del servidor Ubuntu:

| Servicio | Nombre Contenedor | URL Externa (Host) | URL Interna (DNS Docker) |
| :--- | :--- | :--- | :--- |
| **Ollama API** | `ollama` | `http://<IP-SERVIDOR>:11434` | `http://ollama:11434` |
| **Open WebUI** | `openwebui` | `http://<IP-SERVIDOR>:3000` | `http://openwebui:8080` |
| **Hermes Agent** | `hermes-agent` | `http://<IP-SERVIDOR>:8000` | `http://hermes-agent:8000` |
| **OpenCode** | `opencode` | `http://<IP-SERVIDOR>:8443` | `http://opencode:8080` |
| **ComfyUI** | `comfyui` | `http://<IP-SERVIDOR>:8188` | `http://comfyui:8188` |
| **YOLO Engine** | `yolo` | `http://<IP-SERVIDOR>:5000` | `http://yolo:5000` |
| **SearXNG** | `searxng` | `http://<IP-SERVIDOR>:8080` | `http://searxng:8080` |
| **RAG Vector DB**| `rag` | `http://<IP-SERVIDOR>:8001` | `http://rag:8000` |

---

## 6. Mantenimiento y Actualización

### 6.1 Copias de Seguridad (Backups)
Para realizar una copia de seguridad completa del estado de la IA (chats, código, modelos, workflows y bases vectoriales):

1. **Detener el stack:**
   ```bash
   cd $HOME/proyecto
   docker compose down
   ```
2. **Crear archivo comprimido de volúmenes:**
   ```bash
   tar -czvf $HOME/backup_pila_ia_$(date +%Y%m%d).tar.gz \
     $HOME/ollama \
     $HOME/openwebui \
     $HOME/hermes \
     $HOME/opencode \
     $HOME/comfyui \
     $HOME/yolo \
     $HOME/searxng \
     $HOME/rag
   ```
3. **Reacondicionar servicios:**
   ```bash
   docker compose up -d
   ```

### 6.2 Actualización de Imágenes y Servicios
Para actualizar las aplicaciones a sus últimas versiones de software:

```bash
cd $HOME/proyecto
docker compose pull
docker compose up -d --remove-orphans
# Limpiar imágenes antiguas o en desuso
docker image prune -f
```

### 6.3 Resolució́n de Problemas Frecuentes

#### A. Error de Permisos en Volúmenes (`Permission Denied`)
Si un contenedor falla al escribir en las carpetas montadas en `$HOME/`:
```bash
sudo chown -R $USER:$USER $HOME/ollama $HOME/openwebui $HOME/hermes $HOME/opencode $HOME/comfyui $HOME/yolo $HOME/searxng $HOME/rag
sudo chmod -R 775 $HOME/ollama $HOME/openwebui $HOME/hermes $HOME/opencode $HOME/comfyui $HOME/yolo $HOME/searxng $HOME/rag
```

#### B. La GPU no es detectada dentro del contenedor
Si al ejecutar `docker exec -it ollama nvidia-smi` arroja un error:
1. Verifique que el servicio `nvidia-persistenced` o los módulos estén activos en el servidor:
   ```bash
   nvidia-smi
   ```
2. Reinicie el demonio de Docker para recargar el runtime de NVIDIA:
   ```bash
   sudo systemctl restart docker
   ```

---

## 7. Guía de Integración Interna

Los servicios interconectados dentro de la red `red-ia` deben comunicarse utilizando los nombres de contenedor como nombres de dominio (DNS de Docker).

```text
                     ┌──────────────────┐
                     │   Open WebUI     │ (Puerto 3000)
                     └────────┬─────────┘
                              │
             ┌────────────────┼────────────────┐
             ▼                ▼                ▼
     ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
     │    Ollama    │ │   SearXNG    │ │  RAG DB      │
     │ (Port 11434) │ │ (Port 8080)  │ │ (ChromaDB)   │
     └───────┬──────┘ └──────────────┘ └──────────────┘
             │
             ├────────────────┐
             ▼                ▼
     ┌──────────────┐ ┌──────────────┐
     │  ComfyUI     │ │ Hermes Agent │
     │ (Port 8188)  │ │ (Port 8000)  │
     └──────────────┘ └──────────────┘
```

### 7.1 Conectar Ollama con Open WebUI
Open WebUI detecta automáticamente a Ollama mediante la variable de entorno configurada en el `docker-openwebui.yml`:
`OLLAMA_BASE_URL=http://ollama:11434`

* **Verificación:** Inicie sesión en Open WebUI (`http://<IP-SERVIDOR>:3000`), vaya a **Admin Panel > Settings > Connections** y compruebe que la dirección esté fijada en `http://ollama:11434`. Al hacer clic en el botón de verificación, se desplegarán los modelos descargados en Ollama.

### 7.2 Integrar SearXNG con Open WebUI (Búsqueda Web en Vivo)
Permite a los modelos LLM realizar búsquedas en tiempo real en Internet a través de SearXNG sin rastreo.

1. Acceda a Open WebUI como Administrador.
2. Navegue a **Admin Panel > Settings > Web Search**.
3. Active la opción **Enable Web Search**.
4. Seleccione como motor de búsqueda (**Search Engine**): `searxng`.
5. En **SearXNG Query URL**, introduzca:
   `http://searxng:8080/search?q=<query>`
6. Guarde los cambios. En cualquier chat, al activar el icono de "Web Search", la IA consultará SearXNG antes de responder.

### 7.3 Interconexión de Hermes Agent con Ollama, SearXNG y ComfyUI
Hermes Agent actúa como un orquestador para ejecutar tareas complejas en la red local. Sus variables de entorno apuntan internamente a:
* **LLM Engine:** `http://ollama:11434`
* **Navegación / Tool Search:** `http://searxng:8080`
* **Generación de Imagen:** `http://comfyui:8188`

En los scripts en Python o workflows dentro del volumen `$HOME/hermes`, las peticiones API deben realizarse usando los endpoints DNS citados (por ejemplo, realizando POST a `http://comfyui:8188/prompt` para lanzar flujos de difusión).

### 7.4 Configuración de OpenCode (IDE Web) con Asistente LLM Local
Para programar desde la interfaz web de OpenCode (`http://<IP-SERVIDOR>:8443`) con ayuda del motor Ollama local:

1. Abra la interfaz web de OpenCode e inicie sesión con el `OPENCODE_PASSWORD`.
2. Abra el gestor de extensiones (Ctrl+Shift+X).
3. Busque e instale una extensión como **Continue** o **Codeium**.
4. En la configuración del plugin (ej. `config.json` de Continue), configure el proveedor local:
   ```json
   {
     "models": [
       {
         "title": "Ollama Local Code",
         "provider": "ollama",
         "model": "codellama",
         "apiBase": "http://ollama:11434"
       }
     ]
   }
   ```

### 7.5 Integración del Módulo RAG (ChromaDB + Ollama)
El servicio RAG amplía las capacidades de respuesta con sus propios documentos privados localizados en `$HOME/rag`.

1. **Embeddings:** Open WebUI o Hermes Agent envían los documentos cargados a ChromaDB (`http://rag:8000`) para su vectorización.
2. **Generación de respuestas:** Durante una consulta con RAG activo, el sistema consulta los vectores en `http://rag:8000`, extrae los contextos relevantes y los inyecta en el prompt enviado a `http://ollama:11434`.

---

## 8. Resumen de Comandos de Operación Rápida

```bash
# Iniciar todo el stack
cd $HOME/proyecto && docker compose up -d

# Detener el stack
cd $HOME/proyecto && docker compose down

# Ver logs globales
cd $HOME/proyecto && docker compose logs -f

# Monitorear consumo de GPU
watch -n 1 nvidia-smi
```
