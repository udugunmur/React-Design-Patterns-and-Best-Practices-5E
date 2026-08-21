# Capítulo 1: Dominando React Server Components (RSC)

Los RSC representan un enfoque dentro de esta evolución, junto con alternativas como la reanudabilidad (*resumability*) de Qwik y la arquitectura de Islas (*Islands architecture*) de Astro. Cambian fundamentalmente la forma en que pensamos sobre el renderizado al permitir que los componentes se rendericen exclusivamente en el servidor, sin enviar su JavaScript al cliente en absoluto.

### Por qué los RSC mejoran el rendimiento

Los RSC ofrecen varias ventajas de rendimiento:

- **Cero huella de JavaScript (*Zero JavaScript footprint*):** Los Server Components se ejecutan únicamente en el servidor y no envían nada de JavaScript al cliente, reduciendo drásticamente el tamaño del bundle.
- **Acceso directo a la base de datos:** Los Server Components pueden acceder directamente a bases de datos y recursos del backend sin necesidad de capas de API intermedias.
- **Reducción de peticiones en cascada (*Reduced Waterfall requests*):** En lugar de esperar a que el cliente cargue, ejecute JavaScript y luego realice llamadas a la API, los Server Components pueden obtener datos directamente durante el renderizado.
- **Tiempo hasta la interactividad (*Time-to-Interactive*) mejorado:** Con menos JavaScript para analizar y ejecutar, las aplicaciones se vuelven interactivas más rápido.
- **Mejora progresiva (*Progressive enhancement*):** Los Server Components funcionan junto a los Client Components, permitiéndote añadir interactividad únicamente donde sea necesario.

Veamos una comparación sencilla. Aquí hay un componente de cliente tradicional que obtiene y muestra datos de usuario:

```tsx
// Traditional Client Component
'use client'
import { useState, useEffect } from 'react'
type User = {
  id: string
  name: string
  email: string
  createdAt: string
}
type Props = {
  userId: string
}
const UserProfile = ({ userId }: Props) => {
  const [user, setUser] = useState<User | null>(null)
  const [loading, setLoading] = useState<boolean>(true)
  useEffect(() => {
    const fetchUser = async () => {
      try {
        const response = await fetch(`/api/users/${userId}`)
        const data: User = await response.json()
        setUser(data)
      } catch (error) {
        console.error('Failed to fetch user', error)
      } finally {
        setLoading(false)
      }
    }
    fetchUser()
  }, [userId])
  if (loading) {
    return <div>Loading user...</div>
  }
  if (!user) {
    return <div>User not found</div>
  }
  return (
    <div className="user-profile">
      <h2>{user.name}</h2>
      <p>Email: {user.email}</p>
      <p>Member since: {new Date(user.createdAt).toLocaleDateString()}</p>
    </div>
  )
}
export default UserProfile
```

Ahora, aquí está la misma funcionalidad con un Server Component:

```tsx
// Server Component
type User = {
  id: string
  name: string
  email: string
  createdAt: string
}
type Props = {
  userId: string
}
async function UserProfile({ userId }: Props) {
  const user = await fetchUserDirectly(userId)
  if (!user) {
    return <div>User not found</div>
  }
  return (
    <div className="user-profile">
      <h2>{user.name}</h2>
      <p>Email: {user.email}</p>
      <p>Member since: {new Date(user.createdAt).toLocaleDateString()}</p>
    </div>
  )
}
// This function runs on the server only
async function fetchUserDirectly(userId: string): Promise<User | null> {
  // Replace with your actual DB logic
  return await db.users.findById(userId)
}
export default UserProfile
```

Observa cómo el Server Component es más limpio, sin hooks `useState` o `useEffect`, y puede acceder directamente a las fuentes de datos en el servidor. Los estados de carga y error se manejan de forma declarativa mediante `Suspense` y `Error Boundaries` en el nivel superior.

### Cómo los RSC reducen el tamaño del bundle de JavaScript

Uno de los beneficios más significativos de los Server Components es su impacto en el tamaño de los bundles. Cuando marcas un componente como Server Component, nada de su código se envía al cliente; únicamente el HTML resultante.

Considera un componente de visualización de datos que utiliza una biblioteca de gráficos pesada como D3.js. Si este componente se renderiza en el cliente, toda la biblioteca D3 debe descargarse, analizarse y ejecutarse en el cliente. Pero como Server Component, el gráfico se genera en el servidor y solo se envía al cliente el SVG/HTML resultante. Este enfoque funciona mejor para visualizaciones estáticas; las características interactivas como tooltips o zoom aún requieren Client Components. Ten en cuenta también que generar gráficos en cada solicitud añade sobrecarga de CPU al servidor, por lo que las estrategias de almacenamiento en caché se vuelven importantes a escala. Demostremos la diferencia de tamaño del bundle con un ejemplo práctico:

```tsx
// Client Component importing a large library
'use client'
import { useState } from 'react'
import * as d3 from 'd3' // 300KB+ of JavaScript
function DataChart({ data }) {
  const [activeIndex, setActiveIndex] = useState(null)
  // d3 rendering logic here
  return <div className="chart-container">...</div>
}
```

Como Server Component:

```tsx
// Server Component
import * as d3 from 'd3' // Never sent to the client

export function DataChart({ data }: { data: { label: string; value: number }[] }) {
  const width = 400
  const height = 200

  const x = d3
    .scaleBand()
    .domain(data.map((d) => d.label))
    .range([0, width])
    .padding(0.1)

  const y = d3
    .scaleLinear()
    .domain([0, d3.max(data, (d) => d.value) ?? 0])
    .range([height, 0])

  return (
    <svg viewBox={`0 0 ${width} ${height}`} width={width} height={height}>
      {data.map((d) => (
        <rect
          key={d.label}
          x={x(d.label)}
          y={y(d.value)}
          width={x.bandwidth()}
          height={height - y(d.value)}
        />
      ))}
    </svg>
  )
} 
```

En el segundo ejemplo, D3 se ejecuta únicamente en el servidor para los cálculos del gráfico, como escalas, marcas de graduación y diseño. Nada de ese código de D3 se envía al cliente. El navegador recibe solo el marcado SVG final renderizado por React, lo que puede reducir significativamente el tamaño del bundle y mejorar el rendimiento de carga de la página.

Hemos establecido los principios fundamentales para organizar aplicaciones con RSC: priorizar por defecto los Server Components para maximizar los beneficios de rendimiento, minimizar estratégicamente el JavaScript del lado del cliente mediante el uso cuidadoso de las directivas `use client`, mantener límites arquitectónicos claros entre la lógica de servidor y cliente, y aprovechar la estructura del sistema de archivos para crear jerarquías de componentes intuitivas y mantenibles.

### Comparación de RSC con SSR y Client Components

Para comprender mejor los RSC, comparémoslos con otros enfoques de renderizado:

| Característica | Client Components | SSR | RSC |
| :--- | :--- | :--- | :--- |
| **Ubicación del renderizado** | Navegador | Servidor, luego Navegador | Servidor |
| **JavaScript enviado al cliente** | Código completo del componente | Código completo del componente | Ninguno |
| **Interactividad** | Completa | Completa (después de la hidratación) | Ninguna (a menos que se envuelva con Client Components) |
| **Obtención de datos** | Lado del cliente (`useEffect`) | Servidor para el HTML inicial, cliente para actualizaciones | Lado del servidor |
| **Impacto en el tamaño del bundle** | El mayor | Grande (requiere hidratación) | El menor |
| **Acceso al backend** | Solo a través de APIs | A través de APIs o base de datos | Directo |
| **SEO** | Pobre (sin SSR, depende del rastreador) | Bueno | Bueno |
| **Streaming SSR** | No aplicable | Soportado | Soportado |
| **Hidratación selectiva** | No aplicable | Soportado | No necesario |

*Tabla 1.1 – Comparación entre Client Components, SSR y RSC*

La diferencia clave entre SSR y RSC es que con SSR todavía enviamos todo el JavaScript del componente al cliente para la hidratación. Con RSC, los server components permanecen en el servidor, enviando únicamente su salida al cliente.

Esto no significa que los client components sean obsoletos; son esenciales para los elementos interactivos de la interfaz de usuario. El verdadero poder proviene de combinar Server y Client Components estratégicamente para obtener lo mejor de ambos mundos.

Esta comparación integral muestra cómo los RSC ocupan una posición única en el panorama del renderizado, ofreciendo los beneficios de SEO de SSR mientras logran el tamaño de bundle más pequeño posible al mantener el código del componente completamente en el servidor. Ahora que entendemos qué hace especiales a los RSC y cómo se comparan con otros enfoques, estamos listos para explorar cómo implementarlos en la práctica.

---

## Configuración de RSC en Next.js 16+

