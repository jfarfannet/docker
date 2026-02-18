# Laboratorio 3: Docker Compose y Aplicaciones Multi-Contenedor
## Duración: 60 minutos

### Objetivos
- Entender Docker Compose y orquestación básica
- Crear aplicaciones con múltiples servicios
- Gestionar redes y volúmenes
- Implementar una aplicación completa (Frontend + Backend + Base de Datos)

---

## Parte 1: Introducción a Docker Compose (10 minutos)

### Paso 1.1: Verificar instalación
```bash
# Verificar versión de Docker Compose
docker compose version
```

### Paso 1.2: Primer archivo docker-compose.yml

Crea un directorio para el proyecto:
```bash
mkdir app-completa-docker
cd app-completa-docker
```

**Archivo: `docker-compose.yml` (versión simple)**
```yaml
version: '3.8'

services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    
  redis:
    image: redis:alpine
```

### Paso 1.3: Comandos básicos
```bash
# Levantar todos los servicios
docker compose up -d

# Ver servicios ejecutándose
docker compose ps

# Ver logs de todos los servicios
docker compose logs

# Ver logs de un servicio específico
docker compose logs web

# Detener todos los servicios
docker compose down

# Ver redes creadas
docker network ls
```

---

## Parte 2: Aplicación con Contador de Visitas (20 minutos)

Vamos a crear una aplicación web que cuenta visitas usando Redis.

### Paso 2.1: Crear la aplicación backend

**Archivo: `app.py`**
```python
from flask import Flask
import redis
import os

app = Flask(__name__)

# Conectar a Redis usando el nombre del servicio
cache = redis.Redis(host='redis', port=6379, decode_responses=True)

def get_hit_count():
    retries = 5
    while True:
        try:
            return cache.incr('hits')
        except redis.exceptions.ConnectionError as exc:
            if retries == 0:
                raise exc
            retries -= 1

@app.route('/')
def hello():
    count = get_hit_count()
    html = f"""
    <html>
    <head>
        <title>Contador Docker</title>
        <style>
            body {{ 
                font-family: Arial; 
                text-align: center; 
                padding: 50px;
                background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
                color: white;
            }}
            .container {{ 
                background: rgba(255,255,255,0.1); 
                padding: 40px; 
                border-radius: 20px;
                backdrop-filter: blur(10px);
            }}
            .count {{ 
                font-size: 72px; 
                font-weight: bold; 
                margin: 20px 0;
                text-shadow: 2px 2px 4px rgba(0,0,0,0.3);
            }}
        </style>
    </head>
    <body>
        <div class="container">
            <h1>🐳 Contador de Visitas Docker</h1>
            <p>Esta página ha sido visitada:</p>
            <div class="count">{count}</div>
            <p>veces</p>
            <hr style="border-color: rgba(255,255,255,0.3);">
            <p>Backend: Python + Flask | Cache: Redis</p>
        </div>
    </body>
    </html>
    """
    return html

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=5000, debug=True)
```

**Archivo: `requirements.txt`**
```
Flask==3.0.0
redis==5.0.1
```

**Archivo: `Dockerfile`**
```dockerfile
FROM python:3.11-alpine

WORKDIR /app

# Instalar dependencias
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copiar código
COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
```

### Paso 2.2: Configurar Docker Compose

**Archivo: `docker-compose.yml`**
```yaml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "5000:5000"
    environment:
      - FLASK_ENV=development
    depends_on:
      - redis
    networks:
      - app-network
    volumes:
      - ./app.py:/app/app.py  # Hot reload en desarrollo

  redis:
    image: redis:alpine
    networks:
      - app-network
    volumes:
      - redis-data:/data  # Persistencia de datos

networks:
  app-network:
    driver: bridge

volumes:
  redis-data:
```

### Paso 2.3: Ejecutar la aplicación
```bash
# Construir y levantar servicios
docker compose up --build -d

# Ver logs en tiempo real
docker compose logs -f

# Probar en el navegador
# http://localhost:5000

# Recargar varias veces para ver el contador aumentar
```

