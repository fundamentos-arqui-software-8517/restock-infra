# Guía de Despliegue en VM de Azure - Plataforma Restock

Esta guía detalla los pasos para realizar un despliegue de nivel de producción "Lift and Shift" de la plataforma **Restock** en una Máquina Virtual (VM) de Azure, configurando el Gateway Nginx con SSL/TLS (Let's Encrypt), persistencia de datos y el broker MQTT.

---

## 1. Arquitectura de Despliegue en Azure

La arquitectura consta de una única red virtual virtualizada (`restock-net`) gestionada por Docker Compose dentro de la VM, aislando los componentes críticos de base de datos e inyectando Nginx como único punto de entrada:

* **Público (Expueto al exterior)**:
  * Puerto `80` (HTTP) y `443` (HTTPS) para el cliente web (Angular) y consumo de API.
  * Puerto `1883` (TCP) para que los sensores y dispositivos IoT reporten mediciones por MQTT.
* **Privado (Aislado internamente)**:
  * MongoDB (`27017`), Redis (`6379`), Mosquitto WebSockets (`9001`) y el Backend Spring Boot (`8080`).

---

## 2. Aprovisionamiento de la VM en Azure

### Configuración Recomendada
* **Sistema Operativo**: Ubuntu 22.04 LTS o Ubuntu 24.04 LTS.
* **Tamaño**: `Standard_B2ms` (2 vCPUs, 8 GB RAM) o mínimo `Standard_B2s` (2 vCPUs, 4 GB RAM).
* **IP Pública**: Configurar como **Estática** en Azure para evitar que cambie al reiniciar la VM.
* **DNS Name Label**: Asignar un nombre DNS en Azure (ej. `restock-platform.eastus.cloudapp.azure.com`) o apuntar un dominio propio (`restock.tudominio.com`) al IP público de la VM.

### Reglas del Grupo de Seguridad de Red (NSG)
Crea las siguientes reglas de entrada en el grupo de seguridad de red en Azure:

| Prioridad | Nombre | Puerto Destino | Protocolo | Origen | Descripción |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 100 | SSH | `22` | TCP | Tu IP Pública | Acceso administrativo seguro |
| 110 | HTTP | `80` | TCP | Any | Tráfico web estándar / Let's Encrypt |
| 120 | HTTPS | `443` | TCP | Any | Tráfico web seguro (Recomendado) |
| 130 | MQTT_TCP | `1883` | TCP | Any | Conexión de sensores IoT físicos |

---

## 3. Preparación de la Máquina Virtual

Conéctate por SSH a la VM e instala las dependencias necesarias:

```bash
# 1. Actualizar el sistema operativo
sudo apt-get update && sudo apt-get upgrade -y

# 2. Instalar Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 3. Habilitar que docker se ejecute sin sudo (opcional pero recomendado)
sudo usermod -aG docker $USER
newgrp docker

# 4. Habilitar el inicio automático de Docker
sudo systemctl enable docker
sudo systemctl start docker

# 5. Instalar Node.js y npm (necesarios para compilar el frontend Angular en la VM si no usas CI/CD externo)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
```

---

## 4. Estructura de Directorios en la VM

Dado que el `docker-compose.yml` busca los archivos compilados del frontend en una ruta relativa adyacente (`../restock-web-application/dist`), debes clonar ambos repositorios en el mismo directorio principal en la VM (por ejemplo, en tu carpeta `/home/azureuser/`):

```
/home/azureuser/
│
├── restock-infra/               <-- Repositorio de Infraestructura (clonado)
│   ├── docker-compose.yml
│   ├── .env
│   ├── nginx/
│   └── mosquitto/
│
└── restock-web-application/     <-- Repositorio del Frontend Angular (clonado)
    ├── package.json
    ├── dist/
    └── ...
```

### Pasos en la VM:

```bash
# Clonar repositorios
git clone https://github.com/tu-usuario/restock-infra.git /home/azureuser/restock-infra
git clone https://github.com/tu-usuario/restock-web-application.git /home/azureuser/restock-web-application

# Entrar al frontend y compilar
cd /home/azureuser/restock-web-application
npm install
npm run build
```

---

## 5. Configuración de Variables de Entorno

En el directorio de infraestructura `/home/azureuser/restock-infra`:

1. Genera el archivo de entorno definitivo:
   ```bash
   cp .env.example .env
   ```
2. Edita el archivo `.env` utilizando un editor como `nano`:
   ```bash
   nano .env
   ```
3. **IMPORTANTE: Define credenciales y secretos seguros en producción**:
   * Cambia `JWT_SECRET` por una cadena de caracteres aleatoria y larga de al menos 256 bits.
   * Completa tus credenciales reales de **Cloudinary** (`CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET`).
   * Mantén `MQTT_BROKER_URL=tcp://mosquitto:1883` para comunicación interna segura de red.
   * Configura la versión definitiva del backend (`BACKEND_IMAGE=julioxc4/restock-web-service:latest`).

---

## 6. Despliegue de la Aplicación

Para descargar las últimas imágenes del backend y levantar la plataforma completa:

```bash
cd /home/azureuser/restock-infra

# Descargar las imágenes oficiales
docker compose pull

# Levantar los contenedores en segundo plano
docker compose up -d
```

Verifica que todos los servicios estén corriendo correctamente:
```bash
docker compose ps
```

---

## 7. Configuración de SSL/TLS (Seguridad HTTPS y WSS)

Para que el navegador permita conexiones WebSocket y de autenticación seguras, es mandatorio habilitar SSL. Usaremos Certbot para obtener certificados gratuitos de Let's Encrypt.

### Paso 1: Instalar Certbot
```bash
sudo apt-get install -y certbot
```

### Paso 2: Obtener el certificado (Modo temporal)
Detén el contenedor de Nginx temporalmente para liberar el puerto 80:
```bash
docker compose stop nginx
sudo certbot certonly --standalone -d restock.tudominio.com
```

### Paso 3: Configurar Nginx para usar SSL
Crea un volumen de certificados en el `docker-compose.yml` para Nginx montando la carpeta `/etc/letsencrypt` de la VM, y actualiza la configuración de Nginx en `default.conf` para escuchar en el puerto `443` y redirigir el puerto `80` a `HTTPS`.

**Ejemplo de bloque en default.conf con SSL:**
```nginx
server {
    listen 80;
    server_name restock.tudominio.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name restock.tudominio.com;

    ssl_certificate /etc/letsencrypt/live/restock.tudominio.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/restock.tudominio.com/privkey.pem;

    # Bloques de /api, /mqtt y / para estáticos aquí...
}
```

Inicia Nginx de nuevo:
```bash
docker compose start nginx
```

---

## 8. Verificación de Telemetría IoT en Producción

Para validar que el broker MQTT está operando correctamente en Azure VM:

1. **Desde tu máquina local (o un sensor físico)**:
   Utiliza un cliente MQTT (como MQTT Explorer o CLI) para conectarte al IP/Dominio de tu VM en el puerto `1883` sin credenciales.
2. **Publicar Telemetría**:
   Publica un payload JSON en el topic `restock/telemetry`:
   ```json
   {
     "macAddress": "00:1A:2B:3C:4D:5E",
     "grossWeight": 250.0
   }
   ```
3. **Verificar Recepción**:
   * Observa los logs del backend en la VM para confirmar el procesamiento:
     ```bash
     docker compose logs -f backend-1
     ```
   * Abre la consola de desarrollador del navegador en el cliente Angular para ver la actualización de pesos reflejada en tiempo real en los listados del inventario.
