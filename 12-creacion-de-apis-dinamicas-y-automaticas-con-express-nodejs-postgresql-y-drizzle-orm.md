# Capítulo 12: Creación de APIs Dinámicas y Automáticas con Express, Node.js, PostgreSQL y Drizzle ORM

La web moderna prospera gracias a las APIs. Cada clic, cada desplazamiento infinito (*infinite scroll*), cada interacción entre tu frontend de React y una base de datos ocurre a través de una API. Pero aquí está el detalle: construir APIs puede ser tedioso. Defines un modelo, escribes operaciones CRUD, manejas la validación, gestionas relaciones y repites este proceso para cada entidad individual en tu aplicación. ¿Qué pasaría si tu API pudiera adaptarse automáticamente a medida que evoluciona tu base de datos? ¿Qué pasaría si agregar una nueva tabla significara que tus endpoints simplemente aparecieran, listos para usar?

Este capítulo explora cómo construir APIs que no solo sean funcionales, sino también inteligentes: APIs que comprenden la estructura de tu base de datos y se generan a sí mismas dinámicamente. Usaremos Express como nuestro framework de servidor, PostgreSQL para un almacenamiento de datos robusto y Drizzle ORM como el puente que hace que todo sea automático. Al final, tendrás un backend que escala con la complejidad de tu aplicación mientras mantiene tu base de código limpia y fácil de mantener.

En este capítulo, cubriremos los siguientes temas:

- Introducción al desarrollo de APIs dinámicas
- Configuración de un servidor Express con Node.js
- Integración de PostgreSQL con Drizzle ORM
- Generación automática de APIs CRUD dinámicas
- Optimización del rendimiento de APIs
- Patrones avanzados y consideraciones del mundo real

---

## Introducción al desarrollo de APIs dinámicas

El desarrollo tradicional de APIs sigue un patrón predecible. Creas un modelo, escribes controladores, defines rutas, manejas la validación, y repites el ciclo una y otra vez. Esto funciona bien para aplicaciones pequeñas, pero a medida que tu proyecto crece, la repetición se vuelve abrumadora. Cada nueva funcionalidad significa duplicar código similar en múltiples archivos.

### ¿Qué hace que una API sea automática y dinámica?

Una API automática comprende el esquema de tu base de datos y genera endpoints sin instrucciones explícitas. Cuando defines una tabla `users` en tu base de datos, rutas como `GET /users`, `POST /users`, `PUT /users/:id` y `DELETE /users/:id` aparecen automáticamente. ¿Agregas una tabla `posts`? Los mismos endpoints se materializan para las publicaciones. ¿Cambias una columna? La API se adapta.

Las APIs dinámicas van más allá; manejan las relaciones de manera inteligente. Cuando un usuario tiene muchas publicaciones, la API sabe cómo obtenerlas juntas. Cuando necesitas filtrar por fecha o buscar por nombre, la API construye las consultas correctas sin que tengas que escribir manejadores personalizados.

Esto no es magia. Se trata de aprovechar el sistema de tipos de TypeScript, la introspección de esquemas de Drizzle y una abstracción bien pensada. El resultado es código que se escribe solo, o que al menos se siente de esa manera.

### Beneficios de usar Express, Node.js, PostgreSQL y Drizzle ORM

Express sigue siendo el estándar de facto para servidores Node.js porque es minimalista, flexible y está probado en batalla. A diferencia de los frameworks pesados que imponen una estructura rígida, Express te da el control. Tú decides cómo organizar tu código, qué middleware utilizar y qué tan compleja o simple debe ser tu API. Esta libertad es un arma de doble filo: si bien acelera la creación de prototipos y se adapta a proyectos pequeños y medianos, las aplicaciones empresariales a gran escala a menudo requieren una gobernanza explícita: convenciones de carpetas, patrones de manejo de errores y pipelines de middleware que los equipos deben establecer por sí mismos.

PostgreSQL proporciona confiabilidad de nivel empresarial con un sólido soporte para consultas complejas, transacciones y tipos de datos enriquecidos como JSON, lo que lo hace ideal para aplicaciones con datos relacionales y requisitos de consistencia. En comparación, las bases de datos NoSQL enfatizan la escalabilidad horizontal, los esquemas flexibles y el rendimiento optimizado para patrones de acceso específicos, a menudo favoreciendo modelos de datos desnormalizados para reducir la complejidad de las consultas. Ambos enfoques tienen fortalezas distintas, y la elección correcta depende de factores como la estructura de los datos, las necesidades de consistencia y la escala esperada.

Drizzle ORM aporta seguridad de tipos a SQL sin la sobrecarga del mapeo objeto-relacional tradicional. Al igual que constructores de consultas como Knex.js, Drizzle te mantiene cerca de SQL en lugar de ocultarlo detrás de capas de abstracción. Lo que lo distingue es su enfoque centrado en TypeScript (*TypeScript-first*): defines tu esquema en TypeScript y Drizzle infiere los tipos en toda tu base de código, desde las definiciones de tablas hasta los resultados de las consultas. Las consultas utilizan una sintaxis legible y encadenable que se compila en SQL eficiente, y las migraciones se generan automáticamente a partir de los cambios en el esquema. El resultado es el poder y la previsibilidad de SQL con una inferencia de tipos de extremo a extremo que detecta errores en tiempo de compilación.

Juntas, estas herramientas crean una experiencia de desarrollo que es a la vez potente y agradable. Pasas menos tiempo escribiendo código repetitivo (*boilerplate*) y más tiempo construyendo funcionalidades.

---

## Configuración de un servidor Express con Node.js

Antes de generar APIs dinámicas, necesitamos un servidor en funcionamiento. Usaremos Express 5, que agrega soporte nativo para `async/await` en los manejadores de rutas, lo que significa que los errores lanzados y las promesas rechazadas se capturan automáticamente sin necesidad de envolver todo en bloques `try/catch`. No es la opción más rápida disponible (Fastify y Hono obtienen mejores resultados en las pruebas de rendimiento), pero el ecosistema, la documentación y la familiaridad de Express lo convierten en una opción pragmática para aprender patrones que se transfieren a cualquier framework de Node.js.

### Instalación de Express 5 y configuración de middleware

La configuración de un servidor Express comienza con la creación de un nuevo proyecto y la instalación de las dependencias adecuadas. Usaremos TypeScript desde el principio porque la seguridad de tipos detecta errores antes de que lleguen a producción.

Comienza creando un directorio para el proyecto e inicializándolo:

```bash
mkdir dynamic-api && cd dynamic-api
npm init -y
```

Esto genera un `package.json` básico. Necesitamos reemplazar su contenido con nuestras dependencias:

```json
{
  "name": "dynamic-api",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "start": "node dist/index.js",
    "build": "tsc -p tsconfig.json",
    "migrate": "tsx src/db/migrate.ts",
    "generate": "drizzle-kit generate --config=./src/drizzle.config.ts"
  },
  "dependencies": {
    "compression": "^1.8.1",
    "cors": "^2.8.5",
    "dotenv": "^17.2.3",
    "drizzle-orm": "^0.45.1",
    "drizzle-zod": "^0.8.3",
    "express": "^5.2.1",
    "express-rate-limit": "^8.2.1",
    "helmet": "^8.1.0",
    "postgres": "^3.4.8",
    "zod": "^4.3.5"
  },
  "devDependencies": {
    "@types/compression": "^1.8.1",
    "@types/cors": "^2.8.19",
    "@types/express": "^5.0.6",
    "@types/node": "^25.0.6",
    "drizzle-kit": "^0.31.8",
    "tsx": "^4.21.0",
    "typescript": "^5.9.3"
  }
}
```

### Configuración de TypeScript

TypeScript necesita un archivo de configuración para saber cómo compilar tu código. Crea `tsconfig.json` en la raíz de tu proyecto:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "nodenext",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "rootDir": "src",
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

