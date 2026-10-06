---
title: "Manual técnico — Pila de IA local en Docker sobre Ubuntu"
author: "Vantrio / Administrador de sistemas"
date: "2026-10-06"
category: "IA local / Docker / DevOps"
version: "1.0"
tags: [markdown, ia, docker, docker-compose, ubuntu, nvidia, cuda, ollama, openwebui, hermes, opencode, comfyui, yolo, searxng, rag]
---

# Manual técnico: despliegue de una pila de IA local en Docker sobre Ubuntu

## 0. Propósito y alcance

Este documento transforma la especificación del proyecto SDD en un procedimiento operativo para desplegar una pila de Inteligencia Artificial local sobre **Ubuntu Server**, utilizando **Docker Compose**, una **GPU NVIDIA** y una red Docker común denominada `red-ia`.

El objetivo es que un administrador de sistemas principiante pueda:

1. Preparar Ubuntu.
2. Instalar Docker Engine y Docker Compose.
3. Instalar el driver NVIDIA y NVIDIA Container Toolkit.
4. Crear la estructura de directorios.
5. Crear la red Docker común.
6. Desplegar cada servicio en su propio contenedor.
7. Configurar la comunicación interna mediante DNS de Docker.
8. Verificar el acceso a la GPU.
9. Comprobar los servicios y sus logs.
10. Realizar copias de seguridad y actualizaciones.
11. Diagnosticar los errores más habituales.

### Servicios del proyecto

| Servicio | Contenedor | Puerto interno efectivo | Puerto externo | Función |
|---|---|---:|---:|---|
| Ollama | `ollama` | 11434 | 11434 | Motor de LLM y API |
| Open WebUI | `openwebui` | 8080 | 3000 | Interfaz web |
| Hermes Agent | `hermes-agent` | 8642 | 8000 | Agente autónomo / API |
| OpenCode | `opencode` | 4096 | 8443 | Entorno de desarrollo web |
| ComfyUI | `comfyui` | 8188 | 8188 | Generación de imágenes/vídeo |
| YOLO | `yolo` | 5000 | 5000 | API de visión artificial |
| SearXNG | `searxng` | 8080 | 8080 | Metabuscador |
| RAG | `rag` | 8000 | 11435 | API RAG |

> **Importante:** la especificación original asignaba `11434` también al RAG. Eso produciría una colisión de puertos en el host porque Ollama ya publica `11434`. En este manual el RAG utiliza `11435` en el host y `8000` dentro del contenedor. La comunicación Docker no depende del puerto publicado en el host.

---

# 1. Prerrequisitos e instalación base

## 1.1. Requisitos de hardware

La especificación establece:

- Ubuntu Server 24.04 o 26.04.
- GPU NVIDIA.
- Driver NVIDIA.
- Docker Engine.
- Docker Compose.
- Preferentemente arquitectura `amd64/x86_64`.

Una RTX 4060 es adecuada para el escenario descrito, pero la cantidad y tamaño de los modelos que pueden ejecutarse simultáneamente dependerán principalmente de la VRAM disponible.

## 1.2. Comprobar el sistema

```bash
cat /etc/os-release
uname -a
uname -m
free -h
df -h
```

Comprobar la GPU:

```bash
lspci | grep -i nvidia
```

Si el driver NVIDIA ya está instalado:

```bash
nvidia-smi
```

Debe aparecer la GPU, versión del driver, memoria utilizada y versión CUDA soportada por el driver.

---

# 2. Instalación del driver NVIDIA

## 2.1. Actualizar Ubuntu

```bash
sudo apt update
sudo apt upgrade -y
sudo reboot
```

Tras reiniciar:

```bash
sudo apt update
```

## 2.2. Detectar el driver recomendado

Ubuntu puede indicar el driver recomendado mediante:

```bash
ubuntu-drivers devices
```

Instalar el recomendado:

```bash
sudo ubuntu-drivers autoinstall
```

Reiniciar:

```bash
sudo reboot
```

Comprobar:

```bash
nvidia-smi
```

> No es necesario instalar el CUDA Toolkit completo del host para utilizar GPU desde Docker. Para este proyecto es especialmente importante que el **driver NVIDIA del host** y el **NVIDIA Container Toolkit** estén correctamente instalados.

---

# 3. Instalación de Docker Engine

## 3.1. Eliminar paquetes Docker potencialmente conflictivos

```bash
sudo apt remove -y docker.io docker-compose docker-compose-v2 docker-doc \
  docker-buildx docker-ce docker-ce-cli containerd runc podman-docker
```

Si alguno no está instalado, `apt` puede indicarlo; no supone un problema.

## 3.2. Instalar dependencias

```bash
sudo apt update
sudo apt install -y ca-certificates curl
```

Crear el directorio de claves:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

Descargar la clave oficial:

```bash
sudo curl -fsSL \
  https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Crear el repositorio:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

Actualizar:

```bash
sudo apt update
```

## 3.3. Instalar Docker Engine y Compose

```bash
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

Activar Docker:

```bash
sudo systemctl enable --now docker
```

Comprobar:

```bash
sudo systemctl status docker
docker --version
docker compose version
```

Prueba básica:

```bash
sudo docker run --rm hello-world
```

## 3.4. Permitir utilizar Docker sin `sudo`

Añadir el usuario actual al grupo `docker`:

```bash
sudo usermod -aG docker "$USER"
```

Cerrar sesión y volver a entrar.

Comprobar:

```bash
docker ps
```

> El grupo `docker` concede privilegios elevados equivalentes en la práctica a acceso administrativo al host. Solo debe utilizarse con usuarios de confianza.

---

# 4. Instalación de NVIDIA Container Toolkit

## 4.1. Dependencias

```bash
sudo apt-get update
sudo apt-get install -y --no-install-recommends \
  ca-certificates \
  curl \
  gnupg2
```

## 4.2. Repositorio NVIDIA

```bash
curl -fsSL \
  https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor \
  -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
```

```bash
curl -s -L \
  https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

Actualizar:

```bash
sudo apt-get update
```

Instalar:

```bash
sudo apt-get install -y nvidia-container-toolkit
```

## 4.3. Configurar Docker

```bash
sudo nvidia-ctk runtime configure --runtime=docker
```

Reiniciar Docker:

```bash
sudo systemctl restart docker
```

## 4.4. Prueba GPU dentro de un contenedor

```bash
docker run --rm --gpus all \
  nvidia/cuda:12.6.2-base-ubuntu24.04 \
  nvidia-smi