Next.js utiliza el App Router como su arquitectura de React Server Components. En un proyecto con App Router, los archivos bajo `app/` son Server Components por defecto a menos que optes por el comportamiento del lado del cliente mediante la directiva `use client`. Esto convierte a Next.js en una de las formas más sencillas de comenzar a trabajar con RSC. Puedes crear un nuevo proyecto con:

```bash
npx create-next-app@latest my-rsc-app
cd my-rsc-app
```

Durante la configuración, se te harán varias preguntas. Para RSC, asegúrate de:

- Usar el App Router (no el Pages Router)
- Seleccionar tu solución de estilos preferida
- Elegir si deseas utilizar TypeScript

Una vez creado el proyecto, tendrás una estructura que admite Server Components por defecto. Cada componente dentro del directorio `app` es un Server Component a menos que se especifique lo contrario.

La estructura de carpetas básica se verá de la siguiente manera:

```text
my-rsc-app/
  - app/
     - layout.tsx
     - page.tsx
     - [...other routes]
  - components/
  - public/
  - next.config.ts
  - package.json
```

En Next.js 16, el directorio `app` utiliza el App Router, que cuenta con soporte integrado para RSC. Todos los componentes dentro de este directorio son Server Components de forma predeterminada, y no necesitas marcarlos explícitamente como tales.

### Cómo funciona el renderizado en RSC

Un error común es pensar que los Server Components simplemente renderizan HTML y envían ese HTML al navegador. Ese no es el panorama completo. React renderiza los Server Components en un flujo de datos especial llamado **React Server Component Payload**, también conocido como el formato **Flight**. Next.js luego utiliza ese payload junto con los Client Components para producir la interfaz de usuario final y el HTML prerenderizado cuando corresponde.

Esta distinción es importante porque RSC no es simplemente SSR con componentes en el servidor. Los Server Components son un modelo de renderizado independiente que permite al servidor resolver árboles de componentes, eliminar el código exclusivo del servidor del bundle del cliente y enviar un payload compacto que describe lo que el cliente debe renderizar.

### `use server`: Funciones de Servidor y Server Actions

Es importante no confundir RSC con Server Actions:

- **RSC** es el modelo de renderizado.
- `use server` marca **Server Functions**.
- **Server Actions** son un flujo de trabajo para formularios y un patrón de mutaciones construido sobre las funciones de servidor.

En la práctica, `use server` se utiliza para definir código que debe ejecutarse únicamente en el servidor. Puedes colocarlo en línea dentro de una función asíncrona, o en la parte superior de un archivo para marcar las funciones exportadas como exclusivas del servidor. En la documentación de Next.js, se describen como funciones del lado del servidor que se pueden invocar desde formularios o desde código de cliente a través del mecanismo de acciones del framework.

### Entendiendo `use server` y cómo funciona

La directiva `use server` es una parte fundamental del paradigma de RSC. Marca las funciones que deben ejecutarse exclusivamente en el servidor, incluso si se invocan desde componentes de cliente.

Hay dos formas de utilizar la directiva `use server`:

A nivel de función:

```tsx
'use client'
// This component is a Client Component
export default function Form() {
  async function handleSubmit(formData: FormData) {
    await submitForm(formData)
  }
  return <form action={handleSubmit}>...</form>
}
// But this function runs on the server
async function submitForm(formData) { 
  'use server'
  // Server-side validation and processing
  const data = Object.fromEntries(formData)
  await db.saveData(data)
  // ...
}
```

A nivel de archivo (afecta a todas las funciones exportadas):

```tsx
'use server'
// All functions in this file run on the server
export async function createPost(data) {
  await db.posts.create(data)
  revalidatePath('/posts')
}
export async function deletePost(id) {
  await db.posts.delete(id)
  revalidatePath('/posts')
}
```

Cuando el cliente llama a una función de servidor, Next.js serializa los argumentos, los envía al servidor, ejecuta la función y devuelve el resultado al cliente. Esto crea un puente seguro entre el código del cliente y del servidor sin necesidad de construir endpoints de API independientes.

### Escribiendo tu primer Server Component

Vamos a crear un Server Component sencillo que obtiene y muestra publicaciones de blog:

```tsx
// app/posts/page.tsx
// No 'use client' directive - Server Component by default
async function getPosts() {
  // In a real app, this would be a database query
  const res = await fetch('https://jsonplaceholder.typicode.com/posts', {
    cache: 'no-store' // Don't cache this data
  })
  if (!res.ok) {
    throw new Error('Failed to fetch posts')
  }
  return res.json()
}
export default async function PostsPage() {
  // Data fetching happens directly in the component
  const posts = await getPosts()
  return (
    <div className="posts-container">
      <h1>Recent Posts</h1>
      <div className="posts-grid">
        {posts.slice(0, 12).map(post => (
          <article key={post.id} className="post-card">
            <h2>{post.title}</h2>
            <p>{post.body.substring(0, 100)}...</p>
            <a href={`/posts/${post.id}`}>Read more</a>
          </article>
        ))}
      </div>
    </div>
  )
}
```

Este Server Component:

- Obtiene datos directamente sin `useEffect` ni `useState`.
- Utiliza la sintaxis `async/await` a nivel del componente.
- Renderiza las publicaciones en el servidor.
- Envía únicamente el HTML resultante al cliente.

### Depuración y solución de problemas en la configuración de RSC

Al trabajar con RSC, es posible que encuentres algunos problemas comunes:

**Error: No se pueden usar hooks en Server Components:**

```tsx
// This will cause an error
function BrokenServerComponent() {
  // useState is not allowed in Server Components
  const [count, setCount] = useState(0)
  return <div>{count}</div>
}
```

*Solución:* Mueve el componente al cliente o divídelo en partes de cliente y servidor:

```tsx
'use client'
// Now it's a Client Component and can use hooks
function CounterComponent() {
  const [count, setCount] = useState(0)
  return <div>{count}</div>
}
```

**Error: Los controladores de eventos no son compatibles en Server Components:**

```tsx
// This will cause an error
function BrokenButtonComponent() {
  return <button onClick={() => alert('Clicked!')}>Click me</button>
}
```

*Solución:* Mueve los elementos interactivos a Client Components:

```tsx
// Client component
'use client'
function InteractiveButton() {
  return <button onClick={() => alert('Clicked!')}>Click me</button>
}
// Server Component
export default function Page() {
  return (
    <div>
      <h1>My Page</h1>
      <InteractiveButton />
    </div>
  )
}
```

**Importar Client Components en Server Components:** ¡Esto es totalmente compatible! Puedes importar un Client Component dentro de un Server Component:

```tsx
// ServerComponent
import ClientComponent from './ClientComponent'
export default function ServerComponent() {
  return (
    <div>
      <h1>Server Component</h1>
      <ClientComponent /> {/* This works! */}
    </div>
  )
}
// ClientComponent
'use client'
export default function ClientComponent() {
  return <button onClick={() => alert('Clicked!')}>Click me</button>
}
```

**Manejo de errores al obtener datos:** Utiliza límites de error (*error boundaries*) y los archivos `error.js` de Next.js:

```tsx
// app/posts/error.tsx
'use client'
import { useEffect } from 'react'
export default function Error({ error, reset }) {
  useEffect(() => {
    // Log the error to an error reporting service
    console.error(error)
  }, [error])
  return (
    <div className="error-container">
      <h2>Something went wrong!</h2>
      <button onClick={() => reset()}>Try again</button>
    </div>
  )
}
```

La depuración de RSC requiere un cambio en la forma de rastrear problemas. Debido a que estos componentes se ejecutan en el servidor, tus sentencias `console.log` se mostrarán en tu terminal o en los registros del servidor, no en las herramientas de desarrollo del navegador.

Para solucionar problemas de manera eficaz, estructura tu enfoque de depuración en torno a estas prácticas técnicas:

- **Registro en el lado del servidor (*Server-side logging*):** Envuelve las llamadas asíncronas de obtención de datos en bloques `try-catch`. Registra los detalles del error con marcas de tiempo y contexto. Para configuraciones complejas, ve más allá de los registros de texto plano e implementa un registro estructurado en JSON.
- **Aislar fallos:** Los errores no controlados en el servidor detendrán por completo el renderizado del componente. Utiliza los error boundaries de Next.js (archivos `error.tsx`) para capturar estas excepciones a nivel de segmento y mostrar interfaces de respaldo (*fallback UIs*).
- **Herramientas de Node.js:** Trata tus componentes de React como servicios de backend. Utiliza herramientas de depuración estándar de Node.js, incluyendo sentencias `debugger` y utilidades de rastreo de peticiones, para monitorizar el rendimiento.
- **Comprobaciones preventivas:** Valida las variables de entorno en tiempo de ejecución y gestiona los casos límite de forma explícita. Prueba los escenarios de fallo localmente para asegurarte de que tus mecanismos de reserva se activen correctamente.

