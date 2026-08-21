# Capítulo 8: React Router 7

El panorama del enrutamiento en React ha experimentado una transformación notable con React Router 7, marcando una de las actualizaciones más significativas en la historia del framework. La integración de Remix en React Router ha incorporado el renderizado en el servidor (*server-side rendering* o SSR), patrones avanzados de carga de datos y optimizaciones de rendimiento mejoradas directamente en la librería central de enrutamiento. Esta evolución representa no solo una actualización de versión, sino un cambio fundamental en cómo concebimos el enrutamiento en las aplicaciones modernas de React.

React Router siempre ha sido el estándar de facto para el enrutamiento del lado del cliente en aplicaciones React, pero el desarrollo web exigía más. Las aplicaciones necesitaban mejor SEO, cargas iniciales de página más rápidas y un manejo de datos más sofisticado. El equipo de Remix ya había resuelto muchos de estos desafíos con su framework full-stack de React, y la decisión de fusionar las capacidades de Remix en React Router 7 ha creado una solución potente y unificada.

En este capítulo, exploraremos cómo React Router 7 cierra la brecha entre el renderizado del lado del cliente y del lado del servidor, permitiéndote construir aplicaciones que son a la vez eficientes y escalables. Profundizaremos en los nuevos patrones de carga de datos, los mecanismos de manejo de formularios y las capacidades de streaming que hacen de React Router 7 una solución integral para el desarrollo web moderno.

En este capítulo, cubriremos los siguientes temas:

- Instalación y configuración de React Router
- Definición de rutas y diseños anidados (*nested layouts*)
- Agregar parámetros a las rutas y manejo de datos dinámicos
- React Router v7 y la integración de Remix

---

## Requisitos técnicos

Para completar este capítulo, necesitarás lo siguiente:

- Node.js 24
- Cursor o VSCode

---

## Instalación y configuración de React Router

La configuración de React Router 7 comienza entendiendo que ya no es solo una librería de enrutamiento del lado del cliente. El proceso de instalación ahora incluye consideraciones tanto para entornos de cliente como de servidor, lo que refleja las capacidades ampliadas del framework:

```bash
npm install react-router @react-router/node @react-router/serve
```

React Router 7 incluye sus propias definiciones de TypeScript, por lo que no necesitas instalar `@types/react-router`. De hecho, el paquete separado `@types/react-router` apunta a versiones anteriores de React Router y no debe usarse para una configuración de React Router 7.

La idea más importante en la configuración inicial es que React Router 7 admite múltiples modos de enrutamiento. Aún puedes usarlo como un enrutador SPA exclusivo para el cliente con rutas declarativas o data routers, pero también admite enrutamiento estilo framework con renderizado en el servidor, loaders, actions y otras características full-stack. SSR no es obligatorio a menos que estés utilizando el modo framework de React Router.

Al leer un data router o una configuración estilo framework, concéntrate en tres responsabilidades: el `element` define la UI, el `loader` obtiene datos antes del renderizado y el `action` maneja envíos de formularios u otras mutaciones. Una vez que estos roles están claros, la configuración general del enrutador se vuelve mucho más fácil de entender.

Una vez instalado, puedes configurar tu enrutador definiendo rutas con sus componentes, loaders y actions asociados:

```tsx
// app/router.tsx
import {
  createBrowserRouter,
  type ActionFunctionArgs,
  type LoaderFunctionArgs,
  type RouteObject,
} from 'react-router'
import { RootLayout } from './components/RootLayout'
import { HomePage } from './pages/HomePage'
import { ProductPage } from './pages/ProductPage'

const routes = [
  {
    path: '/',
    element: <RootLayout />,
    children: [
      {
        index: true,
        element: <HomePage />,
        loader: homeLoader,
      },
      {
        path: 'products/:id',
        element: <ProductPage />,
        loader: productLoader,
        action: productAction,
      },
    ],
  },
] satisfies RouteObject[]

export const router = createBrowserRouter(routes)

async function homeLoader() {
  return { message: 'Welcome to React Router 7!' }
}

async function productLoader({ params }: LoaderFunctionArgs) {
  if (!params.id) {
    throw new Response('Product ID is required', { status: 400 })
  }

  const response = await fetch(`/api/products/${params.id}`)

  if (!response.ok) {
    throw new Response('Product not found', { status: response.status })
  }

  return response.json()
}

async function productAction({ request }: ActionFunctionArgs) {
  const formData = await request.formData()

  return {
    success: true,
    data: Object.fromEntries(formData),
  }
}
```

El proceso de configuración ahora incluye el establecimiento de los puntos de entrada de la aplicación para entornos de cliente y servidor. Esta configuración de modo dual es lo que permite a React Router 7 proporcionar capacidades de renderizado en el servidor mientras mantiene la experiencia de navegación fluida del lado del cliente que los usuarios esperan.

```tsx
// app/entry.client.tsx
import { StrictMode } from 'react'
import { hydrateRoot } from 'react-dom/client'
import { RouterProvider } from 'react-router/dom'
import { router } from './router'

hydrateRoot(
  document,
  <StrictMode>
    <RouterProvider router={router} />
  </StrictMode>
)
```

