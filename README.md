# CRM ORM/ODM Lab

API REST de un CRM básico que combina un ORM (Sequelize + PostgreSQL) y un ODM (Mongoose + MongoDB).

## Stack

- Node.js 22, Express 5, CommonJS
- Sequelize + PostgreSQL 16 (`User`, `Company`, `Contact`)
- Mongoose + MongoDB 7 (`Activity`)
- Jest + Supertest
- GitHub Codespaces, Dev Containers, Docker Compose
- Supervisor (`npm run dev`)

## Arquitectura

```text
GitHub Codespace
│
├── app       Node.js 22  ──┬── Sequelize ──> postgres (PostgreSQL)
│                           └── Mongoose  ──> mongo    (MongoDB)
├── postgres
└── mongo
```

La aplicación se conecta por nombre de servicio (`postgres`, `mongo`). Las credenciales de desarrollo llegan como variables de entorno definidas en `.devcontainer/docker-compose.yml` (ver `.env.example`).

## Iniciar el Codespace

1. En GitHub: **Code → Codespaces → Create codespace on main**.
2. Espera a que se levanten los tres servicios (`app`, `postgres`, `mongo`). `postCreateCommand` ejecuta `npm install`.

## Instalar dependencias

```bash
npm install
```

## Seed y reset

```bash
npm run seed    # inserta datos deterministas (3 users, 4 companies, 8 contacts, 10 activities)
npm run reset   # elimina y recrea tablas/base de datos y vuelve a sembrar
```

## Iniciar la API

```bash
npm start       # node ./bin/www
npm run dev     # supervisor ./bin/www
```

Servidor en el puerto `3000` (variable `PORT`).

## Pruebas

```bash
npm test
```

Cada suite restablece PostgreSQL y MongoDB antes de ejecutarse y cierra las conexiones al terminar.