En la práctica, depurar RSC significa tratar tu código de frontend con el mismo rigor que los endpoints del backend. Un registro sólido te ayuda a identificar la causa raíz en la terminal, mientras que los error boundaries aíslan el fallo en el cliente, manteniendo funcional el resto de la aplicación.

### Caché en Next.js 16+

El almacenamiento en caché en Next.js 16 debe explicarse por separado de los RSC y las Server Actions. Los RSC definen cómo se renderizan los componentes en el servidor y cómo se transmite el Payload de RSC al cliente, mientras que el almacenamiento en caché controla si los datos, la salida de los componentes o los resultados de las funciones pueden reutilizarse entre solicitudes. En el App Router, el almacenamiento en caché moderno se centra en los **Cache Components** y la directiva `use cache`, que cubriremos en detalle en una sección posterior.

Por ahora, basta con saber que la obtención de datos no se almacena en caché de forma predeterminada en el modelo actual de Cache Components. Si deseas que una ruta, componente o función sea reutilizable entre solicitudes, debes marcarla explícitamente como almacenable en caché con `use cache`. Next.js puede aplicar esta directiva a nivel de archivo para que cada exportación en el archivo se almacene en caché, o en línea en la parte superior de una función o componente individual para que solo esa unidad específica se almacene en caché.

Un ejemplo sencillo se ve así:

```tsx
import { cacheTag } from 'next/cache'

type Product = {
  id: string
  name: string
  price: number
}

async function getProducts(): Promise<Product[]> {
  'use cache'
  cacheTag('products')

  const res = await fetch('https://api.example.com/products')
  if (!res.ok) {
    throw new Error('Failed to fetch products')
  }

  return res.json() as Promise<Product[]>
}

export default async function ProductsPage() {
  const products = await getProducts()

  return (
    <main>
      <h1>Products</h1>
      <ul>
        {products.map((product) => (
          <li key={product.id}>
            {product.name} — ${product.price}
          </li>
        ))}
      </ul>
    </main>
  )
}
```

En este ejemplo, `getProducts()` se marca explícitamente como almacenable en caché, y la etiqueta `products` se puede utilizar posteriormente para la invalidación. La invalidación basada en etiquetas es parte del modelo de revalidación actual en Next.js. Puedes asociar los resultados almacenados en caché con etiquetas y luego invalidar todas las entradas coincidentes cuando cambien los datos subyacentes.

Después de una mutación, normalmente actualizas el contenido almacenado en caché mediante la revalidación. Las dos APIs más comunes son `revalidatePath()` y `revalidateTag()`. `revalidatePath()` invalida las entradas de caché para una ruta específica, mientras que `revalidateTag()` invalida las entradas asociadas con una etiqueta de caché. En el App Router, `revalidatePath()` se puede invocar en Server Functions y Route Handlers, y la regeneración ocurre en la siguiente solicitud.

Por ejemplo:

```tsx
'use server'
import { revalidateTag } from 'next/cache'
type CreateProductInput = {
  name: string
  price: number
}

export async function createProduct(input: CreateProductInput): Promise<void> {
  // Validate and sanitize before persistence
  await db.product.create({ data: input })

  revalidateTag('products', 'max')
}
```

El almacenamiento en caché en Next.js 16 merece su propio modelo mental. Las explicaciones más antiguas a menudo giraban en torno a `cache: 'no-store'` y las opciones de segmento de ruta, pero ese ya no es el mejor punto de partida. En Next.js 16, el enfoque moderno se construye en torno a Cache Components, la directiva `use cache` y la revalidación explícita.

Una forma sencilla de pensarlo es la siguiente: los RSC determinan dónde se ejecuta la lógica de los componentes, `use server` define funciones exclusivas del servidor para escrituras y mutaciones, y `use cache` le indica a Next.js que una parte del trabajo puede reutilizarse entre solicitudes. Cuando los datos cambian, actualizas la salida afectada con herramientas como `revalidatePath()` o `revalidateTag()`.

Esa separación hace que el sistema sea mucho más fácil de razonar. El renderizado, las mutaciones, el almacenamiento en caché y la revalidación están relacionados, pero no son la misma preocupación. Comprender esa distinción es la clave para escribir aplicaciones de Next.js claras y predecibles.

---

## Escritura de RSC optimizados

La directiva `use server` es particularmente útil para:

- **Envíos de formularios:** Procesar datos de formularios de forma segura en el servidor.
- **Autenticación:** Gestionar operaciones de inicio/cierre de sesión.
- **Operaciones de base de datos:** Realizar operaciones CRUD (Crear, Leer, Actualizar y Eliminar).
- **Validación en el lado del servidor:** Validar datos antes de su almacenamiento.
- **Operaciones de archivos:** Subir o procesar archivos.

Implementemos el envío de un formulario utilizando Server Actions (que utilizan `use server`):

```tsx
// app/contact/page.tsx
// Server Component
export default function ContactPage() {
  return (
    <div className="contact-page">
      <h1>Contact Us</h1>
      <form action={submitContactForm}>
        <div className="form-group">
          <label htmlFor="name">Name</label>
          <input type="text" id="name" name="name" required />
        </div>
        <div className="form-group">
          <label htmlFor="email">Email</label>
          <input type="email" id="email" name="email" required />
        </div>
        <div className="form-group">
          <label htmlFor="message">Message</label>
          <textarea id="message" name="message" rows="5" required></textarea>
        </div>
        <button type="submit">Send Message</button>
      </form>
    </div>
  )
}
// Server Action
async function submitContactForm(formData) {
  'use server'
  // Extract form values
  const name = formData.get('name')
  const email = formData.get('email')
  const message = formData.get('message')
  // Validate inputs
  if (!name || !email || !message) {
    throw new Error('All fields are required')
  }
  // Process the submission (e.g., save to database, send email)
  try {
    await saveContactMessage({ name, email, message })
    // Redirect to success page
    redirect('/contact/success')
  } catch (error) {
    console.error('Failed to submit form:', error)
    throw new Error('Failed to submit. Please try again.')
  }
}
// Backend function (never exposed to client)
async function saveContactMessage(data) {
  // Save to database, send notification emails, etc.
}
```

Este enfoque es poderoso porque:

- El formulario se renderiza como un Server Component.
- El envío del formulario es manejado por una función de servidor.
- No se necesita JavaScript en el cliente para el formulario en sí.
- Puedes añadir componentes de cliente para una validación mejorada si es necesario.

### Reducción del JavaScript del lado del cliente con RSC

Una estrategia clave para optimizar el rendimiento es minimizar la cantidad de JavaScript enviado al cliente. Aquí hay enfoques prácticos:

**Mantén dependencias grandes en el lado del servidor:** Si necesitas usar librerías grandes como `date-fns`, `lodash` o procesadores de markdown, intenta usarlas en Server Components:

```tsx
// blog-post.tsx (Server Component)
import { marked } from 'marked' // A markdown processor
import { format } from 'date-fns' // Date formatting library

export default async function BlogPost({ id }) {
  const post = await fetchPost(id)
 
  // Process markdown on the server
  const htmlContent = marked(post.content)
 
  // Format dates on the server
  const formattedDate = format(new Date(post.publishedAt), 'MMMM dd, yyyy')
 
  return (
    <article>
      <h1>{post.title}</h1>
      <time>{formattedDate}</time>
     
      {/* Rendered HTML is sent to the client, not the markdown processor */}
      <div dangerouslySetInnerHTML={{ __html: htmlContent }} />
    </article>
  )
}
```

**Crea islas de interactividad:** Haz que la mayor parte de tu interfaz se renderice en el servidor, con pequeñas islas de interactividad en el cliente:

```tsx
// page.tsx (Server Component)
import LikeButton from './like-button'
export default async function ProductPage({ productId }) {
  const product = await fetchProduct(productId)
  const reviews = await fetchReviews(productId)

  return (
    <div className="product-page">
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <div className="product-price">${product.price}</div>  
      {/* Only this button is a Client Component */}
      <LikeButton productId={productId} />
      <h2>Reviews</h2>
      <div className="reviews-list">
        {reviews.map(review => (
          <div key={review.id} className="review">
            <p>{review.text}</p>
            <p>- {review.author}</p>
          </div>
        ))}
      </div>
    </div>
  )
}

// like-button.tsx
'use client'

import { useState } from 'react'
import { likeProduct } from './actions'

export default function LikeButton({ productId }) {
  const [liked, setLiked] = useState(false)
  return (
    <button
      onClick={async () => {
        await likeProduct(productId)
        setLiked(true)
      }}
      className={liked ? 'liked' : ''}
    >
      {liked ? 'Liked!' : 'Like'}
    </button>
  )
}

// actions.js
'use server'
export async function likeProduct(productId) {
  // Server-side logic to record the like
  await db.likes.create({ productId })
}
```

### Gestión del renderizado de componentes entre servidor y cliente

