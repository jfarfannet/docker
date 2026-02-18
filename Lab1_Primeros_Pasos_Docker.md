# Laboratorio 1: Primeros Pasos con Docker
## Duración: 45 minutos

### Objetivos
- Verificar la instalación de Docker
- Ejecutar tu primer contenedor
- Comprender el ciclo de vida de los contenedores
- Gestionar contenedores e imágenes

---

## Parte 1: Verificación de Docker (5 minutos)

### Paso 1.1: Verificar instalación
```bash
# Verificar versión de Docker
docker --version

# Ver información del sistema Docker
docker info

# Probar que Docker funciona correctamente
docker run hello-world
```

**¿Qué sucedió?**
- Docker descargó la imagen `hello-world` desde Docker Hub
- Creó un contenedor a partir de esa imagen
- Ejecutó el contenedor que imprimió un mensaje
- El contenedor se detuvo automáticamente

### Paso 1.2: Entender lo que pasó
```bash
# Ver las imágenes descargadas
docker images

# Ver todos los contenedores (incluyendo los detenidos)
docker ps -a
```

---

## Parte 2: Trabajando con Contenedores (15 minutos)

### Paso 2.1: Ejecutar un servidor web Nginx
```bash
# Ejecutar Nginx en modo detached (segundo plano)
docker run -d -p 8080:80 --name mi-nginx nginx

# Verificar que está ejecutándose
docker ps
```

**Abrir en el navegador:** `http://localhost:8080`

### Paso 2.2: Explorar el contenedor
```bash
# Ver los logs del contenedor
docker logs mi-nginx

# Ver estadísticas en tiempo real
docker stats mi-nginx

# Inspeccionar detalles del contenedor
docker inspect mi-nginx
```

### Paso 2.3: Interactuar con el contenedor
```bash
# Ejecutar un comando dentro del contenedor
docker exec mi-nginx ls /usr/share/nginx/html

# Entrar al contenedor de forma interactiva
docker exec -it mi-nginx bash

# Dentro del contenedor, ejecutar:
ls /usr/share/nginx/html
cat /usr/share/nginx/html/index.html
exit
```

### Paso 2.4: Modificar el contenido
```bash
# Copiar un archivo al contenedor
echo "<h1>¡Hola desde Docker!</h1>" > index.html
docker cp index.html mi-nginx:/usr/share/nginx/html/index.html

# Verificar el cambio en el navegador
# Recargar http://localhost:8080
```

### Paso 2.5: Gestión del contenedor
```bash
# Detener el contenedor
docker stop mi-nginx

# Verificar que ya no está en ejecución
docker ps

# Iniciar el contenedor nuevamente
docker start mi-nginx

# Reiniciar el contenedor
docker restart mi-nginx

# Detener y eliminar el contenedor
docker stop mi-nginx
docker rm mi-nginx

# Verificar que fue eliminado
docker ps -a
```

---

## Parte 3: Trabajando con Imágenes (10 minutos)

### Paso 3.1: Explorar Docker Hub
```bash
# Buscar imágenes de Python
docker search python

# Descargar una imagen específica
docker pull python:3.11-alpine

# Listar todas las imágenes locales
docker images
```

### Paso 3.2: Ejecutar diferentes versiones
```bash
# Ejecutar Python 3.11 de forma interactiva
docker run -it python:3.11-alpine python

# Dentro de Python, ejecutar:
print("¡Hola desde un contenedor Docker!")
import sys
print(f"Versión de Python: {sys.version}")
exit()
```

### Paso 3.3: Limpiar imágenes
```bash
# Ver el espacio que ocupan las imágenes
docker images

# Eliminar una imagen específica
docker rmi hello-world

# Ver imágenes huérfanas (sin usar)
docker images -f dangling=true

# Limpiar imágenes no utilizadas
docker image prune
```

---

## Parte 4: Ejercicio Práctico (10 minutos)