### Paso 2.4: Explorar la red
```bash
# Ver detalles de la red
docker network inspect app-completa-docker_app-network

# Ejecutar comando en el contenedor web
docker compose exec web ping redis

# Ver datos en Redis
docker compose exec redis redis-cli
# Dentro de redis-cli:
GET hits
INCR hits
GET hits
exit
```

---

## Parte 3: Aplicación Completa - Lista de Tareas (25 minutos)

Ahora crearemos una aplicación TODO con:
- **Frontend**: React (interfaz web)
- **Backend**: Python FastAPI (API REST)
- **Base de Datos**: PostgreSQL

### Paso 3.1: Estructura del proyecto
```bash
mkdir todo-app
cd todo-app
mkdir backend frontend
```

### Paso 3.2: Backend (FastAPI)

**Archivo: `backend/main.py`**
```python
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import psycopg2
from psycopg2.extras import RealDictCursor
import os

app = FastAPI()

# Configurar CORS para permitir peticiones desde el frontend
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Modelo de datos
class Todo(BaseModel):
    title: str
    completed: bool = False

# Conexión a la base de datos
def get_db_connection():
    conn = psycopg2.connect(
        host=os.getenv("DB_HOST", "db"),
        database=os.getenv("DB_NAME", "tododb"),
        user=os.getenv("DB_USER", "todouser"),
        password=os.getenv("DB_PASSWORD", "todopass")
    )
    return conn

# Inicializar tabla
@app.on_event("startup")
async def startup():
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute("""
        CREATE TABLE IF NOT EXISTS todos (
            id SERIAL PRIMARY KEY,
            title TEXT NOT NULL,
            completed BOOLEAN DEFAULT FALSE
        )
    """)
    conn.commit()
    cur.close()
    conn.close()

# Endpoints
@app.get("/")
async def root():
    return {"message": "TODO API is running"}

@app.get("/todos")
async def get_todos():
    conn = get_db_connection()
    cur = conn.cursor(cursor_factory=RealDictCursor)
    cur.execute("SELECT * FROM todos ORDER BY id")
    todos = cur.fetchall()
    cur.close()
    conn.close()
    return todos

@app.post("/todos")
async def create_todo(todo: Todo):
    conn = get_db_connection()
    cur = conn.cursor(cursor_factory=RealDictCursor)
    cur.execute(
        "INSERT INTO todos (title, completed) VALUES (%s, %s) RETURNING *",
        (todo.title, todo.completed)
    )
    new_todo = cur.fetchone()
    conn.commit()
    cur.close()
    conn.close()
    return new_todo

@app.put("/todos/{todo_id}")
async def update_todo(todo_id: int):
    conn = get_db_connection()
    cur = conn.cursor(cursor_factory=RealDictCursor)
    cur.execute(
        "UPDATE todos SET completed = NOT completed WHERE id = %s RETURNING *",
        (todo_id,)
    )
    updated_todo = cur.fetchone()
    conn.commit()
    cur.close()
    conn.close()
    if not updated_todo:
        raise HTTPException(status_code=404, detail="Todo not found")
    return updated_todo

@app.delete("/todos/{todo_id}")
async def delete_todo(todo_id: int):
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute("DELETE FROM todos WHERE id = %s", (todo_id,))
    deleted = cur.rowcount
    conn.commit()
    cur.close()
    conn.close()
    if deleted == 0:
        raise HTTPException(status_code=404, detail="Todo not found")
    return {"message": "Todo deleted"}
```

**Archivo: `backend/requirements.txt`**
```
fastapi==0.109.0
uvicorn[standard]==0.27.0
psycopg2-binary==2.9.9
pydantic==2.5.3
```