Al construir una aplicación con RSC, debes pensar cuidadosamente qué partes de tu aplicación deben renderizarse en el servidor versus el cliente. Aquí hay algunas pautas:

| Server Components para | Client Components para |
| :--- | :--- |
| Contenido estático | Elementos interactivos de interfaz de usuario |
| Obtención de datos (*Data fetching*) | Cualquier elemento que utilice hooks de React |
| Contenido crítico para SEO | Componentes que necesitan APIs del navegador |
| Secciones que usan dependencias pesadas | Actualizaciones en tiempo real con WebSockets |

*Tabla 1.2 – Server Components vs Client Components*

Veamos cómo estructurar una página de dashboard con este enfoque híbrido:

```tsx
// app/dashboard/page.tsx (Server Component)
import DashboardStats from './dashboard-stats' // Server Component
import RecentActivity from './recent-activity' // Server Component
import UserSettings from './user-settings' // Client Component
import LiveNotifications from './live-notifications' // Client Component

export default async function DashboardPage() {
  const user = await getCurrentUser()
  const stats = await fetchUserStats(user.id)
  const recentActivity = await fetchRecentActivity(user.id)
 
  return (
    <div className="dashboard">
      <h1>Welcome back, {user.name}</h1>
     
      {/* Server Components - static or data-intensive */}
      <DashboardStats stats={stats} />
      <RecentActivity activity={recentActivity} />
     

      {/* Client Components - interactive elements */}
      <UserSettings user={user} />
      <LiveNotifications userId={user.id} />
    </div>
  )
}
```

### Manejo de efectos secundarios y dependencias de servidor en RSC

Los Server Components pueden interactuar directamente con recursos del lado del servidor, pero debes gestionarlos adecuadamente:

**Conexiones a bases de datos:**

```tsx
// lib/db.ts - Shared database utility
import { Pool } from 'pg'

let pool

if (!global.pool) {
  global.pool = new Pool({
    connectionString: process.env.DATABASE_URL,
    ssl: process.env.NODE_ENV === 'production'
  })
}

pool = global.pool

export async function query(text, params) {
  return pool.query(text, params)
}

// Using in a Server Component
async function ProductList() {
  const { rows } = await query('SELECT * FROM products ORDER BY created_at DESC LIMIT 10')
 
  return (
    <div>
      <h2>Latest Products</h2>
      <ul>
        {rows.map(product => (
          <li key={product.id}>{product.name} - ${product.price}</li>
        ))}
      </ul>
    </div>
  )
}
```

**Consideraciones sobre el almacenamiento en caché:**

```tsx
import { cacheLife, cacheTag } from 'next/cache'

type User = {
  name: string
  bio: string
}

async function UserProfile({ id }: { id: string }) {
  'use cache'
  cacheTag(`user:${id}`)
  cacheLife('hours')

  const user = await fetch(`https://api.example.com/users/${id}`, {
    cache: 'force-cache', // Cache indefinitely until revalidated
  }).then((res) => res.json() as Promise<User>)

  return (
    <div className="profile">
      <h1>{user.name}</h1>
      <p>{user.bio}</p>
    </div>
  )
}
```

El almacenamiento en caché en Next.js ha evolucionado hacia un modelo altamente explícito y de inclusión voluntaria (*opt-in*). La obtención de datos ahora no se almacena en caché de forma predeterminada para garantizar que los usuarios siempre reciban datos actualizados.
Si estás siguiendo tutoriales más antiguos de la era de Next.js 13/14, ten en cuenta que su comportamiento de almacenamiento en caché automático ya no se aplica.
Next.js 16 lleva esta filosofía más allá al introducir Cache Components. En lugar de configurar el almacenamiento en caché por cada solicitud `fetch` individual, ahora declaras el almacenamiento en caché en torno al trabajo lógico en sí mediante la directiva `use cache`. En este enfoque moderno, marcas explícitamente las operaciones que se almacenarán en caché, las ajustas con asistentes como `cacheLife` y `cacheTag`, y refrescas los resultados obsoletos usando `revalidatePath` o `revalidateTag`. Aunque existen opciones heredadas como `unstable_cache` para compatibilidad con versiones anteriores, la directiva `use cache` es la base definitiva para construir aplicaciones modernas de Next.js:

```tsx
type Stats = {
  activeUsers: number
  visits: number
}

// Fresh on every request
async function DynamicStats() {
  const stats = await fetch('https://api.example.com/stats').then(
    (res) => res.json() as Promise<Stats>
  )

  return (
    <div>
      <h2>Current Stats</h2>
      <p>Active Users: {stats.activeUsers}</p>
      <p>Total Visits: {stats.visits}</p>
    </div>
  )
}
```

**Manejo de errores:**
Los Server Components deben gestionar los errores de forma elegante:

```tsx
async function ProductDetails({ id }) {
  try {
    const product = await fetchProduct(id)
   
    return (
      <div>
        <h1>{product.name}</h1>
        <p>{product.description}</p>
      </div>
    )
  } catch (error) {
    // This will be rendered on the server
    return (
      <div className="error-state">
        <h2>Failed to load product</h2>
        <p>Please try again later</p>
      </div>
    )
  }
}
```

Hemos cubierto los patrones esenciales para construir aplicaciones híbridas con RSC: elegir estratégicamente entre Server y Client Components, gestionar recursos del servidor como bases de datos, implementar estrategias de almacenamiento en caché adecuadas y manejar errores con elegancia. Estos patrones forman la base para construir aplicaciones React robustas y de alto rendimiento que aprovechan lo mejor del renderizado en servidor y cliente.

---

## Diferenciación entre componentes de servidor y de cliente

Es esencial comprender la diferencia entre estas dos directivas:

- `use client`: Marca un componente o archivo para que se ejecute en el cliente, habilitando la interactividad y las APIs del navegador.
- `use server`: Marca una función para que se ejecute en el servidor, incluso cuando se llama desde el código del cliente.

Así es como funcionan juntos:

```tsx
// Button.tsx
'use client'

import { useState } from 'react'
import { incrementCounter } from './actions'

export default function Button() {
  const [count, setCount] = useState(0)
 
  const handleClick = async () => {
    // Call a server function from client code
    const newCount = await incrementCounter(count)
    setCount(newCount)
  }
 
  return (
    <button onClick={handleClick}>
      Count: {count}
    </button>
  )
}

// actions.ts
'use server'

export async function incrementCounter(currentCount) {
  // This runs on the server
  console.log('Incrementing counter on server')
  // Could also update a database here
  return currentCount + 1
}
```

Las conclusiones clave son:

- Los Server Components son los predeterminados en el App Router.
- Los Client Components se marcan explícitamente con `use client`.
- Las funciones de servidor se marcan explícitamente con `use server`.
- Los Client Components pueden llamar a funciones de servidor.
- Los Server Components no pueden usar hooks ni controladores de eventos.

### Mezcla de componentes de servidor y cliente en una aplicación Next.js

Una de las fortalezas del modelo RSC es la capacidad de mezclar sin problemas Server y Client Components. Veamos un ejemplo práctico:

```tsx
// app/products/[id]/page.tsx (Server Component)
import ProductDetails from './product-details' // Server Component
import AddToCartButton from './add-to-cart-button' // Client Component
import RelatedProducts from './related-products' // Server Component
import CustomerReviews from './customer-reviews' // Server Component
import ReviewForm from './review-form' // Client Component
import { fetchProduct, fetchRelatedProducts, fetchProductReviews } from '@/lib/api'

interface PageProps {
  params: Promise<{ id: string }>
}

export default async function ProductPage({ params }: PageProps) {
  const { id: productId } = await params

  const [product, relatedProducts, reviews] = await Promise.all([
    fetchProduct(productId),
    fetchRelatedProducts(productId),
    fetchProductReviews(productId),
  ])

  return (
    <div>
      <ProductDetails product={product} />
      <AddToCartButton productId={product.id} />
      <RelatedProducts products={relatedProducts} />
      <CustomerReviews reviews={reviews} />
      <ReviewForm productId={product.id} />
    </div>
  )
}
```

Este enfoque te permite:

- Cargar datos de productos en el servidor.
- Renderizar la mayor parte de la interfaz en el servidor.
- Añadir elementos interactivos solo donde sea necesario.
- Mantener pequeño el bundle del cliente.
- Mejorar el rendimiento de la carga inicial de la página.

### Estrategias para pasar datos entre componentes de servidor y cliente

Al trabajar tanto con Server como con Client Components, necesitas estrategias para compartir datos:

**Paso de Props (Servidor → Cliente):** La forma más directa es pasar datos como props:

```tsx
// Server Component
import UserProfile from './user-profile'
import { fetchUser } from '@/lib/api'

interface User {
  id: string
  name: string
  bio: string
}

export default async function Page() {
  const user: User = await fetchUser()
  return <UserProfile user={user} />
}



// Client Component
'use client'

