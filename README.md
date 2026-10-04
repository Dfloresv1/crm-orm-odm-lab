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

**1. Dos motores.**
Activity es buena candidata para una base documental porque su campo metadata cambia de estructura según el tipo de actividad, y MongoDB permite guardar esa variabilidad sin forzar un esquema rígido y fijo. Company y Contact, por otro lado, tienen una estructura fija y bien definida, además de una relación clara entre sí, lo cual se modela de forma natural con tablas relacionales y llaves foráneas en PostgreSQL.

**2. ORM vs ODM.**
Un ORM traduce entre objetos de código y tablas de una base de datos relacional; en este proyecto es Sequelize, usado con PostgreSQL. Un ODM hace lo mismo pero para bases de datos documentales; aquí es Mongoose, usado con MongoDB. La diferencia importante es que Sequelize trabaja con esquemas fijos y relaciones explícitas, mientras que Mongoose permite campos flexibles como metadata, que puede tener distinta forma en cada documento.

**3. Configuración por variables de entorno.**
Las credenciales se definen en variables de entorno, no en el código JavaScript, para no dejar contraseñas ni datos sensibles expuestos en el repositorio. En este Codespace, los hosts que usa la app para conectarse no son localhost porque la app corre en un contenedor separado de los contenedores de PostgreSQL y MongoDB; cada servicio vive en su propio contenedor dentro de la red de Docker Compose, así que se conectan usando el nombre del servicio en lugar de localhost.

**4. Asociaciones.**
En models/sequelize/index.js existe una relación uno a muchos: -Company.hasMany(Contact, { foreignKey: 'companyId', as: 'contacts', onDelete: 'CASCADE' })-. La llave foránea es companyId, y vive en la tabla de Contact. El alias -as: 'contacts'- sirve para nombrar esa relación al consultarla con Sequelize; por ejemplo, en el include que usamos en companies.js (-include: { model: Contact, as: 'contacts' }-), ese alias debe coincidir exactamente con el definido en la asociación, o Sequelize no sabe qué relación cargar.

**5. Eager loading.**
Si trajera la compañía y luego hiciera una segunda consulta aparte para sus contactos, estaría haciendo dos viajes distintos a la base de datos. Usando include en la misma consulta, Sequelize genera una sola consulta SQL con un JOIN que trae la compañía y sus contactos juntos. Es preferible el eager loading porque reduce la cantidad de conexiones y consultas a la base de datos, haciendo la operación más rápida y eficiente.

**6. Instancia vs consulta.**
En update de contactos primero se busca el registro con findByPk y luego se modifica con contact.update(...). Esto devuelve la instancia ya actualizada, lista para responderla directamente en el JSON. La alternativa, Model.update({...}, { where }), actualiza directamente en la base de datos sin traer el registro a memoria primero, pero no devuelve el objeto actualizado unicamente indica cuántas filas se modificaron. La ventaja de usar la instancia es que puedes devolver el dato actualizado sin hacer una consulta extra; la ventaja de Model.update directo es que es más eficiente cuando solo ocupas modificar el registro.

**7. Esquema flexible.**
En models/mongoose/activity.js, el campo metadata usa el tipo Mixed (o Schema.Types.Mixed), que le permite aceptar cualquier estructura de objeto sin que Mongoose la valide contra un esquema fijo. Esto es lo que permite que CALL, EMAIL y MEETING guarden campos completamente distintos en el mismo campo metadata. La desventaja es que se pierde la validación automática.

**8. Sin ref.**
contactId y userId en Activity son simples números porque Contact y User no viven en MongoDB, sino en PostgreSQL, son bases de datos completamente distintas, y ref/populate de Mongoose solo funciona para relacionar documentos dentro de la misma base MongoDB. La consecuencia es que no hay integridad referencial automática entre ambos motores.

**9. Documento actualizado.**
La actualización devolvía el documento tal como estaba antes del cambio, porque findByIdAndUpdate() por defecto en Mongoose regresa la versión anterior del documento. Para que devolviera el documento ya actualizado, agregué la opción { new: true } en la llamada, y además { runValidators: true } para que Mongoose validara los nuevos datos contra el esquema antes de guardarlos.

**10. Pruebas de comportamiento.**
Probar el comportamiento en lugar de la implementación te deja cambiar cómo está escrito el código internamente sin que las pruebas se rompan, siempre y cuando el resultado final siga siendo el correcto. Esto da más libertad para refactorizar o mejorar el código sin miedo a romper los tests, porque las pruebas verifican el contrato de la API, no los detalles internos de cómo se logró ese resultado.

**11. Repetibilidad.**
tests/setup.js restablece las bases de datos a los datos de prueba antes de cada suite, y limpia conexiones o datos creados después de cada una. Esto es necesario porque si una prueba crea, modifica o borra datos y esos cambios persistieran para la siguiente ejecución, los resultados de npm test dependerían del estado previo de la base de datos, haciendo que las pruebas fueran impredecibles. Al restablecer el estado antes de cada suite, se garantiza que npm test dé siempre el mismo resultado sin importar cuántas veces se ejecute.

**12. Tu experiencia.**
El reto más difícil para mí fue el 05 (incluir los contactos de una compañía con Sequelize), no por la lógica del include en sí, sino porque olvidé importar Contact en la primera línea del archivo (const { Company } = require('../models/sequelize')). El código de la función se veía correcto a simple vista, pero al correr la prueba obtuve un error ReferenceError: Contact is not defined, con un status 500 en lugar del 200 esperado. Estuve un rato probando pero no hallaba el error hasta que recorde que no habia importado Contact.

## Evidencia
![npm test con las 9 suites en verde](<Captura de pantalla 2026-10-04 000327.png>)