**Archivo: `backend/Dockerfile`**
```dockerfile
FROM python:3.11-alpine

WORKDIR /app

# Dependencias del sistema para psycopg2
RUN apk add --no-cache postgresql-dev gcc python3-dev musl-dev

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY main.py .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

### Paso 3.3: Frontend (HTML simple)

**Archivo: `frontend/index.html`**
```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>TODO App - Docker</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #028090 0%, #02C39A 100%);
            min-height: 100vh;
            padding: 20px;
        }
        .container {
            max-width: 600px;
            margin: 0 auto;
            background: white;
            border-radius: 20px;
            box-shadow: 0 20px 60px rgba(0,0,0,0.3);
            padding: 30px;
        }
        h1 {
            color: #028090;
            margin-bottom: 30px;
            text-align: center;
        }
        .input-group {
            display: flex;
            gap: 10px;
            margin-bottom: 20px;
        }
        input[type="text"] {
            flex: 1;
            padding: 15px;
            border: 2px solid #e0e0e0;
            border-radius: 10px;
            font-size: 16px;
        }
        button {
            padding: 15px 30px;
            background: #028090;
            color: white;
            border: none;
            border-radius: 10px;
            cursor: pointer;
            font-size: 16px;
            font-weight: bold;
        }
        button:hover {
            background: #026873;
        }
        .todo-list {
            list-style: none;
        }
        .todo-item {
            display: flex;
            align-items: center;
            padding: 15px;
            margin-bottom: 10px;
            background: #f5f5f5;
            border-radius: 10px;
            transition: all 0.3s;
        }
        .todo-item:hover {
            background: #e0e0e0;
        }
        .todo-item.completed {
            opacity: 0.6;
        }
        .todo-item.completed .todo-text {
            text-decoration: line-through;
        }
        .todo-text {
            flex: 1;
            margin-left: 15px;
            font-size: 16px;
        }
        .delete-btn {
            background: #ff4757;
            padding: 8px 15px;
            font-size: 14px;
        }
        .delete-btn:hover {
            background: #ee5a6f;
        }
        input[type="checkbox"] {
            width: 20px;
            height: 20px;
            cursor: pointer;
        }
        .footer {
            text-align: center;
            margin-top: 20px;
            color: #666;
            font-size: 14px;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>📝 Lista de Tareas Docker</h1>
        
        <div class="input-group">
            <input type="text" id="todoInput" placeholder="Escribe una nueva tarea...">
            <button onclick="addTodo()">Agregar</button>
        </div>
        
        <ul class="todo-list" id="todoList"></ul>
        
        <div class="footer">
            <p>Backend: FastAPI | DB: PostgreSQL | Frontend: HTML+JS</p>
        </div>
    </div>

    <script>
        const API_URL = 'http://localhost:8000';

        async function loadTodos() {
            try {
                const response = await fetch(`${API_URL}/todos`);
                const todos = await response.json();
                renderTodos(todos);
            } catch (error) {
                console.error('Error loading todos:', error);
            }
        }

        function renderTodos(todos) {
            const list = document.getElementById('todoList');
            list.innerHTML = '';
            
            todos.forEach(todo => {
                const li = document.createElement('li');
                li.className = `todo-item ${todo.completed ? 'completed' : ''}`;
                
                li.innerHTML = `
                    <input type="checkbox" 
                           ${todo.completed ? 'checked' : ''} 
                           onchange="toggleTodo(${todo.id})">
                    <span class="todo-text">${todo.title}</span>
                    <button class="delete-btn" onclick="deleteTodo(${todo.id})">Eliminar</button>
                `;
                
                list.appendChild(li);
            });
        }

        async function addTodo() {
            const input = document.getElementById('todoInput');
            const title = input.value.trim();
            
            if (!title) return;
            
            try {
                await fetch(`${API_URL}/todos`, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ title, completed: false })
                });
                
                input.value = '';
                loadTodos();
            } catch (error) {
                console.error('Error adding todo:', error);
            }
        }

        async function toggleTodo(id) {
            try {
                await fetch(`${API_URL}/todos/${id}`, { method: 'PUT' });
                loadTodos();
            } catch (error) {
                console.error('Error toggling todo:', error);
            }
        }

        async function deleteTodo(id) {
            try {
                await fetch(`${API_URL}/todos/${id}`, { method: 'DELETE' });
                loadTodos();
            } catch (error) {
                console.error('Error deleting todo:', error);
            }
        }

        // Cargar tareas al iniciar
        loadTodos();
        
        // Permitir agregar con Enter
        document.getElementById('todoInput').addEventListener('keypress', (e) => {
            if (e.key === 'Enter') addTodo();
        });
    </script>