```

Si aparece la GPU dentro del contenedor, el paso crítico de GPU está funcionando.

---

# 5. Estructura del proyecto

El proyecto se ubicará en:

```text
$HOME/proyecto
```

Crear estructura:

```bash
mkdir -p "$HOME/proyecto"
cd "$HOME/proyecto"
```

Crear directorios:

```bash
mkdir -p \
  ollama \
  openwebui \
  hermes \
  opencode \
  comfyui/models \
  comfyui/input \
  comfyui/output \
  comfyui/user \
  comfyui/custom_nodes \
  yolo/data \
  searxng/core-config \
  searxng/data \
  rag/data \
  rag/app \
  services/yolo \
  services/rag \
  scripts \
  backups
```

Estructura final:

```text
$HOME/proyecto/
├── .env
├── README.md
├── docker_ollama.yml
├── docker_openwebui.yml
├── docker_hermes-agent.yml
├── docker_opencode.yml
├── docker_comfyui.yml
├── docker_yolo.yml
├── docker_searxng.yml
├── docker_rag.yml
│
├── ollama/
│
├── openwebui/
│
├── hermes/
│
├── opencode/
│
├── comfyui/
│   ├── models/
│   ├── input/
│   ├── output/
│   ├── user/
│   └── custom_nodes/
│
├── yolo/
│   └── data/
│
├── searxng/
│   ├── core-config/
│   └── data/
│
├── rag/
│   └── data/
│
├── services/
│   ├── yolo/
│   │   ├── Dockerfile
│   │   ├── requirements.txt
│   │   └── app.py
│   └── rag/
│       ├── Dockerfile
│       ├── requirements.txt
│       └── app.py
│
├── scripts/
└── backups/
```

---

# 6. Red Docker `red-ia`

Crear la red:

```bash
docker network create \
  --driver bridge \
  red-ia
```

Comprobar:

```bash
docker network inspect red-ia
```

Todos los servicios utilizarán esta red.

Dentro de Docker, el nombre del contenedor funciona como DNS:

```text
ollama:11434
openwebui:8080
hermes-agent:8642
opencode:4096
comfyui:8188
yolo:5000
searxng:8080
rag:8000
```

No utilizar `localhost` para comunicar un contenedor con otro. Por ejemplo, desde Open WebUI:

```text
http://ollama:11434
```

y no:

```text
http://localhost:11434
```

---

# 7. Fichero `.env`

Crear:

```bash
nano "$HOME/proyecto/.env"
```

Contenido inicial:

```dotenv
# ============================================================
# Proyecto IA local
# ============================================================

TZ=Europe/Madrid

# Host
PROJECT_DIR=${HOME}/proyecto

# Puertos publicados en el host
OLLAMA_PORT=11434
OPENWEBUI_PORT=3000
HERMES_PORT=8000
OPENCODE_PORT=8443
COMFYUI_PORT=8188
YOLO_PORT=5000
SEARXNG_PORT=8080
RAG_PORT=11435

# Puertos internos
OLLAMA_INTERNAL_PORT=11434
OPENWEBUI_INTERNAL_PORT=8080
HERMES_INTERNAL_PORT=8642
OPENCODE_INTERNAL_PORT=4096
COMFYUI_INTERNAL_PORT=8188
YOLO_INTERNAL_PORT=5000
SEARXNG_INTERNAL_PORT=8080
RAG_INTERNAL_PORT=8000

# GPU
NVIDIA_VISIBLE_DEVICES=all

# Open WebUI
WEBUI_SECRET_KEY=CAMBIAR_POR_UN_SECRETO_LARGO_Y_ALEATORIO
OLLAMA_BASE_URL=http://ollama:11434

# Hermes
HERMES_UID=1000
HERMES_GID=1000

# OpenCode
OPENCODE_SERVER_USERNAME=opencode
OPENCODE_SERVER_PASSWORD=CAMBIAR_POR_UNA_PASSWORD_LARGA

# RAG
RAG_OLLAMA_URL=http://ollama:11434
RAG_EMBEDDING_MODEL=nomic-embed-text
RAG_LLM_MODEL=llama3.2

# YOLO
YOLO_MODEL=yolo11n.pt
```

Generar un secreto seguro para `WEBUI_SECRET_KEY`:

```bash
openssl rand -hex 32
```

Generar una contraseña:

```bash
openssl rand -base64 32
```

No guardar `.env` en un repositorio Git.

Añadir:

```bash
printf ".env\nbackups/\n" >> .gitignore
```

---

# 8. Ollama

## 8.1. Fichero `docker_ollama.yml`

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped

    ports:
      - "${OLLAMA_PORT}:11434"

    volumes:
      - "${PROJECT_DIR}/ollama:/root/.ollama"

    environment:
      TZ: "${TZ}"
      NVIDIA_VISIBLE_DEVICES: "${NVIDIA_VISIBLE_DEVICES}"

    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

## 8.2. Arranque

```bash
cd "$HOME/proyecto"
docker compose -f docker_ollama.yml up -d
```

Comprobar:

```bash
docker ps --filter name=ollama
docker logs --tail 100 ollama
```

Comprobar API:

```bash
curl http://localhost:11434/api/tags
```

## 8.3. Descargar un modelo

Ejemplo:

```bash
docker exec -it ollama ollama pull llama3.2
```

Comprobar:

```bash
docker exec -it ollama ollama list
```

Probar:

```bash
docker exec -it ollama ollama run llama3.2
```

---

# 9. Open WebUI

Open WebUI utilizará Ollama mediante:

```text
http://ollama:11434
```

## 9.1. `docker_openwebui.yml`

```yaml
services:
  openwebui:
    image: ghcr.io/open-webui/open-webui:cuda
    container_name: openwebui
    restart: unless-stopped

    ports:
      - "${OPENWEBUI_PORT}:8080"

    volumes:
      - "${PROJECT_DIR}/openwebui:/app/backend/data"

    environment:
      TZ: "${TZ}"
      WEBUI_SECRET_KEY: "${WEBUI_SECRET_KEY}"
      OLLAMA_BASE_URL: "${OLLAMA_BASE_URL}"
      ENABLE_OLLAMA_API: "true"

    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

    depends_on:
      - ollama

    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