Las opciones clave aquí: `target` y `module` establecidos en `ES2022` nos brindan características modernas de JavaScript como `top-level await` y sintaxis nativa de ESM. `moduleResolution: "nodenext"` le indica a TypeScript que resuelva las importaciones de la forma en que lo hace Node.js. El valor heredado `"Node"` no puede resolver importaciones con extensión `.js` ni mapas de exportaciones. Con `"type": "module"` en `package.json` y módulos `ES2022`, `"nodenext"` es obligatorio para que las compilaciones y el tiempo de ejecución funcionen correctamente. `strict: true` habilita todas las opciones estrictas de verificación de tipos; esto detecta más errores en tiempo de compilación pero requiere tipado explícito en lugares donde TypeScript no puede inferir. `outDir` y `rootDir` separan tus archivos fuente de la salida compilada, manteniendo el proyecto organizado.

El arreglo `include` le dice a TypeScript que compile todo lo que esté bajo `src/`, mientras que `exclude` evita que procese dependencias o su propia salida. Con esta configuración, ejecutar `npm run build` compila tu TypeScript a JavaScript en la carpeta `dist/`, y `npm run dev` utiliza `tsx` para ejecutar TypeScript directamente durante el desarrollo sin un paso de compilación separado.

### Creación de nuestro servidor Express

Nuestro servidor necesita una estructura limpia. Crearemos un archivo de servidor principal que maneje toda la lógica de inicialización en un solo lugar:

```typescript
// src/index.ts
import dotenv from 'dotenv';
import express, { Express, Request, Response, NextFunction } from 'express';
import cors from 'cors';
import helmet from 'helmet';
import compression from 'compression';

dotenv.config();

const app: Express = express();
const PORT = process.env.PORT || 3000;

// Security middleware
app.use(helmet());
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || 'http://localhost:3000',
  credentials: true
}));

// Performance middleware
app.use(compression());
app.use('/api', apiLimiter)
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true, limit: '10mb' }));

// Health check endpoint
app.get('/health', (_req: Request, res: Response) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

// Error handling middleware
app.use((err: Error, _req: Request, res: Response, _next: NextFunction) => {
  console.error('Server error:', err);
  res.status(500).json({
    error: 'Internal server error',
    message: process.env.NODE_ENV === 'development' ? err.message : undefined
  });
});

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

export default app;
```

Este servidor proporciona un buen punto de partida para producción, incluyendo encabezados de seguridad a través de Helmet, CORS para solicitudes de orígenes cruzados, compresión para cargas más pequeñas y un manejo estructurado de errores. El orden del middleware sigue importando: seguridad primero, luego el análisis de solicitudes, seguido de tus rutas y, finalmente, el manejo centralizado de errores, lo que garantiza tanto la seguridad como la mantenibilidad a medida que crece tu aplicación.

### Configuración de variables de entorno y archivos de configuración

Las variables de entorno mantienen los datos confidenciales fuera de tu base de código. Las credenciales de la base de datos, las claves de API y las opciones de configuración pertenecen a los archivos `.env`, y nunca deben confirmarse en el control de versiones:

```env
// .env.example
PORT=3000
NODE_ENV=development
DATABASE_URL=postgresql://user:password@localhost:5432/myapp
ALLOWED_ORIGINS=http://localhost:3000
CACHE_TTL=3600
```

### Manejo de solicitudes y respuestas de la API

Express maneja solicitudes a través de manejadores de rutas, pero podemos crear funciones auxiliares que estandaricen las respuestas en toda nuestra API. Los formatos de respuesta consistentes hacen que tu API sea predecible y más fácil de consumir:

```typescript
// src/utils/responses.ts
import { Response } from 'express';

// Standardized API response structure ensures consistent JSON shape
// across all endpoints, making client-side parsing predictable
export interface ApiResponse<T = any> {
  success: boolean;
  data?: T;
  error?: string;
  meta?: {
    total?: number;    // Total records (for paginated responses)
    page?: number;     // Current page number
    limit?: number;    // Items per page
  };
}

// Sends successful response with optional pagination metadata
// Usage: sendSuccess(res, users) or sendSuccess(res, user, 201, { total: 100 })
export const sendSuccess = <T>(
  res: Response,
  data: T,
  statusCode: number = 200,
  meta?: ApiResponse['meta']
): Response => {
  return res.status(statusCode).json({
    success: true,
    data,
    ...(meta && { meta }) // Only include meta if provided
  } as ApiResponse<T>);
};

// Sends error response with message and status code
// Usage: sendError(res, 'Validation failed', 422)
export const sendError = (
  res: Response,
  message: string,
  statusCode: number = 400
): Response => {
  return res.status(statusCode).json({
    success: false,
    error: message
  } as ApiResponse);
};

// Convenience wrapper for 404 responses
// Usage: sendNotFound(res, 'User') → { success: false, error: 'User not found' }
export const sendNotFound = (res: Response, resource: string = 'Resource'): Response => {
  return sendError(res, `${resource} not found`, 404);
};
```

Resumen del código:
- `ApiResponse<T>`: Un contrato de TypeScript para todas las respuestas: `{ success, data?, error?, meta? }`. `meta` es información de paginación opcional.
- `sendSuccess<T>`: Envía un JSON 2xx con `{ success: true, data, meta? }`. Es genérico (`<T>`), por lo que tus datos están completamente tipados. El bloque `meta` se agrega condicionalmente a través de un operador spread.
- `sendError`: Envía un JSON 4xx/5xx con `{ success: false, error }`. El estado predeterminado es 400, pero puedes pasar cualquier código.
- `sendNotFound`: Contenedor conveniente alrededor de `sendError` que devuelve un 404 con `"${resource} not found"`.

Los genéricos te permiten escribir funciones que funcionan con cualquier tipo mientras preservan la información de tipos en todo momento. El `<T>` en `sendSuccess<T>` es un parámetro de tipo, un marcador de posición que TypeScript completa en función de lo que realmente pasas. Cuando llamas a `sendSuccess(res, { id: 1, name: 'Carlos' })`, TypeScript infiere que `T` es `{ id: number; name: string }` y traslada ese tipo a la estructura de respuesta. Sin genéricos, perderías la seguridad de tipos al usar `any`, o necesitarías funciones separadas para cada forma de datos posible. Los genéricos te brindan código reutilizable sin sacrificar las comprobaciones en tiempo de compilación.

Efecto neto: Las respuestas de tu API son consistentes, tipadas y predecibles, y el frontend siempre puede bifurcar según `success` y leer `meta` para la paginación cuando esté presente.

Estas funciones auxiliares garantizan que cada respuesta de la API siga la misma estructura. Los desarrolladores de frontend pueden confiar en el campo `success` para determinar si una solicitud tuvo éxito, y el objeto `meta` proporciona información de paginación cuando es necesario.

Con una estructura de respuesta consistente y predecible establecida, el siguiente paso es hacer que los datos detrás de esas respuestas sean igual de confiables. Combinaremos la solidez de PostgreSQL con los tipos de TypeScript de extremo a extremo de Drizzle ORM para que nuestras consultas se validen en tiempo de compilación, nuestros modelos permanezcan estrechamente alineados con el esquema de la base de datos y los resultados que devuelva la API sean tan confiables como el formato que los envuelve.

---

## Integración de PostgreSQL con Drizzle ORM

PostgreSQL nos brinda una base sólida para almacenar datos, pero las consultas SQL sin procesar son propensas a errores, carecen de seguridad de tipos y un solo error de concatenación de cadenas puede abrir la puerta a ataques de inyección SQL. Drizzle ORM cierra esta brecha al proporcionar consultas parametrizadas de forma predeterminada y tipos de TypeScript que se mantienen alineados con el esquema de tu base de datos, detectando nombres de columnas no coincidentes y errores de tipo en tiempo de compilación en lugar de en los registros de producción.

### Configuración de una base de datos PostgreSQL

Antes de escribir cualquier código, necesitas una instancia de PostgreSQL en ejecución. Docker es una de las opciones más rápidas para el desarrollo si ya lo tienes instalado; mantiene a PostgreSQL aislado de tu sistema y hace que la limpieza sea trivial. Si prefieres una instalación nativa, funciona igual de bien; simplemente actualiza tu `DATABASE_URL` en consecuencia. Crea un archivo `docker-compose.yml` en la raíz de tu proyecto:

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: development
      POSTGRES_DB: myapp_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