interface User {
  id: string
  name: string
  bio: string
}

interface UserProfileProps {
  user: User
}

export default function UserProfile({ user }: UserProfileProps) {
  return (
    <div>
      <h1>{user.name}</h1>
      <button onClick={() => alert(`Hello, ${user.name}!`)}>
        Say Hello
      </button>
    </div>
  )
}
```

**Server Actions (Cliente → Servidor):** Para enviar datos del cliente al servidor:

```tsx
// actions.ts (Server Actions)
'use server'
import { db } from '@/lib/db'

interface UserData {
  name: string
  bio: string
}

interface UpdateResult {
  id: string
  name: string
  bio: string
}

export async function updateUserProfile(
  userId: string,
  data: UserData,
): Promise<UpdateResult> {
  await db.users.update({ where: { id: userId }, data })
  return db.users.findUniqueOrThrow({ where: { id: userId } })
}

// profile-form.tsx (Client Component)
'use client'

import { useState } from 'react'
import { updateUserProfile } from './actions'

export default function ProfileForm({ user }) {
  const [name, setName] = useState(user.name)
  const [bio, setBio] = useState(user.bio)
 
  const handleSubmit = async (e) => {
    e.preventDefault()
    const updatedUser = await updateUserProfile(user.id, { name, bio })
    // Do something with updated user...
  }
 
  return (
    <form onSubmit={handleSubmit}>
      <input
        value={name}
        onChange={(e) => setName(e.target.value)}
      />
      <textarea
        value={bio}
        onChange={(e) => setBio(e.target.value)}
      />
      <button type="submit">Update Profile</button>
    </form>
  )
}
```

**Context con React Cache (Estado Compartido):** Para una gestión de estado más compleja:

```tsx
// lib/user-cache.ts
'use server'

import { cache } from 'react'
import { db } from '@/lib/db'

export interface User {
  id: string
  name: string
  bio: string
}

export const getUser = cache(async (userId: string): Promise<User> => {
  return db.users.findUniqueOrThrow({ where: { id: userId } })
})

// context/user-provider.tsx
'use client'

import { createContext, useContext, useState, useEffect } from 'react'

const UserContext = createContext(null)

export function UserProvider({ initialUser, children }) {
  const [user, setUser] = useState(initialUser)
  return (
    <UserContext.Provider value={{ user, setUser }}>
      {children}
    </UserContext.Provider>
  )
}

export function useUser() {
  return useContext(UserContext)
}

// app/user/[id]/layout.tsx (Server Component)
import type { ReactNode } from 'react'
import { UserProvider } from '@/context/user-provider'
import { getUser } from '@/lib/user-cache'

interface LayoutProps {
  params: Promise<{ id: string }>
  children: ReactNode
}

export default async function UserLayout({ params, children }: LayoutProps) {
  const { id } = await params
  const user = await getUser(id)

  return <UserProvider initialUser={user}>{children}</UserProvider>
}
```

### Consideraciones de rendimiento al combinar lógica de servidor y cliente

Al mezclar Server y Client Components, ten en cuenta estas consideraciones de rendimiento:

**Límites de los componentes:** Crea límites claros entre las partes de servidor y cliente:

```tsx
// Bad: Fine-grained mixing
function ProductPage() {
  return (
    <div>
      <ServerComponent1 />
      <ClientComponent1 />
      <ServerComponent2 />
      <ClientComponent2 />
      <ServerComponent3 />
      <ClientComponent3 />
    </div>
  )
}
// Better: Chunked by logical sections
function ProductPage() {
  return (
    <div>
      <ProductInfo /> {/* Server Component with all product details */}
      <InteractiveSection /> {/* Client Component with interactive elements */}
      <RelatedProducts /> {/* Server Component with all related products */}
    </div>
  )
}
```

**Evita el paso excesivo de props (*prop drilling*) a través de Client Components:**

```tsx
// Bad: Server data passes through client component
async function Page() {
  const user = await getUser()
 
  return (
    <ClientWrapper>
      <ServerComponent user={user} />
    </ClientWrapper>
  )
}

// Better: Keep server data flow on server
async function Page() {
  const user = await getUser()
 
  return (
    <>
      <ServerComponent user={user} />
      <ClientWrapper />
    </>
  )
}
```

**Usa límites de Suspense estratégicamente:**

```tsx
import { Suspense } from 'react'

function ProductPage() {
  return (
    <div>
      <h1>Product Details</h1>
     
      {/* Critical UI loads first */}
      <ProductBasicInfo />
     
      {/* Less critical UI can suspend */}
      <Suspense fallback={<p>Loading reviews...</p>}>
        <ProductReviews />
      </Suspense>
     
      <Suspense fallback={<p>Loading recommendations...</p>}>
        <RecommendedProducts />
      </Suspense>
    </div>
  )
}
```

Para maximizar el rendimiento al combinar Server y Client Components, concéntrate en crear límites arquitectónicos claros agrupando la funcionalidad relacionada en lugar de alternar entre componentes de servidor y cliente a lo largo de tu árbol de componentes. Evita pasar datos obtenidos en el servidor a través de Client Components como props, lo que fuerza una serialización innecesaria y puede anular los beneficios del renderizado en el servidor; en su lugar, mantén los flujos de datos del servidor completamente en el lado del servidor y permite que los Client Components manejen su propio estado e interacciones. Implementa límites de `Suspense` estratégicamente alrededor de Server Components no críticos, como reseñas, recomendaciones o datos analíticos, lo que permite que tu interfaz de usuario principal se renderice de inmediato mientras que el contenido menos esencial se carga progresivamente. Considera los efectos de cascada de red de tu arquitectura de componentes, minimiza el tamaño del bundle de JavaScript manteniendo la lógica interactiva en Client Components enfocados y aprovecha las funciones concurrentes de React para priorizar las rutas críticas de renderizado. Recuerda que el objetivo es ofrecer la carga inicial de página más rápida posible mientras se mantiene una interactividad enriquecida donde sea necesario: mide tus Core Web Vitals con regularidad y ajusta tus límites de servidor/cliente basándote en datos de rendimiento reales de los usuarios y no en suposiciones.

---

## Obtención eficiente de datos con RSC

Una de las mayores ventajas de los Server Components es la capacidad de obtener datos directamente sin `useEffect`:

```tsx
// Traditional Client Component data fetching
function ClientProductList() {
  const [products, setProducts] = useState([])
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState(null)
 
  useEffect(() => {
    async function fetchProducts() {
      try {
        const response = await fetch('/api/products')
        if (!response.ok) {
          throw new Error('Failed to fetch products')
        }
        const data = await response.json()
        setProducts(data)
      } catch (err) {
        setError(err.message)
      } finally {
        setLoading(false)
      }
    }
   
    fetchProducts()
  }, [])
 
  if (loading) return <div>Loading products...</div>
  if (error) return <div>Error: {error}</div>
 
  return (
    <div className="products-grid">
      {products.map(product => (
        <div key={product.id} className="product-card">
          <h3>{product.name}</h3>
          <p>${product.price}</p>
        </div>
      ))}
    </div>
  )
}

// Server Component data fetching
async function ServerProductList() {
  // Direct data fetching - no useEffect, no loading state
  const products = await fetchProducts()
 
  return (
    <div className="products-grid">
      {products.map(product => (
        <div key={product.id} className="product-card">
          <h3>{product.name}</h3>
          <p>${product.price}</p>
        </div>
      ))}
    </div>
  )
}

// Server-side function (not exposed to client)
async function fetchProducts() {
  // This could be a direct database query
  // For example purposes, we're still using fetch
  const response = await fetch('https://api.example.com/products', {
    // Next.js specific options
    cache: 'no-store'
  })
 
  if (!response.ok) {
    throw new Error('Failed to fetch products')
  }
 
  return response.json()
}
```

Beneficios de la obtención de datos en Server Components:

- Sin estados de carga que gestionar.
- Sin arreglos de dependencias de `useEffect` que mantener.
- Sin riesgo de cascadas (*waterfalls* de múltiples llamadas secuenciales a APIs).
- Acceso directo a bases de datos si es necesario.
- Mejor manejo de errores con `try/catch`.
- Los datos están disponibles antes de que el componente se renderice.

También puedes obtener datos en paralelo para un mejor rendimiento:

```tsx
async function DashboardPage() {
  // Start all fetch requests in parallel
  const productsPromise = fetchProducts()
  const usersPromise = fetchUsers()
  const statsPromise = fetchStats()
  // Wait for all to complete
  const [products, users, stats] = await Promise.all([
    productsPromise,
    usersPromise,
    statsPromise
  ])
 
  return (
    <div>
      <ProductsSection products={products} />
      <UsersSection users={users} />
      <StatsSection stats={stats} />
    </div>
  )
}
```

Hemos dominado los patrones esenciales de obtención de datos para Server Components: obtener datos directamente sin hooks de cliente como `useEffect`, optimizar el rendimiento a través de solicitudes paralelas con `Promise.all` y `Promise.allSettled`, habilitar cargas de página progresivas con React Suspense para streaming, y construir aplicaciones resilientes a través de un manejo integral de errores utilizando bloques `try/catch` y archivos `error.js`.

### Combinación de RSC con streaming y Suspense para cargas de página más rápidas

Los React Server Components funcionan perfectamente con la función Suspense de React para permitir el streaming. Esto permite al servidor enviar partes de la página a medida que están listas, en lugar de esperar a que se completen todos los datos:

```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react'
import Loading from './loading'

