# Restock Infrastructure

Este repositorio es el director de orquesta de la plataforma Restock. No contiene codigo fuente, sino la configuracion centralizada necesaria para desplegar todos los microservicios (Backend, Frontend, Base de Datos y Cache) juntos utilizando Docker Compose.

---

## El Flujo de Trabajo (Para Desarrolladores)

La regla principal es: **Aqui no se programa**. Este repositorio solo se usa para probar la integracion final de todos los componentes o para realizar despliegues.

### 1. Desarrollo Local
Si estas trabajando en una nueva funcionalidad o corrigiendo un bug (ya sea en Angular o Spring Boot), debes hacerlo en tus respectivos repositorios (`restock-web-app` o `restock-web-service`).

### 2. Integracion Continua (CI)
Cuando finalizas tu tarea y haces push a las ramas principales (`develop` o `main`), los flujos de GitHub Actions tomaran tu codigo, lo compilaran y crearan una imagen inmutable en Docker Hub (por ejemplo, `julioxc4/restock-web-service:latest`).

### 3. Orquestacion (Este repositorio)
Una vez que la imagen oficial esta publicada en Docker Hub, cualquier desarrollador puede venir a este repositorio para descargar esa nueva version y levantar la plataforma completa con todos sus servicios interconectados.

---

## Requisitos Previos

* Docker Desktop instalado y en ejecucion.
* Git instalado.

---

## Configuracion Inicial

Si es la primera vez que clonas este repositorio en tu maquina, debes configurar las variables de entorno locales:

1. Crea una copia del archivo plantilla:
```bash
   cp .env.example .env
```
2. Abre el archivo `.env` y completa los valores reales (credenciales de la base de datos, secretos de JWT, claves de Cloudinary, etc.). Estos valores no se suben al control de versiones por seguridad. Solicita los secretos al administrador del repositorio.

---

## Comandos Principales

### Levantar toda la plataforma

Descarga las ultimas imagenes desde Docker Hub y enciende todos los contenedores en segundo plano:

```bash
docker compose pull
docker compose up -d
```

### Ver los registros (Logs)

Para observar las peticiones en tiempo real o depurar errores:

```bash
docker compose logs -f
```

Para ver los registros de un solo servicio en especifico (por ejemplo, el backend):

```bash
docker compose logs -f backend
```

### Apagar la plataforma

Detiene todos los contenedores sin eliminar los datos persistentes:

```bash
docker compose down
```

### Reinicio limpio (Destructivo)

Si necesitas una instalacion limpia y deseas borrar todos los datos almacenados en los volumenes de MongoDB y Redis:

```bash
docker compose down -v
```

---

## Como integrar cambios nuevos de tu equipo

Si tu o un companero acaban de hacer un merge en el backend o en el frontend y desean ver esos cambios reflejados en la integracion local:

1. Abre tu terminal en la raiz de este repositorio.
2. Descarga las imagenes mas recientes:
```bash
   docker compose pull
```
3. Aplica los cambios:
```bash
   docker compose up -d
```

> Docker Compose es inteligente: detectara que hay una nueva version de una imagen especifica, apagara unicamente ese contenedor y lo volvera a levantar con la nueva version. El resto de los servicios seguiran ejecutandose sin interrupcion.