</body>
</html>
```

**Archivo: `frontend/Dockerfile`**
```dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### Paso 3.4: Docker Compose completo

**Archivo: `docker-compose.yml` (en la raíz de todo-app)**
```yaml
version: '3.8'

services:
  # Base de datos PostgreSQL
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: tododb
      POSTGRES_USER: todouser
      POSTGRES_PASSWORD: todopass
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - todo-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U todouser -d tododb"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Backend API
  backend:
    build: ./backend
    ports:
      - "8000:8000"
    environment:
      DB_HOST: db
      DB_NAME: tododb
      DB_USER: todouser
      DB_PASSWORD: todopass
    depends_on:
      db:
        condition: service_healthy
    networks:
      - todo-network
    volumes:
      - ./backend:/app

  # Frontend
  frontend:
    build: ./frontend
    ports:
      - "3000:80"
    depends_on:
      - backend
    networks:
      - todo-network

networks:
  todo-network:
    driver: bridge

volumes:
  postgres-data:
```

### Paso 3.5: Ejecutar la aplicación completa
```bash
# Construir y levantar todos los servicios
docker compose up --build -d

# Ver logs de todos los servicios
docker compose logs -f

# Ver solo logs del backend
docker compose logs -f backend

# Verificar que todos los servicios están running
docker compose ps

# Probar la aplicación:
# - Frontend: http://localhost:3000
# - Backend API: http://localhost:8000/todos
```

---

## Parte 4: Comandos Avanzados de Docker Compose (5 minutos)

```bash
# Escalar un servicio (ejemplo: 3 instancias del backend)
docker compose up -d --scale backend=3

# Reconstruir un servicio específico
docker compose build backend

# Reiniciar un servicio
docker compose restart backend

# Ver uso de recursos en tiempo real
docker compose top

# Ejecutar comando en un servicio
docker compose exec backend python -c "print('Hello from backend')"

# Ver variables de entorno de un servicio
docker compose exec backend env

# Detener sin eliminar contenedores
docker compose stop

# Iniciar servicios detenidos
docker compose start

# Detener y eliminar todo (contenedores, redes, volúmenes)
docker compose down -v
```

---

## Parte 5: Ejercicio Final - Mejoras (Opcional)

### Desafío: Agrega un servicio de caché

Mejora la aplicación TODO agregando Redis como caché para las consultas frecuentes.

**Pistas:**
1. Agrega un servicio `redis` en docker-compose.yml
2. Modifica el backend para usar Redis
3. Cachea la lista de todos por 30 segundos

**Solución parcial:**
```yaml
# En docker-compose.yml, agregar:
  redis:
    image: redis:alpine
    networks:
      - todo-network
```

```python
# En backend/main.py, agregar:
import redis
import json

redis_client = redis.Redis(host='redis', port=6379, decode_responses=True)

@app.get("/todos")
async def get_todos():
    # Intentar obtener del caché
    cached = redis_client.get('todos')
    if cached:
        return json.loads(cached)
    
    # Si no está en caché, consultar DB
    conn = get_db_connection()
    cur = conn.cursor(cursor_factory=RealDictCursor)
    cur.execute("SELECT * FROM todos ORDER BY id")
    todos = cur.fetchall()
    cur.close()
    conn.close()
    
    # Guardar en caché por 30 segundos
    redis_client.setex('todos', 30, json.dumps(todos))
    
    return todos
```