Ejecuta `docker-compose up -d` para iniciar el contenedor en modo desacoplado (*detached*); se ejecuta en segundo plano y sobrevive al cierre de la terminal. La primera ejecución descarga la imagen de PostgreSQL, lo que puede tardar un minuto dependiendo de tu conexión. Estos son los comandos esenciales para administrar tu contenedor:

- `docker ps`: Enumera todos los contenedores activos junto con sus nombres, puertos y tiempo de actividad. Úsalo para verificar que PostgreSQL se esté ejecutando y exponiendo el puerto 5432.
- `docker-compose down`: Detiene y elimina el contenedor. Agrega el indicador `-v` para eliminar también el volumen de datos, útil para comenzar de nuevo, pero destruye tu base de datos.
- `docker-compose stop` y `docker-compose start`: Pausan y reanudan el contenedor sin eliminarlo, preservando tus datos entre sesiones.

La cadena de conexión se convierte en: `postgresql://myuser:mypassword@localhost:5432/myapp_dev`, que va en tu archivo `.env`.

### Instalación y configuración de Drizzle ORM

Drizzle requiere algunos paquetes para funcionar con PostgreSQL y TypeScript. Instálalos junto con la herramienta CLI para migraciones:

```bash
npm install drizzle-orm postgres
npm install -D drizzle-kit
```

Crea un archivo de configuración de Drizzle que le indique dónde vive tu esquema y cómo conectarse a la base de datos:

```typescript
// src/drizzle.config.ts
import 'dotenv/config'
import { defineConfig } from 'drizzle-kit'

const url = process.env.DATABASE_URL!

export default defineConfig({
  schema: './src/db/schema.ts',
  out: './src/db/migrations',
  dialect: 'postgresql',
  dbCredentials: {
    url
  }
})
```

Ahora necesitamos una conexión de base de datos que nuestra aplicación utilizará. Drizzle trabaja con el paquete `postgres` para crear un grupo de conexiones (*connection pool*):

```typescript
// src/db/index.ts
import dotenv from 'dotenv'
import { drizzle } from 'drizzle-orm/postgres-js'
import postgres from 'postgres'
import * as schema from './schema'

dotenv.config()

const url = process.env.DATABASE_URL!
const queryClient = postgres(url)

export const db = drizzle(queryClient, { schema })
export type Database = typeof db
```

Esta conexión se reutiliza en toda tu aplicación. El agrupamiento de conexiones (*connection pooling*) ocurre automáticamente, manejando la mayoría de los escenarios de fábrica. Dicho esto, los entornos de producción con mucho tráfico aún pueden agotar el grupo si las solicitudes superan las conexiones disponibles; es posible que debas ajustar el tamaño del grupo, implementar colas de solicitudes o agregar tiempos de espera de conexión según tu carga.

### Definición de modelos y esquemas de base de datos con TypeScript

Los esquemas de Drizzle son archivos TypeScript que definen la estructura de tu base de datos. Cada tabla se convierte en un objeto tipado que puedes consultar con soporte completo de autocompletado:

```typescript
// src/db/schema.ts
import { boolean, customType, index, integer, pgTable, text, timestamp, varchar } from 'drizzle-orm/pg-core'
import { relations, sql } from 'drizzle-orm'

// Custom tsvector type for PostgreSQL full-text search
const tsvector = customType<{ data: string }>({
  dataType() {
    return 'tsvector'
  }
})

export const users = pgTable(
  'users',
  {
    id: integer('id').primaryKey().generatedAlwaysAsIdentity(),
    email: varchar('email', { length: 255 }).notNull().unique(),
    name: varchar('name', { length: 255 }).notNull(),
    bio: text('bio'),
    isActive: boolean('is_active').notNull().default(true),
    createdAt: timestamp('created_at', { withTimezone: false }).notNull().defaultNow(),
    updatedAt: timestamp('updated_at', { withTimezone: false }).notNull().defaultNow()
  },
  (table) => [
    index('users_email_idx').on(table.email),
    index('users_active_idx').on(table.isActive),
    index('users_created_at_idx').on(table.createdAt)
  ]
)

export const posts = pgTable(
  'posts',
  {
    id: integer('id').primaryKey().generatedAlwaysAsIdentity(),
    title: varchar('title', { length: 255 }).notNull(),
    content: text('content').notNull(),
    published: boolean('published').notNull().default(false),
    authorId: integer('author_id')
      .notNull()
      .references(() => users.id, { onDelete: 'cascade' }),
    searchVector: tsvector('search_vector').generatedAlwaysAs(
      sql`to_tsvector('english', coalesce(title, '') || ' ' || coalesce(content, ''))`
    ),
    createdAt: timestamp('created_at', { withTimezone: false }).notNull().defaultNow(),
    updatedAt: timestamp('updated_at', { withTimezone: false }).notNull().defaultNow()
  },
  (table) => [
    index('posts_author_idx').on(table.authorId),
    index('posts_published_idx').on(table.published),
    index('posts_created_at_idx').on(table.createdAt),
    index('posts_author_published_idx').on(table.authorId, table.published),
    index('posts_search_idx').using('gin', table.searchVector)
  ]
)

export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts)
}))

export const postsRelations = relations(posts, ({ one }) => ({
  author: one(users, {
    fields: [posts.authorId],
    references: [users.id]
  })
}))
```

El esquema define dos tablas con una relación de uno a muchos. Los usuarios pueden tener muchas publicaciones y cada publicación pertenece a un usuario. Drizzle infiere los tipos de TypeScript a partir de este esquema automáticamente; cuando consultas `users`, obtienes un objeto tipado con `id`, `email`, `name` y todos los demás campos.

Las definiciones de `relations` le indican a Drizzle cómo se conectan las tablas. Cuando consultas un usuario, puedes incluir sus publicaciones con una opción simple. Cuando consultas publicaciones, puedes incluir al autor. No se requieren uniones (*joins*) manuales.

### Automatización de migraciones de base de datos con Drizzle

Las migraciones rastrean los cambios en el esquema de tu base de datos a lo largo del tiempo. Cuando modificas una tabla, Drizzle genera un archivo de migración que describe el cambio. Estos archivos residen en el control de versiones junto con tu código.

Genera una migración después de crear tu esquema:

```bash
npm run generate
```

Esto crea archivos SQL en el directorio `drizzle`. Cada archivo contiene los comandos SQL necesarios para actualizar tu base de datos. Para aplicar las migraciones:

```bash
npm run migrate
```

Drizzle compara tu esquema con el estado actual de la base de datos y aplica solo los cambios necesarios. ¿Agregas una columna? Genera una sentencia `ALTER TABLE`. ¿Renombras un campo? Crea la migración adecuada. ¿Eliminas una tabla? La migración lo maneja de manera segura, pero de manera segura aquí significa que ejecuta SQL explícito y predecible, no que respalde automáticamente tus datos. Las estrategias de copias de seguridad y reversión siguen siendo tu responsabilidad y deben formar parte de tu proceso de despliegue.

Para tener un mayor control, crea un script de migración que se ejecute cuando se inicie tu aplicación:

```typescript
// src/db/migrate.ts
import dotenv from 'dotenv'
import { drizzle } from 'drizzle-orm/postgres-js'
import { migrate } from 'drizzle-orm/postgres-js/migrator'
import postgres from 'postgres'

dotenv.config()

const runMigrations = async () => {
  const migrationClient = postgres(process.env.DATABASE_URL!, { max: 1 })
  if (!migrationClient) {
    throw new Error('DATABASE_URL is not set')
  }

  const migrationDb = drizzle(migrationClient)

  console.log('Running migrations...')
  await migrate(migrationDb, { migrationsFolder: './src/db/migrations' })
  console.log('Migrations completed')

  await migrationClient.end()
}

if (import.meta.url === `file://${process.argv[1]}`) {
  runMigrations()
    .then(() => process.exit(0))
    .catch((error) => {
      console.error('Migration failed:', error)
      process.exit(1)
    })
}
```

Ejecuta este script con `npm run migrate` antes de iniciar tu servidor. En producción, ejecuta las migraciones como parte de tu pipeline de despliegue. Nunca omitas las migraciones; garantizan que la estructura de tu base de datos coincida con tu código.

---

## Generación automática de APIs CRUD dinámicas

Ahora viene la parte emocionante: construir una API que se genera a sí misma. Crearemos un sistema que lee tu esquema de Drizzle y crea endpoints CRUD para cada tabla automáticamente.

### Generación dinámica de endpoints basados en modelos de base de datos

La clave para las APIs automáticas es la introspección. Drizzle proporciona acceso a tu esquema en tiempo de ejecución, lo que significa que podemos iterar sobre las tablas y crear rutas dinámicamente:

```typescript
// src/api/factory.ts
import type { Router } from 'express'
import { and, eq, getTableColumns, ilike, or, sql } from 'drizzle-orm'
import type { AnyPgTable } from 'drizzle-orm/pg-core'
import { db } from '../db/index'
import * as schema from '../db/schema'
import { sendError, sendNotFound, sendSuccess } from '../utils/responses'
import { buildPaginationMeta, parsePaginationParams } from '../utils/pagination'
import { isTable, EXPOSED_TABLES, searchableColumns, asyncHandler } from '../utils/table'
import { validate } from '../middleware/validate'
import { getValidationSchema } from '../validation/schemas'

