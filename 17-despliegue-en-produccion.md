# Capítulo 17: Despliegue en Producción

Hay algo satisfactorio en tomar una aplicación que has estado desarrollando localmente y verla cobrar vida en Internet. El viaje desde `localhost:3000` hasta una URL de producción representa más que una transición técnica; es el momento en que tu trabajo se vuelve accesible para usuarios reales.

En este capítulo, veremos cómo desplegar una aplicación de Next.js en Vercel, la plataforma creada por el mismo equipo que construyó Next.js. La profunda integración de Vercel con Next.js hace que el despliegue sea sencillo: características como Server Components, Edge Functions e ISR funcionan de inmediato sin configuración adicional. Sin embargo, esta conveniencia viene con compensaciones que vale la pena considerar. La estrecha integración de la plataforma puede conducir al bloqueo del proveedor (*vendor lock-in*), algunas funciones de Next.js funcionan mejor (o solo) en Vercel, y migrar a otro host más tarde puede requerir cambios en el código. Si la independencia de la plataforma es importante para tu proyecto, evalúa alternativas como AWS Amplify, Cloudflare Pages u opciones autoalojadas desde el principio.

### Una nota sobre la optimización

Este capítulo cubre características de rendimiento como el almacenamiento en caché de CDN, el despliegue en el edge y la optimización de imágenes. Estas importan más para aplicaciones orientadas al consumidor donde los milisegundos afectan la experiencia del usuario y las tasas de conversión. Las herramientas internas, los paneles de administración y las aplicaciones de bajo tráfico pueden no necesitar una optimización agresiva; un despliegue más simple en una sola región podría ser perfectamente adecuado. El rendimiento de la CDN también depende de factores fuera de tu control: la ubicación del servidor de origen, las tasas de aciertos de caché (*cache hit rates*) según tus patrones de tráfico y la distribución geográfica de tus usuarios. No asumas que el despliegue global en el edge automáticamente hace que todo sea más rápido; mide tu caso de uso específico.

En este capítulo, cubriremos los siguientes temas:

- Comprensión del panorama de producción
- Compilación para producción
- Registro en Vercel
- Creación de nuestro primer proyecto
- Configuración de un dominio
- Despliegue en producción
- Optimización del rendimiento en producción
- Monitorización y resolución de problemas
- Trabajo con la CLI de Vercel
- Mejora continua

---

## Comprensión del panorama de producción

Antes de sumergirnos en los detalles del despliegue, vale la pena tomarse un momento para comprender qué significa realmente la producción. Cuando desarrollamos localmente, trabajamos en un entorno optimizado para ciclos de retroalimentación rápidos y depuración. El reemplazo de módulos en caliente (*Hot Module Replacement*) actualiza nuestros cambios al instante, los mensajes de error son detallados y útiles, y no nos preocupamos por el tamaño de los archivos o la optimización.

La producción es diferente. Cada kilobyte importa. Cada milisegundo de tiempo de carga afecta la experiencia del usuario y, por extensión, los resultados del negocio. El código que se ejecuta en producción se ha transformado, optimizado, minificado y despojado de las comodidades de desarrollo. Las imágenes se optimizan, JavaScript se divide en fragmentos óptimos y todo se sirve a través de una red global de entrega de contenido (CDN) para garantizar la entrega más rápida posible a los usuarios de todo el mundo.

Next.js maneja gran parte de esta complejidad automáticamente, pero como desarrolladores, necesitamos entender la transformación por la que pasa nuestra aplicación durante el proceso de compilación. Creemos una aplicación de Next.js lista para producción para ver esto en acción.

---

## Compilación para producción

Antes de que podamos desplegar cualquier cosa, debemos asegurarnos de que nuestra aplicación esté lista para producción. Comencemos con una aplicación de Next.js realista que demuestre patrones modernos y mejores prácticas. Este no es solo un ejemplo de "hola mundo"; es el tipo de aplicación que realmente querrías desplegar:

```tsx
// app/page.tsx
import { Suspense } from 'react'
import Hero from '@/components/Hero'

export const metadata = {
  title: 'Modern Store - Quality Products',
  description: 'Discover our curated collection of premium products',
}

export default function HomePage() {
  return (
    <main className="min-h-screen bg-gradient-to-b from-slate-50 to-white">
      {/* Hero renders immediately */}
      <Hero />

      {/* Products stream in when ready */}
      <Suspense fallback={<ProductGridSkeleton />}>
        <ProductGridLoader />
      </Suspense>
    </main>
  )
}

// Async component that fetches its own data
// Suspense catches the promise and shows fallback until resolved
async function ProductGridLoader() {
  const products = await getProducts()
  return <ProductGrid products={products} />
}

function ProductGridSkeleton() {
  return (
    <div className="container mx-auto px-4 py-12">
      <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
        {[...Array(6)].map((_, i) => (
          <div
            key={i}
            className="bg-slate-200 animate-pulse h-64 rounded-lg"
          />
        ))}
      </div>
    </div>
  )
}
```

Esta página de inicio demuestra varios patrones listos para producción: el componente asíncrono `ProductGridLoader` obtiene datos en el servidor y transmite HTML cuando está listo, la exportación de `metadata` proporciona información de SEO que Next.js inyecta en el `<head>`, y el límite de `Suspense` muestra un esqueleto mientras se cargan los productos sin impedir que el `Hero` se renderice de inmediato. Estas son convenciones de App Router, sin necesidad de configuración especial.

Nuestro componente `ProductGrid` maneja la lógica de visualización con tipos de TypeScript adecuados y consideraciones de accesibilidad:

```tsx
// components/ProductGrid.tsx
import Image from 'next/image'
import Link from 'next/link'
import ProductFilter from './ProductFilter'

interface Product {
  id: string
  name: string
  price: number
  image: string
  category: string
}

interface ProductGridProps {
  products: Product[]
}

export default function ProductGrid({ products }: ProductGridProps) {
  const categories = ['all', ...new Set(products.map(p => p.category))]

  return (
    <section
      className="container mx-auto px-4 py-12"
      aria-label="Product catalog"
    >
      {/* Only the filter is a Client Component */}
      <ProductFilter categories={categories} products={products} />
    </section>
  )
}
```

Luego creamos el `ProductFilter` como un componente de cliente:

```tsx
// components/ProductFilter.tsx
'use client'

import { useState } from 'react'
import ProductCard from './ProductCard'

interface Product {
  id: string
  name: string
  price: number
  image: string
  category: string
}

interface ProductFilterProps {
  categories: string[]
  products: Product[]
}

export default function ProductFilter({ categories, products }: ProductFilterProps) {
  const [filter, setFilter] = useState('all')
  const filtered = filter === 'all' ? products : products.filter(p => p.category === filter)

  return (
    <>
      <div
        role="group"
        aria-label="Filter products by category"
        className="flex gap-2 mb-8 flex-wrap"
      >
        {categories.map(cat => (
          <button
            key={cat}
            onClick={() => setFilter(cat)}
            aria-pressed={filter === cat}
            className={`px-4 py-2 rounded-full transition-all focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2 ${
              filter === cat
                ? 'bg-blue-600 text-white shadow-lg'
                : 'bg-white text-slate-700 hover:bg-slate-100'
            }`}
          >
            {cat.charAt(0).toUpperCase() + cat.slice(1)}
          </button>
        ))}
      </div>

      <div aria-live="polite" aria-atomic="true" className="sr-only">
        {filtered.length} products shown
      </div>

      <ul role="list" className="grid grid-cols-1 md:grid-cols-3 gap-6">
        {filtered.map(product => (
          <ProductCard key={product.id} product={product} />
        ))}
      </ul>
    </>
  )
}
```

Este componente muestra la interactividad del lado del cliente con accesibilidad integrada. La directiva `'use client'` le indica a Next.js que este componente necesita APIs del navegador (`useState` para el filtrado), pero los datos iniciales del producto aún provienen del servidor a través de props. Los botones de filtro usan `aria-pressed` para comunicar el estado de alternancia a los lectores de pantalla, y los anillos de foco visibles aseguran que los usuarios de teclado puedan navegar por la interfaz. La región en vivo oculta anuncia los cambios en el recuento de productos cuando se aplican los filtros; sin esto, los usuarios de lectores de pantalla no sabrían que la cuadrícula se actualizó. La cuadrícula de productos utiliza elementos semánticos `<ul>` y `<li>`, y cada enlace combina el nombre del producto y el precio en su `aria-label` ya que la imagen es puramente decorativa.

La configuración del entorno es crucial para los despliegues en producción. Next.js utiliza variables de entorno ampliamente, y comprender cómo administrarlas en diferentes entornos es esencial:

```typescript
// lib/api.ts
const API_URL = process.env.API_URL || 'http://localhost:3000/api'

interface ApiProduct {
  id: string
  name: string
  price: number
  image: string
  category: string
}

export async function getProducts(): Promise<ApiProduct[]> {
  try {
    const res = await fetch(`${API_URL}/products`, {
      // ISR: Incremental Static Regeneration
      // - First request: fetches from API, caches the result
      // - Subsequent requests within 3600s: serves cached version instantly
      // - After 3600s: serves stale cache, triggers background revalidation
      // - Next request gets fresh data
      next: { revalidate: 3600 }
    })

    if (!res.ok) {
      throw new Error(`Failed to fetch products: ${res.status}`)
    }

    return res.json()
  } catch (error) {
    console.error('Error fetching products:', error)
    // Return empty array so the page still renders
    return []
  }
}

export async function getProduct(id: string): Promise<ApiProduct | null> {
  try {
    const res = await fetch(`${API_URL}/products/${id}`, {
      next: { revalidate: 3600 }
    })

    if (!res.ok) return null
    return res.json()
  } catch (error) {
    console.error(`Error fetching product ${id}:`, error)
    return null
  }
}

// Other caching options:
//
// next: { revalidate: 0 } - No caching, always fetch fresh
// next: { revalidate: false } - Cache forever (static)
// cache: 'no-store' - Opt out of caching entirely
// cache: 'force-cache' - Cache until manually invalidated
//
// For on-demand revalidation (e.g., after CMS update):
// import { revalidatePath, revalidateTag } from 'next/cache'
// revalidatePath('/products')
// revalidateTag('products') - requires: next: { tags: ['products'] }
```

El valor `revalidate: 3600` habilita la Regeneración Estática Incremental (*Incremental Static Regeneration*, ISR). A diferencia de la generación estática tradicional donde el contenido se fija en el momento de la compilación, ISR te permite actualizar páginas estáticas sin reconstruir todo el sitio. El número está en segundos; 3600 significa una hora. Durante esa hora, cada usuario obtiene la respuesta en caché instantáneamente. Después de una hora, la siguiente solicitud aún sirve la caché obsoleta (por lo que sigue siendo rápida), pero desencadena una actualización en segundo plano. Una vez que se completa la actualización, las solicitudes posteriores obtienen los datos actualizados. Esto te brinda el rendimiento de las páginas estáticas con contenido casi actualizado. Para los datos que cambian con frecuencia (precios de acciones, recuentos de inventario), usa un valor más bajo o `revalidate: 0`. Para contenido que rara vez cambia (publicaciones de blog, documentación), usa valores más altos o `revalidate: false` para un almacenamiento en caché indefinido.

---

## Registro en Vercel

Esta sección explica cómo desplegar tu aplicación de Next.js en Vercel. Estamos utilizando Vercel porque su integración con Next.js es fluida; funciones como Server Components, ISR y Edge Functions funcionan sin configuración adicional. Sin embargo, no es la única opción:
- **Netlify** ofrece despliegues similares basados en Git con buen soporte para Next.js.
- **Cloudflare Pages** proporciona despliegue en el edge con generosos límites gratuitos.
- **AWS Amplify** te brinda más control si ya estás en el ecosistema de AWS.
- **Railway** es amigable para los desarrolladores para aplicaciones full-stack con bases de datos.

Cada uno tiene ventajas y desventajas en precios, características y flexibilidad; evalúa según tus necesidades.

El nivel gratuito de Vercel incluye 100 GB de ancho de banda por mes, 6,000 minutos de compilación, 100,000 invocaciones de funciones serverless y una duración máxima de función de 10 segundos. Esto cubre proyectos personales y aplicaciones de bajo tráfico cómodamente. Los proyectos comerciales con mayor tráfico o funciones de equipo requieren planes de pago. Consulta su página de precios para conocer los límites actuales, ya que estos cambian periódicamente.