## 9.2. Arranque

```bash
docker compose -f docker_openwebui.yml up -d
```

Logs:

```bash
docker logs -f openwebui
```

Acceso:

```text
http://IP_DEL_SERVIDOR:3000
```

La URL interna entre contenedores es:

```text
http://openwebui:8080
```

---

# 10. Hermes Agent

La especificación original utiliza el nombre `hermes-agent` y el puerto externo `8000`.

La versión actual de Hermes Agent expone su gateway en el puerto interno `8642`; por ello se conserva `8000` como puerto del host mediante:

```text
8000:8642
```

## 10.1. `docker_hermes-agent.yml`

```yaml
services:
  hermes-agent:
    image: nousresearch/hermes-agent:latest
    container_name: hermes-agent
    restart: unless-stopped

    command: gateway run

    ports:
      - "${HERMES_PORT}:${HERMES_INTERNAL_PORT}"

    volumes:
      - "${PROJECT_DIR}/hermes:/opt/data"

    environment:
      TZ: "${TZ}"
      HERMES_UID: "${HERMES_UID}"
      HERMES_GID: "${HERMES_GID}"

    depends_on:
      - ollama
      - searxng
      - comfyui

    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

## 10.2. Configuración de Ollama para Hermes

Hermes debe utilizar la API compatible con OpenAI de Ollama:

```text
http://ollama:11434/v1
```

El modelo debe ser uno que exista previamente en Ollama.

La configuración de Hermes se conserva en:

```text
$HOME/proyecto/hermes
```

## 10.3. Arranque

```bash
docker compose -f docker_hermes-agent.yml up -d
```

Logs:

```bash
docker logs -f hermes-agent
```

Comprobar desde Hermes que Ollama responde:

```bash
docker exec hermes-agent \
  curl -s http://ollama:11434/api/tags
```

---

# 11. OpenCode

OpenCode dispone de modo web mediante:

```bash
opencode web
```

El servidor interno utiliza el puerto `4096` en la configuración actual.

El requisito del proyecto mantiene el puerto externo `8443`:

```text
8443:4096
```

## 11.1. `docker_opencode.yml`

```yaml
services:
  opencode:
    image: ghcr.io/anomalyco/opencode:2.0.7
    container_name: opencode
    restart: unless-stopped

    command:
      - web
      - --hostname
      - 0.0.0.0
      - --port
      - "4096"

    ports:
      - "${OPENCODE_PORT}:${OPENCODE_INTERNAL_PORT}"

    working_dir: /workspace

    volumes:
      - "${PROJECT_DIR}/opencode:/root/.local"
      - "${PROJECT_DIR}/opencode/config:/root/.config/opencode"
      - "${PROJECT_DIR}:/workspace/proyecto"

    environment:
      TZ: "${TZ}"
      OPENCODE_SERVER_USERNAME: "${OPENCODE_SERVER_USERNAME}"
      OPENCODE_SERVER_PASSWORD: "${OPENCODE_SERVER_PASSWORD}"

    depends_on:
      - ollama

    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

## 11.2. Acceso

```text
http://IP_DEL_SERVIDOR:8443
```

> El puerto `8443` es un puerto HTTP en este diseño. No debe interpretarse como HTTPS solo por utilizar el número 8443. Para TLS real debe añadirse un reverse proxy con certificado.

## 11.3. Conexión con Ollama

OpenCode puede trabajar con proveedores compatibles. Para una configuración con Ollama se debe configurar el proveedor de Ollama en `opencode.json` según la versión instalada de OpenCode.

La URL del servicio desde Docker es:

```text
http://ollama:11434
```

No usar:

```text
http://localhost:11434
```

---

# 12. ComfyUI

ComfyUI utiliza GPU NVIDIA.

## 12.1. `docker_comfyui.yml`

```yaml
services:
  comfyui:
    image: ghcr.io/lecode-official/comfyui-docker:latest
    container_name: comfyui
    restart: unless-stopped

    ports:
      - "${COMFYUI_PORT}:8188"

    environment:
      TZ: "${TZ}"
      USER_ID: "1000"
      GROUP_ID: "1000"

    volumes:
      - "${PROJECT_DIR}/comfyui/models:/opt/comfyui/models"
      - "${PROJECT_DIR}/comfyui/custom_nodes:/opt/comfyui/custom_nodes"
      - "${PROJECT_DIR}/comfyui/input:/opt/comfyui/input"
      - "${PROJECT_DIR}/comfyui/output:/opt/comfyui/output"
      - "${PROJECT_DIR}/comfyui/user:/opt/comfyui/user"

    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

## 12.2. Arranque

```bash
docker compose -f docker_comfyui.yml up -d
```

Acceso:

```text
http://IP_DEL_SERVIDOR:8188
```

Logs:

```bash
docker logs -f comfyui
```

---

# 13. YOLO

La especificación define YOLO como una API de visión artificial en el puerto `5000`, pero no proporciona una imagen Docker ni una implementación concreta de API.

Por ello se utiliza una pequeña API FastAPI propia sobre Ultralytics. Esto convierte la especificación en un servicio reproducible en Docker sin inventar una imagen externa de YOLO.

## 13.1. `services/yolo/requirements.txt`

```text
fastapi
uvicorn[standard]
python-multipart
ultralytics
pillow
```

## 13.2. `services/yolo/Dockerfile`

```dockerfile
FROM ultralytics/ultralytics:latest

WORKDIR /app

COPY requirements.txt /app/requirements.txt
RUN pip install --no-cache-dir -r /app/requirements.txt

COPY app.py /app/app.py

EXPOSE 5000

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "5000"]
```

## 13.3. `services/yolo/app.py`

```python
import os
from pathlib import Path

from fastapi import FastAPI, File, UploadFile, HTTPException
from ultralytics import YOLO
from PIL import Image

app = FastAPI(title="YOLO API", version="1.0")

MODEL_NAME = os.getenv("YOLO_MODEL", "yolo11n.pt")
MODEL_PATH = Path("/models") / MODEL_NAME

model = YOLO(str(MODEL_PATH) if MODEL_PATH.exists() else MODEL_NAME)


@app.get("/health")
def health():
    return {
        "status": "ok",
        "model": MODEL_NAME
    }