export default function DashboardPage() {
  return (
    <div className="dashboard">
      <h1>Dashboard</h1>
     
      {/* Critical UI renders first */}
      <UserWelcome />
     
      {/* These components can load in parallel */}
      <div className="dashboard-grid">
        <Suspense fallback={<Loading type="stats" />}>
          <Stats />
        </Suspense>
       
        <Suspense fallback={<Loading type="recent-orders" />}>
          <RecentOrders />
        </Suspense>
       
        <Suspense fallback={<Loading type="notifications" />}>
          <Notifications />
        </Suspense>
      </div>
    </div>
  )
}

// Each component can fetch its own data
async function Stats() {
  // This fetch might take a while
  const stats = await fetchStats()
 
  return (
    <div className="stats-card">
      <h2>Statistics</h2>
      <p>Revenue: ${stats.revenue}</p>
      <p>Orders: {stats.orders}</p>
      <p>Customers: {stats.customers}</p>
    </div>
  )
}

async function RecentOrders() {
  // This might be a separate API call
  const orders = await fetchRecentOrders()
 
  return (
    <div className="orders-card">
      <h2>Recent Orders</h2>
      <ul>
        {orders.map(order => (
          <li key={order.id}>
            Order #{order.id} - ${order.total}
          </li>
        ))}
      </ul>
    </div>
  )
}

// loading.tsx component
export default function Loading({ type }) {
  return (
    <div className={`loading-skeleton ${type}`}>
      <div className="loading-header"></div>
      <div className="loading-body"></div>
    </div>
  )
}
```

Este enfoque tiene varios beneficios:

- El usuario ve el contenido progresivamente a medida que está disponible.
- La interfaz de usuario crítica aparece primero.
- Las consultas de datos más lentas no bloquean toda la página.
- Cada componente puede cargarse de forma independiente.
- La página se vuelve interactiva en etapas.

El streaming funciona mediante:

- Envío del HTML para el layout principal y cualquier contenido disponible al instante.
- Reemplazo de los fallbacks de Suspense con contenido real a medida que los datos están disponibles.
- Envío de JavaScript únicamente para los Client Components.

### Estrategias de almacenamiento en caché con RSC para evitar peticiones innecesarias

Un almacenamiento en caché eficaz es fundamental para el rendimiento. En Next.js, dispones de varias opciones de almacenamiento en caché:

**Memoización de peticiones (*Request memoization*):** React memoiza automáticamente las solicitudes `fetch` con la misma URL y opciones dentro de un mismo árbol de React Server Components:

```tsx
async function ProductInfo({ id }) {
  // This fetch is automatically memoized
  const product = await fetch(`https://api.example.com/products/${id}`)
    .then(res => res.json())
 
  return <div>{product.name}</div>
}

async function ProductPrice({ id }) {
  // This won't cause a second fetch - it reuses the cached result
  const product = await fetch(`https://api.example.com/products/${id}`)
    .then(res => res.json())
 
  return <div>${product.price}</div>
}

export default function ProductPage({ id }) {
  return (
    <>
      <ProductInfo id={id} />
      <ProductPrice id={id} /> {/* Uses cached data */}
    </>
  )
}
```

**Caché de datos de Next.js (*Next.js data cache*):** Next.js proporciona opciones de almacenamiento en caché para `fetch`:

```tsx
// Cache data until manually invalidated (default)
async function getProduct(id) {
  const res = await fetch(`https://api.example.com/products/${id}`)
  return res.json()
}
// Cache data for 60 seconds
async function getVolatileData() {
  const res = await fetch('https://api.example.com/stats', {
    next: { revalidate: 60 } // Seconds
  })
  return res.json()
}
// Never cache this data
async function getLiveData() {
  const res = await fetch('https://api.example.com/live-stats', {
    cache: 'no-store'
  })
  return res.json()
}
```

**React Cache para funciones personalizadas:** Para el acceso a datos que no sea mediante `fetch`, utiliza la función `cache` de React:

```tsx
// lib/utils/database.ts
import { cache } from 'react'
import { db } from '@/lib/db'

export const getUser = cache(async (userId) => {
  // This database query will be cached
  return await db.users.findUnique({
    where: { id: userId }
  })
})
export const getProducts = cache(async ({ category, limit = 10 }) => {
  return await db.products.findMany({
    where: { category },
    take: limit,
    orderBy: { createdAt: 'desc' }
  })
})
```

**Estrategias de revalidación:** Next.js ofrece varias formas de invalidar la caché:

```tsx
// Time-based revalidation
fetch('https://api.example.com/products', {
  next: { revalidate: 3600 } // Revalidate every hour
})

// On-demand revalidation in Server Actions
'use server'

import { revalidatePath, revalidateTag } from 'next/cache'

export async function createProduct(data) {
  // Create the product in the database
  await db.products.create({ data })
 
  // Invalidate all product pages
  revalidatePath('/products')
 
  // Or use tags for more granular control
  revalidateTag('products', 'max')
}

// Using tags with fetch
fetch('https://api.example.com/products', {
  next: { tags: ['products'] }
})
```

**Combinación de diferentes estrategias de caché:** Para un rendimiento óptimo, combina estrategias según las características de los datos:

```tsx
async function ShopPage() {
  // Static data - long cache
  const categories = await getCategories()
 
  // Semi-dynamic - revalidate periodically
  const featuredProducts = await getFeaturedProducts()
 
  // Highly dynamic - no cache
  const flashSales = await getActiveFlashSales()
 
  return (
    <div>
      <CategoriesList categories={categories} />
      <FeaturedProducts products={featuredProducts} />
      <FlashSales sales={flashSales} />
    </div>
  )
}

async function getCategories() {
  // Categories rarely change - cache until manually invalidated
  return await fetch('https://api.example.com/categories').then(res => res.json())
}

async function getFeaturedProducts() {
  // Featured products change daily - revalidate every hour
  return await fetch('https://api.example.com/featured', {
    next: { revalidate: 3600 }
  }).then(res => res.json())
}

async function getActiveFlashSales() {
  // Flash sales change constantly - never cache
  return await fetch('https://api.example.com/flash-sales', {
    cache: 'no-store'
  }).then(res => res.json())
}
```

Hemos explorado el conjunto completo de herramientas de almacenamiento en caché para aplicaciones con RSC: memoización automática de solicitudes para deduplicación, caché de datos de Next.js con opciones de revalidación flexibles, la función `cache` de React para fuentes de datos personalizadas e invalidación estratégica de caché mediante `revalidatePath` y `revalidateTag`. El ejemplo final demuestra cómo combinar cuidadosamente estas estrategias según la volatilidad de los datos: usando almacenamiento en caché a largo plazo para contenido estático, revalidación basada en tiempo para datos semidinámicos y sin almacenamiento en caché para información en tiempo real.

---

## ¿Qué hay de nuevo en React 19.2?

El equipo de React ha estado trabajando intensamente. React 19.2 llega con una colección de características que se sienten menos como cambios revolucionarios y más como el framework finalmente poniéndose al día con lo que los desarrolladores han estado pidiendo todo el tiempo. Hay una cierta elegancia en este lanzamiento; no se trata de reinventar React, sino de refinar la experiencia del desarrollador y abordar los desafíos de rendimiento del mundo real que han afectado a las aplicaciones de producción durante años.

React y React Native están en transición hacia la **React Foundation**, una organización independiente bajo la Linux Foundation con una junta directiva que incluye a Amazon, Meta, Microsoft, Vercel y otras empresas importantes. Si bien Meta mantiene su compromiso con una asociación de cinco años que incluye más de 3 millones de dólares en financiación, este cambio transforma a React de un proyecto liderado por Meta a un ecosistema neutral, impulsado por la comunidad y con gobernanza independiente.

### El componente `<Activity />`

A primera vista, `<Activity />` parece pertenecer a la misma familia que los spinners, skeletons y otros indicadores de carga. No es así. Su propósito es más sutil y más poderoso. `<Activity />` sirve para preservar la interfaz de usuario en segundo plano, no para indicar que hay un trabajo en progreso.

Cuando una actividad está oculta, React elimina sus hijos de la vista, limpia sus Effects y mantiene su estado en memoria. Eso significa que la interfaz puede desaparecer sin ser descartada. Cuando vuelve a ser visible, regresa en el mismo estado que tenía antes. El usuario no tiene que empezar de nuevo. La interfaz de usuario simplemente regresa.

Esto hace que `<Activity />` sea especialmente útil para partes de la interfaz que entran y salen de la vista pero que no deberían perder su lugar: una barra lateral llena de filtros, un panel de pestañas, un panel lateral de detalles o una sección de configuración que el usuario puede volver a visitar un momento después. En lugar de desmontar ese subárbol y reconstruirlo desde cero, puedes ocultarlo y restaurarlo cuando sea necesario:

```tsx
import { Activity, useState } from 'react'

