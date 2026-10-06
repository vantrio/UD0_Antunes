[manual_ia_local_docker.md](https://github.com/user-attachments/files/33092396/manual_ia_local_docker.md)
---
title: "Manual de instalación, configuración y operación: pila de IA local en Docker sobre Ubuntu Server"
version: "1.0"
date: "2026-10-06"
category: "Administración de sistemas / IA local"
tags: [docker, ubuntu, nvidia, ollama, open-webui, hermes-agent, opencode, comfyui, yolo, searxng, rag]
---

# Manual de instalación, configuración y operación
## Pila de IA local con Docker sobre Ubuntu Server (GPU NVIDIA)

- **Versión:** 1.0
- **Fecha:** 2026-10-06
- **Destinatario:** administrador de sistemas novato
- **Basado en:** especificación SDD `proyecto_ia.md` (v1.0)
- **Sistema de referencia:** Ubuntu Server 24.04 LTS o 26.04 LTS, GPU NVIDIA GeForce RTX 4060

---

## Índice

0. [Antes de empezar](#0-antes-de-empezar)
1. [Prerrequisitos e instalación base](#1-prerrequisitos-e-instalación-base)
2. [Estructura del proyecto](#2-estructura-del-proyecto)
3. [Ficheros de configuración (`docker-<servicio>.yml`)](#3-ficheros-de-configuración)
4. [Fichero de entorno `.env`](#4-fichero-de-entorno-env)
5. [Despliegue y verificación](#5-despliegue-y-verificación)
6. [Mantenimiento y actualización](#6-mantenimiento-y-actualización)
7. [Guía interna de integración entre servicios](#7-guía-interna-de-integración-entre-servicios)
8. [Comprobación de los criterios de aceptación](#8-comprobación-de-los-criterios-de-aceptación)
9. [Anexo: chuleta de comandos](#9-anexo-chuleta-de-comandos)

---

## 0. Antes de empezar

### 0.1 Qué vas a montar

Ocho servicios, cada uno en su propio contenedor Docker, conectados por una red privada llamada `red-ia`. Solo tres de ellos usan la GPU (Ollama, ComfyUI y YOLO); el resto son aplicaciones web o servicios de apoyo.

```text
                    ┌──────────────────────── red-ia (bridge) ────────────────────────┐
 Navegador          │                                                                  │
 ──► :3000 ───────► │  openwebui ──► ollama:11434   (LLMs en GPU)                      │
                    │      │  ├────► hermes-agent:8000  (agente, API OpenAI)           │
                    │      │  ├────► searxng:8080       (búsqueda web)                 │
                    │      │  ├────► comfyui:8188       (imágenes/vídeo en GPU)        │
                    │      │  └────► rag:6333           (base vectorial Qdrant)        │
                    │                                                                  │
 ──► :8000 ───────► │  hermes-agent ─► ollama, searxng, comfyui                        │
 ──► :8443 ───────► │  opencode ─────► ollama, searxng (vía MCP)                       │
 ──► :5000 ───────► │  yolo (API de visión artificial en GPU)                          │
                    └──────────────────────────────────────────────────────────────────┘
```

### 0.2 Convenciones del manual

- Los comandos se ejecutan en una terminal del servidor (por SSH o en consola) con **tu usuario normal**. Cuando haga falta privilegios verás `sudo` delante.
- `~` y `$HOME` significan tu carpeta personal (por ejemplo `/home/pablo`). El proyecto vive en `~/proyecto`.
- Para crear ficheros se usa el editor `nano`: escribe `nano nombre_fichero`, pega el contenido, guarda con `Ctrl+O` + `Intro` y sal con `Ctrl+X`.
- Las líneas que empiezan por `#` dentro de un bloque de comandos son comentarios: no hace falta escribirlas.
- Donde veas `TU_IP` sustituye por la IP del servidor (`hostname -I`). Si abres el navegador en el propio servidor, usa `localhost`.

### 0.3 Ajustes realizados respecto a la especificación

Al convertir la especificación en un sistema que funcione he tenido que resolver varias incoherencias. Están aquí para que sepas qué se cambió y por qué.

| # | Punto de la especificación | Problema | Solución adoptada |
| :-: | :--- | :--- | :--- |
| 1 | **RAG** con puerto `11434` | Es el mismo puerto que usa Ollama: colisión en el host (incumple el criterio 6.3). | El servicio `rag` es una base de datos vectorial **Qdrant** y usa el puerto **6333**. Open WebUI la usa como almacén de vectores. |
| 2 | URLs internas de la sección 3.1 (`openwebui:3000`, `opencode:8443`, `hermesagent:8000`) | Entre contenedores se usa el **nombre del contenedor** y el puerto **interno**, no el externo. Además, `hermesagent` no coincide con el nombre de contenedor `hermes-agent`. | URLs corregidas: `http://openwebui:8080`, `http://opencode:8080`, `http://hermes-agent:8000`. |
| 3 | Ficheros `docker-<servicio>.yml` (sección 5) frente a `docker_<servicio>.yml` (criterio 6.1) | Dos nombres distintos para lo mismo. | Se usa el guion: `docker-<servicio>.yml`. |
| 4 | Criterio 6.2: «todos los contenedores usan GPU» | SearXNG, Open WebUI, Hermes Agent, OpenCode y Qdrant no ejecutan cargas CUDA; darles GPU solo gastaría VRAM. | Ollama, ComfyUI y YOLO reservan la GPU. Los demás no. La dependencia «GPU driver» de SearXNG se elimina. |
| 5 | Hermes Agent en el puerto `8000` | La API de Hermes escucha por defecto en `8642`. | Se fuerza `API_SERVER_PORT=8000`. |
| 6 | OpenCode en el puerto externo `8443` | `8443` suele asociarse a HTTPS, pero OpenCode sirve HTTP. | Se mantiene `8443`, pero se accede con `http://`. |
| 7 | ComfyUI y YOLO | No hay una imagen lista que cumpla lo pedido. | `build/comfyui` y `build/yolo` contienen sus propios `Dockerfile`. YOLO incluye una pequeña API en FastAPI. |
| 8 | Volúmenes (sección 3.2) | La especificación pide «mapear» carpetas del host. | Se usan *bind mounts* en `$HOME/<servicio>`, tal como indica la tabla. |
| 9 | Dependencias «ConfyUI» | Errata. | Se entiende **ComfyUI**. |

### 0.4 Requisitos orientativos de hardware

| Recurso | Mínimo razonable | Comentario |
| :--- | :--- | :--- |
| GPU | NVIDIA con 8 GB de VRAM (RTX 4060) | Los tres servicios con GPU **comparten** esos 8 GB. No los exprimas a la vez. |
| RAM | 16 GB | 32 GB si vas a usar Hermes Agent con contextos grandes. |
| Disco | 150 GB libres | Las imágenes ocupan decenas de GB y cada modelo de 4 a 10 GB. |
| Red | Acceso a Internet | Para descargar paquetes, imágenes y modelos. |

---

## 1. Prerrequisitos e instalación base

> Los pasos siguientes sirven para Ubuntu Server **24.04 LTS** y **26.04 LTS**. Los repositorios de Docker publican paquetes para ambas versiones.

### 1.1 Comprobaciones iniciales

```bash
lsb_release -a              # versión de Ubuntu (debe ser 24.04 o 26.04)
uname -r                    # versión del kernel
lspci | grep -i nvidia      # el sistema debe "ver" la tarjeta gráfica
free -h                     # memoria RAM
df -h /                     # espacio libre en disco
```

Si `lspci` no muestra ninguna línea con NVIDIA, la GPU no está detectada: revisa BIOS/hardware antes de seguir.

### 1.2 Actualizar el sistema e instalar utilidades

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg ubuntu-drivers-common openssl jq
sudo reboot
```

- `apt update` descarga la lista de paquetes disponibles; `apt upgrade` instala las actualizaciones.
- `jq` sirve para leer respuestas JSON en los tests y `openssl` para generar contraseñas aleatorias.
- Tras el `reboot`, vuelve a conectarte.

### 1.3 Instalar el controlador (driver) de NVIDIA

```bash
ubuntu-drivers devices          # muestra el driver recomendado para tu tarjeta
sudo ubuntu-drivers install     # instala el recomendado
sudo reboot
```

Después de reiniciar, comprueba que el driver funciona:

```bash
nvidia-smi
```

Debe aparecer una tabla con el nombre de la tarjeta (por ejemplo *NVIDIA GeForce RTX 4060*), la versión del driver y la VRAM. Si da error, no avances: sin driver no hay GPU en los contenedores.

> **Si el equipo tiene Secure Boot activado** (`mokutil --sb-state`), durante la instalación te pedirá crear una contraseña de «MOK». En el siguiente arranque aparecerá una pantalla azul (*MOK management*): elige **Enroll MOK** e introduce esa contraseña. Si te la saltas, el driver no se cargará.

**Sobre CUDA.** Las imágenes de Ollama, ComfyUI y YOLO traen sus propias librerías CUDA. En el servidor solo necesitas el **driver**. Si además quieres el compilador `nvcc` en el host (opcional, para pruebas):

```bash
sudo apt install -y nvidia-cuda-toolkit
nvcc --version
```

### 1.4 Instalar Docker Engine y Docker Compose (repositorio oficial)

**Paso 1. Eliminar paquetes que podrían entrar en conflicto** (si no tienes ninguno, no hará nada):

```bash
sudo apt remove -y $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc 2>/dev/null | cut -f1)
```

**Paso 2. Añadir la clave y el repositorio de Docker:**

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

El bloque `Suites:` detecta solo el nombre de tu versión (`noble` en 24.04, `resolute` en 26.04).

**Paso 3. Instalar Docker y Compose:**

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

**Paso 4. Comprobar la instalación:**

```bash
sudo docker run --rm hello-world
docker compose version
```

Si ves el mensaje «Hello from Docker!» y una versión de Compose v2 (o superior), está correcto.

### 1.5 Usar Docker sin `sudo`

```bash
sudo usermod -aG docker $USER
```

Cierra la sesión SSH y vuelve a entrar (o ejecuta `newgrp docker`). Comprueba con:

```bash
docker ps
```

> **Seguridad:** pertenecer al grupo `docker` equivale a tener permisos de administrador sobre el servidor. Añade solo a usuarios de confianza.

### 1.6 Instalar el NVIDIA Container Toolkit (GPU dentro de Docker)

Este componente permite que los contenedores usen la GPU.

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor --yes -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit

# Registrar el runtime de NVIDIA en Docker y reiniciar
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

**Prueba final de la GPU en Docker:**

```bash
docker run --rm --gpus all ubuntu nvidia-smi
```

Debe mostrar la misma tabla que `nvidia-smi` en el host. Si aparece, la base está lista.

### 1.7 Nota sobre el cortafuegos

Docker publica los puertos saltándose las reglas de `ufw`: un puerto publicado como `0.0.0.0:3000` queda accesible desde toda la red aunque `ufw` diga lo contrario. Por eso este proyecto usa la variable `BIND_IP` (sección 4):

- `BIND_IP=0.0.0.0` → accesible desde otros equipos de la red (valor por defecto).
- `BIND_IP=127.0.0.1` → solo accesible desde el propio servidor (por ejemplo, con túnel SSH).

**No expongas estos servicios directamente a Internet.** Hermes Agent y OpenCode pueden ejecutar comandos y modificar archivos.

---
## 2. Estructura del proyecto

### 2.1 Árbol de directorios

El proyecto se divide en dos zonas:

- **`~/proyecto`**: la *configuración* del stack (ficheros `.yml`, `.env`, Dockerfiles, scripts). Es pequeña y se puede versionar con Git.
- **`~/<servicio>`**: los *datos persistentes* de cada servicio, tal como define la especificación. Así puedes borrar y recrear los contenedores sin perder modelos, chats ni configuraciones.

```text
$HOME/
├── proyecto/                           ← configuración del stack
│   ├── .env                            ← variables de todos los servicios
│   ├── docker-ollama.yml
│   ├── docker-openwebui.yml
│   ├── docker-hermes-agent.yml
│   ├── docker-opencode.yml
│   ├── docker-comfyui.yml
│   ├── docker-yolo.yml
│   ├── docker-searxng.yml
│   ├── docker-rag.yml
│   ├── build/                          ← imágenes que construimos nosotros
│   │   ├── comfyui/Dockerfile
│   │   ├── yolo/
│   │   │   ├── Dockerfile
│   │   │   └── app.py                  ← API REST de YOLO (FastAPI)
│   │   └── opencode/Dockerfile
│   ├── scripts/
│   │   ├── check.sh                    ← comprobación de todos los servicios
│   │   ├── backup.sh                   ← copia de seguridad
│   │   └── update.sh                   ← actualización
│   └── backups/                        ← copias generadas por backup.sh
│
├── ollama/                             ← modelos LLM descargados
├── openwebui/                          ← usuarios, chats, prompts, configuración
├── hermes/                             ← config.yaml, .env y datos de los agentes
├── opencode/
│   ├── config/                         ← opencode.json
│   ├── data/                           ← sesiones y credenciales
│   └── projects/                       ← código de tus proyectos
├── comfyui/
│   ├── models/                         ← checkpoints, loras, vae... (modelos generativos)
│   ├── input/                          ← imágenes de entrada
│   ├── output/                         ← imágenes y vídeos generados
│   ├── user/                           ← workflows (prompts) y ajustes de la interfaz
│   └── custom_nodes/                   ← extensiones
├── yolo/
│   ├── models/                         ← pesos (.pt) de YOLO
│   └── config/                         ← configuración de Ultralytics
├── searxng/
│   ├── config/                         ← settings.yml
│   └── cache/                          ← caché de búsquedas
└── rag/
    ├── storage/                        ← base vectorial Qdrant
    ├── snapshots/                      ← instantáneas de Qdrant (backups)
    └── docs/                           ← documentos fuente que quieras aportar al RAG
```

### 2.2 Crear los directorios

```bash
mkdir -p ~/proyecto/{build/{comfyui,yolo,opencode},scripts,backups}
mkdir -p ~/ollama ~/openwebui ~/hermes
mkdir -p ~/opencode/{config,data,projects}
mkdir -p ~/comfyui/{input,output,user,custom_nodes}
mkdir -p ~/comfyui/models/{checkpoints,clip,clip_vision,configs,controlnet,diffusers,diffusion_models,embeddings,gligen,hypernetworks,loras,style_models,text_encoders,unet,upscale_models,vae,vae_approx}
mkdir -p ~/yolo/{models,config}
mkdir -p ~/searxng/{config,cache}
mkdir -p ~/rag/{storage,snapshots,docs}
cd ~/proyecto
```

> Las subcarpetas de `comfyui/models` hay que crearlas porque, al montar una carpeta vacía del host sobre `/app/models`, ComfyUI no encontraría los directorios donde busca los modelos.

### 2.3 Puertos (sin colisiones)

| Servicio | Contenedor | Puerto interno | Puerto en el host | Variable de `.env` |
| :--- | :--- | :-: | :-: | :--- |
| Ollama | `ollama` | 11434 | **11434** | `OLLAMA_PORT` |
| Open WebUI | `openwebui` | 8080 | **3000** | `OPENWEBUI_PORT` |
| Hermes Agent | `hermes-agent` | 8000 | **8000** | `HERMES_PORT` |
| OpenCode | `opencode` | 8080 | **8443** | `OPENCODE_PORT` |
| ComfyUI | `comfyui` | 8188 | **8188** | `COMFYUI_PORT` |
| YOLO | `yolo` | 5000 | **5000** | `YOLO_PORT` |
| SearXNG | `searxng` | 8080 | **8080** | `SEARXNG_PORT` |
| RAG (Qdrant) | `rag` | 6333 | **6333** (solo `127.0.0.1`) | `RAG_PORT` |

Los puertos **del host** son todos distintos (11434, 3000, 8000, 8443, 8188, 5000, 8080, 6333). Que Open WebUI, OpenCode y SearXNG usen el 8080 *dentro* del contenedor no es un problema: cada contenedor tiene su propio espacio de red.

Antes de desplegar, comprueba que ningún otro programa del servidor usa esos puertos (no debe mostrar nada):

```bash
sudo ss -tulpn | grep -E ':(11434|3000|8000|8443|8188|5000|8080|6333)\b'
```

### 2.4 Red `red-ia` y nombres DNS internos

Todos los contenedores se conectan a la red Docker `red-ia` (tipo *bridge*). En una red definida por el usuario, Docker incorpora un DNS interno: cada contenedor se localiza por su **nombre de contenedor** y se accede a su **puerto interno**.

| Servicio | Nombre de contenedor | URL interna (desde otro contenedor) |
| :--- | :--- | :--- |
| Ollama | `ollama` | `http://ollama:11434` |
| Open WebUI | `openwebui` | `http://openwebui:8080` |
| Hermes Agent | `hermes-agent` | `http://hermes-agent:8000` |
| OpenCode | `opencode` | `http://opencode:8080` |
| ComfyUI | `comfyui` | `http://comfyui:8188` |
| YOLO | `yolo` | `http://yolo:5000` |
| SearXNG | `searxng` | `http://searxng:8080` |
| RAG (Qdrant) | `rag` | `http://rag:6333` (la usa Open WebUI) |

Estas URLs **solo funcionan dentro de la red `red-ia`**. Desde tu navegador usarás `http://TU_IP:PUERTO_DEL_HOST`.

### 2.5 Volúmenes de datos (carpetas del host montadas en los contenedores)

| Servicio | Carpeta en el host | Ruta en el contenedor | Contenido |
| :--- | :--- | :--- | :--- |
| Ollama | `~/ollama` | `/root/.ollama` | Modelos LLM |
| Open WebUI | `~/openwebui` | `/app/backend/data` | Usuarios, chats, prompts, configuración |
| Hermes Agent | `~/hermes` | `/opt/data` | Configuración y datos de los agentes |
| OpenCode | `~/opencode/config` | `/root/.config/opencode` | `opencode.json` |
| OpenCode | `~/opencode/data` | `/root/.local/share/opencode` | Sesiones y credenciales |
| OpenCode | `~/opencode/projects` | `/workspace` | Código de los proyectos |
| ComfyUI | `~/comfyui/{models,input,output,user,custom_nodes}` | `/app/{models,input,output,user,custom_nodes}` | Modelos, entradas, resultados, workflows, extensiones |
| YOLO | `~/yolo` | `/data` | Pesos, configuración y datasets |
| SearXNG | `~/searxng/config` | `/etc/searxng` | `settings.yml` |
| SearXNG | `~/searxng/cache` | `/var/cache/searxng` | Caché de resultados |
| RAG (Qdrant) | `~/rag/storage` | `/qdrant/storage` | Vectores indexados |
| RAG (Qdrant) | `~/rag/snapshots` | `/qdrant/snapshots` | Instantáneas |

---

## 3. Ficheros de configuración

### 3.0 Reglas comunes y creación de la red

Todos los ficheros `docker-<servicio>.yml` siguen las mismas reglas:

- Son **autónomos**: cada uno define su servicio y declara la red `red-ia` como *externa* (existe fuera de los ficheros).
- Se **fusionan** al arrancar gracias a la variable `COMPOSE_FILE` del fichero `.env` (sección 4), de modo que `docker compose up -d` levanta todo el stack. Cada fichero también puede usarse solo: `docker compose -f docker-ollama.yml up -d`.
- Usan variables de `.env` con valor por defecto (`${VARIABLE:-valor}`), por lo que el stack arranca aunque falte alguna.
- `restart: unless-stopped` hace que los servicios arranquen solos tras reiniciar el servidor.
- `${HOME}` se sustituye por tu carpeta personal. **Ejecuta Docker con tu usuario, no con `sudo`**, o `${HOME}` apuntaría a `/root`.
- Los servicios con GPU incluyen este bloque, que reserva la tarjeta NVIDIA:

```yaml
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: all
          capabilities: [gpu]
```

**Crear la red `red-ia`** (una sola vez):

```bash
docker network create --driver bridge red-ia
docker network ls | grep red-ia
```

### 3.1 `docker-ollama.yml`: motor de LLMs (GPU)

Ollama descarga y ejecuta los modelos de lenguaje y los ofrece mediante una API HTTP.

Crea el fichero: `nano ~/proyecto/docker-ollama.yml`

```yaml
# docker-ollama.yml - Motor de LLMs locales y servidor de API (GPU NVIDIA)
services:
  ollama:
    image: ollama/ollama:${OLLAMA_TAG:-latest}
    container_name: ollama
    restart: unless-stopped
    ports:
      - "${BIND_IP:-0.0.0.0}:${OLLAMA_PORT:-11434}:11434"
    environment:
      TZ: ${TZ:-Europe/Madrid}
      OLLAMA_HOST: "0.0.0.0:11434"
      OLLAMA_KEEP_ALIVE: ${OLLAMA_KEEP_ALIVE:-5m}
      OLLAMA_CONTEXT_LENGTH: ${OLLAMA_CONTEXT_LENGTH:-8192}
      OLLAMA_NUM_PARALLEL: ${OLLAMA_NUM_PARALLEL:-1}
      OLLAMA_MAX_LOADED_MODELS: ${OLLAMA_MAX_LOADED_MODELS:-1}
      OLLAMA_FLASH_ATTENTION: ${OLLAMA_FLASH_ATTENTION:-1}
      OLLAMA_KV_CACHE_TYPE: ${OLLAMA_KV_CACHE_TYPE:-q8_0}
      NVIDIA_VISIBLE_DEVICES: all
      NVIDIA_DRIVER_CAPABILITIES: compute,utility
    volumes:
      - ${HOME}/ollama:/root/.ollama
    networks:
      - red-ia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    healthcheck:
      test: ["CMD", "ollama", "list"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 20s

networks:
  red-ia:
    external: true
    name: red-ia
```

Qué hace cada parte importante:

- `OLLAMA_KEEP_ALIVE`: cuánto tiempo mantiene un modelo cargado en VRAM tras la última petición. Con 5 minutos la VRAM se libera sola para ComfyUI o YOLO.
- `OLLAMA_CONTEXT_LENGTH`: tamaño de contexto por defecto (cuánto «recuerda» el modelo en una conversación). A mayor contexto, más VRAM.
- `OLLAMA_NUM_PARALLEL=1` y `OLLAMA_MAX_LOADED_MODELS=1`: una petición y un modelo a la vez, para no agotar los 8 GB.
- `OLLAMA_FLASH_ATTENTION` y `OLLAMA_KV_CACHE_TYPE=q8_0`: reducen el consumo de memoria del contexto.

### 3.2 `docker-openwebui.yml`: interfaz web tipo ChatGPT

Open WebUI es la puerta de entrada para los usuarios: chatea con Ollama y Hermes, busca en Internet con SearXNG, genera imágenes con ComfyUI y consulta documentos mediante RAG.

Crea el fichero: `nano ~/proyecto/docker-openwebui.yml`

```yaml
# docker-openwebui.yml - Interfaz web para LLMs (Ollama, Hermes, SearXNG, ComfyUI, RAG)
services:
  openwebui:
    image: ghcr.io/open-webui/open-webui:${OPENWEBUI_TAG:-main}
    container_name: openwebui
    restart: unless-stopped
    ports:
      - "${BIND_IP:-0.0.0.0}:${OPENWEBUI_PORT:-3000}:8080"
    environment:
      TZ: ${TZ:-Europe/Madrid}
      WEBUI_NAME: ${WEBUI_NAME:-IA Local}
      WEBUI_SECRET_KEY: ${WEBUI_SECRET_KEY}
      # --- Ollama (LLMs locales)
      OLLAMA_BASE_URL: http://ollama:11434
      # --- Hermes Agent (API compatible con OpenAI)
      OPENAI_API_BASE_URL: http://hermes-agent:8000/v1
      OPENAI_API_KEY: ${HERMES_API_KEY}
      # --- SearXNG (búsqueda web)
      ENABLE_WEB_SEARCH: "true"
      WEB_SEARCH_ENGINE: searxng
      SEARXNG_QUERY_URL: "http://searxng:8080/search?q=<query>"
      # --- ComfyUI (generación de imágenes)
      ENABLE_IMAGE_GENERATION: "true"
      IMAGE_GENERATION_ENGINE: comfyui
      COMFYUI_BASE_URL: http://comfyui:8188
      # --- RAG: vectores en Qdrant, embeddings con Ollama
      VECTOR_DB: qdrant
      QDRANT_URI: http://rag:6333
      RAG_EMBEDDING_ENGINE: ollama
      RAG_EMBEDDING_MODEL: ${RAG_EMBEDDING_MODEL:-nomic-embed-text}
      RAG_OLLAMA_BASE_URL: http://ollama:11434
    volumes:
      - ${HOME}/openwebui:/app/backend/data
    networks:
      - red-ia
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://localhost:8080/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 90s

networks:
  red-ia:
    external: true
    name: red-ia
```

Notas:

- Dentro del contenedor, Open WebUI escucha en el `8080`; el `3000` es el puerto del host.
- `WEBUI_SECRET_KEY` firma las sesiones. Se genera en la sección 4.
- **Importante:** casi todas estas variables son *configuración persistente*: Open WebUI solo las lee **la primera vez** que arranca y después las guarda en su base de datos. Si cambias una variable más tarde y no surte efecto, modifícala en *Panel de administración → Ajustes* o añade `ENABLE_PERSISTENT_CONFIG: "false"` al servicio.

### 3.3 `docker-hermes-agent.yml`: agente autónomo

Hermes Agent es el agente de Nous Research. Usa Ollama como «cerebro», SearXNG para buscar y ComfyUI para generar imágenes. Se ejecuta en modo *gateway* y expone una API compatible con OpenAI.

Crea el fichero: `nano ~/proyecto/docker-hermes-agent.yml`

```yaml
# docker-hermes-agent.yml - Agente autónomo (API compatible con OpenAI en el puerto 8000)
services:
  hermes-agent:
    image: nousresearch/hermes-agent:${HERMES_TAG:-latest}
    container_name: hermes-agent
    restart: unless-stopped
    command: gateway run
    ports:
      - "${BIND_IP:-0.0.0.0}:${HERMES_PORT:-8000}:8000"
    environment:
      TZ: ${TZ:-Europe/Madrid}
      API_SERVER_ENABLED: "true"
      API_SERVER_HOST: 0.0.0.0
      API_SERVER_PORT: "8000"
      API_SERVER_KEY: ${HERMES_API_KEY}
      SEARXNG_URL: http://searxng:8080
    volumes:
      - ${HOME}/hermes:/opt/data
    shm_size: "1gb"
    networks:
      - red-ia

networks:
  red-ia:
    external: true
    name: red-ia
```

Notas:

- La API de Hermes está **desactivada por defecto**: `API_SERVER_ENABLED=true` la activa y `API_SERVER_KEY` es la clave (*Bearer*) que deberán usar los clientes.
- Los datos y la configuración (`config.yaml`, `.env`) viven en `/opt/data`, es decir, en `~/hermes`.
- El modelo y el proveedor de IA se configuran una vez (sección 7.2).

### 3.4 `docker-opencode.yml`: IDE de código en el navegador

OpenCode es un agente de programación con interfaz web. Construimos una imagen propia a partir de la oficial para añadir `git` y el servidor MCP de SearXNG.

Crea el fichero: `nano ~/proyecto/docker-opencode.yml`

```yaml
# docker-opencode.yml - Entorno de desarrollo de código (interfaz web)
services:
  opencode:
    build:
      context: ./build/opencode
    image: proyecto/opencode:local
    container_name: opencode
    restart: unless-stopped
    command: ["web", "--hostname", "0.0.0.0", "--port", "8080"]
    working_dir: /workspace
    ports:
      - "${BIND_IP:-0.0.0.0}:${OPENCODE_PORT:-8443}:8080"
    environment:
      TZ: ${TZ:-Europe/Madrid}
      BROWSER: none
      OPENCODE_SERVER_USERNAME: ${OPENCODE_USERNAME:-opencode}
      OPENCODE_SERVER_PASSWORD: ${OPENCODE_PASSWORD}
      SEARXNG_URL: http://searxng:8080
    volumes:
      - ${HOME}/opencode/config:/root/.config/opencode
      - ${HOME}/opencode/data:/root/.local/share/opencode
      - ${HOME}/opencode/projects:/workspace
    networks:
      - red-ia

networks:
  red-ia:
    external: true
    name: red-ia
```

Crea el Dockerfile: `nano ~/proyecto/build/opencode/Dockerfile`

```dockerfile
# build/opencode/Dockerfile
# Imagen oficial de OpenCode + git + Node.js + servidor MCP de SearXNG
FROM ghcr.io/anomalyco/opencode:latest

USER root
RUN apk add --no-cache git openssh-client bash curl nodejs npm \
 && npm install -g mcp-searxng
```

Notas:

- `command: web ...` arranca la interfaz web. La contraseña (`OPENCODE_SERVER_PASSWORD`) activa la autenticación básica HTTP; el usuario por defecto es `opencode`.
- La imagen oficial está basada en Alpine (por eso `apk`). Si el *build* fallase en esa línea, consulta la sección 6.4.
- La configuración que conecta OpenCode con Ollama y SearXNG se escribe en `~/opencode/config/opencode.json` (sección 7.3).

### 3.5 `docker-comfyui.yml`: imágenes y vídeo (GPU)

ComfyUI es una interfaz de nodos para modelos generativos (Stable Diffusion, Flux, etc.). Se construye desde el código oficial con PyTorch para CUDA.

Crea el fichero: `nano ~/proyecto/docker-comfyui.yml`

```yaml
# docker-comfyui.yml - Generación, edición y procesamiento de imágenes y vídeo (GPU NVIDIA)
services:
  comfyui:
    build:
      context: ./build/comfyui
      args:
        COMFYUI_REF: ${COMFYUI_REF:-master}
    image: proyecto/comfyui:local
    container_name: comfyui
    restart: unless-stopped
    command: python main.py --listen 0.0.0.0 --port 8188 ${COMFYUI_EXTRA_ARGS:-}
    ports:
      - "${BIND_IP:-0.0.0.0}:${COMFYUI_PORT:-8188}:8188"
    environment:
      TZ: ${TZ:-Europe/Madrid}
      NVIDIA_VISIBLE_DEVICES: all
      NVIDIA_DRIVER_CAPABILITIES: compute,utility
    volumes:
      - ${HOME}/comfyui/models:/app/models
      - ${HOME}/comfyui/input:/app/input
      - ${HOME}/comfyui/output:/app/output
      - ${HOME}/comfyui/user:/app/user
      - ${HOME}/comfyui/custom_nodes:/app/custom_nodes
    networks:
      - red-ia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8188/system_stats')"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 90s

networks:
  red-ia:
    external: true
    name: red-ia
```

Crea el Dockerfile: `nano ~/proyecto/build/comfyui/Dockerfile`

```dockerfile
# build/comfyui/Dockerfile
FROM python:3.12-slim

ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

RUN apt-get update \
 && apt-get install -y --no-install-recommends git ffmpeg libgl1 libglib2.0-0 \
 && rm -rf /var/lib/apt/lists/*

# PyTorch con CUDA 12.8 (las ruedas incluyen las librerías CUDA; el driver lo aporta el host)
RUN pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu128

ARG COMFYUI_REF=master
RUN git clone --depth 1 --branch ${COMFYUI_REF} https://github.com/comfyanonymous/ComfyUI.git /app
WORKDIR /app
RUN pip install -r requirements.txt

EXPOSE 8188
CMD ["python", "main.py", "--listen", "0.0.0.0", "--port", "8188"]
```

Notas:

- `--listen 0.0.0.0` hace que ComfyUI acepte conexiones desde fuera del contenedor.
- `COMFYUI_EXTRA_ARGS` (en `.env`) permite añadir opciones, por ejemplo `--lowvram` si te quedas sin VRAM.
- El primer *build* es largo (descarga PyTorch, unos 2-3 GB).
- ComfyUI **no trae modelos**: hay que copiar al menos un *checkpoint* en `~/comfyui/models/checkpoints` (sección 5.4).
- Para tarjetas de 8 GB, los modelos de Stable Diffusion 1.5 y SDXL son los más cómodos.

### 3.6 `docker-yolo.yml`: visión artificial (GPU)

YOLO detecta objetos en imágenes. Como no hay un «servidor YOLO» estándar, construimos una API REST mínima con FastAPI sobre la imagen oficial de Ultralytics.

Crea el fichero: `nano ~/proyecto/docker-yolo.yml`

```yaml
# docker-yolo.yml - API de visión artificial (detección de objetos) con GPU NVIDIA
services:
  yolo:
    build:
      context: ./build/yolo
    image: proyecto/yolo:local
    container_name: yolo
    restart: unless-stopped
    ports:
      - "${BIND_IP:-0.0.0.0}:${YOLO_PORT:-5000}:5000"
    environment:
      TZ: ${TZ:-Europe/Madrid}
      YOLO_MODEL: ${YOLO_MODEL:-yolo11n.pt}
      YOLO_MODELS_DIR: /data/models
      YOLO_CONFIG_DIR: /data/config
      NVIDIA_VISIBLE_DEVICES: all
      NVIDIA_DRIVER_CAPABILITIES: compute,utility
    volumes:
      - ${HOME}/yolo:/data
    networks:
      - red-ia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:5000/health')"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 90s

networks:
  red-ia:
    external: true
    name: red-ia
```

Crea el Dockerfile: `nano ~/proyecto/build/yolo/Dockerfile`

```dockerfile
# build/yolo/Dockerfile
FROM ultralytics/ultralytics:latest

ENV PIP_BREAK_SYSTEM_PACKAGES=1 \
    PYTHONUNBUFFERED=1

RUN pip install --no-cache-dir fastapi "uvicorn[standard]" python-multipart

WORKDIR /app
COPY app.py /app/app.py

EXPOSE 5000
ENTRYPOINT []
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "5000"]
```

Crea la API: `nano ~/proyecto/build/yolo/app.py`

```python
# build/yolo/app.py - API REST mínima para detección de objetos con YOLO
import io
import os

import torch
from fastapi import FastAPI, File, Query, UploadFile
from fastapi.responses import Response
from PIL import Image
from ultralytics import YOLO

MODELS_DIR = os.getenv("YOLO_MODELS_DIR", "/data/models")
MODEL_NAME = os.getenv("YOLO_MODEL", "yolo11n.pt")

# Los pesos se descargan (la primera vez) y se guardan en el volumen /data
os.makedirs(MODELS_DIR, exist_ok=True)
os.chdir(MODELS_DIR)

DEVICE = 0 if torch.cuda.is_available() else "cpu"
model = YOLO(MODEL_NAME)

app = FastAPI(title="YOLO API", version="1.0")


@app.get("/health")
def health():
    return {
        "status": "ok",
        "model": MODEL_NAME,
        "device": str(DEVICE),
        "cuda": torch.cuda.is_available(),
    }


@app.post("/detect")
async def detect(file: UploadFile = File(...), conf: float = Query(0.25, ge=0.0, le=1.0)):
    """Devuelve las detecciones en JSON."""
    img = Image.open(io.BytesIO(await file.read())).convert("RGB")
    result = model.predict(img, conf=conf, device=DEVICE, verbose=False)[0]
    detections = []
    for box in result.boxes:
        class_id = int(box.cls[0])
        detections.append(
            {
                "class_id": class_id,
                "class_name": result.names[class_id],
                "confidence": round(float(box.conf[0]), 4),
                "bbox_xyxy": [round(float(v), 1) for v in box.xyxy[0].tolist()],
            }
        )
    return {"count": len(detections), "detections": detections}


@app.post("/detect/image")
async def detect_image(file: UploadFile = File(...), conf: float = Query(0.25, ge=0.0, le=1.0)):
    """Devuelve la imagen con las cajas dibujadas (JPEG)."""
    img = Image.open(io.BytesIO(await file.read())).convert("RGB")
    result = model.predict(img, conf=conf, device=DEVICE, verbose=False)[0]
    annotated = result.plot()[..., ::-1]  # BGR -> RGB
    buffer = io.BytesIO()
    Image.fromarray(annotated).save(buffer, format="JPEG")
    return Response(content=buffer.getvalue(), media_type="image/jpeg")
```

Notas:

- La API ofrece `GET /health`, `POST /detect` (JSON) y `POST /detect/image` (imagen anotada). Documentación interactiva en `http://TU_IP:5000/docs`.
- La primera vez descarga los pesos `yolo11n.pt` (necesita Internet) y los guarda en `~/yolo/models`.
- Cambia `YOLO_MODEL` en `.env` para usar otro tamaño (`yolo11s.pt`, `yolo11m.pt`...).

### 3.7 `docker-searxng.yml`: metabuscador privado

SearXNG consulta varios buscadores sin rastrearte y devuelve los resultados en HTML o JSON. Las IA usan el formato JSON.

Crea el fichero: `nano ~/proyecto/docker-searxng.yml`

```yaml
# docker-searxng.yml - Metabuscador privado
services:
  searxng:
    image: searxng/searxng:${SEARXNG_TAG:-latest}
    container_name: searxng
    restart: unless-stopped
    ports:
      - "${BIND_IP:-0.0.0.0}:${SEARXNG_PORT:-8080}:8080"
    environment:
      TZ: ${TZ:-Europe/Madrid}
      SEARXNG_BASE_URL: http://${SERVER_HOST:-localhost}:${SEARXNG_PORT:-8080}/
    volumes:
      - ${HOME}/searxng/config:/etc/searxng:rw
      - ${HOME}/searxng/cache:/var/cache/searxng:rw
    networks:
      - red-ia
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
      - DAC_OVERRIDE
    logging:
      driver: json-file
      options:
        max-size: "1m"
        max-file: "1"
    healthcheck:
      test: ["CMD-SHELL", "wget -q --spider http://127.0.0.1:8080/healthz || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

networks:
  red-ia:
    external: true
    name: red-ia
```

Crea la configuración: `nano ~/searxng/config/settings.yml`

```yaml
# ~/searxng/config/settings.yml
use_default_settings: true

general:
  instance_name: "SearXNG IA local"
  debug: false

search:
  safe_search: 0
  autocomplete: ""
  formats:
    - html
    - json

server:
  port: 8080
  bind_address: "0.0.0.0"
  secret_key: "CAMBIAR_POR_CLAVE_ALEATORIA"
  limiter: false
  image_proxy: true
```

Sustituye la clave de ejemplo por una aleatoria:

```bash
sed -i "s|CAMBIAR_POR_CLAVE_ALEATORIA|$(openssl rand -hex 32)|" ~/searxng/config/settings.yml
grep secret_key ~/searxng/config/settings.yml
```

Notas:

- `formats: [html, json]` es **imprescindible**: sin `json`, Open WebUI, Hermes y OpenCode recibirán un error 403.
- `limiter: false` desactiva el limitador de peticiones: es lo adecuado en una red privada y evita necesitar Valkey/Redis.
- `cap_drop: ALL` + `cap_add` limita los privilegios del contenedor a lo mínimo que necesita para arrancar.

### 3.8 `docker-rag.yml`: base vectorial para RAG

**RAG** (*Retrieval-Augmented Generation*) amplía un LLM con documentos privados: los documentos se trocean, se convierten en vectores (*embeddings*) y se guardan en una base vectorial; cuando haces una pregunta, se recuperan los fragmentos más parecidos y se entregan al modelo como contexto.

En este proyecto el servicio `rag` es **Qdrant** (la base vectorial). Los *embeddings* los calcula Ollama con el modelo `nomic-embed-text`, y Open WebUI coordina todo el proceso.

Crea el fichero: `nano ~/proyecto/docker-rag.yml`

```yaml
# docker-rag.yml - Base de datos vectorial (Qdrant) para RAG
services:
  rag:
    image: qdrant/qdrant:${QDRANT_TAG:-latest}
    container_name: rag
    restart: unless-stopped
    ports:
      - "${RAG_BIND_IP:-127.0.0.1}:${RAG_PORT:-6333}:6333"
    environment:
      TZ: ${TZ:-Europe/Madrid}
      QDRANT__TELEMETRY_DISABLED: "true"
    volumes:
      - ${HOME}/rag/storage:/qdrant/storage
      - ${HOME}/rag/snapshots:/qdrant/snapshots
    networks:
      - red-ia

networks:
  red-ia:
    external: true
    name: red-ia
```

Notas:

- Qdrant se publica solo en `127.0.0.1` (`RAG_BIND_IP`) porque, sin clave, cualquiera que llegue al puerto podría leer o borrar los vectores. Open WebUI accede igualmente por la red interna.
- No se activa API key en Qdrant: con conexión HTTP sin cifrar el cliente que usa Open WebUI no es compatible con ella.

---
## 4. Fichero de entorno `.env`

Docker Compose lee automáticamente el fichero `.env` situado en la carpeta del proyecto. Aquí se centralizan **todas las variables** de todos los servicios: puertos, versiones de imagen, claves y ajustes. Así cambias un valor en un solo sitio.

Crea el fichero: `nano ~/proyecto/.env`

```bash
# ~/proyecto/.env - Variables de entorno de todo el stack
# ---------------------------------------------------------------------------
# Compose: nombre del proyecto y ficheros que se fusionan con "docker compose up -d"
# ---------------------------------------------------------------------------
COMPOSE_PROJECT_NAME=proyecto
COMPOSE_FILE=docker-ollama.yml:docker-searxng.yml:docker-rag.yml:docker-comfyui.yml:docker-yolo.yml:docker-hermes-agent.yml:docker-opencode.yml:docker-openwebui.yml

# ---------------------------------------------------------------------------
# General
# ---------------------------------------------------------------------------
TZ=Europe/Madrid
# IP del servidor (se usa para la URL base de SearXNG). Se rellena más abajo con un comando.
SERVER_HOST=localhost
# Interfaz donde se publican los puertos: 0.0.0.0 = toda la red, 127.0.0.1 = solo el servidor
BIND_IP=0.0.0.0

# ---------------------------------------------------------------------------
# Ollama (LLMs locales, GPU)
# ---------------------------------------------------------------------------
OLLAMA_TAG=latest
OLLAMA_PORT=11434
OLLAMA_KEEP_ALIVE=5m
OLLAMA_CONTEXT_LENGTH=8192
OLLAMA_NUM_PARALLEL=1
OLLAMA_MAX_LOADED_MODELS=1
OLLAMA_FLASH_ATTENTION=1
OLLAMA_KV_CACHE_TYPE=q8_0
# Modelos que se descargan en la sección 5.4 (ejemplos; mira ollama.com/library para otros)
OLLAMA_CHAT_MODEL=qwen3:8b
OLLAMA_CODE_MODEL=qwen2.5-coder:7b

# ---------------------------------------------------------------------------
# Open WebUI
# ---------------------------------------------------------------------------
OPENWEBUI_TAG=main
OPENWEBUI_PORT=3000
WEBUI_NAME="IA Local"
WEBUI_SECRET_KEY=CAMBIAR
RAG_EMBEDDING_MODEL=nomic-embed-text

# ---------------------------------------------------------------------------
# Hermes Agent
# ---------------------------------------------------------------------------
HERMES_TAG=latest
HERMES_PORT=8000
HERMES_API_KEY=CAMBIAR

# ---------------------------------------------------------------------------
# OpenCode
# ---------------------------------------------------------------------------
OPENCODE_PORT=8443
OPENCODE_USERNAME=opencode
OPENCODE_PASSWORD=CAMBIAR

# ---------------------------------------------------------------------------
# ComfyUI (GPU)
# ---------------------------------------------------------------------------
COMFYUI_REF=master
COMFYUI_PORT=8188
# Opciones extra de arranque, por ejemplo --lowvram si falta VRAM (puede quedar vacío)
COMFYUI_EXTRA_ARGS=

# ---------------------------------------------------------------------------
# YOLO (GPU)
# ---------------------------------------------------------------------------
YOLO_PORT=5000
YOLO_MODEL=yolo11n.pt

# ---------------------------------------------------------------------------
# SearXNG
# ---------------------------------------------------------------------------
SEARXNG_TAG=latest
SEARXNG_PORT=8080

# ---------------------------------------------------------------------------
# RAG (Qdrant)
# ---------------------------------------------------------------------------
QDRANT_TAG=latest
RAG_PORT=6333
# Qdrant no tiene autenticación: se publica solo en el propio servidor
RAG_BIND_IP=127.0.0.1
```

**Generar las claves** (sustituye cada `CAMBIAR` por un valor aleatorio) y rellenar la IP del servidor:

```bash
cd ~/proyecto
sed -i "s|^WEBUI_SECRET_KEY=.*|WEBUI_SECRET_KEY=$(openssl rand -hex 32)|"   .env
sed -i "s|^HERMES_API_KEY=.*|HERMES_API_KEY=$(openssl rand -hex 24)|"       .env
sed -i "s|^OPENCODE_PASSWORD=.*|OPENCODE_PASSWORD=$(openssl rand -hex 12)|" .env
sed -i "s|^SERVER_HOST=.*|SERVER_HOST=$(hostname -I | awk '{print $1}')|"   .env
chmod 600 .env
grep -E "SECRET|API_KEY|PASSWORD|SERVER_HOST" .env
```

> **Guarda las claves en un gestor de contraseñas.** El `.env` contiene secretos: nunca lo subas a un repositorio. Si usas Git, añade `.env` a `.gitignore`.
>
> Todos los valores generados son hexadecimales (sin caracteres especiales) a propósito, para evitar problemas de interpretación en Compose.

---

## 5. Despliegue y verificación

### 5.1 Comprobar la configuración antes de arrancar

```bash
cd ~/proyecto
ls -la                                    # deben verse .env y los 8 ficheros docker-*.yml
docker compose config --quiet && echo "Sintaxis correcta"
docker compose config --services          # lista de servicios detectados
```

- `docker compose config --quiet` valida la sintaxis y la sustitución de variables. No debe mostrar errores ni avisos del tipo *«The X variable is not set»*.
- Deben aparecer 8 servicios: `ollama`, `searxng`, `rag`, `comfyui`, `yolo`, `hermes-agent`, `opencode` y `openwebui`.
- Recuerda: si el comando falla por no encontrar ficheros, no estás en `~/proyecto`.

### 5.2 Red, descarga de imágenes y construcción

```bash
docker network inspect red-ia > /dev/null 2>&1 || docker network create --driver bridge red-ia
docker compose pull --ignore-buildable    # descarga las imágenes oficiales
docker compose build                      # construye ComfyUI, YOLO y OpenCode
```

La primera vez es lento (decenas de GB y, según tu conexión, de 15 a 40 minutos). `build` mostrará mucho texto: es normal. Si termina sin la palabra *ERROR*, continúa.

### 5.3 Arranque de los servicios

Opción recomendada, **por fases**, para detectar problemas pronto:

```bash
docker compose up -d ollama searxng rag          # servicios base
docker compose ps                                 # espera a que estén "Up" / "healthy"
docker compose up -d comfyui yolo                 # servicios con GPU
docker compose up -d hermes-agent opencode openwebui
```

Opción rápida, todo de una vez:

```bash
docker compose up -d
```

`-d` significa *detached*: los contenedores se ejecutan en segundo plano.

### 5.4 Descargar modelos

**Modelos de lenguaje (Ollama).** Los nombres salen de `.env` (`OLLAMA_CHAT_MODEL`, `OLLAMA_CODE_MODEL` y `RAG_EMBEDDING_MODEL`); cámbialos allí si prefieres otros modelos:

```bash
set -a; source .env; set +a                          # carga las variables de .env en esta terminal
docker exec ollama ollama pull "${OLLAMA_CHAT_MODEL}"     # chat general (qwen3:8b)
docker exec ollama ollama pull "${OLLAMA_CODE_MODEL}"     # programación para OpenCode (qwen2.5-coder:7b)
docker exec ollama ollama pull "${RAG_EMBEDDING_MODEL}"   # embeddings para el RAG (nomic-embed-text)
docker exec ollama ollama list                       # comprobar
```

> Con 8 GB de VRAM, los modelos de 7-8 mil millones de parámetros son el límite cómodo. Existen modelos más nuevos en [ollama.com/library](https://ollama.com/library): prueba también versiones más pequeñas si algo va lento.

**Modelo generativo para ComfyUI.** Necesita al menos un *checkpoint*:

```bash
cd ~/comfyui/models/checkpoints
curl -L -O https://huggingface.co/Comfy-Org/stable-diffusion-v1-5-archive/resolve/main/v1-5-pruned-emaonly-fp16.safetensors
cd ~/proyecto
```

Si el enlace dejase de existir, busca «stable diffusion 1.5 safetensors» en huggingface.co y guarda el fichero `.safetensors` en esa misma carpeta.

**Pesos de YOLO.** No hace falta hacer nada: `yolo11n.pt` se descarga solo la primera vez que arranca el contenedor (queda en `~/yolo/models`).

### 5.5 Estado y registros (logs)

```bash
docker compose ps                         # estado de todos los servicios
docker compose logs --tail=50 openwebui   # últimas 50 líneas de un servicio
docker compose logs -f ollama             # seguir el log en directo (Ctrl+C para salir)
docker compose logs --tail=20             # resumen de todos
docker stats --no-stream                  # CPU y memoria por contenedor
```

Un servicio sano aparece como `Up` o `Up (healthy)`. `Restarting` indica que el contenedor falla al arrancar: mira su log.

### 5.6 URLs de acceso

| Servicio | URL desde tu navegador / red | Qué verás |
| :--- | :--- | :--- |
| Open WebUI | `http://TU_IP:3000` | Pantalla de registro (el primer usuario es el administrador) |
| Ollama | `http://TU_IP:11434` | Texto «Ollama is running» |
| Hermes Agent | `http://TU_IP:8000/v1/models` | Requiere cabecera `Authorization: Bearer <HERMES_API_KEY>` |
| OpenCode | `http://TU_IP:8443` | Pide usuario y contraseña (`OPENCODE_*` de `.env`) |
| ComfyUI | `http://TU_IP:8188` | Editor de nodos |
| YOLO | `http://TU_IP:5000/docs` | Documentación interactiva de la API |
| SearXNG | `http://TU_IP:8080` | Buscador |
| RAG (Qdrant) | `http://localhost:6333/dashboard` | Panel de Qdrant (solo desde el propio servidor) |

> Open WebUI: al registrar el primer usuario será administrador. Después, en *Panel de administración → Ajustes → General*, desactiva «Permitir nuevos registros» si no quieres que se creen cuentas nuevas.

### 5.7 Verificar el uso de la GPU

```bash
nvidia-smi                                # GPU y procesos desde el host
docker exec ollama nvidia-smi             # la GPU es visible dentro del contenedor
docker exec ollama ollama run qwen3:8b "Responde solo: hola"
docker exec ollama ollama ps              # ¿en qué se está ejecutando el modelo?
watch -n 1 nvidia-smi                     # monitor en vivo (Ctrl+C para salir)
```

La columna **PROCESSOR** de `ollama ps` debe indicar **`100% GPU`**. Si pone algo como `35%/65% CPU/GPU`, el modelo no cabe entero en la VRAM y irá más lento (usa un modelo más pequeño o reduce `OLLAMA_CONTEXT_LENGTH`).

Para ComfyUI y YOLO:

```bash
docker exec comfyui python -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name(0))"
docker exec yolo python -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name(0))"
```

Ambos deben imprimir `True NVIDIA GeForce RTX 4060` (o tu modelo).

### 5.8 Pruebas de funcionamiento por servicio

Ejecuta estas pruebas desde el servidor, en `~/proyecto`.

```bash
set -a; source .env; set +a      # carga las variables de .env en esta terminal

# 1) Ollama: API y generación
curl -s http://localhost:${OLLAMA_PORT}/api/tags | jq '.models[].name'
curl -s http://localhost:${OLLAMA_PORT}/api/generate \
  -d '{"model":"qwen3:8b","prompt":"Di hola en una frase","stream":false}' | jq -r .response

# 2) Open WebUI
curl -s http://localhost:${OPENWEBUI_PORT}/health

# 3) Hermes Agent (API compatible con OpenAI)
curl -s http://localhost:${HERMES_PORT}/v1/models -H "Authorization: Bearer ${HERMES_API_KEY}" | jq .

# 4) OpenCode (autenticación básica)
curl -s -o /dev/null -w "HTTP %{http_code}\n" -u "${OPENCODE_USERNAME}:${OPENCODE_PASSWORD}" http://localhost:${OPENCODE_PORT}/

# 5) ComfyUI: debe mostrar la GPU
curl -s http://localhost:${COMFYUI_PORT}/system_stats | jq '.devices'

# 6) YOLO: estado y detección sobre una imagen de ejemplo
curl -s http://localhost:${YOLO_PORT}/health | jq .
curl -L -o /tmp/bus.jpg https://ultralytics.com/images/bus.jpg
curl -s -F "file=@/tmp/bus.jpg" http://localhost:${YOLO_PORT}/detect | jq '.count, .detections[0]'

# 7) SearXNG en formato JSON
curl -s "http://localhost:${SEARXNG_PORT}/search?q=ubuntu&format=json" | jq '.results[0].title'

# 8) RAG (Qdrant)
curl -s http://localhost:${RAG_PORT}/readyz
curl -s http://localhost:${RAG_PORT}/collections | jq .
```

Resultados esperados:

| Prueba | Resultado correcto |
| :--- | :--- |
| 1 | Lista de modelos y una frase de respuesta |
| 2 | `{"status":true}` |
| 3 | JSON con el modelo `hermes-agent` |
| 4 | `HTTP 200` |
| 5 | JSON con el nombre de tu GPU |
| 6 | `"cuda": true`; en `/detect`, un recuento de objetos (el autobús de ejemplo contiene personas y un autobús) |
| 7 | Un título de resultado |
| 8 | Respuesta «all shards are ready» y una lista de colecciones (vacía al principio) |

### 5.9 Script de comprobación `check.sh`

Automatiza lo anterior, incluidas las **conexiones entre contenedores** por la red `red-ia`.

Crea el fichero: `nano ~/proyecto/scripts/check.sh`

```bash
#!/usr/bin/env bash
# ~/proyecto/scripts/check.sh - Comprobación rápida de todos los servicios
cd "$(dirname "$0")/.." || exit 1
set -a; source .env; set +a

host="${BIND_IP}"; [ "${host}" = "0.0.0.0" ] && host="localhost"
rag_host="${RAG_BIND_IP}"; [ "${rag_host}" = "0.0.0.0" ] && rag_host="localhost"
auth_hermes="Authorization: Bearer ${HERMES_API_KEY}"
ok=0; fallo=0

probar() {
  local nombre="$1"; shift
  if "$@" > /dev/null 2>&1; then
    printf "  [ OK  ] %s\n" "${nombre}"; ok=$((ok + 1))
  else
    printf "  [FALLO] %s\n" "${nombre}"; fallo=$((fallo + 1))
  fi
}

echo "== Servicios (puertos del host) =="
probar "Ollama       :${OLLAMA_PORT}"    curl -fsS "http://${host}:${OLLAMA_PORT}/api/tags"
probar "Open WebUI   :${OPENWEBUI_PORT}" curl -fsS "http://${host}:${OPENWEBUI_PORT}/health"
probar "Hermes Agent :${HERMES_PORT}"    curl -fsS -H "${auth_hermes}" "http://${host}:${HERMES_PORT}/v1/models"
probar "OpenCode     :${OPENCODE_PORT}"  curl -fsS -u "${OPENCODE_USERNAME}:${OPENCODE_PASSWORD}" -o /dev/null "http://${host}:${OPENCODE_PORT}/"
probar "ComfyUI      :${COMFYUI_PORT}"   curl -fsS "http://${host}:${COMFYUI_PORT}/system_stats"
probar "YOLO         :${YOLO_PORT}"      curl -fsS "http://${host}:${YOLO_PORT}/health"
probar "SearXNG JSON :${SEARXNG_PORT}"   curl -fsS "http://${host}:${SEARXNG_PORT}/search?q=test&format=json"
probar "RAG Qdrant   :${RAG_PORT}"       curl -fsS "http://${rag_host}:${RAG_PORT}/readyz"

echo "== Conexiones internas (red-ia, por nombre de contenedor) =="
probar "openwebui -> ollama"       docker exec openwebui curl -fsS http://ollama:11434/api/tags
probar "openwebui -> searxng"      docker exec openwebui curl -fsS "http://searxng:8080/search?q=test&format=json"
probar "openwebui -> rag"          docker exec openwebui curl -fsS http://rag:6333/readyz
probar "openwebui -> comfyui"      docker exec openwebui curl -fsS http://comfyui:8188/system_stats
probar "openwebui -> hermes-agent" docker exec openwebui curl -fsS -H "${auth_hermes}" http://hermes-agent:8000/v1/models
probar "opencode  -> ollama"       docker exec opencode curl -fsS http://ollama:11434/api/tags
probar "opencode  -> searxng"      docker exec opencode curl -fsS "http://searxng:8080/search?q=test&format=json"

echo "== GPU =="
nvidia-smi --query-gpu=name,memory.used,memory.total --format=csv,noheader || echo "  nvidia-smi no disponible"

echo
echo "Resultado: ${ok} correctas, ${fallo} con fallo"
[ "${fallo}" -eq 0 ]
```

Dale permisos de ejecución y lánzalo:

```bash
chmod +x ~/proyecto/scripts/check.sh
~/proyecto/scripts/check.sh
```

> Antes de configurar Hermes (sección 7.2) es normal que falle la prueba de Hermes Agent.

### 5.10 Parar, arrancar y reiniciar

```bash
docker compose stop                 # para todos (conserva contenedores y datos)
docker compose start                # arranca los parados
docker compose restart ollama       # reinicia solo un servicio
docker compose down                 # elimina contenedores y red de compose (los datos en ~/<servicio> se conservan)
docker compose up -d                # vuelve a crearlos
```

**Con los servicios arriba, continúa con la sección 7** para conectarlos entre sí (configurar Hermes y OpenCode, ajustar Open WebUI).

---

## 6. Mantenimiento y actualización

### 6.1 Copias de seguridad

Qué se copia: la configuración del proyecto y los datos pequeños e importantes (usuarios y chats de Open WebUI, configuración de Hermes, OpenCode, SearXNG, vectores del RAG, resultados y workflows de ComfyUI...). Los **modelos** (`~/ollama`, `~/comfyui/models`) pesan decenas de GB y se pueden volver a descargar, por lo que solo se incluyen si pides `--con-modelos`.

El script usa un contenedor `alpine` para leer los ficheros: así puede copiar los datos creados por los contenedores como *root* **sin usar `sudo`** (y puede lanzarse desde `cron`).

Crea el fichero: `nano ~/proyecto/scripts/backup.sh`

```bash
#!/usr/bin/env bash
# ~/proyecto/scripts/backup.sh - Copia de seguridad de configuración y datos
# Uso: ./scripts/backup.sh [--con-modelos]
set -euo pipefail

PROYECTO="${HOME}/proyecto"
DESTINO="${PROYECTO}/backups"
FECHA="$(date +%Y%m%d_%H%M%S)"
RETENCION_DIAS=14

CARPETAS="proyecto openwebui hermes opencode searxng rag yolo comfyui/user comfyui/custom_nodes comfyui/input comfyui/output"
if [ "${1:-}" = "--con-modelos" ]; then
  CARPETAS="${CARPETAS} ollama comfyui/models"
fi

mkdir -p "${DESTINO}"
cd "${PROYECTO}"

echo ">> Parando servicios para obtener una copia consistente..."
docker compose stop
trap 'echo ">> Arrancando servicios..."; docker compose start' EXIT

echo ">> Creando ${DESTINO}/datos_${FECHA}.tar.gz"
docker run --rm \
  -v "${HOME}:/origen:ro" \
  -v "${DESTINO}:/destino" \
  alpine \
  tar czf "/destino/datos_${FECHA}.tar.gz" -C /origen --exclude=proyecto/backups ${CARPETAS}

echo ">> Borrando copias de más de ${RETENCION_DIAS} días..."
find "${DESTINO}" -name 'datos_*.tar.gz' -mtime +"${RETENCION_DIAS}" -delete

ls -lh "${DESTINO}" | tail -n 5
```

```bash
chmod +x ~/proyecto/scripts/backup.sh
~/proyecto/scripts/backup.sh                 # copia normal
~/proyecto/scripts/backup.sh --con-modelos   # incluye modelos (muy grande)
```

**Automatizar** (cada domingo a las 03:00): ejecuta `crontab -e` y añade, cambiando `USUARIO` por tu nombre de usuario:

```text
0 3 * * 0 /home/USUARIO/proyecto/scripts/backup.sh >> /home/USUARIO/proyecto/backups/backup.log 2>&1
```

**Restaurar una copia:**

```bash
cd ~/proyecto
docker compose down
docker run --rm -v "${HOME}:/destino" -v "${HOME}/proyecto/backups:/copias:ro" alpine \
  tar xzf /copias/datos_AAAAMMDD_HHMMSS.tar.gz -C /destino
docker compose up -d
```

> Una copia que nunca se ha probado no es una copia. Restaura en una máquina de pruebas al menos una vez.
> Guarda además una copia **fuera del servidor** (disco externo o `scp` a otro equipo).

### 6.2 Actualización de servicios

Crea el fichero: `nano ~/proyecto/scripts/update.sh`

```bash
#!/usr/bin/env bash
# ~/proyecto/scripts/update.sh - Actualiza imágenes y recrea los contenedores
set -euo pipefail
cd "${HOME}/proyecto"

echo ">> Copia de seguridad previa"
./scripts/backup.sh

echo ">> Descargando imágenes nuevas"
docker compose pull --ignore-buildable

echo ">> Reconstruyendo imágenes propias (ComfyUI, YOLO, OpenCode)"
docker compose build --pull

echo ">> Recreando los contenedores que han cambiado"
docker compose up -d

echo ">> Limpiando imágenes antiguas"
docker image prune -f

docker compose ps
```

```bash
chmod +x ~/proyecto/scripts/update.sh
~/proyecto/scripts/update.sh
```

Para actualizar **un solo servicio**:

```bash
docker compose pull ollama
docker compose up -d ollama
```

**Fijar versiones y volver atrás.** Con `latest` cada actualización puede traer cambios inesperados. Una vez el sistema funcione bien, anota la versión que usas (`docker image ls`) y fíjala en `.env`, por ejemplo `OLLAMA_TAG=0.12.3` o `OPENWEBUI_TAG=v0.6.0` (usa números de versión reales publicados en cada proyecto). Para volver atrás, restaura el valor anterior y ejecuta `docker compose up -d <servicio>`.

**Actualizar el sistema operativo:**

```bash
sudo apt update && sudo apt upgrade -y
sudo reboot            # necesario si se actualiza el kernel o el driver NVIDIA
nvidia-smi             # comprobar que la GPU sigue operativa tras reiniciar
~/proyecto/scripts/check.sh
```

### 6.3 Supervisión y limpieza

```bash
docker compose ps                  # estado
docker stats --no-stream           # consumo de CPU y RAM
watch -n 2 nvidia-smi              # uso de GPU
df -h /                            # espacio en disco
docker system df                   # espacio que usa Docker
docker image prune -f              # elimina imágenes huérfanas
docker builder prune -f            # elimina caché de construcción
```

**Limitar el tamaño de los logs** (opcional pero recomendable). Este comando **añade** la rotación a la configuración existente de Docker sin borrar lo que escribió el NVIDIA Container Toolkit:

```bash
sudo cp /etc/docker/daemon.json /etc/docker/daemon.json.bak
sudo jq '. + {"log-driver":"json-file","log-opts":{"max-size":"10m","max-file":"3"}}' /etc/docker/daemon.json.bak | sudo tee /etc/docker/daemon.json > /dev/null
sudo systemctl restart docker
cd ~/proyecto && docker compose up -d --force-recreate
```

### 6.4 Resolución de errores comunes

| Síntoma | Causa probable | Solución |
| :--- | :--- | :--- |
| `permission denied while trying to connect to the Docker daemon socket` | Tu usuario no está en el grupo `docker` | `sudo usermod -aG docker $USER` y volver a iniciar sesión |
| `network red-ia declared as external, but could not be found` | La red no existe | `docker network create --driver bridge red-ia` |
| `could not select device driver "" with capabilities: [[gpu]]` | El NVIDIA Container Toolkit no está instalado o configurado | Repite la sección 1.6: `sudo nvidia-ctk runtime configure --runtime=docker && sudo systemctl restart docker` |
| `Failed to initialize NVML: Driver/library version mismatch` | Se actualizó el driver y no se reinició | `sudo reboot` |
| `nvidia-smi: command not found` o no lista GPU | Driver sin instalar, o bloqueado por Secure Boot | Repite la sección 1.3 y revisa *Enroll MOK* |
| `port is already allocated` / `address already in use` | Otro programa usa ese puerto del host | `sudo ss -tulpn \| grep :PUERTO`; cambia el puerto en `.env` y ejecuta `docker compose up -d` |
| `The container name "/xxx" is already in use` | Existe un contenedor antiguo con ese nombre | `docker rm -f xxx` y `docker compose up -d` |
| `docker compose` avisa de variables sin definir | Estás fuera de `~/proyecto` o falta el `.env` | `cd ~/proyecto` y comprueba `ls -la .env` |
| Open WebUI no lista modelos de Ollama | Conexión interna o configuración ya guardada | `docker exec openwebui curl -s http://ollama:11434` debe responder «Ollama is running». Revisa *Panel de administración → Ajustes → Conexiones* (las variables solo se leen la primera vez) |
| Las respuestas de Ollama son muy lentas | El modelo no cabe en VRAM y usa CPU | `docker exec ollama ollama ps`; usa un modelo menor o baja `OLLAMA_CONTEXT_LENGTH` |
| `CUDA out of memory` en ComfyUI o YOLO | Un modelo de Ollama ocupa la VRAM | `docker exec ollama ollama stop NOMBRE_MODELO`, o `OLLAMA_KEEP_ALIVE=0` en `.env`; para ComfyUI, `COMFYUI_EXTRA_ARGS=--lowvram` |
| ComfyUI dice que no hay *checkpoints* | Falta el modelo en la carpeta correcta | Fichero `.safetensors` en `~/comfyui/models/checkpoints`; recarga la página |
| SearXNG devuelve 403 con `format=json` | `settings.yml` no incluye `json` en `formats` | Edítalo (sección 3.7) y `docker compose restart searxng` |
| SearXNG no arranca | `secret_key` sin cambiar o permisos | `docker compose logs searxng`; revisa la clave y los permisos (ver abajo) |
| Hermes responde 401 | La clave no coincide | Usa exactamente `HERMES_API_KEY` de `.env` |
| Hermes rechaza el modelo por contexto (< 64K) o usa otro proveedor | Contexto de Ollama pequeño o config sin aplicar | Sección 7.2 |
| OpenCode no carga o pide credenciales | Se usa `https://` o credenciales incorrectas | Usa `http://TU_IP:8443` y el usuario/clave de `.env` |
| El *build* de OpenCode falla en `apk` | La imagen base no es Alpine en esa versión | Sustituye `apk add --no-cache ...` por `apt-get update && apt-get install -y git openssh-client curl nodejs npm` (si es Debian) |
| Disco lleno | Imágenes, caché o modelos | `docker system df`, `docker image prune -f`, `docker builder prune -f`; borra modelos que no uses |

#### Problemas de permisos (muy frecuentes)

Los contenedores de este proyecto se ejecutan como *root*, de modo que los ficheros que crean en `~/ollama`, `~/openwebui`, `~/comfyui`, etc. pertenecen a *root*. Es normal, pero puedes encontrarte con:

- **No puedes editar, copiar o borrar** esos ficheros desde tu usuario. Comprueba el propietario y devuélvelo:

  ```bash
  ls -ld ~/ollama ~/openwebui ~/hermes ~/opencode ~/comfyui ~/yolo ~/rag
  sudo chown -R $USER:$USER ~/openwebui ~/hermes ~/opencode ~/comfyui ~/yolo ~/rag ~/ollama
  ```

- **SearXNG** es la excepción: al arrancar, ajusta él mismo la propiedad de `~/searxng` a su usuario interno (UID 977 en la imagen actual). Para editar su `settings.yml` usa `sudo nano ~/searxng/config/settings.yml`. Si no arranca por permisos: `sudo chown -R 977:977 ~/searxng` y `docker compose restart searxng`.
- **`.env` ilegible o con permisos abiertos:** `chmod 600 ~/proyecto/.env`.
- **Scripts que no se ejecutan** (`Permission denied`): `chmod +x ~/proyecto/scripts/*.sh`.
- **Nunca uses `chmod -R 777`** para «arreglarlo»: da acceso total a todos los usuarios del sistema.
- **No ejecutes `docker compose` con `sudo`**: `${HOME}` apuntaría a `/root` y se crearían datos en otra ubicación.

---
## 7. Guía interna de integración entre servicios

Esta sección explica **cómo se conecta exactamente cada servicio con los demás**. Recuerda la regla de oro: dentro de la red `red-ia` se usa `http://NOMBRE_CONTENEDOR:PUERTO_INTERNO`.

### 7.0 Mapa de conexiones

| # | Origen (cliente) | Destino (servidor) | URL interna | Dónde se configura |
| :-: | :--- | :--- | :--- | :--- |
| 1 | Open WebUI | Ollama | `http://ollama:11434` | `docker-openwebui.yml` (`OLLAMA_BASE_URL`) |
| 2 | Open WebUI | Hermes Agent | `http://hermes-agent:8000/v1` | `docker-openwebui.yml` (`OPENAI_API_BASE_URL`) |
| 3 | Open WebUI | SearXNG | `http://searxng:8080/search?q=<query>` | `docker-openwebui.yml` (`SEARXNG_QUERY_URL`) |
| 4 | Open WebUI | ComfyUI | `http://comfyui:8188` | `docker-openwebui.yml` + workflow en la interfaz |
| 5 | Open WebUI | RAG (Qdrant) y Ollama (embeddings) | `http://rag:6333`, `http://ollama:11434` | `docker-openwebui.yml` (`VECTOR_DB`, `QDRANT_URI`, `RAG_*`) |
| 6 | Hermes Agent | Ollama | `http://ollama:11434/v1` | `~/hermes/config.yaml` (sección 7.2) |
| 7 | Hermes Agent | SearXNG | `http://searxng:8080` | `docker-hermes-agent.yml` (`SEARXNG_URL`) |
| 8 | Hermes Agent | ComfyUI | `http://comfyui:8188` | Skill `comfyui` de Hermes (sección 7.3) |
| 9 | OpenCode | Ollama | `http://ollama:11434/v1` | `~/opencode/config/opencode.json` (sección 7.4) |
| 10 | OpenCode | SearXNG | `http://searxng:8080` | `opencode.json` → servidor MCP (sección 7.4) |
| 11 | Cualquier contenedor | YOLO | `http://yolo:5000` | Sin configuración: API REST (sección 7.8) |

### 7.1 Ollama ↔ Open WebUI

Ya está conectado por la variable `OLLAMA_BASE_URL=http://ollama:11434`.

1. Abre `http://TU_IP:3000` y crea la cuenta de administrador (el primer registro).
2. Arriba a la izquierda, el selector de modelos debe mostrar `qwen3:8b`, `qwen2.5-coder:7b`, etc.
3. Escribe un mensaje: la primera respuesta tarda unos segundos porque el modelo se carga en la GPU.

Si no aparecen modelos: *Panel de administración → Ajustes → Conexiones → Ollama*. La URL debe ser `http://ollama:11434` (no `localhost`: dentro del contenedor `localhost` es el propio Open WebUI). Pulsa el botón de verificar conexión.

### 7.2 Ollama ↔ Hermes Agent (y SearXNG ↔ Hermes Agent)

Hermes Agent es exigente con el **contexto**: en las versiones probadas se niega a trabajar con menos de **64.000 tokens**, y el contexto por defecto de Ollama es mucho menor. Hay que crear una variante del modelo con contexto ampliado y decirle a Hermes que la use.

> **Aviso de hardware:** 64.000 tokens de contexto consumen mucha memoria. En una RTX 4060 de 8 GB conviene un modelo pequeño (4B) y aun así Ollama puede repartir parte del trabajo con la CPU y la RAM, de modo que irá más lento que Open WebUI. Es una limitación del hardware, no un error de configuración.

**Paso 1. Crear el modelo con contexto ampliado:**

```bash
docker exec ollama ollama pull qwen3:4b
docker exec ollama sh -c 'printf "FROM qwen3:4b\nPARAMETER num_ctx 65536\n" > /tmp/Modelfile && ollama create qwen3-4b-64k -f /tmp/Modelfile'
docker exec ollama ollama list
```

Debe aparecer `qwen3-4b-64k`.

**Paso 2. Configurar Hermes con el asistente.** Para el contenedor (Hermes reescribe su configuración mientras funciona) y lanza el asistente:

```bash
cd ~/proyecto
docker compose stop hermes-agent
docker compose run --rm hermes-agent setup
```

Cuando pregunte:

| Pregunta | Respuesta |
| :--- | :--- |
| Proveedor | **Custom endpoint** (endpoint compatible con OpenAI) |
| URL base | `http://ollama:11434/v1` |
| Clave API | cualquier texto (por ejemplo `no-key`): Ollama no la usa |
| Modelo | `qwen3-4b-64k` |
| Longitud de contexto | `65536` |

El asistente guarda el resultado en `~/hermes/config.yaml`.

*Alternativa manual:* edita `~/hermes/config.yaml` (con el contenedor parado) y asegúrate de que contiene estas claves, sin borrar el resto del fichero:

```yaml
model:
  default: "qwen3-4b-64k"
  provider: "custom"
  base_url: "http://ollama:11434/v1"
  context_length: 65536

web:
  search_backend: "searxng"
```

**Paso 3. SearXNG.** La variable `SEARXNG_URL=http://searxng:8080` ya está en `docker-hermes-agent.yml`. SearXNG solo **busca**; Hermes necesitará además un proveedor de «extracción» de páginas si quieres que lea el contenido completo de las URL (se configura con `hermes tools`, apartado *Web Search & Extract*).

**Paso 4. Arrancar y probar:**

```bash
docker compose up -d hermes-agent
docker compose logs --tail=30 hermes-agent

set -a; source .env; set +a
curl -s http://localhost:${HERMES_PORT}/v1/chat/completions \
  -H "Authorization: Bearer ${HERMES_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"model":"hermes-agent","messages":[{"role":"user","content":"Hola, ¿quién eres?"}]}' | jq -r '.choices[0].message.content'
```

**Paso 5. Usar Hermes desde Open WebUI.** La conexión ya está definida (`OPENAI_API_BASE_URL` y `OPENAI_API_KEY`). En el selector de modelos aparecerá **`hermes-agent`**. Si no aparece, revisa *Panel de administración → Ajustes → Conexiones → OpenAI API*: URL `http://hermes-agent:8000/v1` y clave igual a `HERMES_API_KEY`.

**Si Hermes ignora tu configuración** (en los registros aparece que usa *openrouter* u otro proveedor): comprueba que no haya claves de otros proveedores en `~/hermes/.env`, reinicia con `docker compose restart hermes-agent` y revisa lo que ve Hermes con `docker exec hermes-agent hermes config show`.

### 7.3 ComfyUI ↔ Hermes Agent

Hermes dispone de una *skill* llamada `comfyui` que maneja ComfyUI a través de su API REST (envía *workflows*, recoge las imágenes generadas y puede iterar sobre el prompt). Según la versión, viene incluida o se instala con:

```bash
docker exec -it hermes-agent hermes skills install official/creative/comfyui
```

Requisitos:

- ComfyUI funcionando y con al menos un *checkpoint* (sección 5.4).
- Indicarle a la skill que ComfyUI está en **`http://comfyui:8188`** (desde el contenedor de Hermes, `localhost` no sirve). Revisa en la documentación de la skill el parámetro o la variable exacta de tu versión: <https://hermes-agent.nousresearch.com/docs/user-guide/skills/optional/creative/creative-comfyui>
- Hermes y ComfyUI comparten los 8 GB de VRAM con Ollama: si falla por memoria, consulta la sección 6.4.

### 7.4 Ollama ↔ OpenCode y SearXNG ↔ OpenCode

OpenCode se configura con un fichero JSON. Crea `~/opencode/config/opencode.json`:

```bash
nano ~/opencode/config/opencode.json
```

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama (local)",
      "options": {
        "baseURL": "http://ollama:11434/v1"
      },
      "models": {
        "qwen2.5-coder:7b": { "name": "Qwen2.5 Coder 7B" },
        "qwen3:8b": { "name": "Qwen3 8B" }
      }
    }
  },
  "model": "ollama/qwen2.5-coder:7b",
  "mcp": {
    "searxng": {
      "type": "local",
      "command": ["mcp-searxng"],
      "environment": {
        "SEARXNG_URL": "http://searxng:8080"
      },
      "enabled": true
    }
  }
}
```

Qué significa:

- `provider.ollama`: define a Ollama como proveedor mediante su API compatible con OpenAI (`/v1`). Los nombres de `models` deben coincidir con los de `ollama list`.
- `mcp.searxng`: registra un servidor MCP (*Model Context Protocol*) que da a OpenCode la herramienta de búsqueda web. El programa `mcp-searxng` ya se instaló en la imagen (`build/opencode/Dockerfile`).

Aplica y comprueba:

```bash
cd ~/proyecto
docker compose restart opencode
docker exec opencode opencode mcp ls          # debe listar "searxng"
```

Entra en `http://TU_IP:8443` (usuario y contraseña de `.env`), elige el modelo *Qwen2.5 Coder 7B* y pide algo que obligue a buscar en Internet.

> **Contexto para programar:** los agentes de código necesitan más contexto que un chat. Si OpenCode «olvida» el principio o falla al usar herramientas, sube `OLLAMA_CONTEXT_LENGTH` en `.env` (por ejemplo a `16384`) y ejecuta `docker compose up -d ollama`. Más contexto significa más VRAM.

### 7.5 SearXNG ↔ Open WebUI

La conexión viene preconfigurada con `WEB_SEARCH_ENGINE=searxng` y `SEARXNG_QUERY_URL=http://searxng:8080/search?q=<query>`.

1. Comprueba la configuración en *Panel de administración → Ajustes → Búsqueda web*: activada, motor **SearXNG** y la URL anterior. Si aquí no coincide, corrígela y guarda (la interfaz manda sobre las variables, ver nota de la sección 3.2).
2. En un chat nuevo, pulsa el icono **+** junto al cuadro de texto y activa **Búsqueda web**.
3. Pregunta por algo reciente. La respuesta incluirá fuentes.

Si falla, lo más habitual es que SearXNG no tenga el formato `json` activo: ejecuta `docker exec openwebui curl -s "http://searxng:8080/search?q=test&format=json" | head -c 200`; si devuelve un 403 o HTML, revisa la sección 3.7.

### 7.6 ComfyUI ↔ Open WebUI

Open WebUI envía a ComfyUI un *workflow* completo, así que hay que exportarlo desde ComfyUI y asociar sus nodos.

1. Abre ComfyUI (`http://TU_IP:8188`) y comprueba que el workflow por defecto genera una imagen con tu *checkpoint*.
2. En los ajustes de ComfyUI (icono del engranaje) activa las **opciones de modo desarrollador**. Aparecerá la opción **Save (API Format)** en el menú. Guarda el fichero `workflow_api.json`.
3. En Open WebUI: *Panel de administración → Ajustes → Imágenes*.
   - Motor de generación de imágenes: **ComfyUI**.
   - URL base: `http://comfyui:8188`.
   - Pega o carga el contenido de `workflow_api.json` en el campo del workflow.
   - En el mapeo de nodos, indica el **ID de nodo** (el número que aparece en el JSON) que corresponde a: el texto del *prompt* positivo (nodo `CLIPTextEncode`), el modelo (nodo `CheckpointLoaderSimple`), el tamaño, los pasos y la semilla (`KSampler`, `EmptyLatentImage`).
4. Guarda. En un chat, usa el botón de **generar imagen** bajo una respuesta o la herramienta de imágenes.

> Los nombres exactos de los campos pueden variar entre versiones de Open WebUI; lo importante es que cada ID de nodo apunte al nodo correcto del JSON.

### 7.7 RAG: Qdrant + Ollama ↔ Open WebUI

Flujo completo del RAG en este proyecto:

```text
Documento ──► Open WebUI (trocea) ──► Ollama: nomic-embed-text (vectores) ──► Qdrant (rag:6333)
Pregunta  ──► Open WebUI ──► Ollama (vector de la pregunta) ──► Qdrant (fragmentos parecidos) ──► LLM responde
```

Las variables ya están definidas en `docker-openwebui.yml`: `VECTOR_DB=qdrant`, `QDRANT_URI=http://rag:6333`, `RAG_EMBEDDING_ENGINE=ollama`, `RAG_EMBEDDING_MODEL=nomic-embed-text` y `RAG_OLLAMA_BASE_URL=http://ollama:11434`.

Pasos para usarlo:

1. Comprueba que el modelo de embeddings está descargado: `docker exec ollama ollama list` (debe aparecer `nomic-embed-text`).
2. En Open WebUI: *Espacio de trabajo → Conocimiento* → crear una colección y subir documentos (PDF, TXT, MD...). Puedes guardar los originales en `~/rag/docs` como archivo.
3. En un chat, escribe `#` y elige la colección, o asóciala a un modelo, y haz preguntas sobre el contenido.
4. Verifica que Qdrant ha recibido los vectores:

   ```bash
   curl -s http://localhost:6333/collections | jq .
   ```

   Aparecerán colecciones creadas por Open WebUI.

> Si cambias el modelo de embeddings, los documentos ya cargados quedan indexados con el modelo anterior y hay que volver a subirlos.

### 7.8 YOLO como servicio para el resto

YOLO no depende de ningún otro servicio: es una API REST que cualquier contenedor de `red-ia` puede llamar.

```bash
# Desde otro contenedor (por nombre):
docker exec opencode curl -s http://yolo:5000/health

# Detección sobre una imagen:
curl -s -F "file=@foto.jpg" http://localhost:5000/detect | jq .

# Imagen con las cajas dibujadas:
curl -s -F "file=@foto.jpg" http://localhost:5000/detect/image -o resultado.jpg
```

Un agente (Hermes o OpenCode) puede usar esta API desde sus herramientas de terminal pidiéndole que ejecute estas llamadas `curl`.

### 7.9 Diagnóstico de la conectividad interna

```bash
# ¿Qué contenedores están conectados a la red y con qué IP?
docker network inspect red-ia --format '{{range .Containers}}{{.Name}}  {{.IPv4Address}}{{"\n"}}{{end}}'

# ¿Resuelve el DNS interno los nombres?
docker exec openwebui getent hosts ollama searxng rag comfyui hermes-agent

# ¿Responde cada servicio dentro de la red?
~/proyecto/scripts/check.sh
```

Si un nombre no resuelve, ese contenedor no está en `red-ia` (¿está parado? `docker compose ps`).

---

## 8. Comprobación de los criterios de aceptación

### 8.1 Criterios de la especificación

| Criterio | Cómo se cumple | Cómo comprobarlo |
| :--- | :--- | :--- |
| **6.1** Los ficheros `docker-<servicio>.yml` son funcionales y sin errores sintácticos | Ocho ficheros autónomos con la misma estructura | `cd ~/proyecto && docker compose config --quiet && echo OK`<br>`for f in docker-*.yml; do docker compose -f $f config --quiet && echo "$f OK"; done` |
| **6.2** Contenedores con librerías NVIDIA/CUDA, usando principalmente GPU | Ollama, ComfyUI y YOLO reservan la GPU (son los únicos con carga CUDA). Open WebUI, Hermes, OpenCode, SearXNG y Qdrant no tienen cargas que aprovechen una GPU (ver sección 0.3, punto 4) | `docker exec ollama ollama ps` (100 % GPU)<br>`docker exec comfyui python -c "import torch; print(torch.cuda.is_available())"`<br>`docker exec yolo python -c "import torch; print(torch.cuda.is_available())"` |
| **6.3** Sin colisiones de puertos | Puertos del host distintos: 11434, 3000, 8000, 8443, 8188, 5000, 8080, 6333 (el RAG se movió del 11434 al 6333) | `docker ps --format 'table {{.Names}}\t{{.Ports}}'` |
| **6.4** Explicaciones paso a paso para un administrador novato | Cada sección parte de cero, explica los comandos y las comprobaciones | Seguir el manual en un servidor limpio, de la sección 1 a la 7 |

### 8.2 Lista de verificación final

- [ ] `nvidia-smi` muestra la GPU en el host y `docker run --rm --gpus all ubuntu nvidia-smi` también.
- [ ] `docker compose ps` muestra los 8 servicios en `Up` (los que tienen *healthcheck*, en `healthy`).
- [ ] `~/proyecto/scripts/check.sh` termina sin fallos (tras configurar Hermes).
- [ ] `ollama ps` indica `100% GPU` con el modelo de chat.
- [ ] Open WebUI responde, busca en Internet (SearXNG), genera una imagen (ComfyUI) y contesta sobre un documento subido (RAG).
- [ ] OpenCode abre en `http://TU_IP:8443` y usa Ollama.
- [ ] Hay una copia de seguridad hecha con `backup.sh` y probada.

### 8.3 Seguridad mínima

- Los servicios son para una **red privada**: no los publiques en Internet ni reenvíes puertos del router hacia ellos.
- Hermes Agent y OpenCode ejecutan acciones en tu nombre: protégelos con las claves del `.env` y úsalos solo con usuarios de confianza.
- Desactiva los registros abiertos de Open WebUI tras crear las cuentas necesarias.
- Mantén `.env` con permisos `600` y fuera de Git.
- Si necesitas acceso desde fuera, usa una VPN o un túnel SSH (`ssh -L 3000:localhost:3000 usuario@servidor`) y publica los puertos con `BIND_IP=127.0.0.1`.

---

## 9. Anexo: chuleta de comandos

| Tarea | Comando |
| :--- | :--- |
| Ver estado | `docker compose ps` |
| Arrancar todo | `docker compose up -d` |
| Arrancar un servicio | `docker compose up -d ollama` |
| Parar todo (sin borrar) | `docker compose stop` |
| Reiniciar un servicio | `docker compose restart openwebui` |
| Eliminar contenedores (datos intactos) | `docker compose down` |
| Ver logs en directo | `docker compose logs -f NOMBRE` |
| Entrar a un contenedor | `docker exec -it NOMBRE bash` (en `opencode` y `searxng` usa `sh`) |
| Validar la configuración | `docker compose config --quiet` |
| Reconstruir imágenes propias | `docker compose build` |
| Descargar imágenes nuevas | `docker compose pull --ignore-buildable` |
| Modelos instalados | `docker exec ollama ollama list` |
| Descargar un modelo | `docker exec ollama ollama pull NOMBRE` |
| Modelos cargados en GPU | `docker exec ollama ollama ps` |
| Descargar un modelo de la GPU | `docker exec ollama ollama stop NOMBRE` |
| Estado de la GPU | `nvidia-smi` |
| Comprobar todo el stack | `~/proyecto/scripts/check.sh` |
| Copia de seguridad | `~/proyecto/scripts/backup.sh` |
| Actualizar | `~/proyecto/scripts/update.sh` |
| Espacio que usa Docker | `docker system df` |
| Puertos en uso | `sudo ss -tulpn` |

---

*Fin del manual.*