@app.post("/predict")
async def predict(file: UploadFile = File(...)):
    if not file.content_type or not file.content_type.startswith("image/"):
        raise HTTPException(
            status_code=400,
            detail="El fichero debe ser una imagen"
        )

    contents = await file.read()

    tmp = Path("/tmp") / file.filename
    tmp.write_bytes(contents)

    image = Image.open(tmp).convert("RGB")
    results = model.predict(image, verbose=False)

    detections = []

    for result in results:
        names = result.names

        if result.boxes is None:
            continue

        for box in result.boxes:
            cls = int(box.cls[0])
            confidence = float(box.conf[0])

            detections.append({
                "class_id": cls,
                "class_name": names[cls],
                "confidence": confidence,
                "xyxy": [float(x) for x in box.xyxy[0].tolist()]
            })

    return {
        "model": MODEL_NAME,
        "detections": detections
    }
```

## 13.4. `docker_yolo.yml`

```yaml
services:
  yolo:
    build:
      context: ./services/yolo

    container_name: yolo
    restart: unless-stopped

    ports:
      - "${YOLO_PORT}:5000"

    environment:
      TZ: "${TZ}"
      YOLO_MODEL: "${YOLO_MODEL}"

    volumes:
      - "${PROJECT_DIR}/yolo/data:/data"
      - "${PROJECT_DIR}/yolo/models:/models"

    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

Crear también el directorio de modelos:

```bash
mkdir -p "$HOME/proyecto/yolo/models"
```

Arrancar:

```bash
docker compose -f docker_yolo.yml up -d --build
```

Comprobar:

```bash
curl http://localhost:5000/health
```

---

# 14. SearXNG

SearXNG no necesita GPU para realizar búsquedas. La especificación lo incluye junto al resto de servicios, pero el driver NVIDIA no es una dependencia funcional de SearXNG.

## 14.1. Configuración mínima

Crear:

```bash
nano "$HOME/proyecto/searxng/core-config/settings.yml"
```

Ejemplo:

```yaml
use_default_settings: true

general:
  instance_name: "SearXNG IA Local"

server:
  bind_address: "0.0.0.0"
  port: 8080
  secret_key: "CAMBIAR_POR_UN_SECRETO_LARGO"
  limiter: false
  image_proxy: true

search:
  safe_search: 0
  autocomplete: ""
  formats:
    - html
    - json
```

Generar secreto:

```bash
openssl rand -hex 32
```

Sustituir el valor de `secret_key`.

## 14.2. `docker_searxng.yml`

```yaml
services:
  searxng:
    image: searxng/searxng:latest
    container_name: searxng
    restart: unless-stopped

    ports:
      - "${SEARXNG_PORT}:8080"

    volumes:
      - "${PROJECT_DIR}/searxng/core-config:/etc/searxng"
      - "${PROJECT_DIR}/searxng/data:/var/cache/searxng"

    environment:
      TZ: "${TZ}"

    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

Arrancar:

```bash
docker compose -f docker_searxng.yml up -d
```

Acceso:

```text
http://IP_DEL_SERVIDOR:8080
```

Prueba:

```bash
curl -I http://localhost:8080
```

---

# 15. RAG

## 15.1. Objetivo

El servicio RAG permite:

1. Recibir documentos.
2. Dividirlos en fragmentos.
3. Generar embeddings mediante Ollama.
4. Almacenar los embeddings en una base vectorial.
5. Recuperar los fragmentos más relevantes.
6. Utilizar Ollama para generar una respuesta basada en el contexto recuperado.

El RAG es una técnica de integración y no un LLM independiente.

## 15.2. Dependencias

`services/rag/requirements.txt`:

```text
fastapi
uvicorn[standard]
python-multipart
chromadb
httpx
pypdf
```

## 15.3. Dockerfile

`services/rag/Dockerfile`:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

RUN apt-get update \
    && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt /app/requirements.txt

RUN pip install --no-cache-dir -r /app/requirements.txt

COPY app.py /app/app.py

EXPOSE 8000

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

## 15.4. API RAG

`services/rag/app.py`:

```python
import os
from pathlib import Path

import chromadb
import httpx

from fastapi import FastAPI, UploadFile, File, HTTPException
from pypdf import PdfReader

app = FastAPI(title="RAG API", version="1.0")

DATA_DIR = Path("/data")
DATA_DIR.mkdir(parents=True, exist_ok=True)

OLLAMA_URL = os.getenv(
    "RAG_OLLAMA_URL",
    "http://ollama:11434"
)

EMBEDDING_MODEL = os.getenv(
    "RAG_EMBEDDING_MODEL",
    "nomic-embed-text"
)

LLM_MODEL = os.getenv(
    "RAG_LLM_MODEL",
    "llama3.2"
)

client = chromadb.PersistentClient(path=str(DATA_DIR / "chroma"))
collection = client.get_or_create_collection("documentos")


def split_text(text: str, size: int = 1000, overlap: int = 150):
    chunks = []
    start = 0

    while start < len(text):
        end = start + size
        chunks.append(text[start:end])
        start += size - overlap

    return chunks


async def embedding(text: str):
    async with httpx.AsyncClient(timeout=120) as http:
        response = await http.post(
            f"{OLLAMA_URL}/api/embed",
            json={
                "model": EMBEDDING_MODEL,
                "input": text
            }
        )
        response.raise_for_status()
        data = response.json()

    return data["embeddings"][0]


@app.get("/health")
def health():
    return {
        "status": "ok",
        "ollama": OLLAMA_URL,
        "embedding_model": EMBEDDING_MODEL,
        "llm_model": LLM_MODEL
    }


@app.post("/ingest")
async def ingest(file: UploadFile = File(...)):
    if not file.filename:
        raise HTTPException(status_code=400, detail="Nombre de fichero vacío")

    content = await file.read()

    if file.filename.lower().endswith(".pdf"):
        tmp = DATA_DIR / file.filename
        tmp.write_bytes(content)

        reader = PdfReader(str(tmp))
        text = "\n".join(page.extract_text() or "" for page in reader.pages)

    elif file.filename.lower().endswith((".txt", ".md")):
        text = content.decode("utf-8", errors="replace")

    else:
        raise HTTPException(
            status_code=400,
            detail="Solo se admiten PDF, TXT y MD"
        )

    chunks = split_text(text)

    ids = []
    documents = []
    embeddings = []

    for index, chunk in enumerate(chunks):
        ids.append(f"{file.filename}-{index}")
        documents.append(chunk)
        embeddings.append(await embedding(chunk))

    collection.upsert(
        ids=ids,
        documents=documents,
        embeddings=embeddings,
        metadatas=[
            {"source": file.filename, "chunk": i}
            for i in range(len(chunks))
        ]
    )

    return {
        "filename": file.filename,
        "chunks": len(chunks)
    }