## Endpoints

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/health` | Health check |
| GET | `/users` | Listar usuarios |
| GET | `/users/:id` | Obtener usuario |
| POST | `/users` | Crear usuario |
| PUT | `/users/:id` | Actualizar usuario |
| DELETE | `/users/:id` | Eliminar usuario |
| GET | `/companies` | Listar compañías (`?industry=`) |
| GET | `/companies/:id` | Obtener compañía |
| POST | `/companies` | Crear compañía |
| PUT | `/companies/:id` | Actualizar compañía |
| DELETE | `/companies/:id` | Eliminar compañía |
| GET | `/contacts` | Listar contactos |
| GET | `/contacts/:id` | Obtener contacto |
| POST | `/contacts` | Crear contacto |
| PUT | `/contacts/:id` | Actualizar contacto |
| DELETE | `/contacts/:id` | Eliminar contacto |
| GET | `/activities` | Listar actividades (`?type=`) |
| GET | `/activities/:id` | Obtener actividad |
| POST | `/activities` | Crear actividad |
| PUT | `/activities/:id` | Actualizar actividad |
| DELETE | `/activities/:id` | Eliminar actividad |

Los errores se devuelven como JSON: `{ "error": "Contact not found" }`.

## Respuestas 
1. Cada actividad guarda ciertos aspectos, por ejemplo, una llamada guarda duration y result. Así un documento puede tener los campos que necesiten sin la necesidad que toods los registros compartan las mismas columnas.
Company y Contact encajan en PostgreSQL porque hay una relación clara, ya que cada contacto pertenece a una compañía y se especifica mediante un companyId, y una base relacional puede asegurar ese vínculo y permite consultas que cruzan tablas.
2. Un ORM es una técnica que traduce clases y objetos en tablas, filas y columnas de una base de datos relacional. Un ODM, es como el ORM pero diseñado para bases de datos NoSQL orientadas a documentos (como JSON/BSON). En este proyecto se observan dependencias como Sequelize y Mongoose los cuales son ejemplos de estas librerías. ORM - Sequelize ODM - Mongoose. 
3. Las credenciales de las bases estan en el archivo .env.example ahi se definen DB_HOST: postgres con un DB_PORT: 5432 y un MONGODB_URI=mongodb://mongo:27017/crm 
Estas no se escriben en los .js porque cuando subimos un codigo y la contraseña esta dentro, cualquiera podria verla. 
También cada base corre su propio contenedor y desde el contenedor app, el localhost es el mismo contenedor donde no hay Postgres ni Mongo por eso usan el nombre de cada contenedor. 
4. Una compañía tiene muchos contactos, y cada contacto pertenece a ua compañia. La llave foranea es companyId y vive en la tabla de contactos, porque es el lado que guarda el id de uno. El alias as:'contacts' es solo la manera en que sequelize nombra esa relación. 
5. Traer la compañía y hacer una segunda consulta es bueno, pero, ¿que sucede cuando los casos crecen y ahora necesitan listar 1000 compañias con sus contactos? Necesitariamos mas recursos(más viajes, más espera y más carga), porque con el enfoque A tendriamos 1 consulta para las compañias y otra para cada compañia, es decir, dos consultas frente a una. Es mejor el enfoque con include porque en una sola consulta te puede mostrar lo mismo y es menos recursos.
6. En la instancia su ventaja es que el contact no devuelve nada. Su desventaja es que hace más consultas una para buscar y otra para actualizar. Además si no existe regresa el 404 antes de actualizar. 
Con el update la ventaja es que hace solo una consulta pero devuelve el numero de filas que afecto y necesitariamos una consulta para devolver un contacto.
7. Metadata es de tipo 'mongoose.Schema.Types.Mixed', que significa que es un tipo que puede aceptar cualquier estructura por ejemplo que una llamada guarda unos cambios, un email otros y una reunion otros. La desventaja frente a definir cada campo con su tipo es que nada valida que 'duration' sea un numero o 'opened' sea un booleano.
8. ref y populate funcionan entre modelos dentro de MongoDB. Los ususarios y contactos estan en PostgreSQL, así que mongo no puede hacer populate porque User y Contact son modelos de Mongoose y 'contactId' y 'userId' son solo numeros sueltos. Entre ambas bases no existe una llave foranea que las conecte, así que si alguien borra un User en Postgre las actividades continuaran en mongo. 
9. findByIdAndUpdate guardaba el cambio en MongoDb, pero en 'activity' lo que entregaba era el documento sin modificarlo, es por eso que PUT respondia con los datos anteriores aunque la base ya tuviera nuevos. Al cambiar con la opcion 'new: true' hace una validacion para que mongoose devuelva el documento ya actualizado. Y 'runValidaators:true' hace validaciones del esquema a los datos que se escriben. 
10. LAs pruebas verifican la respuesta de la API, no los metodos que se usaron dentro. Con pruebas de comportamiento, mientras la API responda lo mismo, las pruebas siguen pasando. Un cliente que consume 'GET/contacts' no sabe y tempoco le interesa si usamos findAll.
11. En test/setup.js antes de cada suite, lo que hace es que se conecta a PostgreSQL y a MongoDB y llama a reset() y deja las bases con los datos iniciales. En el Despues de cada suite, cierra las conexiones a las bases. Antes de cada suite, tests se conecta a las bases y usa el reset() para que todo comience con los datos conocidos y es por eso que npm test da el mismo resultado. Despues cierra las conexiones. Sin el reset los cambios se quedarian guardados. 
12.El reto más difícil para mí fue el 8, porque la actualización sí se guardaba en MongoDB, pero la respuesta del PUT devolvía el documento como estaba antes del cambio. Eso me hizo revisar que sucedia con el 'findByIdAndUpdate' y vi que por defecto devuelve el documento original. Lo resolví agregando la opción 'new: true' para que devuelva el documento actualizado, y 'runValidators: true' para que se validen los datos.

## Evidencia