El archivo de entrada del cliente es intencionalmente pequeño porque su responsabilidad es acotada: hidrata la aplicación después de que el servidor ya ha renderizado el HTML inicial. En otras palabras, este archivo no define el comportamiento de enrutamiento ni la lógica de obtención de datos; simplemente activa la aplicación interactiva en el navegador. Esta separación es importante porque React Router 7 divide las responsabilidades de manera más explícita entre el renderizado del servidor y la interactividad del cliente.

Mientras que el punto de entrada del cliente maneja la hidratación y la interactividad del navegador, la arquitectura de React Router 7 requiere una configuración complementaria del lado del servidor para completar el panorama full-stack. La configuración del servidor trabaja en conjunto con el cliente, generando el HTML inicial que luego el cliente hidrata, creando una experiencia transparente donde los usuarios ven el contenido de inmediato mientras JavaScript se carga en segundo plano.

El punto de entrada del servidor es más fácil de entender si te concentras en su propósito más que en cada detalle de implementación: su trabajo es generar el HTML inicial en el servidor y transmitirlo en streaming al navegador lo más rápido posible.

```tsx
// app/entry.server.tsx
import { PassThrough } from 'stream'
import type { AppLoadContext, EntryContext } from 'react-router'
import { createReadableStreamFromReadable } from '@react-router/node'
import { renderToPipeableStream } from 'react-dom/server'
import { RouterProvider, createStaticRouter } from 'react-router'

export default function handleRequest(
  request: Request,
  responseStatusCode: number,
  responseHeaders: Headers,
  entryContext: EntryContext,
  loadContext: AppLoadContext
) {
  const router = createStaticRouter(
    entryContext.routeModules,
    entryContext.staticHandlerContext
  )

  return new Promise((resolve, reject) => {
    let shellRendered = false
    const { pipe, abort } = renderToPipeableStream(
      <RouterProvider router={router} />,
      {
        onShellReady() {
          shellRendered = true
          const body = new PassThrough()
          responseHeaders.set('Content-Type', 'text/html')
          resolve(
            new Response(createReadableStreamFromReadable(body), {
              headers: responseHeaders,
              status: responseStatusCode
            })
          )
          pipe(body)
        },
        onShellError(error: unknown) {
          reject(error)
        },
        onError(error: unknown) {
          console.error(error)
          responseStatusCode = 500
        }
      }
    )

    setTimeout(abort, 5000)
  })
}
```

Esta configuración del servidor habilita el renderizado en el servidor mediante streaming, lo que significa que tu aplicación puede comenzar a enviar contenido al navegador antes de que se hayan cargado todos los datos, mejorando significativamente el rendimiento percibido.

---

## Definición de rutas y diseños anidados (*nested layouts*)

React Router 7 mantiene el patrón familiar de rutas anidadas mientras lo mejora con nuevas capacidades. El concepto de layouts se vuelve más potente cuando se combina con la carga de datos y el renderizado en el servidor, creando una arquitectura de enrutamiento más sofisticada.

Los layouts anidados se entienden mejor como estructuras de interfaz compartidas (*shared UI shells*). Una ruta padre define una estructura que varias rutas secundarias tienen en común, como encabezados, navegación o barras laterales, mientras que la ruta secundaria completa el contenido específico de la página. El componente `Outlet` es el mecanismo crítico aquí: marca el lugar exacto donde se renderiza el contenido de la ruta secundaria dentro del layout principal:

```tsx
// app/components/RootLayout.tsx
import { Outlet, useNavigation, useLoaderData } from 'react-router'
import { Suspense } from 'react'

interface RootLayoutData {
  user: { name: string; email: string } | null
  notifications: Array<{ id: string; message: string }>
}

export function RootLayout() {
  const { user, notifications } = useLoaderData() as RootLayoutData
  const navigation = useNavigation()
  const isLoading = navigation.state === 'loading'

  return (
    <div className="min-h-screen bg-gray-50">
      <header className="bg-white shadow-sm border-b border-gray-200">
        <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
          <div className="flex justify-between items-center h-16">
            <div className="flex items-center">
              <h1 className="text-xl font-semibold text-gray-900">
                My Application
              </h1>
            </div>
            <div className="flex items-center space-x-4">
              {notifications.length > 0 && (
                <span className="inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium bg-red-100 text-red-800">
                  {notifications.length}
                </span>
              )}
              {user ? (
                <span className="text-sm text-gray-700">
                  Welcome, {user.name}
                </span>
              ) : (
                <a
                  href="/login"
                  className="text-sm text-blue-600 hover:text-blue-500"
                >
                  Sign In
                </a>
              )}
            </div>
          </div>
        </div>
      </header>

      <main className="max-w-7xl mx-auto py-6 px-4 sm:px-6 lg:px-8">
        {isLoading && (
          <div className="fixed top-0 left-0 right-0 h-1 bg-blue-600 animate-pulse" />
        )}
        <Suspense fallback={
          <div className="flex justify-center py-12">
            <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-blue-600"></div>
          </div>
        }>
          <Outlet />
        </Suspense>
      </main>
    </div>
  )
}
```

Crear layouts más específicos para diferentes secciones de tu aplicación permite una mejor organización y experiencia de usuario. Así es como puedes estructurar un layout de panel de control (*dashboard*) que incluye navegación específica para esa sección:

```tsx
// app/components/DashboardLayout.tsx
import { NavLink, Outlet, useMatches } from 'react-router'
import { clsx } from 'clsx'

const navigation = [
  { name: 'Overview', href: '/dashboard', icon: '📊' },
  { name: 'Analytics', href: '/dashboard/analytics', icon: '📈' },
  { name: 'Reports', href: '/dashboard/reports', icon: '📋' },
  { name: 'Settings', href: '/dashboard/settings', icon: '⚙️' }
]
export function DashboardLayout() {
  const matches = useMatches()
  const currentPath = matches[matches.length - 1]?.pathname

  return (
    <div className="flex h-full">
      <nav className="w-64 bg-white shadow-sm border-r border-gray-200">
        <div className="p-6">
          <h2 className="text-lg font-medium text-gray-900 mb-6">
            Dashboard
          </h2>
          <ul className="space-y-2">
            {navigation.map((item) => (
              <li key={item.name}>
                <NavLink
                  to={item.href}
                  className={({ isActive }) =>
                    clsx(
                      'flex items-center px-3 py-2 text-sm font-medium rounded-md transition-colors',
                      isActive
                        ? 'bg-blue-50 text-blue-700 border-r-2 border-blue-700'
                        : 'text-gray-600 hover:text-gray-900 hover:bg-gray-50'
                    )
                  }
                >
                  <span className="mr-3">{item.icon}</span>
                  {item.name}
                </NavLink>
              </li>
            ))}
          </ul>
        </div>
      </nav>

      <div className="flex-1 overflow-auto">
        <Outlet />
      </div>
    </div>
  )
}
```

En aplicaciones más grandes, este patrón ayuda a aislar responsabilidades: el layout raíz maneja la estructura global, mientras que el layout del dashboard maneja la navegación y presentación específicas del dashboard.

El siguiente árbol de rutas muestra cómo se combinan layouts anidados, loaders y rutas secundarias para reflejar una jerarquía de aplicación real:

```tsx
// app/router-advanced.tsx
export const advancedRouter = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    loader: rootLoader,
    children: [
      {
        index: true,
        element: <HomePage />,
      },
      {
        path: 'dashboard',
        element: <DashboardLayout />,
        loader: dashboardLoader,
        children: [
          {
            index: true,
            element: <DashboardOverview />,
            loader: overviewLoader,
          },
          {
            path: 'analytics',
            element: <AnalyticsPage />,
            loader: analyticsLoader,
          },
          {
            path: 'reports',
            element: <ReportsLayout />,
            children: [
              {
                index: true,
                element: <ReportsList />,
                loader: reportsLoader,
              },
              {
                path: ':reportId',
                element: <ReportDetail />,
                loader: reportDetailLoader,
              }
            ]
          }
        ]
      }
    ]
  }
])

async function rootLoader() {
  return {
    user: await getCurrentUser(),
    notifications: await getNotifications()
  }
}

async function dashboardLoader() {
  return {
    stats: await getDashboardStats(),
    recentActivity: await getRecentActivity()
  }
}
```

Esta estructura anidada permite que cada layout cargue sus propios datos de forma independiente mientras mantiene la relación jerárquica entre las diferentes partes de tu aplicación. Los datos fluyen hacia abajo a través del árbol de componentes, lo que permite a los componentes secundarios acceder tanto a sus propios datos como a los datos de los loaders principales.

---

## Agregar parámetros a las rutas y manejo de datos dinámicos

El enrutamiento dinámico en React Router 7 se ha mejorado con capacidades de carga de datos más potentes. Los parámetros de ruta se vuelven especialmente potentes cuando se combinan con loaders porque permiten que la URL determine directamente qué datos se obtienen antes del renderizado. En la práctica, esto significa que la ruta ya no es solo un comparador de rutas; se convierte en un límite de datos.

Aquí hay un ejemplo concreto del uso de parámetros de ruta para impulsar tanto la carga de datos como las interacciones del usuario dentro de una página de detalles del producto:

```tsx
// app/pages/ProductPage.tsx
import { useLoaderData, Form, useActionData, useNavigation } from 'react-router'

interface Product {
  id: string
  name: string
  description: string
  price: number
  image: string
  reviews: Array<{
    id: string
    rating: number
    comment: string
    author: string
  }>
}

interface ActionData {
  success?: boolean
  error?: string
}

export function ProductPage() {
  const product = useLoaderData() as Product
  const actionData = useActionData() as ActionData
  const navigation = useNavigation()
  const isSubmitting = navigation.state === 'submitting'

  return (
    <div className="max-w-4xl mx-auto grid grid-cols-1 lg:grid-cols-2 gap-8">
      <div>
        <img
          src={product.image}
          alt={product.name}
          className="w-full h-96 object-cover rounded-lg shadow-md"
        />
      </div>

      <div className="space-y-6">
        <div>
          <h1 className="text-3xl font-bold text-gray-900">{product.name}</h1>
          <p className="text-2xl font-semibold text-blue-600 mt-2">
            ${product.price.toFixed(2)}
          </p>
          <p className="text-gray-600 mt-4">{product.description}</p>
        </div>

        <div className="border-t border-gray-200 pt-6">
          <h3 className="text-lg font-medium text-gray-900 mb-4">Add a Review</h3>
          <Form method="post" className="space-y-4">
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-2">
                Rating
              </label>
              <select
                name="rating"
                className="block w-full border border-gray-300 rounded-md px-3 py-2"
                required
              >
                <option value="">Select a rating</option>
                {[5, 4, 3, 2, 1].map((rating) => (
                  <option key={rating} value={rating}>{rating} stars</option>
                ))}
              </select>
            </div>
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-2">
                Comment
              </label>
              <textarea
                name="comment"
                rows={4}
                className="block w-full border border-gray-300 rounded-md px-3 py-2"
                placeholder="Share your thoughts..."
                required
              />
            </div>
            <button
              type="submit"
              disabled={isSubmitting}
              className="w-full px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700 disabled:bg-gray-400"
            >
              {isSubmitting ? 'Submitting...' : 'Submit Review'}
            </button>
          </Form>
        </div>

        {actionData?.success && (
          <div className="p-4 bg-green-50 border border-green-200 rounded-md">
            <p className="text-green-800">Review submitted successfully!</p>
          </div>
        )}
      </div>
    </div>
  )
}
```