@app.get("/search")
async def search(q: str, n: int = 5):
    vector = await embedding(q)

    result = collection.query(
        query_embeddings=[vector],
        n_results=n
    )

    return result


@app.get("/ask")
async def ask(q: str, n: int = 5):
    vector = await embedding(q)

    result = collection.query(
        query_embeddings=[vector],
        n_results=n
    )

    documents = result.get("documents", [[]])[0]
    context = "\n\n---\n\n".join(documents)

    prompt = f"""Responde utilizando únicamente el contexto proporcionado.

Contexto:
{context}

Pregunta:
{q}
"""

    async with httpx.AsyncClient(timeout=300) as http:
        response = await http.post(
            f"{OLLAMA_URL}/api/generate",
            json={
                "model": LLM_MODEL,
                "prompt": prompt,
                "stream": False
            }
        )
        response.raise_for_status()
        data = response.json()

    return {
        "answer": data.get("response", ""),
        "sources": result.get("metadatas", [[]])[0]
    }
```

## 15.5. `docker_rag.yml`

```yaml
services:
  rag:
    build:
      context: ./services/rag

    container_name: rag
    restart: unless-stopped

    ports:
      - "${RAG_PORT}:8000"

    volumes:
      - "${PROJECT_DIR}/rag/data:/data"

    environment:
      TZ: "${TZ}"
      RAG_OLLAMA_URL: "${RAG_OLLAMA_URL}"
      RAG_EMBEDDING_MODEL: "${RAG_EMBEDDING_MODEL}"
      RAG_LLM_MODEL: "${RAG_LLM_MODEL}"

    depends_on:
      - ollama

    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

## 15.6. Preparar modelos de RAG

Descargar el modelo de embeddings:

```bash
docker exec -it ollama ollama pull nomic-embed-text
```

Descargar el modelo generativo:

```bash
docker exec -it ollama ollama pull llama3.2
```

Arrancar RAG:

```bash
docker compose -f docker_rag.yml up -d --build
```

Comprobar:

```bash
curl http://localhost:11435/health
```

---

# 16. Despliegue completo

## 16.1. Comprobar sintaxis

Antes de arrancar:

```bash
cd "$HOME/proyecto"

docker compose -f docker_ollama.yml config
docker compose -f docker_openwebui.yml config
docker compose -f docker_hermes-agent.yml config
docker compose -f docker_opencode.yml config
docker compose -f docker_comfyui.yml config
docker compose -f docker_yolo.yml config
docker compose -f docker_searxng.yml config
docker compose -f docker_rag.yml config
```

Si no aparece ningún error, las definiciones Compose son sintácticamente válidas.

## 16.2. Arranque recomendado

Primero la red:

```bash
docker network inspect red-ia >/dev/null 2>&1 || \
  docker network create --driver bridge red-ia
```

Después Ollama:

```bash
docker compose -f docker_ollama.yml up -d
```

Después SearXNG:

```bash
docker compose -f docker_searxng.yml up -d
```

Después ComfyUI:

```bash
docker compose -f docker_comfyui.yml up -d
```

Después Open WebUI:

```bash
docker compose -f docker_openwebui.yml up -d
```

Después Hermes:

```bash
docker compose -f docker_hermes-agent.yml up -d
```

Después OpenCode:

```bash
docker compose -f docker_opencode.yml up -d
```

Después YOLO:

```bash
docker compose -f docker_yolo.yml up -d --build
```

Finalmente RAG:

```bash
docker compose -f docker_rag.yml up -d --build
```

---

# 17. Comprobación global

## 17.1. Contenedores

```bash
docker ps
```

Deben aparecer:

```text
ollama
openwebui
hermes-agent
opencode
comfyui
yolo
searxng
rag
```

## 17.2. Red

```bash
docker network inspect red-ia
```

## 17.3. GPU

```bash
nvidia-smi
```

Para ver el consumo en tiempo real:

```bash
watch -n 1 nvidia-smi
```

## 17.4. GPU desde Docker

```bash
docker exec ollama nvidia-smi
```

Comprobar también ComfyUI:

```bash
docker exec comfyui nvidia-smi
```

Comprobar YOLO:

```bash
docker exec yolo nvidia-smi
```

---

# 18. Tabla de acceso

| Servicio | URL desde el host |
|---|---|
| Ollama | `http://IP:11434` |
| Open WebUI | `http://IP:3000` |
| Hermes Agent | `http://IP:8000` |
| OpenCode | `http://IP:8443` |
| ComfyUI | `http://IP:8188` |
| YOLO | `http://IP:5000` |
| SearXNG | `http://IP:8080` |
| RAG | `http://IP:11435` |

URLs internas Docker:

| Servicio cliente | Servicio destino |
|---|---|
| Open WebUI | `http://ollama:11434` |
| Hermes | `http://ollama:11434/v1` |
| Hermes | `http://searxng:8080` |
| Hermes | `http://comfyui:8188` |
| OpenCode | `http://ollama:11434` |
| RAG | `http://ollama:11434` |
| YOLO | `http://yolo:5000` |

---

# 19. Pruebas de integración

## 19.1. Ollama

```bash
curl http://localhost:11434/api/tags
```

Debe devolver JSON con los modelos instalados.

## 19.2. Open WebUI → Ollama

Desde el contenedor:

```bash
docker exec openwebui \
  curl -s http://ollama:11434/api/tags
```

## 19.3. Hermes → Ollama

```bash
docker exec hermes-agent \
  curl -s http://ollama:11434/api/tags
```

## 19.4. Hermes → SearXNG

```bash
docker exec hermes-agent \
  curl -s http://searxng:8080
```

## 19.5. OpenCode → Ollama

```bash
docker exec opencode \
  wget -qO- http://ollama:11434/api/tags
```