export function Dashboard() {
  const [showSidebar, setShowSidebar] = useState(true)

  return (
    <div className="layout">
      <button onClick={() => setShowSidebar((v) => !v)}>
        Toggle sidebar
      </button>

      <Activity mode={showSidebar ? 'visible' : 'hidden'}>
        <aside className="sidebar">
          <h2>Filters</h2>
          <label>
            Search
            <input type="text" />
          </label>
        </aside>
      </Activity>

      <main>
        <h1>Dashboard</h1>
        <p>Main content goes here.</p>
      </main>
    </div>
  )
}
```

La propiedad importante aquí es `mode`, que controla si el contenido envuelto es visible u oculto. Esa es la idea central detrás del componente. No hay una prop `pending`, ni existe un modo `type="refresh"`. `<Activity />` no es una primitiva de carga y no decide cómo representar el progreso.

Esa distinción importa porque es fácil confundir `<Activity />` con `useTransition`. Los dos pueden aparecer en conversaciones similares, pero resuelven problemas diferentes. `useTransition` te permite marcar las actualizaciones como no urgentes y te brinda una forma de mostrar comentarios pendientes mientras React está trabajando. `<Activity />`, por el contrario, gestiona la visibilidad mientras preserva el estado. Uno se trata de programar actualizaciones. El otro se trata de mantener viva la interfaz de usuario cuando está temporalmente fuera de la vista.

Visto de esta manera, `<Activity />` se parece menos a un spinner y más a un área de bastidores (*backstage*). La interfaz se aparta de la vista, pero no deja de existir. Cuando regresa, continúa justo donde lo dejó. Ese pequeño cambio en el modelo mental es la clave para entender por qué este componente es importante.

### `useEffectEvent`: el hook que estábamos esperando

Todo desarrollador de React ha escrito este código: un efecto que depende de una función, que depende de props, lo que hace que el efecto se vuelva a ejecutar con demasiada frecuencia. Las soluciones alternativas se han convertido en patrones de "culto a la carga" (*cargo-cult*): cadenas de `useCallback` que hacen que tu componente parezca un juego de Jenga de dependencias, el último patrón de ref con `useRef`, o ese comentario del linter que todos copiamos y pegamos para deshabilitar la advertencia de `exhaustive-deps`.

`useEffectEvent` formaliza este patrón separando lo que cambia de lo que reacciona a los cambios. Es una forma de acceder a las props y al estado más recientes dentro de un efecto sin hacer que el efecto se vuelva a ejecutar cuando esos valores cambian:

```tsx
import { useEffect, useEffectEvent, useState } from 'react'

interface AnalyticsTrackerProps {
  userId: string
  pageName: string
}

export function AnalyticsTracker({ userId, pageName }: AnalyticsTrackerProps) {
  const [sessionDuration, setSessionDuration] = useState(0)

  // This function always sees the latest userId and pageName
  // but doesn't cause effects to re-run when they change
  const logEvent = useEffectEvent((eventName: string, data: object) => {
    analytics.track(eventName, {
      userId,
      pageName,
      timestamp: Date.now(),
      ...data,
    })
  })

  // This effect only runs once on mount
  useEffect(() => {
    const startTime = Date.now()
    logEvent('page_view', { startTime })

    const interval = setInterval(() => {
      const duration = Math.floor((Date.now() - startTime) / 1000)
      setSessionDuration(duration)
      logEvent('heartbeat', { duration })
    }, 30000)

    return () => {
      const endTime = Date.now()
      const totalDuration = Math.floor((endTime - startTime) / 1000)
      logEvent('page_exit', { totalDuration })
      clearInterval(interval)
    }
  }, []) // Empty deps array - this really only runs once now

  return (
    <div className="fixed bottom-4 right-4 bg-slate-800 text-white px-3 py-2
                    rounded-lg text-sm opacity-50">
      Session: {sessionDuration}s
    </div>
  )
}
```

Observa qué tan limpio queda ahora el arreglo de dependencias. El efecto se ejecuta una sola vez, pero `logEvent` siempre tiene acceso a los valores actuales de `userId` y `pageName`. Ya no tendrás que elegir entre corrección y rendimiento. No más cadenas de `useCallback` que te hagan cuestionar tus elecciones profesionales.

### Prerenderizado parcial (*Partial Pre-rendering - PPR*): lo mejor de ambos mundos

El debate entre estático y dinámico ha definido el desarrollo web durante una década. Los generadores de sitios estáticos ofrecen velocidad pero sacrifican interactividad. El renderizado del lado del servidor ofrece contenido dinámico pero a costa del tiempo hasta el primer byte (*TTFB*). El prerenderizado parcial (PPR) dice: ¿por qué no ambos?

La idea es elegante: prerenderizar la estructura estática (*shell*) de tu página en tiempo de compilación, pero dejar huecos donde el contenido dinámico se transmitirá por streaming en el momento de la solicitud. Tus usuarios ven algo al instante, y las partes personalizadas se van completando a medida que llegan:

```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react'

// This component will be pre-rendered at build time
function DashboardShell() {
  return (
    <div className="min-h-screen bg-slate-50">
      <header className="bg-white border-b border-slate-200 px-8 py-4">
        <h1 className="text-2xl font-bold text-slate-900">Dashboard</h1>
      </header>

      <main className="p-8">
        <div className="max-w-7xl mx-auto">
          <div className="grid grid-cols-3 gap-6 mb-8">
            {/* Static promotional cards pre-rendered at build time */}
            <div className="bg-gradient-to-br from-blue-500 to-blue-600
                          p-6 rounded-xl text-white">
              <h3 className="text-lg font-semibold mb-2">New Feature</h3>
              <p className="text-blue-100">
                Check out our latest updates and improvements.
              </p>
            </div>

            {/* Dynamic content will be streamed in */}
            <Suspense fallback={<MetricCardSkeleton />}>
              <UserMetrics />
            </Suspense>

            <Suspense fallback={<MetricCardSkeleton />}>
              <RecentActivity />
            </Suspense>
          </div>

          <Suspense fallback={<FeedSkeleton />}>
            <PersonalizedFeed />
          </Suspense>
        </div>
      </main>
    </div>
  )
}

// This component fetches data at request time
async function UserMetrics() {
  const metrics = await fetchUserMetrics() // Server-side data fetching

  return (
    <div className="bg-white p-6 rounded-xl shadow-sm">
      <h3 className="text-sm font-medium text-slate-600 mb-2">
        Your Progress
      </h3>
      <p className="text-3xl font-bold text-slate-900">{metrics.score}</p>
      <p className="text-sm text-green-600 mt-1">
        +{metrics.improvement}% this week
      </p>
    </div>
  )
}

async function RecentActivity() {
  const activities = await fetchRecentActivity()

  return (
    <div className="bg-white p-6 rounded-xl shadow-sm">
      <h3 className="text-sm font-medium text-slate-600 mb-3">
        Recent Activity
      </h3>
      <div className="space-y-2">
        {activities.slice(0, 3).map((activity) => (
          <div key={activity.id} className="text-sm text-slate-700">
            {activity.description}
          </div>
        ))}
      </div>
    </div>
  )
}

function MetricCardSkeleton() {
  return (
    <div className="bg-white p-6 rounded-xl shadow-sm animate-pulse">
      <div className="h-4 bg-slate-200 rounded w-1/2 mb-3"></div>
      <div className="h-8 bg-slate-200 rounded w-3/4"></div>
    </div>
  )
}

export default DashboardShell
```

Con PPR habilitado en tu configuración de Next.js, esta página se prerenderizará parcialmente. La estructura, navegación, diseño y contenido estático se generan en tiempo de compilación y se sirven instantáneamente desde la CDN. Los componentes dinámicos envueltos en límites de `Suspense` se renderizan bajo demanda cuando el usuario solicita la página, y su contenido se transmite por streaming a medida que está disponible.

La experiencia del usuario es una respuesta inmediata seguida de una mejora progresiva. No más pantallas en blanco mientras se esperan consultas a bases de datos. No más elecciones forzadas entre rendimiento y personalización.

### Agrupación en lotes de límites de Suspense para SSR (*Batching Suspense boundaries*)

El renderizado del lado del servidor siempre ha tenido un problema incómodo: ¿qué sucede cuando tienes múltiples límites de Suspense en una página? ¿Esperas por todos ellos? ¿Los transmites uno por uno? ¿Los envías a medida que se completan?

React 19.2 introduce la agrupación inteligente en lotes (*intelligent batching*). Los límites de Suspense relacionados, aquellos que aparecerían en la pantalla al mismo tiempo, se agrupan y se envían juntos. Esto significa menos viajes de ida y vuelta (*round-trips*), menos saltos de diseño (*layout shifting*) y una experiencia de carga percibida mucho más fluida:

```tsx
// app/article/[slug]/page.tsx
import { Suspense } from 'react'