type TableName = keyof typeof schema

// Use Router type - Express app is compatible since it extends Router's interface
export const registerDynamicRoutes = (app: Router) => {
  const tables = Object.entries(schema).filter(
    ([key, value]) => !key.includes('Relations') && isTable(value) && EXPOSED_TABLES.includes(key)
  ) as [TableName, AnyPgTable][]

  tables.forEach(([tableName, table]) => {
    const resource = String(tableName)
    const basePath = `/api/${resource}`
    const columns = getTableColumns(table)
    const idColumn = columns.id

    if (!idColumn) {
      return
    }

    app.get(
      basePath,
      asyncHandler(async (req, res) => {
        try {
          const pagination = parsePaginationParams(req)
          const { search, ...filters } = req.query

          const conditions = Object.entries(filters)
            .filter(([key]) => key in columns)
            .map(([key, value]) => {
              let parsed: unknown = value
              if (
                typeof value === 'string' &&
                value.trim() !== '' &&
                !Number.isNaN(Number(value))
              ) {
                parsed = Number(value)
              }
              return eq(columns[key] as any, parsed as any)
            })

          if (search && typeof search === 'string') {
            const searchConditions = Object.entries(columns)
              .filter(([column]) => searchableColumns.includes(column))
              .map(([, column]) => ilike(column as any, `%${search}%`))

            if (searchConditions.length === 1) {
              conditions.push(searchConditions[0])
            } else if (searchConditions.length > 1) {
              conditions.push(or(...searchConditions)!)
            }
          }

          const query = db.select().from(table)

          if (conditions.length === 1) {
            query.where(conditions[0])
          } else if (conditions.length > 1) {
            query.where(and(...conditions))
          }

          const records = await query.limit(pagination.limit).offset(pagination.offset)
          const [{ count }] = await db.select({ count: sql<number>`count(*)` }).from(table)

          const meta = buildPaginationMeta(Number(count), pagination)
          sendSuccess(res, records, 200, meta)
        } catch (error) {
          console.error(error)
          sendError(res, 'Failed to fetch records', 500)
        }
      })
    )

    // Additional routes continue below...
  });
};
```

Esta función descubre cada tabla en tu esquema y crea una ruta `GET` para obtener registros. La ruta actualmente realiza una coincidencia exacta de cadenas en cada parámetro de consulta. De modo que `GET /users?name=Alice` funciona como se espera, pero `?isActive=true` no coincidirá con una columna booleana, y las claves repetidas (`?email=a&email=b`) no se tratan como un filtro `IN`. La coerción de valores por tipo de columna y la compatibilidad con operadores de arreglos se dejan como una extensión. Internamente, el bucle de filtro solo maneja valores donde `typeof value === 'string'` y convierte el resultado al operando `eq` de Drizzle como `any`, por lo que cuando Express entrega un arreglo para claves repetidas, el valor elude la verificación, y cualquier entrada sin procesar que se filtre puede manifestarse como un error SQL opaco en lugar de un 400 limpio. Siempre analiza y valida la entrada de consulta a través de un esquema tipado antes de alimentarla a consultas dinámicas.

Agreguemos las operaciones CRUD restantes:

```typescript
// Continuing src/api/factory.ts
    // GET by ID
    app.get(
      `${basePath}/:id`,
      asyncHandler(async (req, res) => {
        const { id } = req.params
        const [record] = await db
          .select()
          .from(table)
          .where(eq(idColumn as any, Number(id)))
          .limit(1)

        if (!record) {
          return sendNotFound(res, resource)
        }

        sendSuccess(res, record)
      })
    )

    // CREATE
    app.post(
      basePath,
      validate(getValidationSchema(resource, 'create')),
      asyncHandler(async (req, res) => {
        const data = req.body
        const [created] = await db.insert(table).values(data).returning()
        sendSuccess(res, created, 201)
      })
    )

    // UPDATE
    app.put(
      `${basePath}/:id`,
      validate(getValidationSchema(resource, 'update')),
      asyncHandler(async (req, res) => {
        const { id } = req.params
        const data = req.body
        const [updated] = await db
          .update(table)
          .set(data)
          .where(eq(idColumn as any, Number(id)))
          .returning()

        if (!updated) {
          return sendNotFound(res, resource)
        }

        sendSuccess(res, updated)
      })
    )

    // DELETE
    app.delete(
      `${basePath}/:id`,
      asyncHandler(async (req, res) => {
        const { id } = req.params
        const deleted = await db
          .delete(table)
          .where(eq(idColumn as any, Number(id)))
          .returning()

        if (deleted.length === 0) {
          return sendNotFound(res, resource)
        }

        sendSuccess(res, { deleted: true })
      })
    )
  })
}
```

Con estas rutas en su lugar, cada tabla en tu base de datos se convierte en una API completamente funcional. ¿Agregas una tabla `posts`? Inmediatamente obtienes endpoints para crear, leer, actualizar y eliminar publicaciones. ¿Agregas una tabla `comments`? Lo mismo. La API crece con tu base de datos.

Registra estas rutas en tu archivo principal de servidor:

```typescript
// Add to src/index.ts after middleware
import { registerDynamicRoutes } from './api/factory';

registerDynamicRoutes(app);
```

Tu API ahora es automática. Define una tabla en tu esquema, ejecuta las migraciones y los endpoints aparecerán. Sin controladores, sin archivos de rutas, sin código repetitivo.

### Manejo de relaciones entre tablas

El CRUD dinámico es potente, pero las aplicaciones reales necesitan más que consultas simples. Cuando obtienes un usuario, a menudo también deseas sus publicaciones. Cuando obtienes una publicación, es posible que necesites la información del autor. Las consultas relacionales de Drizzle hacen que esto sea elegante.

Utiliza este fragmento para registrar dos rutas de Express que aprovechan las consultas relacionales de Drizzle: una devuelve un usuario junto con todas sus publicaciones y la otra devuelve una publicación junto con su autor, mientras reutiliza los ayudantes de respuesta estandarizados:

```typescript
// src/api/relational.ts
import type { Express, Request, Response } from 'express'
import { eq } from 'drizzle-orm'
import { db } from '../db/index'
import { posts, users } from '../db/schema'
import { sendError, sendNotFound, sendSuccess } from '../utils/responses'

export const registerRelationalRoutes = (app: Express) => {
  app.get('/api/users/:id/with-posts', async (req: Request, res: Response) => {
    try {
      const { id } = req.params
      const user = await db.query.users.findFirst({
        where: eq(users.id, Number(id)),
        with: { posts: true }
      })

      if (!user) {
        return sendNotFound(res, 'User')
      }

      sendSuccess(res, user)
    } catch (error) {
      console.error(error)
      sendError(res, 'Failed to fetch user with posts', 500)
    }
  })

  app.get('/api/posts/:id/with-author', async (req: Request, res: Response) => {
    try {
      const { id } = req.params
      const post = await db.query.posts.findFirst({
        where: eq(posts.id, Number(id)),
        with: { author: true }
      })

      if (!post) {
        return sendNotFound(res, 'Post')
      }

      sendSuccess(res, post)
    } catch (error) {
      console.error(error)
      sendError(res, 'Failed to fetch post with author', 500)
    }
  })
}
```

La opción `with` le indica a Drizzle que incluya los registros relacionados. Tras bambalinas, genera uniones SQL eficientes. El resultado es una estructura de objetos anidados que coincide exactamente con tus relaciones, con seguridad de tipos y de forma automática.

También puedes hacer esto dinámico. En lugar de codificar rutas de relación fijas (*hardcoded*), inspecciona las definiciones de relaciones en tu esquema y genera rutas para cada relación:

```typescript
// src/api/relational-factory.ts
import type { Router } from 'express'
import { eq, getTableColumns } from 'drizzle-orm'
import { db } from '../db/index'
import * as schema from '../db/schema'
import { sendError, sendNotFound, sendSuccess } from '../utils/responses'
import { EXPOSED_TABLES, isTable, getRelationNames, asyncHandler } from '../utils/table'