Si la imagen no incluye `wget`, utilizar una herramienta disponible o realizar la prueba desde otro contenedor de diagnóstico conectado a `red-ia`.

## 19.6. RAG → Ollama

```bash
docker exec rag \
  python -c "import urllib.request; print(urllib.request.urlopen('http://ollama:11434/api/tags').read().decode())"
```

## 19.7. YOLO

```bash
curl http://localhost:5000/health
```

Respuesta esperada:

```json
{
  "status": "ok",
  "model": "yolo11n.pt"
}
```

## 19.8. RAG

```bash
curl http://localhost:11435/health
```

---

# 20. Mantenimiento

## 20.1. Ver estado

```bash
docker ps
docker stats
```

## 20.2. Ver logs

```bash
docker logs --tail 200 ollama
docker logs --tail 200 openwebui
docker logs --tail 200 hermes-agent
docker logs --tail 200 opencode
docker logs --tail 200 comfyui
docker logs --tail 200 yolo
docker logs --tail 200 searxng
docker logs --tail 200 rag
```

Seguir logs:

```bash
docker logs -f ollama
```

## 20.3. Reiniciar un servicio

```bash
docker restart ollama
```

O mediante Compose:

```bash
docker compose -f docker_ollama.yml restart
```

## 20.4. Detener un servicio

```bash
docker compose -f docker_ollama.yml down
```

Los directorios montados en el host no se eliminan.

---

# 21. Actualización

## 21.1. Actualizar imágenes

Antes de actualizar:

```bash
cd "$HOME/proyecto"

docker compose -f docker_ollama.yml pull
docker compose -f docker_openwebui.yml pull
docker compose -f docker_hermes-agent.yml pull
docker compose -f docker_opencode.yml pull
docker compose -f docker_comfyui.yml pull
docker compose -f docker_searxng.yml pull
```

Después:

```bash
docker compose -f docker_ollama.yml up -d
docker compose -f docker_openwebui.yml up -d
docker compose -f docker_hermes-agent.yml up -d
docker compose -f docker_opencode.yml up -d
docker compose -f docker_comfyui.yml up -d
docker compose -f docker_searxng.yml up -d
```

Para imágenes construidas localmente:

```bash
docker compose -f docker_yolo.yml build --pull
docker compose -f docker_yolo.yml up -d

docker compose -f docker_rag.yml build --pull
docker compose -f docker_rag.yml up -d
```

## 21.2. Actualizar modelos Ollama

Listar:

```bash
docker exec ollama ollama list
```

Actualizar un modelo:

```bash
docker exec ollama ollama pull llama3.2
```

> No actualizar todos los modelos automáticamente sin comprobar antes el espacio disponible en disco y VRAM.

---

# 22. Copias de seguridad

Los datos persistentes están en:

```text
$HOME/proyecto/
```

Crear una copia:

```bash
cd "$HOME"

tar \
  --exclude='proyecto/backups' \
  -czf \
  "proyecto/backups/proyecto_$(date +%Y-%m-%d_%H-%M-%S).tar.gz" \
  proyecto
```

Comprobar:

```bash
ls -lh "$HOME/proyecto/backups"
```

## 22.1. Backup específico de Ollama

```bash
tar -czf \
  "$HOME/proyecto/backups/ollama_$(date +%Y-%m-%d).tar.gz" \
  "$HOME/proyecto/ollama"
```

## 22.2. Backup de Open WebUI

```bash
tar -czf \
  "$HOME/proyecto/backups/openwebui_$(date +%Y-%m-%d).tar.gz" \
  "$HOME/proyecto/openwebui"
```

## 22.3. Backup de Hermes

```bash
tar -czf \
  "$HOME/proyecto/backups/hermes_$(date +%Y-%m-%d).tar.gz" \
  "$HOME/proyecto/hermes"
```

## 22.4. Backup de ComfyUI

```bash
tar -czf \
  "$HOME/proyecto/backups/comfyui_$(date +%Y-%m-%d).tar.gz" \
  "$HOME/proyecto/comfyui"
```

---

# 23. Errores comunes

## 23.1. `permission denied`

Comprobar propietario:

```bash
ls -la "$HOME/proyecto"
```

Corregir propietario de directorios creados por el usuario:

```bash
sudo chown -R "$USER":"$USER" "$HOME/proyecto"
```

Para Hermes:

```bash
sudo chown -R "$USER":"$USER" "$HOME/proyecto/hermes"
```

Para ComfyUI:

```bash
sudo chown -R "$USER":"$USER" "$HOME/proyecto/comfyui"
```

## 23.2. Docker no permite ejecutar sin `sudo`

Comprobar grupo:

```bash
groups
```

Si no aparece `docker`:

```bash
sudo usermod -aG docker "$USER"
```

Cerrar sesión y volver a entrar.

## 23.3. La GPU no aparece

Primero:

```bash
nvidia-smi
```

Si falla, el problema está en el host/driver.

Si funciona:

```bash
docker run --rm --gpus all \
  nvidia/cuda:12.6.2-base-ubuntu24.04 \
  nvidia-smi
```

Si falla dentro del contenedor:

```bash
nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

## 23.4. Puerto ocupado

Comprobar:

```bash
sudo ss -lntp
```

Por ejemplo:

```bash
sudo ss -lntp | grep ':3000'
```

La lista de puertos de este proyecto debe ser:

```text
11434  Ollama
3000   Open WebUI
8000   Hermes
8080   SearXNG
8188   ComfyUI
8443   OpenCode
5000   YOLO
11435  RAG
```

## 23.5. Ollama funciona pero Open WebUI no conecta

Comprobar desde Open WebUI:

```bash
docker exec openwebui \
  curl -s http://ollama:11434/api/tags
```

Si responde, la red funciona.

Comprobar variable:

```bash
docker inspect openwebui \
  --format '{{range .Config.Env}}{{println .}}{{end}}' \
  | grep OLLAMA