### Desafío: Ejecutar una base de datos PostgreSQL

**Tarea:** Levantar un contenedor de PostgreSQL y conectarte a él.

```bash
# 1. Ejecutar PostgreSQL
docker run -d \
  --name mi-postgres \
  -e POSTGRES_PASSWORD=mi_password_secreto \
  -e POSTGRES_USER=alumno \
  -e POSTGRES_DB=laboratorio \
  -p 5432:5432 \
  postgres:15-alpine

# 2. Verificar que está ejecutándose
docker ps

# 3. Ver los logs
docker logs mi-postgres

# 4. Conectarte a PostgreSQL
docker exec -it mi-postgres psql -U alumno -d laboratorio

# Dentro de PostgreSQL, ejecutar:
\l                    # Listar bases de datos
\dt                   # Listar tablas
CREATE TABLE test (id SERIAL PRIMARY KEY, nombre TEXT);
INSERT INTO test (nombre) VALUES ('Docker es genial');
SELECT * FROM test;
\q                    # Salir
```

### Preguntas de reflexión:
1. ¿Qué sucede con los datos si eliminas el contenedor?
2. ¿Cómo podrías persistir los datos?
3. ¿Qué significa el flag `-e` en el comando docker run?

---

## Parte 5: Limpieza Final (5 minutos)

```bash
# Detener todos los contenedores en ejecución
docker stop $(docker ps -q)

# Eliminar todos los contenedores detenidos
docker container prune -f

# Eliminar imágenes no utilizadas
docker image prune -a -f

# Ver el estado final
docker ps -a
docker images
```

---

## Comandos de Referencia Rápida

| Comando | Descripción |
|---------|-------------|
| `docker run <imagen>` | Crear y ejecutar un contenedor |
| `docker run -d` | Ejecutar en segundo plano (detached) |
| `docker run -p 8080:80` | Mapear puerto 8080 del host al 80 del contenedor |
| `docker run --name <nombre>` | Asignar un nombre al contenedor |
| `docker run -it` | Modo interactivo con terminal |
| `docker ps` | Listar contenedores en ejecución |
| `docker ps -a` | Listar todos los contenedores |
| `docker logs <contenedor>` | Ver logs |
| `docker exec -it <contenedor> bash` | Entrar al contenedor |
| `docker stop <contenedor>` | Detener contenedor |
| `docker start <contenedor>` | Iniciar contenedor detenido |
| `docker rm <contenedor>` | Eliminar contenedor |
| `docker images` | Listar imágenes locales |
| `docker pull <imagen>` | Descargar imagen |
| `docker rmi <imagen>` | Eliminar imagen |

---

## Conceptos Clave Aprendidos

✅ **Imagen vs Contenedor**: Las imágenes son plantillas, los contenedores son instancias en ejecución

✅ **Puertos**: El flag `-p` mapea puertos del host al contenedor

✅ **Variables de entorno**: El flag `-e` permite configurar el contenedor

✅ **Modos de ejecución**: `-d` (detached) vs `-it` (interactivo)

✅ **Ciclo de vida**: run → stop → start → restart → rm

---

## Troubleshooting

**Problema:** "Cannot connect to the Docker daemon"
- **Solución:** Verifica que Docker Desktop esté ejecutándose

**Problema:** "Port is already allocated"
- **Solución:** El puerto ya está en uso. Usa un puerto diferente o detén el proceso que lo está usando

**Problema:** "No space left on device"
- **Solución:** Ejecuta `docker system prune` para limpiar recursos no utilizados

---

## Recursos Adicionales

- [Documentación oficial de Docker](https://docs.docker.com)
- [Docker Hub](https://hub.docker.com)
- [Cheat Sheet de Docker](https://docs.docker.com/get-started/docker_cheatsheet.pdf)

---

## Próximo Paso

En el **Laboratorio 2**, aprenderás a crear tus propias imágenes usando Dockerfile. ¡Prepárate para construir!