export const registerDynamicRelationalRoutes = (app: Router) => {
  const relationEntries = Object.entries(schema).filter(([key]) => key.endsWith('Relations'))

  relationEntries.forEach(([relationKey, relationValue]) => {
    const tableName = relationKey.replace('Relations', '')

    // Skip tables not in the allowlist
    if (!EXPOSED_TABLES.includes(tableName)) {
      return
    }

    const table = (schema as Record<string, unknown>)[tableName]
    if (!table || !isTable(table)) {
      return
    }

    const columns = getTableColumns(table)
    const idColumn = columns.id
    if (!idColumn) {
      return
    }

    const relationNames = getRelationNames(relationValue)

    relationNames.forEach((relationName) => {
      const route = `/api/${tableName}/:id/with-${relationName}`

      app.get(
        route,
        asyncHandler(async (req, res) => {
          try {
            const { id } = req.params
            const query = (db.query as Record<string, any>)[tableName]

            if (!query) {
              return sendError(res, 'Relation not configured', 500)
            }

            const record = await query.findFirst({
              where: eq(idColumn as any, Number(id)),
              with: { [relationName]: true }
            })

            if (!record) {
              return sendNotFound(res, tableName)
            }

            sendSuccess(res, record)
          } catch (error) {
            console.error(error)
            sendError(res, 'Failed to fetch related data', 500)
          }
        })
      )
    })
  })
}
```

Ahora cada relación en tu esquema obtiene automáticamente un endpoint. Define un usuario con publicaciones y obtendrás `/users/:id/with-posts`. Define una publicación con un autor y obtendrás `/posts/:id/with-author`. La API comprende tu modelo de datos y se adapta en consecuencia.

---

## Optimización del rendimiento de APIs

Las APIs dinámicas son convenientes, pero la conveniencia no significa nada si tu API es lenta. A medida que tu aplicación crece, la optimización del rendimiento se vuelve crítica. El almacenamiento en caché y la optimización de consultas pueden transformar una API lenta en una que se sienta instantánea.

A medida que crece el tráfico, los endpoints se ralentizan debido a lecturas repetidas en la base de datos, consultas N+1 y a la realización del mismo trabajo en cada solicitud; esta sección aborda ese problema con técnicas prácticas: almacenamiento en caché, procesamiento por lotes de consultas (*query batching*), paginación, indexación, agrupamiento de conexiones y cargas útiles más ligeras, para que puedas perfilar puntos críticos, agregar almacenamiento en caché específico, reducir viajes de ida y vuelta (*round-trips*) y verificar mejoras con métricas de latencia. Comenzaremos implementando almacenamiento en caché de lectura completa (*read-through caching*) utilizando una pequeña caché en memoria (con orientación sobre Redis para producción), envolviendo un endpoint de lectura consultado con frecuencia, seleccionando TTLs sensatos e invalidando en las escrituras para que las solicitudes repetidas omitan la base de datos y respondan más rápido.

### Implementación de almacenamiento en caché para respuestas más rápidas de la API

Las consultas a la base de datos son costosas; si bien un simple `SELECT` solo puede tomar milisegundos, esos retrasos se acumulan a escala al manejar cientos de solicitudes por segundo. El almacenamiento en caché puede ayudar almacenando los resultados solicitados con frecuencia en la memoria para que las solicitudes repetidas omitan la base de datos, pero no siempre es necesario. Para muchas consultas simples, el agrupamiento de conexiones por sí solo es suficiente para mantener el rendimiento. El almacenamiento en caché es más adecuado para datos de alto tráfico y baja volatilidad, y tiene sus propias compensaciones, incluida la sobrecarga de memoria, la complejidad de la invalidación de la caché y el mantenimiento operativo continuo.

Usaremos una caché simple en memoria para los datos a los que se accede con frecuencia. Para aplicaciones de producción, Redis proporciona una capa de caché robusta, pero una caché en memoria demuestra el concepto con claridad:

```typescript
// src/utils/cache.ts
interface CacheEntry<T> {
  data: T
  timestamp: number
  ttl: number
}

class SimpleCache {
  private cache = new Map<string, CacheEntry<unknown>>()

  set<T>(key: string, data: T, ttlSeconds = 3600): void {
    this.cache.set(key, {
      data,
      timestamp: Date.now(),
      ttl: ttlSeconds * 1000
    })
  }

  get<T>(key: string): T | null {
    const entry = this.cache.get(key)
    if (!entry) {
      return null
    }

    const isExpired = Date.now() - entry.timestamp > entry.ttl
    if (isExpired) {
      this.cache.delete(key)
      return null
    }

    return entry.data as T
  }

  delete(key: string): void {
    this.cache.delete(key)
  }

  clear(): void {
    this.cache.clear()
  }

  invalidatePattern(pattern: string): void {
    const regex = new RegExp(pattern)
    for (const key of this.cache.keys()) {
      if (regex.test(key)) {
        this.cache.delete(key)
      }
    }
  }
}

export const cache = new SimpleCache()
```

Esta caché almacena datos con un tiempo de expiración. Cuando obtienes un usuario, almacenas el resultado en la caché durante una hora. La siguiente solicitud para ese usuario lee de la memoria en lugar de consultar la base de datos. Cuando expira la hora, la entrada de la caché se elimina automáticamente.

Integra el almacenamiento en caché en tus rutas dinámicas:

```typescript
// src/api/cached-factory.ts
import type { Router } from 'express'
import { eq, getTableColumns } from 'drizzle-orm'
import type { AnyPgTable } from 'drizzle-orm/pg-core'
import { db } from '../db/index'
import * as schema from '../db/schema'
import { sendError, sendNotFound, sendSuccess } from '../utils/responses'
import { cache } from '../utils/cache'
import { isTable, inFlight, getWithCache, asyncHandler, EXPOSED_TABLES } from '../utils/table'

type TableName = keyof typeof schema