El ID del producto en la URL determina lo que obtiene el loader, y la página luego renderiza ese recurso. El action extiende la misma ruta manejando los envíos de reseñas, mostrando cómo React Router agrupa el comportamiento de lectura y escritura alrededor del mismo límite de ruta:

```typescript
// app/loaders/productLoader.ts
export async function productLoader({
  params
}: {
  params: { id: string }
}): Promise<Product> {
  const productId = params.id;
 
  if (!productId) {
    throw new Response('Product ID is required', { status: 400 });
  }

  try {
    const [productResponse, reviewsResponse] = await Promise.all([
      fetch(`/api/products/${productId}`),
      fetch(`/api/products/${productId}/reviews`)
    ]);

    if (!productResponse.ok) {
      throw new Response('Product not found', { status: 404 });
    }

    const product = await productResponse.json();
    const reviews = reviewsResponse.ok ? await reviewsResponse.json() : [];
    return { ...product, reviews };
  } catch (error) {
    if (error instanceof Response) throw error;
    throw new Response('Failed to load product', { status: 500 });
  }
}

export async function productAction({
  params,
  request
}: {
  params: { id: string };
  request: Request;
}): Promise<ActionData> {
  const formData = await request.formData();
  const rating = Number(formData.get('rating'));
  const comment = formData.get('comment') as string;

  if (!rating || !comment) {
    return { error: 'Rating and comment are required' };
  }

  try {
    const response = await fetch(`/api/products/${params.id}/reviews`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ rating, comment }),
    });

    if (!response.ok) throw new Error('Failed to submit review');
    return { success: true, review: await response.json() };
  } catch (error) {
    return { error: 'Failed to submit review. Please try again.' };
  }
}
```

React Router 7 también proporciona soporte mejorado para parámetros de búsqueda (*search parameters*) y patrones complejos de URL mediante el hook `useSearchParams`.

---

## React Router v7 y la integración de Remix

La integración de Remix en React Router 7 representa un cambio de paradigma en la forma en que abordamos las aplicaciones de React. Esta fusión incorpora la filosofía de Remix sobre estándares web, mejora progresiva (*progressive enhancement*) y renderizado del lado del servidor directamente en el ecosistema de React Router.

### Cómo Remix se convirtió en parte de React Router

La consecuencia más importante de la integración de Remix es que el enrutamiento, la carga de datos y las mutaciones ahora siguen un modelo unificado. Anteriormente, los desarrolladores a menudo tenían que combinar React Router con herramientas separadas para la obtención de datos y el manejo de formularios. React Router 7 reduce esa fragmentación permitiendo que las rutas definan la UI, los loaders y las actions juntos.

La integración mantiene la compatibilidad con versiones anteriores al tiempo que introduce nuevas capacidades. Las aplicaciones existentes de React Router pueden adoptar gradualmente patrones inspirados en Remix sin requerir reescrituras completas.

El siguiente ejemplo muestra un módulo de ruta con tres partes separadas: el loader obtiene los datos, el action maneja los envíos del formulario de suscripción al boletín y el componente renderiza la UI utilizando los resultados de ambos:

```tsx
// app/routes/blog.tsx - React Router v7 route module
import {
  data,
  Form,
  useActionData,
  useLoaderData,
  type ActionFunctionArgs,
  type LoaderFunctionArgs,
} from 'react-router'

interface BlogPost {
  id: string
  title: string
  content: string
  publishedAt: string
  author: { name: string; avatar: string }
}

interface LoaderData {
  posts: BlogPost[]
  totalPages: number
  currentPage: number
}

interface ActionData {
  success?: boolean
  message?: string
  error?: string
}

// Data loading function - runs before the component renders
export async function loader({ request }: LoaderFunctionArgs) {
  const url = new URL(request.url)
  const page = parseInt(url.searchParams.get('page') || '1', 10)

  const response = await fetch(`/api/blog?page=${page}&limit=10`)

  if (!response.ok) {
    throw data(
      { error: 'Failed to load blog posts' },
      { status: response.status }
    )
  }

  const result = await response.json()

  return data<LoaderData>({
    posts: result.posts,
    totalPages: result.totalPages,
    currentPage: page,
  })
}

// Form handler - processes form submissions
export async function action({ request }: ActionFunctionArgs) {
  const formData = await request.formData()
  const email = formData.get('email')

  if (typeof email !== 'string' || !email.includes('@')) {
    return data<ActionData>(
      { error: 'Valid email is required' },
      { status: 400 }
    )
  }

  try {
    await fetch('/api/newsletter/subscribe', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email }),
    })

    return data<ActionData>({
      success: true,
      message: 'Successfully subscribed!',
    })
  } catch {
    return data<ActionData>(
      { error: 'Failed to subscribe' },
      { status: 500 }
    )
  }
}

export default function BlogPage() {
  const { posts, totalPages, currentPage } = useLoaderData() as LoaderData
  const actionData = useActionData() as ActionData | undefined

  return (
    <div className="mx-auto max-w-4xl px-4 py-8">
      <header className="mb-12 text-center">
        <h1 className="text-4xl font-bold">Our Blog</h1>
        <p className="mt-2 text-muted-foreground">
          Insights, tutorials, and updates
        </p>
      </header>

      <aside className="mb-8 rounded-lg border bg-card p-6">
        <h2 className="mb-4 text-xl font-semibold">
          Subscribe to our newsletter
        </h2>

        <Form method="post" className="flex gap-2">
          <input
            name="email"
            type="email"
            placeholder="your@email.com"
            className="flex-1 rounded-md border px-3 py-2"
            required
          />

          <button
            type="submit"
            className="rounded-md bg-primary px-4 py-2 text-primary-foreground"
          >
            Subscribe
          </button>
        </Form>

        {actionData?.success && (
          <p className="mt-2 text-sm text-green-600">
            {actionData.message}
          </p>
        )}

        {actionData?.error && (
          <p className="mt-2 text-sm text-red-600">
            {actionData.error}
          </p>
        )}
      </aside>

      <div className="space-y-6">
        {posts.map((post) => (
          <article key={post.id} className="rounded-lg border bg-card p-6">
            <div className="mb-4 flex items-center gap-3">
              <img
                src={post.author.avatar}
                alt={post.author.name}
                className="h-10 w-10 rounded-full"
              />

              <div>
                <p className="font-medium">{post.author.name}</p>
                <p className="text-sm text-muted-foreground">
                  {new Date(post.publishedAt).toLocaleDateString()}
                </p>
              </div>
            </div>

            <h2 className="mb-2 text-2xl font-bold">
              <a href={`/blog/${post.id}`} className="hover:underline">
                {post.title}
              </a>
            </h2>

            <p className="mb-4 text-muted-foreground">
              {post.content.substring(0, 200)}...
            </p>

            <a href={`/blog/${post.id}`} className="text-primary hover:underline">
              Read more →
            </a>
          </article>
        ))}
      </div>

      <nav className="mt-8 flex justify-center gap-2">
        {Array.from({ length: totalPages }, (_, i) => i + 1).map((page) => (
          <a
            key={page}
            href={`/blog?page=${page}`}
            className={`rounded px-3 py-1 ${
              page === currentPage
                ? 'bg-primary text-primary-foreground'
                : 'bg-secondary'
            }`}
          >
            {page}
          </a>
        ))}
      </nav>
    </div>
  )
}
```

El valor de este patrón es la **coubicación** (*co-location*): en lugar de dispersar el comportamiento de la ruta en archivos o abstracciones separadas, el módulo de ruta mantiene unidos la carga de datos, el manejo de mutaciones y la presentación.

### Renderizado del lado del servidor con React Router 7

El renderizado del lado del servidor (SSR) en React Router 7 se basa en la base establecida por Remix, proporcionando una experiencia fluida lista para usar. La implementación de SSR se centra en el streaming, la mejora progresiva y las características óptimas de rendimiento:

```tsx
// app/entry.server.tsx - Enhanced SSR configuration
import type { EntryContext } from 'react-router'
import { RemixServer } from 'react-router'
import { renderToReadableStream } from 'react-dom/server'
import { createReadableStreamFromReadable } from '@react-router/node'

export default async function handleRequest(
  request: Request,
  responseStatusCode: number,
  responseHeaders: Headers,
  entryContext: EntryContext
) {
  const controller = new AbortController()
  const timeoutId = setTimeout(() => controller.abort(), 5000)
 
  const stream = await renderToReadableStream(
    <RemixServer context={entryContext} url={request.url} />,
    {
      signal: controller.signal,
      onError(error: unknown) {
        console.error(error)
        responseStatusCode = 500
      }
    }
  )
 
  stream.allReady.then(() => clearTimeout(timeoutId))
  responseHeaders.set('Content-Type', 'text/html')
 
  return new Response(stream, {
    headers: responseHeaders,
    status: responseStatusCode
  })
}
```

El enfoque de streaming garantiza que los usuarios vean el contenido lo más rápido posible, incluso cuando algunas partes de la página aún se estén cargando.

### Streaming y carga de datos diferida (*deferred data fetching*) con loaders

La carga de datos diferida resuelve un problema muy específico: no todos los datos son igualmente importantes para el primer pintado de una página. En React Router 7, los loaders pueden devolver promesas no resueltas directamente (ya no se usa `defer()`). Los datos críticos se esperan con `await` dentro del loader, mientras que los datos no críticos se devuelven como una promesa y se renderizan más tarde con `Suspense` y `Await`:

```tsx
// app/routes/dashboard.analytics.tsx
import { Suspense } from 'react'
import {
  Await,
  useLoaderData,
  type LoaderFunctionArgs,
} from 'react-router'

interface AnalyticsData {
  quickStats: {
    visitors: number
    pageViews: number
    bounceRate: number
    avgSession: string
  }
  chartData: Promise<{
    traffic: Array<{ date: string; visitors: number }>
    sources: Array<{ source: string; percentage: number }>
  }>
  reports: Promise<
    Array<{
      id: string
      name: string
      lastUpdated: string
      status: 'ready' | 'processing' | 'error'
    }>
  >
}

export async function loader({ request }: LoaderFunctionArgs) {
  const requestUrl = new URL(request.url)

  const quickStats = await fetch(
    new URL('/api/analytics/quick-stats', requestUrl.origin)
  ).then((r) => r.json())

  const chartData = fetch(
    new URL('/api/analytics/charts', requestUrl.origin)
  ).then((r) => r.json())

  const reports = fetch(
    new URL('/api/analytics/reports', requestUrl.origin)
  ).then((r) => r.json())

  return {
    quickStats,
    chartData,
    reports,
  }
}

export default function AnalyticsPage() {
  const { quickStats, chartData, reports } = useLoaderData() as AnalyticsData

  return (
    <div className="space-y-8">
      <div className="bg-white shadow rounded-lg">
        <div className="px-6 py-4 border-b border-gray-200">
          <h1 className="text-2xl font-bold text-gray-900">
            Analytics Dashboard
          </h1>
        </div>

<div className="p-6">
  <div className="grid grid-cols-1 md:grid-cols-4 gap-6">
    <StatCard
      title="Visitors"
      value={quickStats.visitors.toLocaleString()}
      icon="👥"
    />
    <StatCard
      title="Page Views"
      value={quickStats.pageViews.toLocaleString()}
      icon="📄"
    />
    <StatCard
      title="Bounce Rate"
      value={`${quickStats.bounceRate}%`}
      icon="📊"
    />
    <StatCard
      title="Avg. Session"
      value={quickStats.avgSession}
      icon="⏱"
    />
  </div>
</div>
      <div className="bg-white shadow rounded-lg">
        <div className="px-6 py-4 border-b border-gray-200">
          <h2 className="text-xl font-semibold text-gray-900">
            Traffic Analysis
          </h2>
        </div>

        <div className="p-6">
          <Suspense fallback={<ChartSkeleton />}>
            <Await resolve={chartData}>
              {(data) => <TrafficCharts data={data} />}
            </Await>
          </Suspense>
        </div>
      </div>

      <div className="bg-white shadow rounded-lg">
        <div className="px-6 py-4 border-b border-gray-200">
          <h2 className="text-xl font-semibold text-gray-900">
            Recent Reports
          </h2>
        </div>

        <div className="p-6">
          <Suspense fallback={<ReportsSkeleton />}>
            <Await resolve={reports}>
              {(reportsData) => <ReportsList reports={reportsData} />}
            </Await>
          </Suspense>
        </div>
      </div>
    </div>
  )
}

function StatCard({
  title,
  value,
  icon,
}: {
  title: string
  value: string
  icon: string
}) {
  return (
    <div className="bg-gray-50 p-4 rounded-lg">
      <div className="flex items-center">
        <span className="text-2xl mr-3">{icon}</span>
        <div>
          <p className="text-sm font-medium text-gray-600">{title}</p>
          <p className="text-2xl font-bold text-gray-900">{value}</p>
        </div>
      </div>
    </div>
  )
}

function ChartSkeleton() {
  return (
    <div className="animate-pulse space-y-4">
      <div className="h-64 bg-gray-200 rounded-lg" />
    </div>
  )
}

function ReportsSkeleton() {
  return (
    <div className="space-y-3">
      {Array.from({ length: 3 }, (_, i) => (
        <div key={i} className="h-16 bg-gray-200 rounded animate-pulse" />
      ))}
    </div>
  )
}

function TrafficCharts({
  data,
}: {
  data: {
    traffic: Array<{ date: string; visitors: number }>
    sources: Array<{ source: string; percentage: number }>
  }
}) {
  return (
    <div className="h-64 bg-blue-50 rounded-lg flex items-center justify-center">
      <p className="text-blue-600">
        Traffic visualization with {data.sources.length} sources
      </p>
    </div>
  )
}

function ReportsList({
  reports,
}: {
  reports: Array<{
    id: string
    name: string
    lastUpdated: string
    status: 'ready' | 'processing' | 'error'
  }>
}) {
  return (
    <div className="space-y-3">
      {reports.map((report) => (
        <div
          key={report.id}
          className="flex items-center justify-between p-4 border border-gray-200 rounded-lg"
        >
          <div className="flex items-center space-x-3">
            <div
              className={`w-3 h-3 rounded-full ${
                report.status === 'ready'
                  ? 'bg-green-500'
                  : report.status === 'processing'
                    ? 'bg-yellow-500'
                    : 'bg-red-500'
              }`}
            />

            <div>
              <h4 className="font-medium text-gray-900">{report.name}</h4>
              <p className="text-sm text-gray-500">
                Updated {report.lastUpdated}
              </p>
            </div>
          </div>

          <button className="px-4 py-2 text-sm bg-blue-600 text-white rounded-md hover:bg-blue-700">
            View Report
          </button>
        </div>
      ))}
    </div>
  )
}
```

### Manejo de envíos de formularios con la API de Actions mejorada