Dirígete a [vercel.com](https://vercel.com/) y haz clic en Sign Up. Vercel ofrece autenticación a través de GitHub, GitLab, Bitbucket o correo electrónico. GitHub proporciona la experiencia más fluida: Vercel detecta automáticamente tus repositorios, configura despliegues de vista previa en pull requests y despliega en cada push. GitLab y Bitbucket funcionan de manera similar, pero pueden requerir pasos adicionales de permisos de repositorio. El registro por correo electrónico funciona, pero implica una integración manual de Git; configurarás las conexiones de repositorio por separado en la configuración del proyecto, lo que agrega fricción al flujo de trabajo de despliegue automatizado.

Una vez autenticado, llegarás al panel de control donde puedes importar proyectos, ver el estado del despliegue y administrar la configuración del equipo. La interfaz es sencilla; tu siguiente paso es conectar un repositorio.

---

## Creación de nuestro primer proyecto

Crear un proyecto en Vercel es donde comienza la magia. Haz clic en el botón Add New Project y verás una lista de tus repositorios. Si aún no has subido tu aplicación de Next.js a Git, ahora es el momento. Vercel necesita conectarse a tu repositorio para configurar el despliegue continuo.

Selecciona tu repositorio de la lista. Vercel es lo suficientemente inteligente como para detectar que estás desplegando una aplicación de Next.js y configurará automáticamente los ajustes de compilación adecuados. Verás una pantalla de configuración que muestra:
- El preajuste del framework (*framework preset*) está configurado en Next.js. Vercel detecta esto desde tu `package.json` y la presencia de archivos de configuración de Next.js.
- El comando de compilación está precompletado como `next build`.
- El directorio de salida está configurado en `.next`.

Estos valores predeterminados son correctos para la mayoría de las aplicaciones de Next.js, pero puedes personalizarlos si tienes una configuración única.

Las variables de entorno merecen especial atención durante la configuración. ¿Recuerdas nuestro `API_URL` de antes? Si tu aplicación depende de variables de entorno, aquí es donde las defines para producción. Haz clic en Add Environment Variable y podrías agregar algo como:

```bash
# Public environment variables (exposed to browser)
NEXT_PUBLIC_API_URL=https://api.yourapp.com

# Database connections for serverless environments
# IMPORTANT: Standard connections without pooling will crash under load
# Each serverless function invocation can create a new connection
# Most databases limit connections (e.g., 20-100 depending on plan)

# Pooled connection - use for most queries (via PgBouncer, Supabase, or Neon pooler)
# Note the different port (6543 for Supabase, 5432 with ?pgbouncer=true for Neon)
DATABASE_URL=postgresql://user:pass@db.example.com:6543/db?pgbouncer=true

# Direct connection - use only for migrations and schema changes
# Migrations often use features (like advisory locks) incompatible with poolers
DATABASE_DIRECT_URL=postgresql://user:pass@db.example.com:5432/db

# Third-party API keys (never prefix with NEXT_PUBLIC_)
STRIPE_SECRET_KEY=sk_live_...
```

Observa el prefijo `NEXT_PUBLIC_` en la URL de la API. Esta es la convención de Next.js para las variables de entorno que deben exponerse al navegador. Las variables sin este prefijo solo están disponibles en el servidor, lo que protege las credenciales confidenciales para que no se filtren al código del lado del cliente.

Antes de hacer clic en Deploy, tómate un momento para revisar la configuración. La rama de Git está configurada en `main` de forma predeterminada; los despliegues se activarán cada vez que hagas un push a esta rama. Puedes agregar ramas de producción adicionales o configurar despliegues de vista previa para pull requests, pero los valores predeterminados son puntos de partida razonables.

Haz clic en Deploy y observa cómo se transmiten los registros de compilación en tiempo real. Aquí es donde verás a Next.js compilando tu aplicación para producción. El proceso normalmente incluye:
- Instalación de dependencias desde tu `package.json`.
- Ejecución del proceso de compilación de Next.js, que compila tu TypeScript, optimiza tu código, genera páginas estáticas y prepara tu aplicación para el despliegue.
- Vercel luego despliega tu aplicación en su red global de edge, configurando automáticamente certificados SSL y la distribución de CDN.

Todo el proceso suele tardar de dos a tres minutos para una aplicación típica de Next.js. Cuando se complete, verás un mensaje de éxito y una URL donde tu aplicación está en vivo. Haz clic en ella. Tu aplicación ya está en Internet.

---

## Configuración de un dominio

La URL generada automáticamente por Vercel es funcional pero no memorable. Para una aplicación de producción, deseas un dominio personalizado. Vercel hace que esto sea notablemente sencillo, ya sea que estés comprando un nuevo dominio o conectando uno existente.

En la configuración de tu proyecto, busca la sección Domains. Aquí puedes agregar dominios personalizados a tu proyecto. Si ya posees un dominio, puedes conectarlo agregándolo aquí y luego actualizando tu configuración de DNS con tu registrador de dominios.

Supongamos que deseas conectar [myapp.com](https://myapp.com/) a tu proyecto de Vercel. Agrega el dominio en la interfaz de Vercel y recibirás instrucciones de configuración de DNS. El enfoque más simple es apuntar el registro A de tu dominio a la dirección IP de Vercel y agregar un registro CNAME para el subdominio `www`:

```dns
# DNS Configuration Options (from best to acceptable)

# OPTION 1: Use Vercel's Nameservers (Recommended)
# Point your domain's nameservers to Vercel directly
# Provides best CDN performance and automatic DDoS protection
# Nameservers: ns1.vercel-dns.com, ns2.vercel-dns.com

# OPTION 2: CNAME Flattening / ALIAS Record (Cloudflare, Route53, DNSimple)
# These providers "flatten" CNAME at the apex (root) domain
# Automatically follows Vercel's DNS changes—no hardcoded IPs
ALIAS @ cname.vercel-dns.com # Root domain (example.com)
CNAME www cname.vercel-dns.com # Subdomain (www.example.com)

# OPTION 3: A Record (Not recommended)
# Hardcoded IP—if Vercel changes IPs, your site goes down
# No automatic failover or DDoS mitigation benefits
A @ 76.76.21.21 # Avoid if possible
CNAME www cname.vercel-dns.com
```

Estos valores exactos provienen de la pantalla de configuración de Vercel, así que utiliza siempre los valores que ellos proporcionan en lugar de copiar estos ejemplos directamente. Los cambios de DNS pueden tardar desde unos minutos hasta 48 horas en propagarse globalmente, aunque generalmente es rápido.

Si aún no posees un dominio, Vercel te permite comprar dominios directamente a través de su panel de control. La integración es fluida, el DNS se configura automáticamente y administras todo en un solo lugar. Sin embargo, la conveniencia tiene un costo: los precios de los dominios de Vercel suelen ser más altos que los de los registradores dedicados. Cloudflare Registrar vende dominios al costo mayorista sin margen de beneficio. Porkbun y Namecheap se encuentran constantemente entre las opciones más económicas. GoDaddy ofrece precios bajos para el primer año pero renovaciones costosas; verifica el precio de renovación antes de comprar. Para la mayoría de los desarrolladores, comprar en un registrador económico y apuntar los servidores de nombres a Vercel toma cinco minutos adicionales pero ahorra dinero anualmente. Si valoras la simplicidad sobre el costo, la compra integrada de Vercel está bien, solo ten en cuenta que estás pagando una prima por ella.

Una vez que tu dominio esté conectado y verificado, Vercel aprovisiona automáticamente un certificado SSL. Esto sucede sin ninguna acción por tu parte. En cuestión de minutos, tu sitio es accesible a través de HTTPS con redirección automática de HTTP a HTTPS. HTTPS requiere un certificado emitido por una Autoridad de Certificación (*Certificate Authority*, CA), una organización de confianza que verifica que tú controlas el dominio. Vercel utiliza Let's Encrypt ([https://letsencrypt.org](https://letsencrypt.org/)), una CA gratuita que emite certificados de confianza para todos los navegadores principales. Vercel se encarga de la verificación del dominio, la emisión del certificado, la instalación y la renovación automática cada 90 días, completamente invisible para ti.

Históricamente, los certificados SSL costaban entre $50 y $200+ al año y requerían instalación manual. Let's Encrypt eliminó esa barrera en 2015, razón por la cual el HTTPS automático gratuito es ahora un estándar en todas las plataformas de alojamiento. Si necesitas certificados de Validación Extendida (*Extended Validation*, EV), del tipo que muestra los nombres de las empresas en el navegador, los aprovisionarás por separado a través de CA de pago como DigiCert. Sin embargo, los navegadores modernos han eliminado en gran medida los indicadores visuales de EV y el cifrado es idéntico. Para la mayoría de las aplicaciones, Let's Encrypt proporciona todo lo que necesitas.

También puedes configurar múltiples dominios para un solo proyecto. Quizás desees que tanto `myapp.com` como `myapp.io` apunten a tu aplicación, o desees crear entornos de staging y producción separados con diferentes dominios. Vercel maneja esto con elegancia: agrega múltiples dominios y configura cuál es el principal.

Configura siempre redirecciones entre las versiones con `www` y sin `www` de tu dominio. Si tanto `www.myapp.com` como `myapp.com` ofrecen contenido idéntico sin redirecciones, los motores de búsqueda los indexan como páginas separadas, creando contenido duplicado que diluye tu posicionamiento SEO. En la configuración de dominio de Vercel, agrega ambas versiones pero establece una como principal. Vercel redirige automáticamente la secundaria al dominio principal. La mayoría de los sitios eligen la versión sin www (`myapp.com`) como principal para obtener URLs más limpias, pero cualquiera de las dos funciona; lo que importa es la coherencia. Esto también se aplica si estás apuntando múltiples TLDs (`.com`, `.io`, `.dev`) al mismo proyecto; elige uno como canónico y redirige los demás.

### Del pull request a producción

Ahora que tenemos nuestro dominio configurado, hablemos del flujo de trabajo de despliegue continuo. Aquí es donde la integración de Git realmente brilla. Cada vez que subes cambios a tu rama principal, Vercel inicia automáticamente un nuevo despliegue. Pero hay más sofisticación aquí de la que podrías esperar.

Cuando abres un pull request en GitHub, Vercel crea automáticamente un despliegue de vista previa (*preview deployment*) con una URL única. Este despliegue de vista previa es una versión completa y aislada de tu aplicación con los cambios de ese pull request. Tu equipo puede revisar, probar e interactuar con los cambios antes de fusionarlos en producción. Esto es invaluable para detectar problemas antes de que lleguen a tus usuarios.

Demostremos esto agregando características listas para producción a nuestra aplicación. Comenzaremos con el seguimiento de analíticas, algo que querrás en producción pero sin saturar tu consola de desarrollo. El enfoque es simple: un componente proveedor que envuelve tu aplicación y rastrea las vistas de página mediante los hooks de enrutamiento de Next.js. Dado que esto requiere APIs del navegador y hooks de React, debe ser un Client Component:

```tsx
// app/analytics/AnalyticsProvider.tsx
'use client'

import { useEffect } from 'react'
import { usePathname, useSearchParams } from 'next/navigation'

export default function AnalyticsProvider({ children }: { children: React.ReactNode }) {
  const pathname = usePathname()
  const searchParams = useSearchParams()

  useEffect(() => {
    // Track page views
    const url = pathname + (searchParams?.toString() ? `?${searchParams.toString()}` : '')
    trackPageView(url)
  }, [pathname, searchParams])

  return <>{children}</>
}

function trackPageView(url: string) {
  // In production, this would send to your analytics service
  if (process.env.NODE_ENV === 'production') {
    console.log('Page view:', url);
    // Example: window.gtag?.('event', 'page_view', { page_path: url });
  }
}
```

Este proveedor de analíticas demuestra un patrón que usarás con frecuencia en producción: comportamiento que es diferente entre desarrollo y producción. La verificación `process.env.NODE_ENV` asegura que no estemos saturando nuestra consola de desarrollo con eventos de analítica mientras seguimos rastreando el comportamiento real del usuario en producción.

La monitorización del rendimiento es otro aspecto crucial de las aplicaciones de producción. Agreguemos un componente que mida e informe sobre las métricas de rendimiento:

```tsx
// components/PerformanceMonitor.tsx
'use client'

import { useEffect } from 'react'

export default function PerformanceMonitor() {
  useEffect(() => {
    if (process.env.NODE_ENV !== 'production') {
      return
    }

    const reportWebVitals = async () => {
      // Dynamic import keeps this out of the main bundle
      const { onCLS, onINP, onFCP, onLCP, onTTFB } = await import('web-vitals')

      // Core Web Vitals (what Google measures)
      onLCP(metric => sendToAnalytics('LCP', metric.value)) // Loading
      onINP(metric => sendToAnalytics('INP', metric.value)) // Interactivity
      onCLS(metric => sendToAnalytics('CLS', metric.value)) // Visual stability

      // Additional useful metrics
      onFCP(metric => sendToAnalytics('FCP', metric.value)) // First paint
      onTTFB(metric => sendToAnalytics('TTFB', metric.value)) // Server response
    }

    reportWebVitals()

    // Capture runtime errors
    const errorHandler = (event: ErrorEvent) => {
      sendToAnalytics('error', {
        message: event.message,
        stack: event.error?.stack,
        url: event.filename,
        line: event.lineno,
        column: event.colno
      })
    }

    window.addEventListener('error', errorHandler)
    return () => window.removeEventListener('error', errorHandler)
  }, [])

  // Renders nothing—this component is purely for side effects
  return null
}

function sendToAnalytics(metric: string, value: any) {
  const endpoint = process.env.NEXT_PUBLIC_ANALYTICS_ENDPOINT
  if (!endpoint) return

  const data = JSON.stringify({ metric, value, timestamp: Date.now() })

  // Beacon API: non-blocking, survives page unload
  if (navigator.sendBeacon) {
    navigator.sendBeacon(endpoint, data)
  } else {
    // Fallback for older browsers
    fetch(endpoint, {
      method: 'POST',
      body: data,
      headers: { 'Content-Type': 'application/json' },
      keepalive: true // Allows request to outlive the page
    }).catch(() => {}) // Fail silently—don't disrupt user experience
  }
}
```

Esto es lo que mide cada métrica:

- **Core Web Vitals** (factores de posicionamiento de Google):
  - **LCP (Largest Contentful Paint)**: Cuánto tiempo tarda en renderizarse el elemento visible más grande (generalmente una imagen destacada o un título). Objetivo: menos de 2.5 segundos. Un LCP lento significa que los usuarios observan páginas incompletas.
  - **INP (Interaction to Next Paint)**: La latencia entre las interacciones del usuario (clics, toques, pulsaciones de teclas) y la siguiente actualización visual, medida a lo largo de toda la sesión e informada como la peor interacción en el percentil 98. Objetivo: menos de 200 ms. Un INP alto significa que la interfaz se siente lenta o no responde. INP reemplazó a FID (First Input Delay) en marzo de 2024 porque FID solo medía la primera interacción, pasando por alto respuestas lentas más adelante en la sesión.
  - **CLS (Cumulative Layout Shift)**: Cuánto cambia inesperadamente el diseño de la página durante la carga. Objetivo: menos de 0.1. Un CLS alto ocurre cuando las imágenes se cargan sin espacio reservado, las fuentes se intercambian y redimensionan el texto, o los anuncios inyectan contenido, lo que hace que los usuarios hagan clic en el lugar equivocado.

- **Métricas adicionales** (útiles para diagnósticos):
  - **FCP (First Contentful Paint)**: Cuánto tiempo tarda el navegador en renderizar el primer fragmento de contenido, texto, imagen o SVG. Esto les dice a los usuarios que algo está sucediendo incluso si la página no está completamente cargada.
  - **TTFB (Time to First Byte)**: Cuánto tiempo tarda el navegador en recibir el primer byte del servidor. Un TTFB lento indica problemas del lado del servidor: consultas lentas a la base de datos, falta de almacenamiento en caché de CDN o infraestructura sobrecargada. Esta es la única métrica que no puedes solucionar únicamente con la optimización del frontend.

En Next.js, el manejo de errores debe implementarse en múltiples niveles:
- Utiliza `error.tsx` para manejar fallos recuperables a nivel de ruta dentro de un segmento específico, permitiendo a los usuarios reintentar o continuar navegando sin perder todo el shell de la aplicación.
- Para los fallos que escapan a esos límites, especialmente los errores que ocurren en el layout raíz o durante la inicialización de la aplicación, define `global-error.tsx`. Este archivo actúa como el respaldo final en toda la aplicación y debe renderizar una interfaz de error mínima y resistente porque reemplaza el layout raíz cuando se activa.

Aquí estamos creando un límite de error a nivel raíz que captura cualquier error no controlado en nuestra aplicación y muestra un mensaje fácil de usar con una opción de reintento:

```tsx
// app/error.tsx
'use client'

import { useEffect } from 'react'

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    // Log error to monitoring service
    console.error('Application error:', error)
  }, [error])

  return (
    <div className="min-h-screen flex items-center justify-center bg-slate-50 px-4">
      <div className="max-w-md w-full bg-white rounded-lg shadow-lg p-8 text-center">
        <div className="w-16 h-16 bg-red-100 rounded-full flex items-center justify-center mx-auto mb-4">
          <svg className="w-8 h-8 text-red-600" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z" />
          </svg>
        </div>
        <h2 className="text-2xl font-bold text-slate-900 mb-2">
          Something went wrong
        </h2>
        <p className="text-slate-600 mb-6">
          We're sorry, but something unexpected happened. Our team has been notified.
        </p>
        <button
          onClick={reset}
          className="bg-blue-600 text-white px-6 py-3 rounded-lg font-medium hover:bg-blue-700 transition-colors"
        >
          Try Again
        </button>
      </div>
    </div>
  )
}
```

La propiedad `digest` en el objeto de error es un hash generado por Next.js para errores del lado del servidor, útil para correlacionar errores reportados por el cliente con registros del servidor sin exponer seguimientos de pila confidenciales a los usuarios. En producción, reemplaza el `console.error` con una llamada a tu servicio de monitorización (Sentry, LogRocket, Datadog) para capturar errores con el contexto completo.

---

## Optimización del rendimiento en producción

Vercel maneja muchas optimizaciones automáticamente, pero comprenderlas te ayuda a tomar mejores decisiones. Cuando tu aplicación de Next.js se compila, ocurren varias transformaciones que mejoran drásticamente el rendimiento.

La división de código (*code splitting*) ocurre automáticamente; Next.js divide tu aplicación en pequeños fragmentos de JavaScript que se cargan bajo demanda. Cuando un usuario visita tu página de inicio, solo descarga el código necesario para esa página. Si navega a la página del producto, ese código se carga en ese momento. Esto mantiene rápidas las cargas iniciales de la página.

El componente `Image` de Next.js, que utilizamos en nuestro `ProductGrid`, optimiza automáticamente las imágenes. Cuando despliegas en Vercel, estas optimizaciones ocurren bajo demanda en el edge. Una imagen subida a 3000x2000 píxeles se redimensiona, comprime y convierte automáticamente a formatos modernos como WebP cuando el navegador del usuario lo admite. El atributo `sizes` que incluimos le dice a Next.js exactamente qué tamaños generar.

El objetivo de esta sección es hacer que estas optimizaciones automáticas de producción sean intencionales, para que puedas controlar las compensaciones (tamaño del paquete frente a flexibilidad, calidad de imagen frente a ancho de banda, almacenamiento en caché frente a frescura) y verificar que realmente estás obteniendo las características de rendimiento que esperas en producción.

Agreguemos un archivo de configuración que ajuste estas optimizaciones:

```javascript
// next.config.mjs
/** @type {import('next').NextConfig} */
const nextConfig = {
  images: {
    formats: ['image/avif', 'image/webp'],
    deviceSizes: [640, 750, 828, 1080, 1200, 1920, 2048, 3840],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
    minimumCacheTTL: 60 * 60 * 24 * 365, // 1 year
    // If you load external images, add one of these:
    // domains: ['example.com'],
    // remotePatterns: [{ protocol: 'https', hostname: 'example.com' }],
  },
  async headers() {
    // NOTE: Tune Content-Security-Policy (CSP) based on what you actually load (analytics, fonts, images, etc.)
    const csp = [
      "default-src 'self'",
      "base-uri 'self'",
      "form-action 'self'",
      "object-src 'none'",
      "frame-ancestors 'self'", // modern clickjacking protection
      "img-src 'self' data: blob: https:",
      "font-src 'self' data: https:",
      "style-src 'self' 'unsafe-inline'", // If you use inline scripts or third-party scripts, you must revisit this.
      "script-src 'self'",
      'upgrade-insecure-requests'
    ].join('; ')

    return [
      {
        source: '/:path*',
        headers: [
          { key: 'Content-Security-Policy', value: csp },
          { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
          { key: 'X-DNS-Prefetch-Control', value: 'on' },
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          // Legacy fallback; CSP frame-ancestors is the modern control
          { key: 'X-Frame-Options', value: 'SAMEORIGIN' },
          { key: 'Strict-Transport-Security', value: 'max-age=63072000; includeSubDomains; preload' },
          // Optional: lock down powerful APIs (adjust as needed)
          { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=(), interest-cohort=()' }
        ]
      }
    ]
  }
}

export default nextConfig
```

Esta configuración aplica algunos valores predeterminados prácticos de producción: prioriza los formatos de imagen modernos (AVIF/WebP) para reducir el tamaño de transferencia, aumenta el TTL de la caché de optimización de imágenes para reducir el trabajo repetido en el edge y agrega un conjunto de encabezados de seguridad de referencia. Lo más importante es que introduce una Política de Seguridad de Contenido (*Content Security Policy*, CSP) para restringir desde dónde se pueden cargar scripts, estilos, imágenes y otros recursos, lo que reduce significativamente el radio de alcance de vulnerabilidades XSS si se produce un error.

Las estrategias de almacenamiento en caché son cruciales para el rendimiento. Next.js y Vercel trabajan juntos para almacenar en caché a múltiples niveles. Las páginas estáticas se almacenan en caché indefinidamente hasta que despliegas una nueva versión. Las rutas de API pueden especificar su propio comportamiento de caché. Las solicitudes `fetch` que escribimos con opciones `revalidate` utilizan la caché en el edge de Vercel para servir respuestas rápidamente manteniendo el contenido razonablemente actualizado.

Las optimizaciones de rendimiento solo importan si se mantienen bajo tráfico real, razón por la cual la monitorización y la resolución de problemas son las siguientes piezas del rompecabezas de producción.

---

## Monitorización y resolución de problemas

Una vez que tu aplicación está en vivo, la monitorización se vuelve esencial. Vercel proporciona analíticas integradas que te muestran vistas de página, páginas principales y principales fuentes de referencia sin ninguna configuración. Para obtener información más detallada, querrás integrar herramientas de monitorización dedicadas.

El panel de control de Vercel te muestra cada despliegue, con registros detallados para cada uno. Si un despliegue falla, los registros te indican exactamente qué salió mal. Los problemas comunes incluyen variables de entorno faltantes, errores de compilación que no se detectaron localmente o conflictos de dependencias.

Los registros en tiempo real están disponibles para tus funciones serverless y rutas de API. En el panel de control de Vercel, navega hasta tu proyecto y busca la pestaña Logs. Aquí ves cada solicitud a tus funciones serverless, con registros de consola, errores y métricas de rendimiento. Esto es invaluable al depurar problemas de producción que no ocurren localmente.

Creemos un endpoint de comprobación de estado (*health check*) que ayude con la monitorización:

```typescript
// app/api/health/live/route.ts
import { NextResponse } from 'next/server'

export const dynamic = 'force-dynamic'
export const revalidate = 0

export async function GET() {
  const payload = {
    timestamp: new Date().toISOString(),
    status: 'ok',
    environment: process.env.NODE_ENV,
    version: process.env.VERCEL_GIT_COMMIT_SHA?.slice(0, 7) || 'unknown',
    region: process.env.VERCEL_REGION || 'unknown',
  }

  return NextResponse.json(payload, {
    status: 200,
    headers: { 'Cache-Control': 'no-store, max-age=0' },
  })
}
```

Este endpoint de comprobación de estado puede ser consultado periódicamente por servicios de monitorización de tiempo de actividad como UptimeRobot o Pingdom para garantizar que tu aplicación esté respondiendo. La respuesta incluye información de diagnóstico útil teniendo cuidado de no exponer detalles confidenciales.

---

## Trabajo con la CLI de Vercel

Si bien el flujo de trabajo de despliegue basado en Git es elegante y automático, hay momentos en los que necesitas un control más directo sobre tus despliegues. Quizás estés probando una solución rápida, desplegando desde una rama local que aún no se ha subido o administrando variables de entorno mediante programación. Aquí es donde la CLI de Vercel se vuelve invaluable.

La instalación de la CLI es sencilla. Abre tu terminal y ejecuta `npm install -g vercel`. Esto te da el comando `vercel` (o `vc` como atajo) disponible globalmente. La primera vez que lo uses, deberás autenticarte con tu cuenta de Vercel ejecutando `vercel login`. Esto abre tu navegador para confirmar la autenticación y, una vez completada, tu terminal estará lista para desplegar.

El caso de uso más simple es desplegar directamente desde tu entorno de desarrollo local. Navega hasta el directorio de tu proyecto de Next.js y ejecuta `vercel`. Eso es todo. La CLI detecta la configuración de tu proyecto, compila tu aplicación y la despliega en una URL de vista previa. Este despliegue de vista previa es independiente de tu sitio de producción, perfecto para probar los cambios antes de confirmarlos en Git.

Cuando estés listo para desplegar, puedes usar `vercel --prod` para enviar directamente desde tu máquina local al dominio de producción. Sin embargo, esto debe evitarse en la mayoría de los proyectos empresariales: puede eludir el pipeline de lanzamiento habitual basado en Git (pull requests, revisión de código, pruebas unitarias de CI, linter y verificación de compilación) y debilita la auditabilidad al convertir el estado local, no el repositorio, en la fuente efectiva de la verdad. En los equipos de producción, el enfoque preferido es desplegar desde la rama principal a través de la integración de Git (o CI utilizando las APIs de despliegue de Vercel), para que cada lanzamiento de producción sea rastreable hasta un commit específico y validado por comprobaciones automatizadas.

La CLI brilla al administrar variables de entorno. En lugar de navegar por el panel de control, puedes agregar variables directamente desde tu terminal. Supongamos que necesitas agregar una nueva clave de API:

```bash
vercel env add STRIPE_SECRET_KEY production
```

La CLI te solicita que ingreses el valor de forma segura; no se mostrará en pantalla. Puedes agregar variables para diferentes entornos (producción, vista previa, desarrollo) e incluso descargar las variables de entorno de producción a tu archivo `.env.local`:

```bash
vercel env pull .env.local
```

La CLI te permite descargar variables de entorno a un archivo local `.env.local` mediante `vercel env pull`, lo que puede simplificar la configuración. Sin embargo, en entornos de equipo y empresariales, esto debe usarse con precaución. Descargar variables de entorno de producción localmente significa distribuir secretos de producción (claves de API, credenciales de base de datos, tokens) en las máquinas de los desarrolladores, lo que aumenta la superficie de ataque y puede violar las políticas de privilegio mínimo y cumplimiento.

Un enfoque más seguro es limitar a los desarrolladores a variables de entorno de desarrollo o vista previa, mantener los secretos de producción restringidos a CI/CD y a la plataforma de alojamiento, y administrar las credenciales altamente confidenciales a través de sistemas dedicados de administración de secretos. El desarrollo local debe basarse en credenciales con alcance limitado o de sandbox en lugar de credenciales de producción.

La CLI también es valiosa para la observabilidad del despliegue y la resolución de problemas:
- Ejecutar `vercel ls` muestra los despliegues recientes junto con sus URLs, entorno de destino (vista previa o producción) y estado, lo que te ayuda a correlacionar rápidamente los problemas con una versión específica.
- Para una investigación más profunda, `vercel inspect <deployment-url>` proporciona la salida de la compilación, los registros de tiempo de ejecución, la configuración detectada y las variables de entorno disponibles en el momento de la compilación.

Juntos, estos comandos actúan como una interfaz ligera de auditoría y depuración, permitiéndote verificar qué commit se desplegó, si se utilizó el entorno correcto y dónde ocurrió una falla de compilación o tiempo de ejecución sin requerir acceso directo al servidor.

Los registros en tiempo real son accesibles a través de `vercel logs [deployment-url]`. Esto transmite registros desde tus funciones serverless y rutas de API directamente a tu terminal. Al depurar un problema de producción, esta retroalimentación inmediata es más rápida que navegar por el panel de control. Incluso puedes seguir los registros en tiempo real con el flag `--follow`, similar a `tail -f` para los registros del servidor.

Antes de usar la CLI, debes autenticarte. Ejecutar `vercel login` inicia tu sesión de forma interactiva y almacena un token de acceso local que la CLI utiliza para comandos futuros, mientras que los entornos de CI/CD utilizan un Vercel Access Token generado proporcionado a través de la variable de entorno `VERCEL_TOKEN`. Lo que puedes desplegar está controlado por el control de acceso basado en roles de Vercel (roles de equipo y proyecto), por lo que los permisos de producción deben restringirse a usuarios aprobados o sistemas de CI. En configuraciones empresariales, esto suele estar respaldado por SSO y registros de auditoría, lo que garantiza que el acceso a la CLI esté autenticado y regulado.

Aquí hay un ejemplo práctico de un flujo de trabajo de despliegue utilizando la CLI. Supongamos que has realizado cambios en tu flujo de pago y deseas probarlos en un entorno similar al de producción antes de fusionarlos con la rama principal:

```bash
# Deploy to a preview URL
vercel

# Test the preview deployment

# Once satisfied, deploy to production
vercel --prod

# If something goes wrong, check the logs
vercel logs --follow
```

La CLI también proporciona comandos para administrar tus proyectos y dominios. Puedes vincular un directorio local a un proyecto de Vercel con `vercel link`, haciendo que los despliegues posteriores sean más rápidos. Puedes listar todos tus proyectos con `vercel projects ls`, e incluso eliminar proyectos antiguos con `vercel remove [project-name]`.

Para los equipos que trabajan con múltiples entornos o flujos de trabajo de despliegue complejos, la CLI se puede integrar en los pipelines de CI/CD. Puedes tener una GitHub Action que ejecute pruebas y luego despliegue con la CLI de Vercel solo si las pruebas se aprueban. O puedes usar la CLI en un script que despliegue en un entorno de staging antes de promocionar a producción.

Una funcionalidad particularmente útil es la capacidad de desplegar ramas o commits específicos. Si necesitas desplegar una versión anterior temporalmente para depuración o comparación, puedes hacer checkout a ese commit y ejecutar `vercel`. Esto crea un nuevo despliegue de vista previa de ese estado de código específico sin afectar tu despliegue de producción.

La CLI también respeta tu archivo de configuración `vercel.json` si tienes uno. Este archivo te permite personalizar el comportamiento del despliegue, configurar redirecciones, establecer encabezados y más. Aquí hay un ejemplo que agrega almacenamiento en caché y redirecciones personalizadas:

```json
{
  "headers": [
    {
      "source": "/fonts/(.*)",
      "headers": [
        {
          "key": "Cache-Control",
          "value": "public, max-age=31536000, immutable"
        }
      ]
    }
  ],
  "redirects": [
    {
      "source": "/old-product/:id",
      "destination": "/products/:id",
      "permanent": true
    }
  ]
}
```

Cuando despliegas con la CLI, estas configuraciones se aplican automáticamente. Esta configuración basada en archivos está versionada junto con tu código, lo que facilita el seguimiento de los cambios y la colaboración con tu equipo.

El comando `vercel dev` de la CLI merece una mención especial. Ejecuta tu aplicación de Next.js localmente pero simula el entorno de producción de Vercel, incluidas las funciones serverless, el middleware de edge y las variables de entorno. Esto ayuda a detectar problemas específicos del despliegue antes de que lleguen a producción. Es como tener un runtime local de Vercel en tu máquina.

Comprender cuándo usar la CLI frente al flujo de trabajo de despliegue integrado con Git de Vercel es importante. Para la mayoría de los equipos, los despliegues activados a través de la integración de Git (por ejemplo, pushes y pull requests a través de GitHub, GitLab o Bitbucket) deben ser los predeterminados: son automáticos, están vinculados al historial de control de versiones, pasan por comprobaciones de CI y producen un registro de despliegue rastreable. La CLI es más adecuada para la gestión de la configuración, la depuración, la inspección de despliegues y escenarios excepcionales de emergencia (*break-glass*), no como la ruta principal de lanzamiento a producción. Utilizados de esta manera, los dos enfoques se complementan en lugar de competir.

---

## Mejora continua

El despliegue no es un evento único; se convierte en parte de tu proceso de entrega continua. Con la integración de Git de Vercel, el flujo de trabajo de tu repositorio impulsa los despliegues automáticamente: las ramas de funcionalidades y los pull requests generan entornos de vista previa para pruebas y revisiones, mientras que las fusiones a la rama principal activan los despliegues de producción. Este vínculo entre el control de versiones, las comprobaciones de CI y el alojamiento crea un pipeline de lanzamiento repetible y auditable, lo que permite a los equipos realizar entregas rápidamente sin sacrificar la confiabilidad o la trazabilidad.

La capacidad de reversión (*rollback*) de Vercel se aplica a la capa de la aplicación: cada despliegue es inmutable y el tráfico se puede cambiar instantáneamente a una compilación anterior, lo que garantiza reversiones de código atómicas sin estados de despliegue parciales. Sin embargo, esto no revierte automáticamente los cambios de infraestructura como las migraciones de esquemas de bases de datos. Si un despliegue de producción incluye una migración que altera o elimina columnas, es posible que una versión anterior de la aplicación ya no sea compatible con el esquema actualizado. Por esta razón, los sistemas de producción deben seguir prácticas de migración compatibles con versiones anteriores (expandir $\rightarrow$ migrar $\rightarrow$ contraer), evitar cambios destructivos en el esquema durante la misma versión que el código que depende de ellos y tratar las migraciones de bases de datos como parte del plan de lanzamiento, no algo cubierto únicamente por la reversión de la plataforma.

A medida que tu aplicación crezca, es posible que alcances los límites del nivel gratuito. El precio basado en el uso de Vercel es transparente y el panel de control te muestra métricas de uso en tiempo real. La mayoría de las aplicaciones se mantienen dentro de los generosos límites del nivel gratuito, pero el camino hacia el escalado es claro si lo necesitas.

La belleza de esta configuración de despliegue es que escala contigo. El mismo flujo de trabajo que despliega un proyecto personal funciona para una startup y continúa funcionando a medida que creces a millones de usuarios. La infraestructura se adapta automáticamente: más tráfico significa que más ubicaciones de edge sirven tu contenido, más instancias de funciones serverless manejan solicitudes y más optimizaciones de imágenes ocurren en el edge.

Tu aplicación ahora está en vivo, monitoreada y lista para servir a usuarios de todo el mundo. El despliegue es automático, el rendimiento está optimizado y tienes las herramientas para comprender y solucionar cualquier problema que surja. Esta es la producción bien hecha, no como un paso final intimidante, sino como una extensión natural de tu flujo de trabajo de desarrollo.

Esa confianza proviene de combinar los despliegues con alertas reales, no solo paneles de control. En la práctica, los equipos conectan los registros y las señales de rendimiento de Vercel a un canal de incidentes, por ejemplo, seguimiento de errores (Sentry, Datadog, New Relic), monitorización del tiempo de actividad (Better Uptime, Pingdom, UptimeRobot) y alertas de infraestructura, y enrutan los fallos críticos a Slack, Microsoft Teams o un sistema de guardias (*on-call*) como PagerDuty u Opsgenie. Esto garantiza que cuando aumentan las tasas de error, la latencia se degrada o un endpoint se cae, las personas adecuadas reciban una notificación de inmediato. Los despliegues de vista previa ayudan a detectar problemas antes del lanzamiento, pero las alertas de producción son lo que cierra el ciclo y convierte la monitorización en una respuesta procesable.

Este es el poder de los flujos de trabajo de despliegue modernos, y está disponible tanto si estás entregando tu primer proyecto secundario como tu centésimo sistema de producción. Sin embargo, estas comodidades vienen con cierto acoplamiento a la plataforma. Las características como las funciones serverless, el middleware de edge, la optimización de imágenes y los comportamientos propietarios de compilación/tiempo de ejecución pueden requerir una refactorización si luego migras a otro proveedor de alojamiento. El costo de la migración depende de qué tan profundamente dependas de las características específicas de la plataforma: el alojamiento estático y los runtimes estándar de Node se trasladan fácilmente, mientras que las funciones de edge, la semántica de almacenamiento en caché propietaria y las integraciones administradas aumentan el esfuerzo. Para los equipos que anticipan requisitos de portabilidad, mantener la lógica principal de la aplicación conforme al estándar del framework, aislar el código específico de la plataforma y documentar las suposiciones de la infraestructura puede reducir la fricción en futuras migraciones.

---

## Resumen

El despliegue hoy en día es drásticamente más accesible de lo que era hace una década, pero no es magia y no está libre de responsabilidad. Las plataformas modernas eliminan el trabajo pesado no diferenciado (aprovisionamiento de servidores, terminación TLS, distribución global), pero los fundamentos aún existen por debajo: redes, capas de almacenamiento en caché, restricciones de tiempo de ejecución, límites de seguridad y modos de falla. La diferencia es que interactúas con ellos en un nivel de abstracción más alto, no que desaparezcan.

El uso de plataformas como Vercel no reemplaza el juicio de ingeniería; cambia dónde se aplica ese juicio. Las decisiones arquitectónicas deficientes, las consultas ineficientes, los bucles ilimitados o la confianza insegura en entradas externas fallarán con la misma rapidez en entornos serverless que en un servidor bare-metal, a veces más rápido, porque la escala amplifica los errores. La abstracción de la infraestructura reduce la carga operativa, pero también reduce la visibilidad, lo que significa que los desarrolladores deben ser deliberados sobre la observabilidad, los presupuestos de rendimiento, los límites de dependencia y el manejo de fallos.

Los ingenieros sólidos entienden ambas capas: pueden realizar entregas rápidamente utilizando plataformas modernas y comprenden lo que esas plataformas están haciendo en su nombre. Saber cómo funciona el almacenamiento en caché de HTTP, qué es un arranque en frío (*cold start*), cómo se comporta un grupo de conexiones de base de datos (*connection pool*), por qué importa la CSP o cómo se propaga el DNS sigue siendo esencial. La abstracción debe aumentar el apalancamiento, no reducir la capacidad.

El objetivo no es convertir a los ingenieros en operadores de plataformas; es permitirles dedicar más tiempo al valor del producto sin dejar de respetar los principios de diseño de sistemas. Los equipos más efectivos utilizan estas plataformas como multiplicadores de fuerza mientras conservan suficiente conocimiento de infraestructura para reconocer límites, diseñar responsablemente y recuperarse cuando las abstracciones gotean.

Tu viaje con React y Next.js no termina con el despliegue, y tampoco se detiene en el límite de la plataforma. La entrega continua es parte de la ingeniería, pero también lo es comprender los sistemas que hacen posible esa entrega.