export const registerCachedRoutes = (app: Router) => {
  const tables = Object.entries(schema).filter(
    ([key, value]) => !key.includes('Relations') && isTable(value) && EXPOSED_TABLES.includes(key)
  ) as [TableName, AnyPgTable][]

  tables.forEach(([tableName, table]) => {
    const columns = getTableColumns(table)
    const idColumn = columns.id

    if (!idColumn) {
      return
    }

    const basePath = `/api/${String(tableName)}`

    app.get(
      `${basePath}/:id`,
      asyncHandler(async (req, res) => {
        try {
          const { id } = req.params
          const cacheKey = `${tableName}:${id}`

          const record = await getWithCache(
            cacheKey,
            async () => {
              const [result] = await db
                .select()
                .from(table)
                .where(eq(idColumn as any, Number(id)))
                .limit(1)

              return result ?? null
            },
            Number(process.env.CACHE_TTL) || 3600
          )

          if (!record) {
            return sendNotFound(res, String(tableName))
          }

          sendSuccess(res, record)
        } catch (error) {
          console.error(error)
          sendError(res, 'Failed to fetch record', 500)
        }
      })
    )
  })
}
```

`registerCachedRoutes` solo registra un manejador `GET /:id` almacenado en caché: no duplica las rutas `POST`, `PUT` o `DELETE` de `registerDynamicRoutes`. Debido a que Express envía a la primera ruta coincidente, registra la versión en caché antes de la factoría dinámica en `index.ts` para que las lecturas en caché tengan prioridad mientras que las escrituras sigan fluyendo a través de la factoría dinámica (y su middleware de validación). Con este orden, la primera solicitud de un usuario por ID llega a la base de datos y almacena el resultado; las siguientes 100 solicitudes de ese mismo usuario leen de la caché instantáneamente, reduciendo el tiempo de respuesta de ~50ms a menos de 1ms:

```typescript
// Add to PUT route in cached-factory.ts
app.put(`${basePath}/:id`, asyncHandler(async (req, res) => {
  try {
    const { id } = req.params;
    const data = req.body;

    const [updated] = await db
      .update(table)
      .set(data)
      .where(eq(idColumn as any, Number(id)))
      .returning();

    if (!updated) {
      return sendNotFound(res, String(tableName));
    }

    // Invalidate cache for this record and any related queries
    cache.delete(`${tableName}:${id}`);
    cache.invalidatePattern(`${tableName}:.*with.*`);

    // Also clear any in-flight requests for this key
    // (edge case: update happens while a read is in progress)
    inFlight.delete(`${tableName}:${id}`);

    sendSuccess(res, updated);
  } catch (error) {
    console.error(error);
    sendError(res, 'Failed to update record', 400);
  }
}));
```

Cuando actualizas un usuario, se elimina la entrada de caché para ese usuario. Cualquier consulta almacenada en caché que incluya al usuario (como "usuario con publicaciones") también se invalida. La siguiente solicitud obtendrá datos nuevos de la base de datos y volverá a llenar la caché.

### Manejo de paginación y optimización de consultas

La paginación evita que tu API devuelva miles de registros a la vez. Sin paginación, una consulta como `GET /posts` podría devolver 10,000 publicaciones, abrumando tanto a tu servidor como al cliente. La paginación divide grandes conjuntos de resultados en páginas manejables.

Ya agregamos paginación básica a nuestras rutas dinámicas, pero podemos hacerla más sofisticada:

```typescript
// src/utils/pagination.ts
import { Request } from 'express'

export interface PaginationParams {
  page: number
  limit: number
  offset: number
}

export interface PaginationMeta {
  page: number
  limit: number
  total: number
  totalPages: number
  hasNext: boolean
  hasPrev: boolean
}

export const parsePaginationParams = (req: Request): PaginationParams => {
  const page = Math.max(1, parseInt(req.query.page as string, 10) || 1)
  const limit = Math.min(100, Math.max(1, parseInt(req.query.limit as string, 10) || 10))
  const offset = (page - 1) * limit

  return { page, limit, offset }
}

export const buildPaginationMeta = (total: number, params: PaginationParams): PaginationMeta => {
  const totalPages = Math.max(1, Math.ceil(total / params.limit))

  return {
    page: params.page,
    limit: params.limit,
    total,
    totalPages,
    hasNext: params.page < totalPages,
    hasPrev: params.page > 1
  }
}
```

Estas utilidades estandarizan la paginación en toda tu API. La función `parsePaginationParams` extrae `page` y `limit` de la cadena de consulta, con valores predeterminados y límites razonables (máximo 100 elementos por página). La función `buildPaginationMeta` crea metadatos que indican a los clientes cuántas páginas existen y si pueden navegar hacia adelante o hacia atrás.

La optimización de consultas va más allá de la paginación. Los índices de la base de datos aceleran drásticamente las consultas, especialmente para las columnas que se buscan con frecuencia:

```typescript
// Add to src/db/schema.ts
import { index } from 'drizzle-orm/pg-core'

export const users = pgTable('users', {
  // ... existing fields
}, (table) => [
  index('users_email_idx').on(table.email),
  index('users_active_idx').on(table.isActive),
  index('users_created_at_idx').on(table.createdAt)
])

export const posts = pgTable('posts', {
  // ... existing fields
}, (table) => [
  index('posts_author_idx').on(table.authorId),
  index('posts_published_idx').on(table.published),
  index('posts_created_at_idx').on(table.createdAt),
  index('posts_author_published_idx').on(table.authorId, table.published),
  index('posts_search_idx').using('gin', table.searchVector)
])
```

Los índices funcionan como los índices de los libros; permiten que la base de datos encuentre filas específicas sin escanear toda la tabla. Si consultas con frecuencia usuarios por correo electrónico, un índice en el correo electrónico hace que esa consulta sea casi instantánea. Si filtras publicaciones por `authorId`, un índice en esa columna es esencial.

Los índices compuestos ayudan cuando filtras por múltiples columnas juntas:

```typescript
// src/db/schema.ts
export const posts = pgTable('posts', {
  // ... existing fields
}, (table) => [
  index('posts_author_idx').on(table.authorId),
  index('posts_published_idx').on(table.published),
  index('posts_created_at_idx').on(table.createdAt),
  index('posts_author_published_idx').on(table.authorId, table.published),
  index('posts_search_idx').using('gin', table.searchVector)
]
)
```

Este índice compuesto optimiza consultas como "encontrar todas las publicaciones publicadas por este autor", un patrón común. Sin el índice, PostgreSQL escanea cada publicación. Con el índice, salta directamente a las filas relevantes.

La funcionalidad de búsqueda se beneficia de las capacidades de búsqueda de texto completo (*full-text search*) de PostgreSQL. En lugar de usar consultas `LIKE` que escanean columnas enteras, la búsqueda de texto completo utiliza índices especializados:

```typescript
// src/api/search.ts
import type { Express, Request, Response } from 'express'
import { sql } from 'drizzle-orm'
import { db } from '../db/index'
import { posts } from '../db/schema'
import { sendError, sendSuccess } from '../utils/responses'
import { buildPaginationMeta, parsePaginationParams } from '../utils/pagination'

export const registerSearchRoutes = (app: Express) => {
  app.get('/api/search/posts', async (req: Request, res: Response) => {
    try {
      const { q } = req.query

      if (!q || typeof q !== 'string') {
        return sendError(res, 'Search query required', 400)
      }

      const pagination = parsePaginationParams(req)

      const results = await db
        .select()
        .from(posts)
        // You can specify the language you want to use
        .where(sql`${posts.searchVector} @@ plainto_tsquery('english', ${q})`)
        .limit(pagination.limit)
        .offset(pagination.offset)

      const [{ count }] = await db
        .select({ count: sql<number>`count(*)` })
        .from(posts)
        .where(sql`${posts.searchVector} @@ plainto_tsquery('english', ${q})`)

      const meta = buildPaginationMeta(Number(count), pagination)

      sendSuccess(res, results, 200, meta)
    } catch (error) {
      console.error(error)
      sendError(res, 'Search failed', 500)
    }
  })
}
```

La búsqueda de texto completo comprende el lenguaje: lematiza palabras, ignora palabras comunes y clasifica los resultados por relevancia. Buscar "running" coincide con "run", "runs" y "runner". Una consulta `LIKE` solo coincide con subcadenas exactas.

Para un rendimiento de búsqueda aún mejor, agrega una columna generada con un índice GIN:

```typescript
// src/db/schema.ts
import { sql } from 'drizzle-orm';

export const posts = pgTable('posts', {
  id: integer('id').primaryKey().generatedAlwaysAsIdentity(),
  title: varchar('title', { length: 255 }).notNull(),
  content: text('content').notNull(),
  searchVector: tsvector('search_vector').generatedAlwaysAs(
    sql`to_tsvector('english', coalesce(title, '') || ' ' || coalesce(content, ''))`
  ),
  // ... other fields
}, (table) => [
  ...other indexes
  index('posts_search_idx').using('gin', table.searchVector)
]
)
```

Esta columna generada se actualiza automáticamente cada vez que cambia el título o el contenido. El índice GIN hace que las consultas de búsqueda sean increíblemente rápidas, incluso con millones de publicaciones. Tu ruta de búsqueda también se vuelve más simple:

```typescript
// Simplified search with generated column
const results = await db
  .select()
  .from(posts)
  .where(sql`${posts.searchVector} @@ plainto_tsquery('english', ${q})`)
  .limit(pagination.limit)
  .offset(pagination.offset)