La API de actions representa el manejo de mutaciones a nivel de ruta. En lugar de conectar una capa de estado del lado del cliente solo para enviar y validar formularios, la ruta misma define cómo se procesan los envíos:

```tsx
// app/routes/contact.tsx
import {
  data,
  redirect,
  type ActionFunctionArgs,
} from 'react-router'
import {
  Form,
  useActionData,
  useLoaderData,
  useNavigation,
} from 'react-router'
import { useState } from 'react'

type LoaderData = {
  departments: Array<{ id: string; name: string }>
  settings: {
    responseTime: string
  }
}

type ActionData = {
  errors?: Record<string, string>
  values?: Record<string, string>
}

export async function loader() {
  const [departments, settings] = await Promise.all([
    fetch('/api/departments').then((r) => r.json()),
    fetch('/api/contact-settings').then((r) => r.json()),
  ])

  return data<LoaderData>({ departments, settings })
}

export async function action({ request }: ActionFunctionArgs) {
  const formData = await request.formData()
  const fields = ['name', 'email', 'subject', 'message', 'department']

  const values = Object.fromEntries(
    fields.map((field) => [field, formData.get(field) as string])
  )

  const errors: Record<string, string> = {}

  const validators = {
    name: (value: string) =>
      (!value || value.length < 2) && 'Name must be at least 2 characters',
    email: (value: string) =>
      (!value || !value.includes('@')) && 'Invalid email address',
    subject: (value: string) =>
      (!value || value.length < 5) && 'Subject must be at least 5 characters',
    message: (value: string) =>
      (!value || value.length < 10) && 'Message must be at least 10 characters',
    department: (value: string) => !value && 'Please select a department',
  }

  Object.entries(validators).forEach(([key, validate]) => {
    const error = validate(values[key])

    if (error) {
      errors[key] = error
    }
  })

  if (Object.keys(errors).length > 0) {
    return data<ActionData>(
      { errors, values },
      { status: 400 }
    )
  }

  try {
    const response = await fetch('/api/contact', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(values),
    })

    if (!response.ok) {
      throw new Error('Submit failed')
    }

    return redirect('/contact/success')
  } catch {
    return data<ActionData>(
      {
        errors: { general: 'Failed to submit. Try again.' },
        values,
      },
      { status: 500 }
    )
  }
}

export default function ContactPage() {
  const { departments, settings } = useLoaderData() as LoaderData
  const actionData = useActionData() as ActionData | undefined
  const navigation = useNavigation()
  const [charCount, setCharCount] = useState(0)

  const isSubmitting = navigation.state === 'submitting'

  const inputClass = (field: string) =>
    `w-full px-3 py-2 border rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500 ${
      actionData?.errors?.[field]
        ? 'border-red-500 bg-red-50'
        : 'border-gray-300'
    }`

  const fields = [
    { name: 'name', label: 'Full Name', type: 'text', placeholder: 'John Doe' },
    { name: 'email', label: 'Email', type: 'email', placeholder: 'john@example.com' },
  ]

  return (
    <div className="max-w-2xl mx-auto bg-white shadow-sm rounded-lg">
      <div className="px-6 py-4 border-b">
        <h1 className="text-2xl font-bold">Contact Us</h1>
        <p className="mt-2 text-gray-600">
          Response time: {settings.responseTime}
        </p>
      </div>

      <Form method="post" className="p-6 space-y-4">
        <div className="grid md:grid-cols-2 gap-4">
          {fields.map(({ name, label, type, placeholder }) => (
            <div key={name}>
              <label className="block text-sm font-medium mb-1">
                {label} *
              </label>

              <input
                name={name}
                type={type}
                placeholder={placeholder}
                required
                defaultValue={actionData?.values?.[name]}
                className={inputClass(name)}
              />

              {actionData?.errors?.[name] && (
                <p className="text-sm text-red-600 mt-1">
                  {actionData.errors[name]}
                </p>
              )}
            </div>
          ))}
        </div>

        <select
          name="department"
          required
          className={inputClass('department')}
          defaultValue={actionData?.values?.department}
        >
          <option value="">Select Department</option>
          {departments.map((department) => (
            <option key={department.id} value={department.id}>
              {department.name}
            </option>
          ))}
        </select>

        <input
          name="subject"
          placeholder="Subject"
          required
          className={inputClass('subject')}
          defaultValue={actionData?.values?.subject}
        />

        <div>
          <label className="block text-sm font-medium mb-1">
            Message * ({charCount}/500)
          </label>

          <textarea
            name="message"
            rows={5}
            maxLength={500}
            required
            className={`${inputClass('message')} resize-none`}
            defaultValue={actionData?.values?.message}
            onChange={(event) => setCharCount(event.target.value.length)}
            placeholder="Details about your inquiry..."
          />
        </div>

        {actionData?.errors?.general && (
          <div className="p-3 bg-red-50 text-red-800 rounded">
            {actionData.errors.general}
          </div>
        )}

        <button
          type="submit"
          disabled={isSubmitting}
          className={`w-full py-2 rounded font-medium ${
            isSubmitting
              ? 'bg-gray-400'
              : 'bg-blue-600 text-white hover:bg-blue-700'
          }`}
        >
          {isSubmitting ? 'Sending...' : 'Send Message'}
        </button>
      </Form>
    </div>
  )
}
```

### División de código basada en rutas y carga perezosa (*lazy loading*)