```

Debe aparecer:

```text
OLLAMA_BASE_URL=http://ollama:11434
```

## 23.6. Un contenedor utiliza `localhost` incorrectamente

Dentro de un contenedor:

```text
localhost
127.0.0.1
```

significan **ese mismo contenedor**.

Para hablar con otro servicio:

```text
http://ollama:11434
http://searxng:8080
http://comfyui:8188
http://rag:8000
```

## 23.7. OpenCode no abre por el navegador

Comprobar:

```bash
docker logs opencode
```

Debe indicar que escucha en:

```text
0.0.0.0:4096
```

Comprobar publicación:

```bash
docker port opencode
```

Debe aparecer una correspondencia equivalente a:

```text
4096/tcp -> 0.0.0.0:8443
```

## 23.8. SearXNG no arranca

Comprobar:

```bash
docker logs searxng
```

Validar configuración:

```bash
docker exec searxng \
  python -m searxng.checker
```

Si la versión instalada no incluye ese comando, revisar:

```bash
docker logs --tail 200 searxng
```

y validar `settings.yml`.

---

# 24. Seguridad

## 24.1. No publicar todos los servicios directamente a Internet

Esta pila está pensada inicialmente para una red local.

Evitar abrir directamente a Internet:

```text
11434
3000
8000
8080
8188
8443
5000
11435
```

Si se necesita acceso remoto, utilizar una VPN o un reverse proxy con autenticación y HTTPS.

## 24.2. OpenCode

OpenCode puede funcionar sin autenticación si no se establece contraseña.

En este proyecto se recomienda:

```dotenv
OPENCODE_SERVER_USERNAME=opencode
OPENCODE_SERVER_PASSWORD=CAMBIAR_POR_UNA_PASSWORD_LARGA
```

## 24.3. Secretos

Nunca guardar en Git:

```text
.env
API keys
contraseñas
tokens
certificados privados
```

## 24.4. Firewall

Docker puede publicar puertos de forma que las reglas UFW tradicionales no se comporten como el administrador espera.

Antes de exponer el servidor fuera de la LAN, diseñar explícitamente la política de filtrado y comprobar la cadena `DOCKER-USER`.

---

# 25. Integración exacta entre servicios

## 25.1. Open WebUI → Ollama

```text
Open WebUI
    |
    | HTTP
    v
http://ollama:11434
    |
    v
Ollama
    |
    v
GPU NVIDIA
```

Variable:

```dotenv
OLLAMA_BASE_URL=http://ollama:11434
```

## 25.2. Hermes → Ollama

```text
Hermes Agent
    |
    | OpenAI-compatible API
    v
http://ollama:11434/v1
    |
    v
Ollama
```

## 25.3. Hermes → SearXNG

```text
Hermes Agent
    |
    v
http://searxng:8080
    |
    v
Internet / motores de búsqueda
```

## 25.4. Hermes → ComfyUI

```text
Hermes Agent
    |
    v
http://comfyui:8188
    |
    v
ComfyUI
    |
    v
GPU NVIDIA
```

La integración exacta de workflows dependerá de los workflows/nodos instalados en ComfyUI.

## 25.5. OpenCode → Ollama

```text
OpenCode
    |
    v
http://ollama:11434
    |
    v
Ollama
    |
    v
GPU
```

La configuración del proveedor debe ajustarse al formato de configuración de la versión instalada de OpenCode.

## 25.6. RAG → Ollama

```text
Cliente
   |
   v
RAG API
   |
   +--> Embeddings --> Ollama
   |
   +--> Vector DB --> ChromaDB
   |
   +--> Prompt + contexto --> Ollama
```

## 25.7. YOLO

```text
Cliente
   |
   v
http://yolo:5000/predict
   |
   v
Ultralytics YOLO
   |
   v
GPU NVIDIA
```

---

# 26. Script de comprobación

Crear:

```bash
nano "$HOME/proyecto/scripts/check_stack.sh"
```

Contenido:

```bash
#!/usr/bin/env bash

set -u

echo "========================================"
echo " Comprobación de pila IA local"
echo "========================================"

echo
echo "[1] Docker"
docker --version
docker compose version

echo
echo "[2] GPU"
nvidia-smi --query-gpu=name,driver_version,memory.total,memory.used \
  --format=csv

echo
echo "[3] Contenedores"
docker ps \
  --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'

echo
echo "[4] Red red-ia"
docker network inspect red-ia \
  --format '{{range .Containers}}{{.Name}}{{"\n"}}{{end}}' 2>/dev/null \
  || echo "La red red-ia no existe"

echo
echo "[5] Ollama"
curl -fsS http://localhost:11434/api/tags \
  >/dev/null && echo "OK" || echo "ERROR"

echo
echo "[6] Open WebUI"
curl -fsSI http://localhost:3000 \
  >/dev/null && echo "OK" || echo "ERROR"

echo
echo "[7] Hermes"
curl -fsS http://localhost:8000 \
  >/dev/null && echo "OK" || echo "REVISAR"

echo
echo "[8] OpenCode"
curl -fsSI http://localhost:8443 \
  >/dev/null && echo "OK" || echo "REVISAR"

echo
echo "[9] ComfyUI"
curl -fsSI http://localhost:8188 \
  >/dev/null && echo "OK" || echo "ERROR"

echo
echo "[10] YOLO"
curl -fsS http://localhost:5000/health \
  >/dev/null && echo "OK" || echo "ERROR"

echo
echo "[11] SearXNG"
curl -fsSI http://localhost:8080 \
  >/dev/null && echo "OK" || echo "ERROR"

echo
echo "[12] RAG"
curl -fsS http://localhost:11435/health \
  >/dev/null && echo "OK" || echo "ERROR"

echo
echo "Comprobación finalizada."
```

Dar permisos:

```bash
chmod +x "$HOME/proyecto/scripts/check_stack.sh"
```

Ejecutar:

```bash
"$HOME/proyecto/scripts/check_stack.sh"
```

---

# 27. Orden recomendado de resolución de problemas

Cuando algo no funcione, seguir siempre este orden:

## Paso 1 — ¿Está Docker funcionando?

```bash
systemctl is-active docker
```

## Paso 2 — ¿Existe el contenedor?

```bash
docker ps -a
```

## Paso 3 — ¿El contenedor está arrancado?

```bash
docker ps
```

## Paso 4 — ¿Hay errores?

```bash
docker logs --tail 200 NOMBRE_CONTENEDOR
```

## Paso 5 — ¿Existe la red?

```bash
docker network inspect red-ia
```

## Paso 6 — ¿Hay conectividad interna?

Ejemplo:

```bash
docker exec openwebui \
  curl -s http://ollama:11434/api/tags