```

La optimización del rendimiento consiste en tomar decisiones inteligentes. Almacena en caché los datos a los que se accede con frecuencia. Indexa las columnas que consultas a menudo. Utiliza la búsqueda de texto completo para consultas de texto. Pagina grandes conjuntos de resultados. Estas prácticas se potencian mutuamente; una API optimizada no solo es más rápida, sino que es escalable.

### Integrándolo todo

Hemos construido un sistema de backend completo que genera APIs automáticamente. Veamos cómo se conectan todas las piezas en un ejemplo final y cohesivo. Esta es la configuración completa de tu servidor:

```typescript
// src/index.ts
import dotenv from 'dotenv'
import express, { type Express, type NextFunction, type Request, type Response } from 'express'
import compression from 'compression'
import cors from 'cors'
import helmet from 'helmet'
import { registerDynamicRoutes } from './api/factory'
import { registerRelationalRoutes } from './api/relational'
import { registerDynamicRelationalRoutes } from './api/relational-factory'
import { registerCachedRoutes } from './api/cached-factory'
import { registerSearchRoutes } from './api/search'
import { apiLimiter } from './middleware/rateLimit'
import { authenticate } from './middleware/auth'

dotenv.config()

const app: Express = express()
const PORT = 3000

app.use(helmet())
app.use(
  cors({
    origin: 'http://localhost:3000',
    credentials: true
  })
)
app.use(compression())
app.use(express.json({ limit: '10mb' }))
app.use(express.urlencoded({ extended: true, limit: '10mb' }))
app.use('/api', authenticate)

app.get('/health', (_req: Request, res: Response) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() })
})

registerCachedRoutes(app)
registerDynamicRoutes(app)
registerRelationalRoutes(app)
registerDynamicRelationalRoutes(app)
registerSearchRoutes(app)

app.use((err: Error, _req: Request, res: Response, _next: NextFunction) => {
  console.error('Server error:', err)
  res.status(500).json({
    error: 'Internal server error',
    message: process.env.NODE_ENV === 'development' ? err.message : undefined
  })
})

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`)
})

export default app
```

El manejador de errores en la parte inferior de `index.ts` solo devuelve `err.message` al cliente cuando `NODE_ENV` está configurado en `development`. Asegúrate de que tu despliegue en producción establezca `NODE_ENV=production`. La mayoría de las plataformas de hosting hacen esto de forma predeterminada, pero el `.env.example` en este proyecto viene con `development`, por lo que una consulta fallida en ese modo puede filtrar SQL interno, nombres de tablas o detalles de la traza de pila (*stack trace*) en el cuerpo de la respuesta 500.

#### Creación de un nuevo usuario

Ejecuta la siguiente solicitud en tu Postman:

```http
POST /api/users
Body:
{
  "email": "carlos@gmail.com",
  "name": "Carlos Santana"
}
```

La solicitud llega a tu ruta dinámica `POST`, que inserta los datos en la tabla `users` y devuelve el nuevo registro con su ID generado. TypeScript detecta errores de tipo durante el desarrollo; si intentas pasar un número donde el esquema espera una cadena, tu editor lo marca de inmediato. Pero TypeScript desaparece en tiempo de ejecución. Un cliente malicioso o defectuoso aún puede enviar `{ "email": 12345 }` a través de HTTP y, sin validación en tiempo de ejecución, esos datos no válidos llegan a tu base de datos. El mismo riesgo se aplica a las columnas booleanas y numéricas: PostgreSQL convierte silenciosamente valores inesperados, por lo que sin validación una carga útil como `{ "isActive": "yes-please" }` puede convertirse en `false` y cambiar silenciosamente una bandera, mientras que `{ "age": "12abc" }` genera un error de base de datos opaco en lugar de un 400 limpio. Es por eso que la validación en tiempo de ejecución en el límite de la API es esencial. TypeScript garantiza que tu código sea internamente consistente, mientras que el middleware de validación protege contra la entrada externa que tu sistema de tipos nunca ve.

#### Creación de una nueva publicación para nuestro nuevo usuario

Después de crear el usuario, creemos la primera publicación usando Postman:

```http
POST /api/posts
Body:
{
  "title": "My First Post",
  "content": "Hello world!",
  "authorId": 1 // the id returned from POST /users
}
```

#### Obtención de un usuario con sus publicaciones

Una vez que creas el usuario y la publicación, ahora puedes obtener la publicación de ese usuario con esta solicitud:

```http
GET /api/users/1/with-posts
```

La ruta relacional consulta la base de datos utilizando la opción `with` de Drizzle, devolviendo el usuario y todas sus publicaciones en una sola consulta optimizada. La respuesta es anidada y segura en tipos:

```json
{
  "success": true,
  "data": {
    "id": 1,
    "email": "carlos@gmail.com",
    "name": "Carlos Santana",
    "posts": [
      {
        "id": 1,
        "title": "My First Post",
        "content": "Hello world!",
        "authorId": 1
      }
    ]
  }
}
```

#### Búsqueda de publicaciones

Si deseas buscar publicaciones y paginarlas, puedes hacer una solicitud como esta:

```http
GET /api/search/posts?q=typescript&page=1&limit=20
```

La ruta de búsqueda utiliza la búsqueda de texto completo de PostgreSQL para encontrar publicaciones relevantes. Los resultados se paginan y la respuesta incluye metadatos sobre los resultados totales y las páginas disponibles.

#### Actualización de una publicación

La siguiente solicitud es para actualizar una publicación específica:

```http
PUT /api/posts/1
Body:
{
  "title": "Updated Title"
}
```

La ruta en caché actualiza la publicación en la base de datos y luego invalida todas las entradas de caché relacionadas con esa publicación. La siguiente solicitud para esta publicación obtendrá datos nuevos.

La base está en su lugar: consultas con seguridad de tipos, generación automática de CRUD y operaciones de base de datos eficientes. Agrega una nueva tabla a tu esquema y estos endpoints aparecerán instantáneamente. Modifica una tabla y TypeScript hará cumplir los cambios en toda tu base de código. Lo que falta son las barandillas de seguridad (*guardrails*): validar los datos entrantes antes de que toquen la base de datos y controlar quién puede acceder a qué. Abordaremos ambos aspectos en las siguientes secciones.

---

## Patrones avanzados y consideraciones del mundo real

Lo que hemos construido maneja la mecánica de las operaciones CRUD, pero no está listo para producción. La autenticación, la validación y la limitación de tasa (*rate limiting*) no son extras opcionales; son requisitos fundamentales que pertenecen junto a tus rutas desde el primer día. Sin ellos, tu API acepta datos con formato incorrecto, atiende solicitudes a cualquiera y no tiene defensa contra el abuso. Agreguemos estas capas mientras preservamos la naturaleza automática de la generación de rutas.

### Adición de middleware de autenticación

El middleware de autenticación intercepta las solicitudes y verifica la identidad del usuario antes de llegar a tus rutas. Un enfoque simple basado en JWT funciona bien:

```typescript
// src/middleware/auth.ts
import { Request, Response, NextFunction } from 'express';
import { sendError } from '../utils/responses';

interface AuthRequest extends Request {
  userId?: number;
}

export const authenticate = async (
  req: AuthRequest,
  res: Response,
  next: NextFunction
): Promise<void> => {
  try {
    const token = req.headers.authorization?.replace('Bearer ', '');

    if (!token) {
      sendError(res, 'Authentication required', 401);
      return;
    }

    // In production, verify JWT and extract user ID
    // For this example, we'll simulate it
    const userId = extractUserIdFromToken(token);

    if (!userId) {
      sendError(res, 'Invalid token', 401);
      return;
    }

    req.userId = userId;
    next();
  } catch (error) {
    sendError(res, 'Authentication failed', 401);
  }
};

const extractUserIdFromToken = (token: string): number | null => {
  // Implement your JWT verification here
  // This is a placeholder
  return parseInt(token) || null;
};
```

Aplica la autenticación selectivamente a las rutas que necesitan protección:

```typescript
// src/api/protected-factory.ts
import { Express } from 'express';
import { authenticate } from '../middleware/auth';
import { registerDynamicRoutes } from './factory';

export const registerProtectedRoutes = (app: Express) => {
  // All routes under /api require authentication
  app.use('/api', authenticate);

  // Register dynamic routes under /api prefix
  registerDynamicRoutes(app);
};
```

Ahora, las solicitudes a `/api/users` requieren un token de autenticación válido, mientras que los endpoints públicos permanecen accesibles.