async function ArticlePage({ params }: { params: { slug: string } }) {
  return (
    <article className="max-w-4xl mx-auto px-8 py-12">
      {/* These boundaries are visually grouped, so React batches them */}
      <Suspense fallback={<HeaderSkeleton />}>
        <ArticleHeader slug={params.slug} />
      </Suspense>

      <div className="mt-8 prose prose-slate max-w-none">
        <Suspense fallback={<ContentSkeleton />}>
          <ArticleContent slug={params.slug} />
        </Suspense>
      </div>

      <aside className="mt-12 border-t border-slate-200 pt-8">
        {/* This is separate, so it might flush independently */}
        <Suspense fallback={<CommentsSkeleton />}>
          <CommentsSection slug={params.slug} />
        </Suspense>
      </aside>
    </article>
  )
}
```

React analiza tu árbol de componentes y toma decisiones inteligentes sobre qué agrupar. La cabecera y el contenido llegan juntos porque forman parte de la experiencia de lectura principal. Los comentarios pueden llegar más tarde; son útiles pero no críticos para el renderizado inicial.

### SSR: soporte de Web Streams para Node

React 18 introdujo dos APIs de streaming para el renderizado del lado del servidor, las cuales continúan siendo la opción recomendada en React 19, cada una optimizada para diferentes entornos:

- `renderToPipeableStream`: Úsalo en entornos Node.js. Utiliza la API de streams nativa de Node, ofrece un mejor rendimiento y admite compresión integrada (gzip, brotli). Sigue siendo la opción recomendada para servidores tradicionales de Node.js:

```tsx
// server.ts (Node.js)
import { renderToPipeableStream } from 'react-dom/server'
import App from './App'

export function handler(req: Request, res: Response) {
  const { pipe } = renderToPipeableStream(<App />, {
    bootstrapScripts: ['/client.js'],
    onShellReady() {
      res.setHeader('Content-Type', 'text/html')
      pipe(res)
    },
    onError(error) {
      console.error('SSR Error:', error)
    },
  })
}
```

- `renderToReadableStream`: Úsalo en entornos de ejecución edge (Cloudflare Workers, Deno, Vercel Edge Functions) que admiten Web Streams pero no las APIs de Node.js:

```tsx
// server.ts (Edge runtime)
import { renderToReadableStream } from 'react-dom/server'
import App from './App'
export async function handler(request: Request) {
  const stream = await renderToReadableStream(<App />, {
    bootstrapScripts: ['/client.js'],
    onError(error) {
      console.error('SSR Error:', error)
    },
  })

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/html'
    }
  })
}
```

La API Web Streams proporciona portabilidad a través de entornos edge, pero no cambies de `renderToPipeableStream` si estás desplegando en Node.js; perderías beneficios de rendimiento y compatibilidad con compresión sin ninguna ganancia. Elige la API que coincida con tu objetivo de despliegue.

### `eslint-plugin-react-hooks` v6: análisis de código más inteligente

Las Reglas de los Hooks siempre han parecido algo mágicas: patrones que React requiere pero que JavaScript por sí solo no impone. El plugin de ESLint ha hecho un trabajo encomiable detectando infracciones, pero también ha sido fuente de frustración debido a falsos positivos y advertencias excesivamente estrictas.

La versión 6 comprende mejor a React. Reconoce `useEffectEvent` y no se queja de sus dependencias. Entiende `cacheSignal` y no te obliga a agregarlo a los arreglos de dependencias. Lo más importante es que ha mejorado en la comprensión de tu intención:

```tsx
import { useEffect, useEffectEvent, useState } from 'react'

interface TimerProps {
  onTick: (count: number) => void
  interval: number
}

export function Timer({ onTick, interval }: TimerProps) {
  const [count, setCount] = useState(0)

  // v6 understands that this doesn't need to be in the deps array
  const handleTick = useEffectEvent((currentCount: number) => {
    onTick(currentCount)
  })

  useEffect(() => {
    const timer = setInterval(() => {
      setCount((c) => {
        const newCount = c + 1
        handleTick(newCount)
        return newCount
      })
    }, interval)

    return () => clearInterval(timer)
  }, [interval]) // Only interval needs to be here - no lint errors!

  return (
    <div className="text-center p-8">
      <div className="text-6xl font-bold text-slate-900 mb-4">{count}</div>
      <div className="text-slate-600">seconds elapsed</div>
    </div>
  )
}
```

El linter finalmente trabaja contigo en lugar de en tu contra. Detecta problemas reales mientras se mantiene al margen cuando estás utilizando los nuevos patrones correctamente.

### Actualización del prefijo predeterminado de `useId`

React 19 cambia el prefijo predeterminado para `useId` de dos puntos (`:r1:`) a un formato compatible con CSS. Esto no se debe a la resistencia contra colisiones, sino a la compatibilidad con la View Transitions API.

La View Transitions API utiliza selectores CSS para hacer coincidir elementos durante las transiciones de página. El antiguo formato de `useId` incluía dos puntos, que son caracteres especiales en CSS (utilizados para pseudoclases como `:hover`). Un ID como `:r1:` requeriría caracteres de escape en los selectores CSS, y la View Transitions API no podía hacer coincidir estos elementos sin trucos adicionales:

```tsx
import { useId } from 'react'

export function FormField({ label, type = 'text' }: FormFieldProps) {
  // React 18: ':r1:' (colon prefix, problematic for CSS selectors)
  // React 19: 'r1' or similar CSS-safe format
  const id = useId()

  return (
    <div>
      <label htmlFor={id}>
        {label}
      </label>
      <input id={id} type={type} />
    </div>
  )
}
```

Sigues utilizando `useId` exactamente como antes, ya que la API en sí no ha cambiado. La diferencia radica en que los IDs generados ahora funcionan a la perfección con funciones basadas en CSS como View Transitions, `document.querySelector` y los selectores de atributos CSS. Si tienes pruebas o instantáneas (*snapshots*) que verifiquen formatos de ID específicos, deberán actualizarse, pero el código de tu aplicación permanece sin cambios.

---

## Resumen

React 19.2 representa una evolución reflexiva más que una revolución: una colección de mejoras que hacen que el framework sea aún mejor en lo que ya hace bien. El componente `<Activity />` te permite mantener partes de la interfaz de usuario montadas pero ocultas mientras se preserva su estado, `useEffectEvent` elimina categorías enteras de errores relacionados con dependencias, y `cacheSignal` proporciona un `AbortSignal` vinculado a la vida útil de las entradas de `cache()` en RSC, facilitando la cancelación de tareas que ya no son necesarias. PPR ayuda a cerrar la brecha entre lo estático y lo dinámico, mientras que mejoras más pequeñas como una mejor agrupación por lotes de Suspense, soporte de Web Streams en Node y un linting más inteligente eliminan la fricción del desarrollo diario. Este es un React que madura, volviéndose más refinado y de alto rendimiento sin abandonar su filosofía central.

Los RSC complementan esta maduración al cambiar fundamentalmente dónde se produce el renderizado. Al ejecutarse exclusivamente en el servidor y enviar un Payload de RSC serializado (en lugar de un paquete de JavaScript completo del lado del cliente) al cliente, los RSC reducen drásticamente el tamaño de los bundles de JavaScript y simplifican la obtención de datos con sintaxis directa `async/await`. Next.js ha simplificado su adopción a través del App Router, donde los componentes son Server Components por defecto y el uso reflexivo de la directiva `use client`, junto con `use server` para Server Actions, te permite diseñar límites de componentes que equilibran el renderizado del servidor y del cliente. Combinados con el streaming a través de Suspense y las estrategias de almacenamiento en caché adecuadas, los Server Components te permiten crear aplicaciones potentes y de alto rendimiento, manteniendo los cálculos pesados en el servidor mientras se mantiene una interactividad enriquecida donde realmente importa.

Al comprender estos patrones y aprovechar tanto las mejoras de React 19.2 como las ventajas arquitectónicas de los Server Components, estarás preparado para crear aplicaciones React modernas que ofrezcan experiencias de usuario óptimas. En el próximo capítulo, exploraremos cómo combinar RSC con Server Actions para crear aplicaciones aún más complejas e interactivas manteniendo los beneficios de rendimiento que hemos discutido aquí.