```

## Paso 7 — ¿La GPU funciona?

```bash
nvidia-smi
```

y:

```bash
docker run --rm --gpus all \
  nvidia/cuda:12.6.2-base-ubuntu24.04 \
  nvidia-smi
```

## Paso 8 — ¿Existe el puerto en el host?

```bash
sudo ss -lntp
```

---

# 28. Criterios de aceptación

## 28.1. Configuración Docker

Los ocho ficheros Compose deben pasar:

```bash
docker compose -f docker_ollama.yml config
docker compose -f docker_openwebui.yml config
docker compose -f docker_hermes-agent.yml config
docker compose -f docker_opencode.yml config
docker compose -f docker_comfyui.yml config
docker compose -f docker_yolo.yml config
docker compose -f docker_searxng.yml config
docker compose -f docker_rag.yml config
```

## 28.2. GPU

Debe funcionar:

```bash
nvidia-smi
```

y:

```bash
docker run --rm --gpus all \
  nvidia/cuda:12.6.2-base-ubuntu24.04 \
  nvidia-smi
```

## 28.3. Ausencia de colisiones

Puertos externos definidos:

```text
3000
5000
8000
8080
8188
8443
11434
11435
```

No hay dos servicios publicando el mismo puerto.

## 28.4. Red

Todos los servicios deben aparecer conectados a:

```text
red-ia
```

## 28.5. Persistencia

Los datos importantes deben permanecer después de:

```bash
docker compose -f <fichero> down
```

y:

```bash
docker compose -f <fichero> up -d
```

## 28.6. Accesibilidad

Debe ser posible acceder a:

```text
http://IP:3000
http://IP:8000
http://IP:8080
http://IP:8188
http://IP:8443
http://IP:5000
http://IP:11435
```

y a la API de Ollama en:

```text
http://IP:11434
```

---

# 29. Observaciones de diseño respecto a la especificación original

## 29.1. Puerto RAG

La especificación original indicaba:

```text
Ollama -> 11434:11434
RAG    -> 11434:11434
```

Esto no es posible simultáneamente en la misma IP del host.

Se ha resuelto:

```text
Ollama -> 11434:11434
RAG    -> 11435:8000
```

## 29.2. Hermes

La especificación indicaba puerto interno `8000`. La implementación actual de Hermes Gateway utiliza `8642`.

Se conserva el requisito de puerto externo `8000`:

```text
8000:8642
```

## 29.3. OpenCode

La especificación indicaba:

```text
8080 interno
8443 externo
```

La implementación actual de OpenCode utiliza `4096` como puerto predeterminado del servidor.

Se utiliza:

```text
8443:4096
```

## 29.4. SearXNG

La especificación asociaba SearXNG a GPU NVIDIA. SearXNG no necesita GPU para efectuar búsquedas. Se mantiene en la misma red, pero sin reservar GPU.

## 29.5. YOLO

La especificación no define una API, imagen o framework concreto. Se ha definido una API mínima con FastAPI + Ultralytics para que el contenedor tenga un contrato operativo claro:

```text
GET  /health
POST /predict
```

## 29.6. RAG

La especificación define RAG como técnica y no como producto concreto. Se ha materializado como servicio FastAPI con:

- ChromaDB persistente.
- Embeddings mediante Ollama.
- Generación mediante Ollama.
- Soporte PDF/TXT/Markdown.
- Endpoints `/ingest`, `/search`, `/ask` y `/health`.

---

# 30. Checklist final del administrador

Marcar cada elemento:

- [ ] Ubuntu instalado y actualizado.
- [ ] GPU NVIDIA detectada.
- [ ] `nvidia-smi` funciona en el host.
- [ ] Docker Engine instalado.
- [ ] `docker compose version` funciona.
- [ ] NVIDIA Container Toolkit instalado.
- [ ] `docker run --gpus all ... nvidia-smi` funciona.
- [ ] Directorio `$HOME/proyecto` creado.
- [ ] Red `red-ia` creada.
- [ ] `.env` creado y protegido.
- [ ] `docker_ollama.yml` validado.
- [ ] `docker_openwebui.yml` validado.
- [ ] `docker_hermes-agent.yml` validado.
- [ ] `docker_opencode.yml` validado.
- [ ] `docker_comfyui.yml` validado.
- [ ] `docker_yolo.yml` validado.
- [ ] `docker_searxng.yml` validado.
- [ ] `docker_rag.yml` validado.
- [ ] Ollama funcionando.
- [ ] Al menos un LLM descargado.
- [ ] Open WebUI funcionando.
- [ ] Hermes funcionando.
- [ ] OpenCode funcionando.
- [ ] ComfyUI funcionando.
- [ ] YOLO respondiendo `/health`.
- [ ] SearXNG funcionando.
- [ ] RAG respondiendo `/health`.
- [ ] GPU visible desde los contenedores que la necesitan.
- [ ] Backups configurados.
- [ ] Contraseñas y secretos modificados.
- [ ] No existen puertos publicados duplicados.
- [ ] No se han expuesto servicios innecesariamente a Internet.

---

# 31. Referencias técnicas

Este manual conserva la arquitectura y requisitos del documento SDD proporcionado y ha sido contrastado con la documentación técnica actual de los componentes principales:

- Docker Engine para Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- Docker Compose: https://docs.docker.com/compose/install/linux/
- NVIDIA Container Toolkit: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html
- Open WebUI: https://docs.openwebui.com/
- SearXNG: https://docs.searxng.org/admin/installation-docker
- Hermes Agent: https://github.com/NousResearch/hermes-agent
- OpenCode: https://opencode.ai/docs/
- Ultralytics YOLO Docker: https://docs.ultralytics.com/guides/docker-quickstart/
- ComfyUI Docker: https://github.com/lecode-official/comfyui-docker

> Las etiquetas `latest` se utilizan para simplificar el despliegue inicial. En un entorno de producción se recomienda fijar versiones concretas o digests y probar las actualizaciones antes de aplicarlas.

---

# 32. Fin del manual

La instalación se considera correctamente desplegada cuando todos los servicios están `Up`, la red `red-ia` permite resolver los nombres de servicio, Ollama responde, la GPU es visible dentro de los contenedores que la necesitan y las pruebas de integración descritas en este documento son satisfactorias.
