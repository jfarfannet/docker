# Laboratorio 2: Construyendo Tu Primera Imagen Docker
## Duración: 45 minutos

### Objetivos
- Entender la estructura de un Dockerfile
- Crear tu propia imagen Docker
- Aplicar buenas prácticas de construcción
- Publicar y versionar imágenes

---

## Parte 1: Estructura de un Dockerfile (10 minutos)

### Paso 1.1: Crear el directorio del proyecto
```bash
# Crear directorio de trabajo
mkdir mi-primera-app-docker
cd mi-primera-app-docker
```

### Paso 1.2: Crear una aplicación web simple

**Archivo: `app.py`**
```python
from flask import Flask
import os
import socket

app = Flask(__name__)

@app.route('/')
def hello():
    html = """
    <html>
    <head><title>Mi App Docker</title></head>
    <body style="font-family: Arial; text-align: center; padding: 50px;">
        <h1 style="color: #028090;">¡Hola desde Docker! 🐳</h1>
        <p><strong>Hostname:</strong> {hostname}</p>
        <p><strong>Visitas:</strong> Esta info vendrá de Redis en el Lab 3</p>
        <hr>
        <p>Aplicación containerizada con éxito</p>
    </body>
    </html>
    """
    return html.format(hostname=socket.gethostname())

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=5000)
```

**Archivo: `requirements.txt`**
```
Flask==3.0.0
```

### Paso 1.3: Crear tu primer Dockerfile

**Archivo: `Dockerfile`**
```dockerfile
# Imagen base - Python oficial
FROM python:3.11-slim

# Metadata
LABEL maintainer="tu-email@ejemplo.com"
LABEL description="Mi primera aplicación Docker"

# Establecer directorio de trabajo
WORKDIR /app

# Copiar archivo de dependencias
COPY requirements.txt .

# Instalar dependencias
RUN pip install --no-cache-dir -r requirements.txt

# Copiar código de la aplicación
COPY app.py .

# Exponer el puerto
EXPOSE 5000

# Comando por defecto
CMD ["python", "app.py"]
```

---

## Parte 2: Construir y Ejecutar (10 minutos)

### Paso 2.1: Construir la imagen
```bash
# Construir la imagen con un tag
docker build -t mi-app:v1.0 .

# Ver la imagen creada
docker images | grep mi-app
```

**Observa el proceso de construcción:**
- Cada línea del Dockerfile crea una capa
- Las capas se cachean para builds futuros más rápidos

### Paso 2.2: Ejecutar el contenedor
```bash
# Ejecutar el contenedor
docker run -d -p 5000:5000 --name mi-app-container mi-app:v1.0

# Verificar que está ejecutándose
docker ps

# Ver los logs
docker logs mi-app-container
```

**Probar la aplicación:**
Abre tu navegador en `http://localhost:5000`

### Paso 2.3: Entender las capas
```bash
# Ver el historial de la imagen (capas)
docker history mi-app:v1.0

# Inspeccionar la imagen
docker inspect mi-app:v1.0
```

---

## Parte 3: Optimización del Dockerfile (15 minutos)

### Paso 3.1: Mejorar con .dockerignore

**Archivo: `.dockerignore`**
```
__pycache__
*.pyc
*.pyo
*.pyd
.Python
env/
venv/
.git
.gitignore
.vscode
*.md
.DS_Store
```

### Paso 3.2: Dockerfile optimizado

**Archivo: `Dockerfile.optimizado`**
```dockerfile
# Usa imagen Alpine para menor tamaño
FROM python:3.11-alpine

# Instala dependencias del sistema en una sola capa
RUN apk add --no-cache gcc musl-dev linux-headers

# Establece directorio de trabajo
WORKDIR /app

# Copia e instala dependencias primero (mejor cache)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copia el código de la aplicación
COPY app.py .

# Crea un usuario no-root para seguridad
RUN adduser -D appuser && chown -R appuser /app
USER appuser

# Expone el puerto
EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:5000')" || exit 1

# Comando de ejecución
CMD ["python", "app.py"]
```