---

## Comandos de Referencia Rápida

| Comando | Descripción |
|---------|-------------|
| `docker compose up` | Levantar servicios |
| `docker compose up -d` | Levantar en segundo plano |
| `docker compose up --build` | Reconstruir imágenes |
| `docker compose down` | Detener y eliminar contenedores |
| `docker compose down -v` | Incluir volúmenes al eliminar |
| `docker compose ps` | Listar servicios |
| `docker compose logs` | Ver logs de todos los servicios |
| `docker compose logs -f servicio` | Seguir logs de un servicio |
| `docker compose exec servicio comando` | Ejecutar comando en servicio |
| `docker compose restart servicio` | Reiniciar servicio |
| `docker compose build` | Construir/reconstruir imágenes |
| `docker compose top` | Procesos en ejecución |
| `docker compose config` | Validar y ver configuración |

---

## Estructura de docker-compose.yml

```yaml
version: '3.8'

services:
  servicio1:
    build: ./directorio          # Construir desde Dockerfile
    # O
    image: imagen:tag             # Usar imagen existente
    
    ports:
      - "host:contenedor"         # Mapeo de puertos
    
    environment:                  # Variables de entorno
      - VAR=valor
    
    volumes:                      # Montajes
      - ./host:/contenedor        # Bind mount
      - nombre-volumen:/ruta      # Named volume
    
    networks:                     # Redes
      - red-personalizada
    
    depends_on:                   # Dependencias
      - servicio2
    
    restart: unless-stopped       # Política de reinicio
    
    healthcheck:                  # Health check
      test: ["CMD", "comando"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  red-personalizada:
    driver: bridge

volumes:
  nombre-volumen:
```

---

## Conceptos Clave Aprendidos

✅ **Orquestación**: Docker Compose gestiona múltiples contenedores como una aplicación

✅ **Redes**: Los servicios se comunican por nombre de servicio (DNS interno)

✅ **Volúmenes**: Persistencia de datos más allá del ciclo de vida del contenedor

✅ **Dependencias**: `depends_on` asegura el orden de inicio

✅ **Health checks**: Verifican que los servicios estén realmente listos

✅ **Variables de entorno**: Configuración sin modificar código

✅ **Escalabilidad**: Múltiples instancias del mismo servicio

---

## Troubleshooting

**Error:** "Cannot connect to database"
```bash
# Verificar que el servicio db esté healthy
docker compose ps
# Esperar a que el health check pase
docker compose logs db
```

**Error:** "Port is already in use"
```bash
# Cambiar el puerto del host en docker-compose.yml
ports:
  - "3001:80"  # Usar 3001 en lugar de 3000
```

**Error:** "No space left on device"
```bash
# Limpiar volúmenes huérfanos
docker volume prune
docker system prune -a --volumes
```

**Error:** CORS en el frontend
```bash
# Asegurarse de que el backend tenga CORS configurado
# Ya está incluido en el ejemplo con CORSMiddleware
```

---

## Recursos Adicionales

- [Documentación Docker Compose](https://docs.docker.com/compose/)
- [Compose file reference](https://docs.docker.com/compose/compose-file/)
- [Awesome Compose](https://github.com/docker/awesome-compose) - Ejemplos de aplicaciones

---

## ¡Felicitaciones! 🎉

Has completado los tres laboratorios de Docker. Ahora puedes:
- ✅ Ejecutar y gestionar contenedores
- ✅ Crear tus propias imágenes con Dockerfile
- ✅ Orquestar aplicaciones multi-contenedor con Docker Compose

### Próximos pasos recomendados:
1. Explora Kubernetes para orquestación a gran escala
2. Implementa CI/CD con Docker
3. Aprende sobre seguridad en contenedores
4. Prueba Docker Swarm para clusters

**¡Sigue practicando y dockerizando todas tus aplicaciones!** 🐳