La división de código (*code splitting*) en React Router 7 permite cargar el código según las necesidades de navegación:

```tsx
// app/routes/dashboard.lazy.tsx - Lazy-loaded route
import { lazy } from 'react';
import type { LoaderFunctionArgs } from 'react-router';

export async function loader({ request }: LoaderFunctionArgs) {
  // This loader runs on the server for SSR
  const dashboardData = await fetch('/api/dashboard/summary');
  return dashboardData.json();
}

// The component is lazy-loaded on the client
export const Component = lazy(() => import('../components/DashboardPage'));

// Optional: Export a loading component
export function HydrateFallback() {
  return (
    <div className="min-h-screen bg-gray-50 flex items-center justify-center">
      <div className="bg-white p-8 rounded-lg shadow-sm">
        <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-blue-600 mx-auto"></div>
        <p className="mt-4 text-gray-600 text-center">Loading dashboard...</p>
      </div>
    </div>
  );
}
```

El loader aún puede ejecutarse en el servidor y preparar los datos requeridos, mientras que el componente en sí se carga perezosamente en el cliente.

El framework también admite una división de código más granular para características específicas dentro de las rutas:

```tsx
// app/components/DashboardPage.tsx
import { Suspense, lazy } from 'react';
import { useLoaderData } from 'react-router';

// Lazy load heavy components
const AdvancedCharts = lazy(() => import('./AdvancedCharts'));
const DataTable = lazy(() => import('./DataTable'));
const ReportsPanel = lazy(() => import('./ReportsPanel'));

export default function DashboardPage() {
  const data = useLoaderData();
 
  return (
    <div className="space-y-8">
      <div className="bg-white p-6 rounded-lg shadow-sm">
        <h1 className="text-2xl font-bold text-gray-900 mb-4">Dashboard</h1>
       
        {/* Critical content loads immediately */}
        <div className="grid grid-cols-1 md:grid-cols-3 gap-6 mb-8">
          <div className="bg-blue-50 p-4 rounded-lg">
            <h3 className="font-medium text-blue-900">Total Users</h3>
            <p className="text-2xl font-bold text-blue-600">{data.users}</p>
          </div>
          <div className="bg-green-50 p-4 rounded-lg">
            <h3 className="font-medium text-green-900">Revenue</h3>
            <p className="text-2xl font-bold text-green-600">${data.revenue}</p>
          </div>
          <div className="bg-purple-50 p-4 rounded-lg">
            <h3 className="font-medium text-purple-900">Orders</h3>
            <p className="text-2xl font-bold text-purple-600">{data.orders}</p>
          </div>
        </div>
      </div>

      {/* Heavy components are lazy loaded */}
      <Suspense fallback={<ChartSkeleton />}>
        <AdvancedCharts data={data.chartData} />
      </Suspense>
     
      <Suspense fallback={<TableSkeleton />}>
        <DataTable data={data.tableData} />
      </Suspense>
     
      <Suspense fallback={<PanelSkeleton />}>
        <ReportsPanel reports={data.reports} />
      </Suspense>
    </div>
  );
}

function ChartSkeleton() {
  return (
    <div className="bg-white p-6 rounded-lg shadow-sm">
      <div className="animate-pulse space-y-4">
        <div className="h-4 bg-gray-200 rounded w-1/4"></div>
        <div className="h-64 bg-gray-200 rounded"></div>
      </div>
    </div>
  );
}

function TableSkeleton() {
  return (
    <div className="bg-white p-6 rounded-lg shadow-sm">
      <div className="animate-pulse space-y-4">
        <div className="h-4 bg-gray-200 rounded w-1/3"></div>
        {Array.from({ length: 5 }, (_, i) => (
          <div key={i} className="h-12 bg-gray-200 rounded"></div>
        ))}
      </div>
    </div>
  );
}

function PanelSkeleton() {
  return (
    <div className="bg-white p-6 rounded-lg shadow-sm">
      <div className="animate-pulse space-y-4">
        <div className="h-4 bg-gray-200 rounded w-1/4"></div>
        <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
          {Array.from({ length: 4 }, (_, i) => (
            <div key={i} className="h-24 bg-gray-200 rounded"></div>
          ))}
        </div>
      </div>
    </div>
  );
}
```

---

## Resumen

React Router 7 se entiende mejor no como una librería de enrutamiento más grande, sino como un framework de aplicación completo para enrutamiento, datos y renderizado. A lo largo de este capítulo, examinamos cómo las rutas ahora definen no solo la navegación, sino también la carga de datos, las mutaciones, el renderizado diferido y la división de código. La idea unificadora en todas estas funciones es que **la ruta se convierte en el límite principal tanto para la interfaz de usuario como para el comportamiento de los datos**, lo que hace que las aplicaciones sean más fáciles de estructurar y razonar a medida que crecen.

La clave para aplicar estos conceptos en la práctica es evitar tratar cada función de forma aislada: los layouts anidados, loaders, actions, datos diferidos y lazy loading forman parte del mismo modelo arquitectónico.

En el próximo capítulo, pasaremos de la arquitectura de enrutamiento y datos a una de las partes más críticas de las aplicaciones modernas: los **formularios**. Aprenderás cómo las funciones más nuevas de React 19, combinadas con herramientas como Zod y React Hook Form, simplifican la creación desde inputs simples hasta flujos complejos de varios pasos.