### Paso 3.3: Comparar tamaños
```bash
# Construir versión optimizada
docker build -f Dockerfile.optimizado -t mi-app:v1.1-alpine .

# Comparar tamaños
docker images | grep mi-app
```

**Observa la diferencia:**
- `mi-app:v1.0` ~100MB (basada en Debian)
- `mi-app:v1.1-alpine` ~50MB (basada en Alpine)

---

## Parte 4: Multi-Stage Builds (10 minutos)

Para aplicaciones que requieren compilación, usa multi-stage builds.

**Ejemplo con Node.js: `Dockerfile.multistage`**
```dockerfile
# Etapa 1: Build
FROM node:18-alpine AS builder
WORKDIR /build
COPY package*.json ./
RUN npm ci --only=production

# Etapa 2: Runtime
FROM node:18-alpine
WORKDIR /app

# Copia solo node_modules desde builder
COPY --from=builder /build/node_modules ./node_modules
COPY . .

# Usuario no-root
USER node

EXPOSE 3000
CMD ["node", "index.js"]
```

**Ventajas:**
- Imagen final solo contiene lo necesario para ejecutar
- No incluye herramientas de build
- Tamaño significativamente menor

---

## Parte 5: Versionado y Tags (5 minutos)

### Paso 5.1: Estrategia de versionado
```bash
# Tag semántico
docker build -t mi-app:1.0.0 .

# Tag latest (última versión)
docker tag mi-app:1.0.0 mi-app:latest

# Tag con SHA del commit (en CI/CD)
docker tag mi-app:1.0.0 mi-app:abc123f

# Ver todos los tags
docker images | grep mi-app
```

### Paso 5.2: Gestión de tags
```bash
# Eliminar un tag específico (no elimina la imagen si hay otros tags)
docker rmi mi-app:v1.0

# Verificar que la imagen sigue existiendo con otros tags
docker images | grep mi-app
```

---

## Parte 6: Ejercicio Práctico (5 minutos)

### Desafío: Crear una imagen de una calculadora web

**Tarea:** Crea una aplicación web simple que sume dos números.

**Archivo: `calculadora.py`**
```python
from flask import Flask, request, render_template_string

app = Flask(__name__)

HTML = """
<!DOCTYPE html>
<html>
<head>
    <title>Calculadora Docker</title>
    <style>
        body { font-family: Arial; max-width: 500px; margin: 50px auto; text-align: center; }
        input { padding: 10px; margin: 5px; width: 100px; }
        button { padding: 10px 20px; background: #028090; color: white; border: none; }
    </style>
</head>
<body>
    <h1>🧮 Calculadora Dockerizada</h1>
    <form method="POST">
        <input type="number" name="num1" placeholder="Número 1" required>
        <span>+</span>
        <input type="number" name="num2" placeholder="Número 2" required>
        <button type="submit">Calcular</button>
    </form>
    {% if resultado %}
        <h2>Resultado: {{ resultado }}</h2>
    {% endif %}
</body>
</html>
"""

@app.route('/', methods=['GET', 'POST'])
def calculadora():
    resultado = None
    if request.method == 'POST':
        num1 = float(request.form['num1'])
        num2 = float(request.form['num2'])
        resultado = num1 + num2
    return render_template_string(HTML, resultado=resultado)

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=8080)
```

**Tu tarea:**
1. Crea el `requirements.txt` necesario
2. Escribe un Dockerfile optimizado (usa Alpine)
3. Construye la imagen con tag `calculadora:1.0`
4. Ejecuta el contenedor en el puerto 8080
5. Prueba la aplicación en tu navegador

**Solución:**
```bash
# requirements.txt
echo "Flask==3.0.0" > requirements.txt

# Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.11-alpine
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY calculadora.py .
EXPOSE 8080
CMD ["python", "calculadora.py"]
EOF

# Construir y ejecutar
docker build -t calculadora:1.0 .
docker run -d -p 8080:8080 --name calc calculadora:1.0

# Probar en http://localhost:8080
```

