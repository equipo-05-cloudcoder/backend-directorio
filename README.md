# Sesión 2: Backend: Docker, MySQL y servicios web con Flask

## Objetivos de la sesión

Al finalizar, el participante podrá:

1. Configurar un `devcontainer.json` con Docker-in-Docker para habilitar contenedores dentro del Codespace.
2. Levantar una instancia de MySQL dentro de un contenedor Docker.
3. Conectar Python a MySQL usando `mysql-connector-python`.
4. Construir una API REST con Flask que implemente las operaciones CRUD.
5. Probar los servicios web desde la terminal con `curl` y desde Postman.
6. Exponer un puerto del Codespace al exterior.

---
## Previo
Revisemos algunos conceptos

Revisar repositorio [ws_basics](https://github.com/CLA-CADI-VER2026/ws_basics)

## Arquitectura para este repositorio

```
┌────────────────────────────────────────────────────────┐
│                    GitHub Codespace                    │
│                                                        │
│  ┌──────────────┐  HTTP/JSON  ┌──────────────────────┐ │
│  │ curl /       │◄───────────►│   Flask :5000        │ │
│  │ Postman      │             │   (ws_estudiantes.py)│ │
│  └──────────────┘             └──────────┬───────────┘ │
│                                          │             │
│                               mysql-connector-python   │
│                                          │             │
│                               ┌──────────▼───────────┐ │
│                               │  MySQL en Docker     │ │
│                               │  (contenedor :3306)  │ │
│                               └──────────────────────┘ │
└────────────────────────────────────────────────────────┘
```

---

## `devcontainer.json` para backend

Guarda este archivo en `.devcontainer/devcontainer.json` en tu repositorio:

```json
{
  "name": "Backend Flask + MySQL",
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu",
  "features": {
    "ghcr.io/devcontainers/features/docker-outside-of-docker:1": {},
    "ghcr.io/devcontainers/features/python:1": {
      "version": "3.12"
    }
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-python.python",
        "ms-python.pylint",
        "humao.rest-client"
      ]
    }
  },
  "remoteUser": "vscode"
}
```

**¿Qué hace la feature `docker-outside-of-docker`?**

Permite ejecutar comandos `docker` dentro del Codespace utilizando el daemon de Docker del host. Esto es necesario para poder levantar contenedores (como MySQL) desde el terminal del Codespace.

**Extensiones instaladas automáticamente:**

| Extensión | Propósito |
|---|---|
| `ms-python.python` | Soporte completo de Python en VS Code |
| `ms-python.pylint` | Verificación de errores y estilo en Python |
| `humao.rest-client` | Probar peticiones HTTP directamente desde VS Code |

---

## Paso 1: Levantar MySQL en Docker

Una vez que tu codespace esté corriendo, en el terminal de tu Codespace, ejecuta:

```bash
docker run \
  --name mysql-container \
  -e MYSQL_ROOT_PASSWORD=contrasena \
  -e MYSQL_DATABASE=curso_db \
  -p 3306:3306 \
  -d mysql:latest
```

**Explicación de los parámetros:**

| Parámetro | Significado |
|---|---|
| `--name mysql-container` | Nombre del contenedor para referenciarlo después |
| `-e MYSQL_ROOT_PASSWORD=contrasena` | Contraseña del usuario `root` de MySQL |
| `-e MYSQL_DATABASE=curso_db` | Crea automáticamente la base de datos `curso_db` |
| `-p 3306:3306` | Mapea el puerto 3306 del contenedor al 3306 del Codespace |
| `-d` | Ejecuta en segundo plano (detached) |
| `mysql:latest` | Imagen oficial de MySQL |

Verifica que el contenedor esté corriendo:

```bash
docker ps
```

Espera ~15 segundos y conéctate al cliente MySQL:

```bash
docker exec -it mysql-container mysql -u root -pcontrasena
```

---

## Paso 2: Crear la tabla

Dentro del cliente MySQL (el prompt cambia a `mysql>`):

```sql
USE curso_db;

CREATE TABLE estudiantes (
  id          INT AUTO_INCREMENT PRIMARY KEY,
  nombre      VARCHAR(100) NOT NULL,
  email       VARCHAR(100) UNIQUE NOT NULL,
  carrera     VARCHAR(100)
);

-- Insertar algunos registros de prueba
INSERT INTO estudiantes (nombre, email, carrera) VALUES
  ('Ana Stark',    'ana@tec.mx',     'ITC'),
  ('Carlos Spider',  'carlos@tec.mx',  'IIS'),
  ('Mary Hens',   'maria@tec.mx',   'IMD');

SELECT * FROM estudiantes;

quit
```

Alternativamente, puedes guardar el SQL en un archivo `esquema.sql` y ejecutarlo así:

```bash
docker exec -i mysql-container mysql -u root -pcontrasena curso_db < esquema.sql
```

---

## Paso 3: Instalar dependencias Python

```bash
pip install mysql-connector-python flask flask-cors
```

| Paquete | Propósito |
|---|---|
| `mysql-connector-python` | Driver oficial de Oracle para conectar Python con MySQL |
| `flask` | Microframework web para construir la API REST |
| `flask-cors` | Extensión que agrega soporte CORS al servidor Flask |

---

## Paso 4: CRUD directo con Python

Crea el archivo `crud_estudiantes.py`:

```python
import mysql.connector

# Configuración de la conexión
conn = mysql.connector.connect(
    host="127.0.0.1",
    user="root",
    password="contrasena",
    database="curso_db",
    port=3306
)
cursor = conn.cursor(dictionary=True)

# CREATE
def crear_estudiante(nombre, email, carrera):
    cursor.execute(
        "INSERT INTO estudiantes (nombre, email, carrera) VALUES (%s, %s, %s)",
        (nombre, email, carrera)
    )
    conn.commit()
    print(f"Estudiante creado con id: {cursor.lastrowid}")

# READ (todos)
def leer_estudiantes():
    cursor.execute("SELECT * FROM estudiantes")
    for e in cursor.fetchall():
        print(e)

# READ (por id)
def leer_estudiante(id):
    cursor.execute("SELECT * FROM estudiantes WHERE id = %s", (id,))
    print(cursor.fetchone())

# UPDATE
def actualizar_carrera(id, nueva_carrera):
    cursor.execute(
        "UPDATE estudiantes SET carrera = %s WHERE id = %s",
        (nueva_carrera, id)
    )
    conn.commit()
    print(f"Filas afectadas: {cursor.rowcount}")

# DELETE
def eliminar_estudiante(id):
    cursor.execute("DELETE FROM estudiantes WHERE id = %s", (id,))
    conn.commit()
    print(f"Filas eliminadas: {cursor.rowcount}")

# Prueba manual
if __name__ == "__main__":
    print("--- Todos los estudiantes ---")
    leer_estudiantes()

    print("\n--- Crear nuevo estudiante ---")
    crear_estudiante("Luis Ramos", "luis@tec.mx", "ITC")

    print("\n--- Leer estudiante 1 ---")
    leer_estudiante(1)

    print("\n--- Actualizar carrera del id 1 ---")
    actualizar_carrera(1, "IIS")
    leer_estudiante(1)

    cursor.close()
    conn.close()
```

Ejecuta:

```bash
python crud_estudiantes.py
```

---

## Paso 5: Servicios web con Flask

Crea el archivo `ws_estudiantes.py`:

```python
from flask import Flask, request, jsonify
from flask_cors import CORS
import mysql.connector

app = Flask(__name__)
CORS(app, origins="*")

def get_connection():
    return mysql.connector.connect(
        host="127.0.0.1",
        user="root",
        password="contrasena",
        database="curso_db",
        port=3306
    )

# GET /estudiantes: obtener todos
@app.route("/estudiantes", methods=["GET"])
def get_all():
    conn = get_connection()
    cursor = conn.cursor(dictionary=True)
    cursor.execute("SELECT * FROM estudiantes")
    resultado = cursor.fetchall()
    cursor.close()
    conn.close()
    return jsonify(resultado), 200

# GET /estudiantes/<id>: obtener uno
@app.route("/estudiantes/<int:id>", methods=["GET"])
def get_one(id):
    conn = get_connection()
    cursor = conn.cursor(dictionary=True)
    cursor.execute("SELECT * FROM estudiantes WHERE id = %s", (id,))
    estudiante = cursor.fetchone()
    cursor.close()
    conn.close()
    if not estudiante:
        return jsonify({"error": "No encontrado"}), 404
    return jsonify(estudiante), 200

# POST /estudiantes: crear
@app.route("/estudiantes", methods=["POST"])
def create():
    data = request.get_json()
    conn = get_connection()
    cursor = conn.cursor()
    cursor.execute(
        "INSERT INTO estudiantes (nombre, email, carrera) VALUES (%s, %s, %s)",
        (data["nombre"], data["email"], data.get("carrera", ""))
    )
    conn.commit()
    nuevo_id = cursor.lastrowid
    cursor.close()
    conn.close()
    return jsonify({"mensaje": "Estudiante creado", "id": nuevo_id}), 201

# PUT /estudiantes/<id>: actualizar
@app.route("/estudiantes/<int:id>", methods=["PUT"])
def update(id):
    data = request.get_json()
    conn = get_connection()
    cursor = conn.cursor()
    cursor.execute(
        "UPDATE estudiantes SET nombre=%s, email=%s, carrera=%s WHERE id=%s",
        (data["nombre"], data["email"], data.get("carrera", ""), id)
    )
    conn.commit()
    cursor.close()
    conn.close()
    return jsonify({"mensaje": "Estudiante actualizado"}), 200

# DELETE /estudiantes/<id>: eliminar
@app.route("/estudiantes/<int:id>", methods=["DELETE"])
def delete(id):
    conn = get_connection()
    cursor = conn.cursor()
    cursor.execute("DELETE FROM estudiantes WHERE id = %s", (id,))
    conn.commit()
    cursor.close()
    conn.close()
    return jsonify({"mensaje": "Estudiante eliminado"}), 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
```

Inicia el servidor:

```bash
python ws_estudiantes.py
```

---

## Paso 6: Probar con `curl`

Abre **otra terminal** (el servidor debe seguir corriendo en la primera):

```bash
# Obtener todos
curl -X GET http://localhost:5000/estudiantes

# Obtener uno por id
curl -X GET http://localhost:5000/estudiantes/1

# Crear
curl -X POST http://localhost:5000/estudiantes \
  -H "Content-Type: application/json" \
  -d '{"nombre": "Pedro Ruiz", "email": "pedro@tec.mx", "carrera": "ITC"}'

# Actualizar
curl -X PUT http://localhost:5000/estudiantes/1 \
  -H "Content-Type: application/json" \
  -d '{"nombre": "Ana Smith", "email": "ana@tec.mx", "carrera": "ITC"}'

# Eliminar
curl -X DELETE http://localhost:5000/estudiantes/4
```

---

## Paso 7: Probar con Postman (puerto público)

1. En VS Code, abre la pestaña **Ports** (junto a la terminal).
2. Localiza el puerto **5000** en la lista.
3. Haz clic derecho → **Port Visibility → Public**.
4. Copia la URL pública (forma: `https://<nombre-codespace>-5000.app.github.dev`).
5. En Postman, usa esa URL como base en lugar de `localhost:5000`.

> **Nota de seguridad:** el puerto público es accesible por cualquier persona con la URL. Úsalo solo para pruebas y cámbialo de vuelta a privado cuando termines.

---

## ¿Qué es CORS y por qué lo habilitamos?

Cuando el frontend (React, puerto 5173) hace una petición al backend (Flask, puerto 5000), el navegador bloquea la respuesta porque detecta que son **orígenes distintos** (diferente puerto = diferente origen). Esto se llama la **Same-Origin Policy**.

`flask-cors` agrega el encabezado `Access-Control-Allow-Origin: *` a todas las respuestas, indicándole al navegador que está permitido recibir esas respuestas.

```python
from flask_cors import CORS
CORS(app, origins="*")   # En producción, especifica solo los dominios autorizados
```

---

## Actividad: crea tu propia entidad

Elige una entidad diferente a `estudiantes` (por ejemplo: `productos`, `libros`, `tareas`) y:

1. Define su tabla SQL con al menos 3 columnas.
2. Crea la tabla en MySQL con `docker exec`.
3. Escribe los 5 endpoints CRUD en Flask.
4. Prueba todos los endpoints con `curl`.

---

## Actividad de cierre: crea tu plantilla de repositorio backend

Con lo que aprendiste en esta sesión puedes construir una plantilla que los alumnos usen como punto de partida para cualquier actividad de backend. La plantilla incluye el entorno de Codespaces preconfigurado y un README que guía al alumno paso a paso.

### Estructura de la plantilla

```
template-backend/
├── .devcontainer/
│   └── devcontainer.json     ← Habilita Docker + Python en el Codespace
├── esquema.sql               ← Script SQL de ejemplo (el alumno lo modificará)
├── ws_ejemplo.py             ← Servicio Flask base (el alumno lo completará)
└── README.md                 ← Instrucciones para el alumno
```

### `devcontainer.json` de la plantilla

```json
{
  "name": "Backend Flask + MySQL",
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu",
  "features": {
    "ghcr.io/devcontainers/features/docker-outside-of-docker:1": {},
    "ghcr.io/devcontainers/features/python:1": {
      "version": "3.12"
    }
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-python.python",
        "ms-python.pylint",
        "humao.rest-client"
      ]
    }
  },
  "remoteUser": "vscode"
}
```

> La feature `docker-outside-of-docker` es la clave: permite al alumno ejecutar `docker run` dentro del Codespace para levantar MySQL sin instalar nada localmente.

### `README.md` para el alumno

El README debe ser una receta de cocina: pasos numerados, comandos listos para copiar. Ejemplo:

```markdown
# [Nombre de la actividad]

## Descripción
[Qué debe construir el alumno]

## Paso 1: abrir el Codespace
1. Haz clic en **Code** → **Codespaces** → **Create codespace on main**.
2. Espera ~60 segundos mientras se configura el entorno (Python + Docker).

## Paso 2: levantar MySQL
En el terminal integrado ejecuta:
\```bash
docker run --name mysql-container \
  -e MYSQL_ROOT_PASSWORD=contrasena \
  -e MYSQL_DATABASE=mi_db \
  -p 3306:3306 -d mysql:latest
\```
Espera 15 segundos y verifica con: `docker ps`

## Paso 3: crear las tablas
\```bash
docker exec -i mysql-container mysql -u root -pcontrasena mi_db < esquema.sql
\```

## Paso 4: instalar dependencias
\```bash
pip install mysql-connector-python flask flask-cors
\```

## Paso 5: [instrucciones específicas de la actividad]
[Lo que el alumno debe implementar]

## Paso 6: probar los servicios
\```bash
python ws_ejemplo.py
# En otra terminal:
curl -X GET http://localhost:5000/[recurso]
\```

## Entregable
[Qué debe entregar y cómo]
```

Usa este prompt con IA para generar el README ajustado a tu actividad:

```
Genera un README.md en español para una actividad de backend llamada "[NOMBRE]".
El alumno usa GitHub Codespaces con Python 3.12 y Docker disponibles.
La base de datos es MySQL corriendo en un contenedor Docker en el puerto 3306.
El backend usa Flask con mysql-connector-python y flask-cors.
Incluye: Descripción, pasos numerados para levantar MySQL, instalar dependencias,
ejecutar el servidor Flask, y probar con curl. Usa bloques de código para todos los comandos.
```

### Activa Template repository

1. En tu repositorio ve a **Settings → General**.
2. Activa **Template repository** y guarda.

### Prueba la plantilla

1. Usa **Use this template → Create a new repository**.
2. Lanza un Codespace desde el nuevo repositorio.
3. Verifica que `docker ps` funcione y que puedas levantar MySQL.
4. Confirma que `pip install flask` no genere errores.

---

## Actividad extra (si queda tiempo)

### Automatizar el arranque con `postCreateCommand`

En lugar de pedirle al alumno que levante MySQL y ejecute el script SQL manualmente, puedes hacerlo automático. Agrega `postCreateCommand` a tu `devcontainer.json`:

```json
{
  "name": "Backend Flask + MySQL",
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu",
  "features": {
    "ghcr.io/devcontainers/features/docker-outside-of-docker:1": {},
    "ghcr.io/devcontainers/features/python:1": { "version": "3.12" }
  },
  "postCreateCommand": "docker run --name mysql-container -e MYSQL_ROOT_PASSWORD=contrasena -e MYSQL_DATABASE=curso_db -p 3306:3306 -d mysql:latest && sleep 30 && docker exec -i mysql-container mysql -u root -pcontrasena curso_db < esquema.sql && pip install -r requirements.txt",
  "customizations": {
    "vscode": {
      "extensions": ["ms-python.python", "ms-python.pylint", "humao.rest-client"]
    }
  },
  "remoteUser": "vscode"
}
```

Crea también un `requirements.txt` en la raíz:

```
mysql-connector-python
flask
flask-cors
```

Con esto el alumno abre el Codespace, espera ~60 segundos y ya puede ejecutar directamente `python ws_estudiantes.py`. El flujo completo queda en un solo paso.

### Agregar una segunda tabla con llave foránea

Extiende el esquema de `estudiantes` agregando una tabla `carreras`:

```sql
CREATE TABLE carreras (
  id     INT AUTO_INCREMENT PRIMARY KEY,
  nombre VARCHAR(100) NOT NULL,
  siglas VARCHAR(10) UNIQUE NOT NULL
);

ALTER TABLE estudiantes
  DROP COLUMN carrera,
  ADD COLUMN carrera_id INT,
  ADD FOREIGN KEY (carrera_id) REFERENCES carreras(id);
```

Agrega el endpoint `GET /carreras` a tu servicio Flask y actualiza el endpoint `GET /estudiantes` para que retorne el nombre de la carrera en lugar del ID usando un `JOIN`:

```python
cursor.execute("""
    SELECT e.id, e.nombre, e.email, c.nombre AS carrera
    FROM estudiantes e
    LEFT JOIN carreras c ON e.carrera_id = c.id
""")
```

Prueba que el resultado incluya el nombre de la carrera y no solo el `carrera_id`.

---

## Recursos adicionales

- [mysql-connector-python docs](https://dev.mysql.com/doc/connector-python/en/)
- [Flask quickstart](https://flask.palletsprojects.com/en/stable/quickstart/)
- [Repositorio de referencia: ejemplo_ws](https://github.com/TMP-DESARROLLO-WEB/ejemplo_ws)
- [Postman: descarga e instalación](https://www.postman.com/downloads/)