### Validación de entrada con Zod y drizzle-zod

Nunca confíes en la entrada del cliente. La validación garantiza que los datos coincidan con tus expectativas antes de que toquen la base de datos. Cubrimos Zod en profundidad en el [Capítulo 9](https://subscription.packtpub.com/book/web-development/9781806108251/9) para la validación de formularios en el lado del cliente; la misma librería funciona igual de bien en el servidor. La verificación de tipos en tiempo de ejecución complementa los tipos en tiempo de compilación de TypeScript.

Escribir manualmente esquemas de Zod para cada tabla socavaría nuestro objetivo de API dinámica; estarías duplicando las definiciones de tu esquema de Drizzle y manteniendo dos fuentes de verdad. El paquete `drizzle-zod` resuelve esto generando esquemas de validación de Zod directamente a partir de las definiciones de tablas de Drizzle. Agrega una columna a tu esquema de Drizzle y tus reglas de validación se actualizarán con ella. La compensación es el control: los esquemas generados hacen cumplir las restricciones de la base de datos, pero no incluirán reglas de negocio como la validación del formato de correo electrónico. Para esos casos, puedes extender los esquemas generados con los métodos `.refine()` o `.extend()` de Zod. Debido a que iteramos sobre el esquema de forma genérica, cada tabla se convierte a `any` antes de pasarse a `createInsertSchema`. Esta es una compensación deliberada, ya que la firma genérica de `drizzle-zod` requiere un tipo de tabla concreto, y las actualizaciones de versiones menores de cualquiera de las librerías pueden endurecer esa restricción, así que fija las versiones de `drizzle-orm`, `drizzle-zod` y `zod` en `package.json` y vuelve a ejecutar `tsc --noEmit` después de cada actualización:

```typescript
// src/validation/schemas.ts
import { createInsertSchema, createUpdateSchema } from 'drizzle-zod'
import * as schema from '../db/schema'
import { isTable } from '../utils/table'
import { z } from 'zod'

// Automatically generate Zod schemas from Drizzle tables
const generateSchemas = () => {
  const schemas: Record<string, { create?: z.ZodSchema; update?: z.ZodSchema }> = {}

  Object.entries(schema)
    .filter(([key, value]) => !key.includes('Relations') && isTable(value))
    .forEach(([tableName, table]) => {
      try {
        schemas[tableName] = {
          create: createInsertSchema(table as any),
          update: createInsertSchema(table as any).partial()
        }
      } catch (e) {
        // Optional: silently skip tables that fail
      }
    })

  return schemas
}

const schemaMap = generateSchemas()

export const getValidationSchema = (table: string, operation: 'create' | 'update') => {
  const schema = schemaMap[table]?.[operation]
  return schema ? { body: schema } : {}
}
```

Crea un middleware de validación que funcione con tus rutas dinámicas:

```typescript
// src/middleware/validate.ts
import type { Request, Response, NextFunction } from 'express'
import type { ZodError, ZodTypeAny } from 'zod'
import { sendError } from '../utils/responses'

type SchemaLike = ZodTypeAny

type ValidateSchemas = {
  body?: SchemaLike
  query?: SchemaLike
  params?: SchemaLike
}

function formatZodError(err: ZodError) {
  // ex: "query.page: Expected number, received nan"
  return err.issues
    .map((i) => {
      const path = i.path.length ? i.path.join('.') : '(root)'
      return `${path}: ${i.message}`
    })
    .join(', ')
}

export function validate(schemas: ValidateSchemas) {
  return (req: Request, res: Response, next: NextFunction): void => {
    try {
      if (schemas.params) {
        req.params = schemas.params.parse(req.params) as Record<string, string>
      }

      if (schemas.query) {
        // Express query values are typically strings / string[] / objects
        // Parse (and coerce if your schema uses z.coerce.*).
        req.query = schemas.query.parse(req.query) as any
      }

      if (schemas.body) {
        req.body = schemas.body.parse(req.body)
      }

      next()
    } catch (err: any) {
      // ZodError has `issues`; your previous code used `error.errors`.
      const zodErr = err as ZodError
      const messages = zodErr?.issues ? formatZodError(zodErr) : String(err?.message ?? err)
      sendError(res, `Validation failed: ${messages}`, 400)
    }
  }
}
```

Integra la validación en tus rutas:

```typescript
import { getValidationSchema } from '../validation/schemas'

...

// Enhanced POST route with validation
app.post(
  `/${tableName}`,
  validate(getValidationSchema(tableName, 'create')),
  async (req: Request, res: Response) => {
    // Route handler here
  }
);
```

El middleware de validación analiza el cuerpo de la solicitud con respecto al esquema. Los datos no válidos devuelven un error 400 con mensajes claros sobre lo que salió mal. Los datos válidos pasan al manejador de la ruta.

### Límite de tasa y limitación de tráfico

La limitación de tasa (*rate limiting*) evita el abuso al restringir la cantidad de solicitudes que un cliente puede realizar en un período de tiempo. Express Rate Limit proporciona una solución sencilla y eficaz:

```typescript
// src/middleware/rateLimit.ts
import rateLimit from 'express-rate-limit';

export const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Limit each IP to 100 requests per window
  message: 'Too many requests, please try again later',
  standardHeaders: true,
  legacyHeaders: false
});

export const strictLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 10, // 10 requests per minute
  message: 'Rate limit exceeded'
});
```

Aplica la limitación de tasa en diferentes niveles:

```typescript
// Apply to all API routes
app.use('/api', apiLimiter);

// Apply stricter limits to auth routes
app.use('/auth', strictLimiter);
```

Esto protege tu API tanto del abuso accidental (como bucles infinitos en el código del cliente) como de ataques maliciosos (abuso simple).

---

## Resumen

Este capítulo explora la creación de APIs escalables y autogeneradas utilizando Express, Node.js, PostgreSQL y Drizzle ORM. En lugar de escribir manualmente operaciones CRUD repetitivas para cada tabla de base de datos, creamos un sistema que inspecciona el esquema de tu base de datos y genera endpoints automáticamente. Al definir tablas en el esquema basado en TypeScript de Drizzle, la factoría de APIs lee estas definiciones en tiempo de ejecución y crea endpoints REST completos (`GET`, `POST`, `PUT` y `DELETE`) para cada tabla. El capítulo recorre la configuración de un servidor Express con el middleware adecuado, la configuración de Drizzle ORM para gestionar bases de datos PostgreSQL con consultas seguras de tipos y la creación de un sistema de generación de rutas dinámicas que elimina el código repetitivo. Las definiciones de relaciones de Drizzle permiten el manejo automático de consultas anidadas, por lo que obtener un usuario con sus publicaciones o una publicación con su autor no requiere código personalizado. El resultado es una API que crece orgánicamente con tu modelo de datos, donde agregar una nueva tabla proporciona inmediatamente un conjunto completo de endpoints en funcionamiento.

Más allá de la generación básica de CRUD, el capítulo cubre aspectos esenciales de producción, incluida la optimización del rendimiento mediante almacenamiento en caché inteligente, paginación para grandes conjuntos de datos y búsqueda de texto completo en PostgreSQL para consultas eficientes. Implementamos una caché en memoria que almacena datos a los que se accede con frecuencia y se invalida automáticamente cuando se actualizan los registros, lo que reduce drásticamente la carga de la base de datos. Los índices de base de datos en columnas consultadas comúnmente transforman los escaneos lentos de tablas en búsquedas instantáneas, mientras que los índices compuestos optimizan los filtros de múltiples columnas. Aprendimos a agregar middleware de autenticación, validación de entrada con esquemas de Zod y limitación de tasa para proteger contra el abuso, todo mientras mantenemos la naturaleza automática de la API. Esta arquitectura representa un cambio fundamental del desarrollo tradicional endpoint por endpoint a un enfoque declarativo donde el esquema de tu base de datos se convierte en la única fuente de verdad, generando automáticamente un backend con seguridad de tipos, de alto rendimiento y fácil de mantener que escala sin esfuerzo con la complejidad de tu aplicación.

En el próximo capítulo, construiremos un sistema completo de internacionalización (i18n) para aplicaciones Next.js.