---

## Mejores Prácticas del Dockerfile

### ✅ HACER
```dockerfile
# 1. Usar imágenes oficiales y específicas
FROM python:3.11-alpine

# 2. Minimizar capas combinando comandos RUN
RUN apk add --no-cache git && \
    pip install --no-cache-dir flask && \
    rm -rf /var/cache/apk/*

# 3. Copiar dependencias antes que el código (mejor cache)
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .

# 4. Usar usuario no-root
USER nobody

# 5. Especificar versiones exactas
pip install flask==3.0.0
```

### ❌ EVITAR
```dockerfile
# 1. NO usar :latest en producción
FROM python:latest  # ❌ Poco predecible

# 2. NO instalar paquetes innecesarios
RUN apt-get install vim git curl wget  # ❌ Aumenta tamaño

# 3. NO ejecutar como root
# Sin especificar USER  # ❌ Riesgo de seguridad

# 4. NO incluir secrets
COPY .env .  # ❌ Secretos en la imagen

# 5. NO crear capas innecesarias
RUN apt-get update
RUN apt-get install python  # ❌ Combínalas en una sola
```

---

## Comandos de Referencia Rápida

| Comando | Descripción |
|---------|-------------|
| `docker build -t nombre:tag .` | Construir imagen |
| `docker build -f Dockerfile.prod .` | Especificar Dockerfile |
| `docker tag imagen:tag nueva:tag` | Crear nuevo tag |
| `docker history imagen` | Ver capas de la imagen |
| `docker inspect imagen` | Detalles de la imagen |
| `docker rmi imagen:tag` | Eliminar imagen/tag |
| `docker build --no-cache .` | Construir sin usar cache |

---

## Instrucciones Dockerfile Esenciales

| Instrucción | Propósito | Ejemplo |
|-------------|-----------|---------|
| `FROM` | Imagen base | `FROM python:3.11-alpine` |
| `WORKDIR` | Directorio de trabajo | `WORKDIR /app` |
| `COPY` | Copiar archivos | `COPY . .` |
| `ADD` | Copiar y extraer | `ADD app.tar.gz /app` |
| `RUN` | Ejecutar comandos al construir | `RUN pip install flask` |
| `CMD` | Comando por defecto | `CMD ["python", "app.py"]` |
| `ENTRYPOINT` | Punto de entrada | `ENTRYPOINT ["python"]` |
| `EXPOSE` | Documentar puertos | `EXPOSE 5000` |
| `ENV` | Variables de entorno | `ENV DEBUG=1` |
| `USER` | Usuario de ejecución | `USER appuser` |
| `HEALTHCHECK` | Check de salud | `HEALTHCHECK CMD curl -f http://localhost/` |

---

## Troubleshooting

**Error:** "No space left on device"
```bash
# Limpiar imágenes no utilizadas
docker system prune -a
```

**Error:** "Cannot locate package"
```bash
# Actualizar índice de paquetes primero
RUN apt-get update && apt-get install -y paquete
```

**Error:** Build muy lento
```bash
# Usar .dockerignore para excluir archivos
# Ordenar COPY para maximizar cache
```

---

## Conceptos Clave Aprendidos

✅ **Capas**: Cada instrucción crea una capa inmutable

✅ **Cache**: Docker reutiliza capas sin cambios

✅ **Orden importa**: Cambia lo que cambia menos frecuentemente primero

✅ **Alpine**: Imágenes ligeras basadas en Alpine Linux

✅ **Multi-stage**: Separa build de runtime para imágenes pequeñas

✅ **Seguridad**: Siempre ejecuta como usuario no-root cuando sea posible

---

## Próximo Paso

En el **Laboratorio 3**, aprenderás a orquestar múltiples contenedores usando Docker Compose. ¡Construiremos una aplicación completa!
