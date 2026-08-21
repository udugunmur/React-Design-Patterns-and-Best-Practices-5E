# Capítulo 3: Manejo Avanzado de Errores y Depuración

Construir aplicaciones robustas en React requiere más que simplemente escribir componentes funcionales y gestionar el estado. A medida que tus aplicaciones crecen en complejidad y escalan para atender a miles o millones de usuarios, el manejo de errores se convierte en un aspecto crítico para ofrecer una experiencia de usuario confiable. React 19 introduce mejoras significativas en los mecanismos de manejo de errores, brindando a los desarrolladores un control más granular sobre cómo se capturan, reportan y gestionan los errores a lo largo del ciclo de vida de la aplicación.

El capítulo cubre los siguientes temas:

- Introducción al manejo de errores en React 19
- Implementación de Error Boundaries en React
- Manejo de errores con `onCaughtError`, `onUncaughtError` y `onRecoverableError`
- Depuración de problemas de hidratación en aplicaciones React renderizadas en el servidor
- Estrategias para depurar aplicaciones React
- Mejores prácticas y consideraciones de producto

---

## Requisitos técnicos

Los lectores deben tener experiencia intermedia con React (más de 6 meses) con una sólida comprensión de hooks, el ciclo de vida de los componentes y los fundamentos de TypeScript. Los prerrequisitos esenciales incluyen conocimientos de React 16+, dominio de JavaScript ES6+ y familiaridad con herramientas de desarrollo modernas como la extensión de navegador React DevTools. Los requisitos de configuración técnica incluyen Node.js 24, React 19, configuración de TypeScript y Tailwind CSS. La experiencia recomendada abarca conceptos de renderizado del lado del servidor (especialmente Next.js), patrones de manejo de errores y conocimientos básicos de optimización de rendimiento. Los lectores deben sentirse cómodos creando aplicaciones de React más allá de tutoriales y estar listos para abordar inquietudes de nivel de producción como la depuración de problemas de hidratación y la implementación de mecanismos sofisticados de recuperación de errores.

---

## Introducción al manejo de errores en React 19

Construir aplicaciones robustas en React requiere más que simplemente escribir componentes funcionales y gestionar el estado. A medida que las aplicaciones crecen en complejidad y escalan para atender a miles de usuarios, el manejo de errores se vuelve crítico para ofrecer experiencias de usuario confiables. React 19 introduce mejoras significativas en los mecanismos de manejo de errores, brindando a los desarrolladores un control más granular sobre cómo se capturan, reportan y gestionan los errores a lo largo del ciclo de vida de la aplicación.

### Comprensión de la importancia de un manejo de errores robusto

El manejo de errores en las aplicaciones React tiene múltiples propósitos más allá de evitar caídas del sistema. Una estrategia de manejo de errores bien implementada garantiza que las aplicaciones sigan funcionando cuando ocurren problemas inesperados, proporciona información significativa a los usuarios y ofrece a los desarrolladores los conocimientos necesarios para diagnosticar y corregir problemas rápidamente.

Considera una aplicación típica de comercio electrónico donde un usuario está realizando el pago. Si ocurre un error en el componente de procesamiento de pagos y no se maneja adecuadamente, toda la aplicación podría colapsar, resultando en ventas perdidas y clientes frustrados. Con los Error Boundaries y estrategias de manejo de errores adecuados, puedes aislar el error en el componente de pago, mostrar un mensaje amigable al usuario y permitirle reintentar mientras mantienes funcional el resto de la aplicación.

El manejo moderno de errores también juega un papel crucial en la observabilidad de la aplicación. Al implementar un seguimiento exhaustivo de errores, puedes identificar patrones en el comportamiento del usuario que conducen a fallos, descubrir casos límite en tu código y abordar proactivamente los problemas antes de que afecten a una base de usuarios más amplia.

### Fuentes comunes de errores en aplicaciones React

Las aplicaciones React pueden encontrar errores de diversas fuentes, cada una de las cuales requiere diferentes estrategias de manejo:

- **Errores del ciclo de vida del componente:** Ocurren durante las fases de montaje, actualización o desmontaje. Estos pueden incluir errores en funciones constructoras, métodos de renderizado o hooks de efectos, como intentar acceder a propiedades de objetos indefinidos o realizar llamadas a APIs que fallan.
- **Errores de operaciones asíncronas:** Son particularmente desafiantes porque a menudo ocurren fuera del alcance normal de los Error Boundaries de React. Las solicitudes de red, las operaciones con temporizadores y los rechazos de promesas pueden fallar silenciosamente o causar estados inesperados en la aplicación si no se gestionan adecuadamente.
- **Errores de hidratación:** Son específicos de aplicaciones renderizadas en el servidor y ocurren cuando el árbol de React en el cliente no coincide con el HTML renderizado por el servidor. Estos errores pueden ser sutiles y difíciles de depurar, manifestándose a menudo como parpadeos de contenido o manipulaciones inesperadas del DOM.
- **Errores de librerías de terceros:** Pueden originarse en dependencias externas. Estos errores a menudo están fuera de tu control directo, pero deben manejarse con elegancia para evitar que colapsen toda la aplicación.
- **Errores de entrada del usuario y validación:** Ocurren cuando los usuarios proporcionan datos no válidos o interactúan con la aplicación de formas no previstas. Aunque no son necesariamente errores de código, requieren un manejo cuidadoso para mantener una buena experiencia de usuario.

### La evolución del manejo de errores en React

El enfoque de React hacia el manejo de errores ha evolucionado significativamente desde sus primeras versiones. Inicialmente, cualquier error de JavaScript dentro de un componente causaba que todo el árbol de componentes de React se desmontara, dejando a los usuarios con una pantalla en blanco.

React 16 introdujo los **Error Boundaries**, permitiendo a los desarrolladores capturar errores de JavaScript en cualquier parte del árbol de componentes, registrarlos y mostrar una interfaz de respaldo (*fallback UI*) en lugar de bloquear toda la aplicación. Sin embargo, los Error Boundaries tenían limitaciones: no podían capturar errores en controladores de eventos, código asíncrono o durante el renderizado del lado del servidor.

React 18 y 19 hacen especialmente importante separar la orquestación del estado de carga del manejo de errores. **Suspense** está diseñado para coordinar el renderizado asíncrono respondiendo a promesas lanzadas, mientras que los **Error Boundaries** manejan errores lanzados. Estos mecanismos son complementarios pero no intercambiables. Un componente que espera datos puede suspenderse y renderizar un fallback de carga, mientras que un componente que falla durante el renderizado debe ser gestionado por un Error Boundary. Tratar a Suspense como una herramienta de recuperación de errores conduce a una arquitectura incorrecta y oculta la verdadera frontera entre la coordinación asíncrona y el aislamiento de fallos.

React 19 introduce tres callbacks de error a nivel raíz, brindándote un control granular sobre diferentes escenarios de error:

```tsx
import { createRoot } from 'react-dom/client'

const root = createRoot(document.getElementById('root')!, {
  // Fires when an error is caught by an Error Boundary
  // The app recovered—the boundary showed its fallback UI
  onCaughtError(error, errorInfo) {
    // Log to your error tracking service
    // User saw a fallback, but app is still functional
    console.log('Caught by boundary:', error)
    trackError(error, { severity: 'warning', recovered: true })
  },
  // Fires when an error is NOT caught by any Error Boundary
  // The app crashed—React unmounted the entire tree
  onUncaughtError(error, errorInfo) {
    // Critical: nothing caught this, app is broken
    console.error('Uncaught error:', error)
    trackError(error, { severity: 'critical', recovered: false })
    showCrashDialog()
  },
  // Fires for errors React recovered from automatically
  // Examples: hydration mismatches, suspended trees that error
  onRecoverableError(error, errorInfo) {
    // React handled it, but something was wrong
    console.warn('Recoverable error:', error)
    trackError(error, { severity: 'info', autoRecovered: true })
  },
})

root.render(<App />)
```

Estos tres callbacks trabajan en conjunto para ofrecerte visibilidad sobre la salud de tu aplicación:

- `onCaughtError`: Tus Error Boundaries están funcionando; el usuario vio un fallback elegante.
- `onUncaughtError`: Algo se escapó; la aplicación colapsó.
- `onRecoverableError`: React lo solucionó automáticamente (como una discrepancia de hidratación), pero debes investigarlo.

Esto reemplaza la necesidad de envolver toda tu aplicación en un Error Boundary de nivel superior únicamente para propósitos de registro (*logging*).

---

## Implementación de Error Boundaries en React

Los Error Boundaries son componentes de React que capturan errores de JavaScript en cualquier parte de su árbol de componentes secundarios, registran esos errores y muestran una interfaz de respaldo en lugar del árbol de componentes que colapsó. Actúan como un mecanismo `try...catch` de JavaScript para componentes de React, proporcionando una forma de manejar errores de manera elegante y evitando que se propaguen y bloqueen toda la aplicación.

### ¿Qué son los Error Boundaries y cómo funcionan?

Los Error Boundaries funcionan implementando métodos específicos del ciclo de vida que React invoca cuando ocurre un error en cualquier componente secundario. En componentes de clase, implementas `getDerivedStateFromError` o `componentDidCatch` (o ambos) para crear un Error Boundary.

Cuando ocurre un error en un componente secundario, React llama a los métodos de manejo de errores del Error Boundary en un orden específico. Primero, se llama a `getDerivedStateFromError` durante la fase de renderizado, lo que te permite actualizar el estado del componente para mostrar una UI de respaldo. Luego, se llama a `componentDidCatch` durante la fase de commit, donde puedes realizar efectos secundarios como registrar el error en un servicio de seguimiento de errores.

Aquí hay una implementación básica de un Error Boundary:

```tsx
interface Props {
  children: ReactNode;
  fallback?: ReactNode;
}

interface State {
  hasError: boolean;
  error?: Error;
}

class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    console.error('Error caught by boundary:', error, errorInfo);
    // Report to error tracking service
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || (
        <div className="error-wrapper">
          <h3 className="error-title">Something went wrong</h3>
          <p className="error-text">Please try refreshing the page.</p>
        </div>
      );
    }

    return this.props.children;
  }
}
```

### Creación de Error Boundaries personalizados para aplicaciones React

Si bien los Error Boundaries básicos proporcionan una funcionalidad esencial, las aplicaciones del mundo real a menudo requieren estrategias de manejo de errores más sofisticadas. Los Error Boundaries personalizados se pueden adaptar a partes específicas de tu aplicación, brindando un manejo contextual de errores y mecanismos de recuperación.

Los Error Boundaries avanzados deben incluir:

- Categorización de errores
- Integración de reportes de errores
- Mecanismos de reintento
- Interfaces de usuario de respaldo contextuales
- Acciones específicas según el error

Veámoslos en detalle.

#### Categorización de errores

Diferentes errores requieren diferentes respuestas. Un tiempo de espera de red puede resolverse al reintentar, pero un fallo en la carga de un chunk (por división de código / *code splitting*) significa que el JavaScript en caché está desactualizado y necesita una recarga completa. Al categorizar los errores en el momento en que se lanzan, tu boundary puede responder apropiadamente:

```ts
// lib/errors.ts
export class NetworkError extends Error {
  constructor(message: string, public statusCode?: number) {
    super(message)
    this.name = 'NetworkError'
  }
}

export class ChunkLoadError extends Error {
  constructor(message: string) {
    super(message)
    this.name = 'ChunkLoadError'
  }
}

export function categorizeError(error: Error): 'network' | 'chunk' | 'render' {
  if (error instanceof NetworkError) return 'network'
  if (error.name === 'ChunkLoadError') return 'chunk'
  if (error.message.includes('Loading chunk')) return 'chunk'
  return 'render'
}
```

#### Integración de reportes de errores

Los errores deben reportarse automáticamente a servicios de monitoreo con suficiente contexto para depurarlos más adelante. El `componentStack` de React te indica qué componentes estaban en el árbol cuando ocurrió el error, lo cual es crítico para rastrear problemas en producción:

```ts
// lib/errorReporting.ts
interface ErrorContext {
  componentStack?: string
  category: string
  route?: string
  timestamp: number
}

export function reportError(error: Error, context: ErrorContext) {
  // Sentry
  if (typeof window !== 'undefined' && window.Sentry) {
    window.Sentry.captureException(error, {
      tags: { category: context.category },
      extra: { componentStack: context.componentStack },
    })
  }

  // Custom endpoint
  if (process.env.NEXT_PUBLIC_ERROR_ENDPOINT) {
    fetch(process.env.NEXT_PUBLIC_ERROR_ENDPOINT, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        message: error.message,
        stack: error.stack,
        ...context
      }),
      keepalive: true,
    }).catch(() => {})
  }
}
```

#### Mecanismos de reintento

El boundary en sí gestiona el estado de error y proporciona capacidades de reintento. La función `retry` limpia el estado de error y vuelve a renderizar a los hijos, dándole al componente fallido otra oportunidad sin perder la posición del usuario en la aplicación. Para errores no recuperables, `reset` desencadena una recarga completa de la página:

```tsx
// components/ErrorBoundary.tsx
'use client'

import { Component, ReactNode } from 'react'
import { categorizeError } from '@/lib/errors'
import { reportError } from '@/lib/errorReporting'
import { ErrorFallback } from './ErrorFallback'

interface Props {
  children: ReactNode
  level: 'page' | 'section' | 'component'
}

interface State {
  hasError: boolean
  error: Error | null
  category: string | null
}

export class ErrorBoundary extends Component<Props, State> {
  state: State = {
    hasError: false,
    error: null,
    category: null
  }

  static getDerivedStateFromError(error: Error): State {
    return {
      hasError: true,
      error,
      category: categorizeError(error)
    }
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    reportError(error, {
      category: categorizeError(error),
      componentStack: errorInfo.componentStack,
      route: typeof window !== 'undefined' ? window.location.pathname : undefined,
      timestamp: Date.now(),
    })
  }

  retry = () => this.setState({ hasError: false, error: null, category: null })
  reset = () => window.location.reload()

  render() {
    if (this.state.hasError && this.state.error) {
      return (
        <ErrorFallback
          error={this.state.error}
          category={this.state.category || 'render'}
          level={this.props.level}
          retry={this.retry}
          reset={this.reset}
        />
      )
    }

    return this.props.children
  }
}
```

#### Interfaces de usuario de respaldo contextuales

Un widget analítico fallido no debería mostrar una página de error a pantalla completa. La prop `level` controla cuánto espacio visual ocupa el fallback: los errores a nivel de página reciben un tratamiento prominente, mientras que los errores a nivel de componente permanecen en línea y discretos:

```tsx
// components/ErrorFallback.tsx
interface FallbackProps {
  error: Error
  category: string
  level: 'page' | 'section' | 'component'
  retry: () => void
  reset: () => void
}

const messages: Record<string, string> = {
  network: 'Please check your internet connection and try again.',
  chunk: 'The app has been updated. Please refresh to get the latest version.',
  render: 'We couldn\'t display this content. Our team has been notified.',
}

export function ErrorFallback({ category, level, retry, reset }: FallbackProps) {
  if (level === 'page') {
    return (
      <div className="min-h-screen flex items-center justify-center bg-slate-50 p-4">
        <div className="max-w-md text-center">
          <h1 className="text-2xl font-bold text-slate-900 mb-2">Something went wrong</h1>
          <p className="text-slate-600 mb-4">{messages[category] || messages.render}</p>
          <ErrorActions category={category} retry={retry} reset={reset} />
        </div>
      </div>
    )
  }

  if (level === 'section') {
    return (
      <div className="bg-red-50 border border-red-200 rounded-lg p-6 my-4">
        <h2 className="text-lg font-semibold text-red-800 mb-2">This section couldn't load</h2>
        <p className="text-slate-600 mb-4">{messages[category] || messages.render}</p>
        <ErrorActions category={category} retry={retry} reset={reset} />
      </div>
    )
  }

  return (
    <span className="inline-flex items-center gap-2 text-sm text-red-600 p-2 bg-red-50 rounded">
      <span>Failed to load</span>
      <button onClick={retry} className="underline">Retry</button>
    </span>
  )
}
```

#### Acciones específicas según el error

Diferentes errores merecen diferentes opciones de recuperación. Los errores de red son a menudo transitorios; reintentar suele funcionar. Los errores de carga de chunks significan que el navegador tiene código obsoleto y necesita una recarga completa. Los errores de renderizado pueden responder de cualquier manera, por lo que ofrecemos un reintento simple:

```tsx
// components/ErrorActions.tsx
interface ActionsProps {
  category: string
  retry: () => void
  reset: () => void
}

export function ErrorActions({ category, retry, reset }: ActionsProps) {
  const btn = 'px-4 py-2 rounded-lg'

  // Chunk errors require full refresh
  if (category === 'chunk') {
    return (
      <button onClick={reset} className={`${btn} bg-blue-600 text-white hover:bg-blue-700`}>
        Refresh page
      </button>
    )
  }

  // Network errors: retry is likely to work
  if (category === 'network') {
    return (
      <div className="flex gap-2 justify-center">
        <button onClick={retry} className={`${btn} bg-blue-600 text-white hover:bg-blue-700`}>
          Try again
        </button>
        <button onClick={reset} className={`${btn} bg-slate-200 text-slate-700 hover:bg-slate-300`}>
          Refresh
        </button>
      </div>
    )
  }

  return (
    <button onClick={retry} className={`${btn} bg-blue-600 text-white hover:bg-blue-700`}>
      Try again
    </button>
  )
}
```

#### Poniéndolo todo junto

Envuelve diferentes partes de tu aplicación con boundaries en los niveles apropiados. Los widgets independientes obtienen boundaries a nivel de componente para que un fallo no afecte a los demás. Las secciones más grandes que fallan juntas reciben tratamiento a nivel de sección. El diseño de la página en sí obtiene un boundary de nivel superior como último recurso:

```tsx
// Usage example
// app/dashboard/page.tsx
import { ErrorBoundary } from '@/components/ErrorBoundary'

export default function DashboardPage() {
  return (
    <div className="grid grid-cols-3 gap-6">
      {/* Each widget handles its own errors */}
      <ErrorBoundary level="component">
        <RevenueWidget />
      </ErrorBoundary>

      <ErrorBoundary level="component">
        <UsersWidget />
      </ErrorBoundary>

      {/* Larger section with section-level fallback */}
      <ErrorBoundary level="section">
        <ActivityFeed />
      </ErrorBoundary>
    </div>
  )
}
```

Esta arquitectura garantiza que los errores se capturen cerca de su origen, se reporten con el contexto completo y se presenten a los usuarios con opciones de recuperación adecuadas, todo sin colapsar la aplicación completa.

### Uso de Error Boundaries para componentes de servidor y cliente

En aplicaciones de React renderizadas en el servidor, las responsabilidades de manejo de errores se dividen entre el framework y los límites de ejecución en el cliente. Los errores lanzados durante el renderizado del servidor, la carga de rutas o las Server Actions suelen ser manejados primero por el framework, mientras que los errores lanzados durante el renderizado del cliente son manejados por los Error Boundaries del cliente. En el App Router de Next.js, esta distinción se vuelve concreta mediante convenciones como `error.tsx` para la recuperación a nivel de segmento de ruta y `global-error.tsx` para fallos a nivel de toda la aplicación. El objetivo práctico no es hacer que los errores de servidor y cliente se comporten de manera idéntica, sino asegurar que ambas rutas produzcan una semántica de recuperación predecible, telemetría útil y retroalimentación consistente para el usuario.

Durante el renderizado del servidor, los errores suelen ser manejados por el framework de servidor (como Next.js), mientras que los errores del lado del cliente son gestionados por los Error Boundaries de React. La clave es asegurar una experiencia de usuario consistente independientemente de dónde ocurra el error.

Las estrategias de respaldo (*fallback*) deben diseñarse según el contexto de ejecución y el alcance del fallo. En el servidor, la prioridad es evitar enviar marcado incompleto o engañoso y devolver un fallback estable para el segmento de ruta afectado. En el cliente, la prioridad cambia a aislar el subárbol fallido, preservar tanta interactividad como sea posible y ofrecer una vía de reintento cuando el fallo es recuperable. Esta distinción es particularmente importante durante la hidratación, donde las discrepancias pueden ser lo suficientemente recuperables como para que React continúe, pero aún indican un error que debe registrarse e investigarse a través de `onRecoverableError`.

Esta sección cubre la construcción de sistemas de manejo de errores en React, comenzando con los fundamentos de `componentDidCatch` y `getDerivedStateFromError`. Implementamos la categorización de errores para distinguir fallos de red de errores de renderizado, añadimos mecanismos de reintento para errores recuperables y creamos UIs de respaldo contextuales que escalan desde mensajes de error en línea hasta fallbacks de página completa.

Estos ejemplos demuestran los patrones y la arquitectura; son puntos de partida, no soluciones de producción para copiar y pegar. Un sistema de manejo de errores en producción también necesitaría integración con tu servicio de monitoreo específico (Sentry, Datadog, LogRocket), correlación con logs del backend, reproducción de sesiones de usuario para depuración, alertas de tasa de errores y estrategias de degradación elegante específicas para las rutas críticas de tu aplicación. Cubrimos las partes específicas de React; la integración con la infraestructura depende de tu stack tecnológico.

---

## Manejo de errores con `onCaughtError`, `onUncaughtError` y `onRecoverableError`

React 19 introduce tres callbacks de error a nivel raíz: `onCaughtError`, `onUncaughtError` y `onRecoverableError`. Estos callbacks mejoran la observabilidad de la aplicación, pero no reemplazan a los Error Boundaries. Los Error Boundaries siguen siendo responsables de renderizar interfaces de respaldo y ofrecer acciones de recuperación como reintentar o remontar un subárbol fallido. Los callbacks raíz tienen un propósito diferente: proporcionan hooks de reporte centralizados para que puedas clasificar fallos, capturar telemetría y distinguir entre errores recuperados, no recuperados y autorrecuperados en la raíz de la aplicación.

### Comprensión de la diferencia entre `onCaughtError`, `onUncaughtError` y `onRecoverableError`

La distinción entre `onCaughtError`, `onUncaughtError` y `onRecoverableError` es esencial para diseñar una estrategia de manejo de errores de nivel de producción. `onCaughtError` se activa cuando un error es interceptado por un Error Boundary, lo que significa que la UI ya se ha degradado elegantemente y React ha preservado el resto del árbol. `onUncaughtError` se activa cuando ningún boundary manejó el fallo y React desmontó la raíz, convirtiendo esto en la clase más severa de error de renderizado del lado del cliente. `onRecoverableError` captura casos donde React detectó un problema, como una discrepancia de hidratación, pero se recuperó automáticamente. Juntos, estos callbacks forman una capa de observabilidad, mientras que los Error Boundaries siguen siendo el mecanismo para el aislamiento y la recuperación de la UI.

Esta separación te permite implementar diferentes estrategias para diferentes tipos de errores. Los errores capturados a menudo representan escenarios de fallo esperados de los que puedes recuperarte elegantemente, mientras que los errores no capturados suelen representar fallos inesperados que requieren atención inmediata.

Así es como configuras estos manejadores al crear tu raíz de React:

```tsx
import { createRoot } from 'react-dom/client'
import * as Sentry from '@sentry/react' // Or your preferred error tracking service

const root = createRoot(document.getElementById('root')!, {
  // Error caught by an Error Boundary - app recovered
  onCaughtError: (error, errorInfo) => {
    Sentry.captureException(error, {
      level: 'warning',
      extra: { componentStack: errorInfo.componentStack },
    })
  },
  // Error NOT caught by any boundary - app crashed
  onUncaughtError: (error, errorInfo) => {
    Sentry.captureException(error, {
      level: 'fatal',
      extra: { componentStack: errorInfo.componentStack },
    })
    // Notify user since the app is broken
    alert('Something went wrong. Please refresh the page.')
  },
  // Error React recovered from automatically (e.g., hydration mismatch)
  onRecoverableError: (error, errorInfo) => {
    Sentry.captureException(error, {
      level: 'info',
      extra: { componentStack: errorInfo.componentStack },
    })
  },
})

root.render(<App />)
```

Estos tres callbacks te dan visibilidad del panorama de errores de tu aplicación. `onCaughtError` se activa cuando tus Error Boundaries funcionan según lo previsto, el usuario vio un fallback, pero la aplicación continuó ejecutándose. `onUncaughtError` significa que algo superó todos los boundaries y colapsó tu aplicación; estos son críticos y justifican una notificación inmediata al usuario. `onRecoverableError` detecta problemas que React manejó en silencio, como discrepancias de hidratación donde el HTML del servidor no coincidía con el renderizado del cliente. Al etiquetar los errores con diferentes niveles de severidad en tu servicio de monitoreo, puedes priorizar correcciones: errores fatales primero, luego advertencias y finalmente problemas de nivel informativo que indican problemas potenciales pero que no afectaron a los usuarios.

### Implementación de `onCaughtError` para el manejo de errores a nivel de componente

El callback `onCaughtError` debe tratarse como un hook de reporte centralizado, no como un mecanismo de control de UI. Para cuando se ejecuta, un Error Boundary ya ha capturado la excepción y ha renderizado su estado de respaldo. En la práctica, esto convierte a `onCaughtError` en el lugar adecuado para reenviar el error a Sentry, Datadog u otro sistema de monitoreo, enriquecerlo con metadatos de ruta y del stack de componentes, y clasificarlo por severidad o categoría. Las decisiones sobre lo que ve el usuario, si se muestra un botón de reintento y cómo se activa la recuperación deben permanecer dentro del propio Error Boundary, donde el subárbol fallido se puede aislar y remontar de forma segura.

Veamos algunas estrategias clave para manejar errores capturados. En la práctica, el manejo de errores capturados debe implementarse como parte del propio Error Boundary en lugar de como pautas abstractas. Un boundary debe renderizar una UI de respaldo contextual y exponer acciones de recuperación explícitas, como reintentar o restablecer el estado. Por ejemplo, un componente de obtención de datos fallido puede renderizar un fallback con un botón de reintento que active un nuevo renderizado, mientras preserva el estado de la UI circundante. Más importante aún, el fallback debe reflejar la naturaleza del fallo: los errores de red transitorios deben fomentar el reintento, mientras que los errores de chunks obsoletos deben desencadenar una recarga completa. Este enfoque garantiza que el manejo de errores sea determinista, impulsado por el usuario y alineado con el modo de fallo subyacente en lugar de depender de patrones de degradación genéricos.

### Uso de `onUncaughtError` para la gestión global de errores

El callback `onUncaughtError` es el hook de telemetría final para fallos que escaparon a todos los Error Boundaries y causaron que React desmontara la raíz. En ese punto, las opciones de recuperación son limitadas porque la aplicación ya no se encuentra en un estado de renderizado confiable. Por esa razón, `onUncaughtError` debe centrarse en el reporte inmediato, la clasificación de caídas y la visibilidad operativa en lugar de intentar una recuperación compleja de la UI. En la mayoría de las aplicaciones, la respuesta adecuada es capturar la carga útil completa del error, preservar cualquier contexto de diagnóstico útil y recurrir a una vía de recuperación a nivel del navegador, como una solicitud de recarga de página o una pantalla de fallo estática servida por el framework.

Un manejo eficaz de errores no capturados debe incluir lo siguiente: cuando un error llega a `onUncaughtError`, el árbol de React ya ha fallado, por lo que la recuperación debe tratarse como una preocupación operativa más que de UI. El enfoque correcto es capturar inmediatamente el error con todo el contexto de diagnóstico, incluyendo el stack del componente, la ruta y los identificadores de sesión de usuario, y reenviarlo a un servicio de monitoreo. En esta etapa, cualquier intento de recuperar parcialmente la interfaz no es confiable, por lo que el respaldo más seguro es guiar al usuario hacia una recarga completa o realizar la transición a un estado de fallo estático. En sistemas de producción, los errores no capturados repetidos se pueden utilizar para activar salvaguardas automatizadas, como deshabilitar temporalmente funciones o redirigir a los usuarios a una página de mantenimiento mientras se investiga el problema.

Esta sección exploró los manejadores `onCaughtError`, `onUncaughtError` y `onRecoverableError` de React para implementar estrategias integrales de gestión de errores.

---

## Depuración de problemas de hidratación en aplicaciones React renderizadas en el servidor

La hidratación es el proceso mediante el cual React toma el control del HTML renderizado en el servidor y lo vuelve interactivo adjuntando escuchadores de eventos e inicializando el estado del lado del cliente. Cuando el árbol de componentes de React en el cliente no coincide con el HTML renderizado por el servidor, ocurren errores de hidratación. Estos errores pueden ser particularmente difíciles de depurar porque a menudo se manifiestan como inconsistencias sutiles en la UI o problemas de rendimiento en lugar de caídas obvias.

### ¿Qué causa los errores de hidratación en React 19?

Los errores de hidratación ocurren cuando el árbol renderizado en el cliente no coincide con el HTML producido durante el renderizado en el servidor. En la práctica, estas discrepancias suelen provenir de salidas no deterministas o lógica específica del entorno, como marcas de tiempo, valores aleatorios, APIs exclusivas del navegador o estado derivado de `localStorage` durante el renderizado. La dificultad radica en que muchos problemas de hidratación no colapsan la aplicación de inmediato; en su lugar, surgen como advertencias, inconsistencias visuales, vinculaciones de eventos obsoletas o reemplazos inesperados del DOM. Por esa razón, la depuración de la hidratación debe centrarse en identificar qué valores difieren entre las pasadas de renderizado del servidor y del cliente, y trasladar la lógica no determinista fuera de la fase de renderizado.

Los errores de hidratación provienen de varias fuentes, cada una de las cuales requiere diferentes enfoques de depuración:

- **Diferencias de entorno servidor-cliente:** Son la fuente más común. Diferentes entornos pueden generar diferentes resultados de renderizado, como diferentes marcas de tiempo, valores aleatorios o APIs específicas del navegador que se llaman durante el renderizado del servidor.
- **Renderizado condicional basado en el estado del cliente:** A menudo causa discrepancias de hidratación. Cuando los componentes se renderizan de manera diferente en el servidor frente al cliente según un estado que solo está disponible en el navegador (como `window.innerWidth` o valores de `localStorage`), se producen errores de hidratación.
- **Incompatibilidades de librerías de terceros:** Pueden introducir problemas de hidratación cuando las librerías no manejan adecuadamente el renderizado del lado del servidor o cuando inyectan contenido que difiere entre los entornos de servidor y cliente.
- **Dependencias de datos asíncronos:** Pueden causar errores de hidratación cuando los componentes renderizados en el servidor dependen de datos que se obtienen de manera diferente o en momentos diferentes entre el renderizado del servidor y del cliente.

### Discrepancias comunes de hidratación y cómo solucionarlas

La forma más efectiva de depurar discrepancias de hidratación es trabajar a partir de una discrepancia concreta servidor/cliente en lugar de categorías abstractas únicamente. Si un componente renderiza `Date.now()`, `Math.random()`, dimensiones del viewport o valores respaldados por almacenamiento durante el renderizado, es probable que el servidor y el cliente produzcan salidas diferentes. El patrón correctivo suele ser directo: diferir la lógica dependiente del tiempo o exclusiva del navegador a `useEffect`, aislar el contenido exclusivo del cliente detrás de un componente wrapper como `ClientOnly`, o suprimir intencionalmente las advertencias solo cuando la discrepancia es conocida, inofensiva e inevitable. El objetivo es una salida de renderizado determinista durante la hidratación, no un enmascaramiento *a posteriori* del marcado inconsistente.

### Depuración de problemas de hidratación en Next.js y otros frameworks SSR

En frameworks como Next.js, la depuración de la hidratación debe abordarse como un flujo de trabajo repetible:

1. Reproduce la discrepancia en desarrollo para que React muestre la advertencia con el contexto del componente.
2. Inspecciona el componente afectado para detectar el uso en tiempo de renderizado de valores inestables, APIs exclusivas del cliente o ramas condicionales que dependan del estado del navegador.
3. Compara lo que el servidor puede conocer en tiempo de renderizado con lo que solo está disponible después de la hidratación.
4. Una vez identificada la fuente de la discrepancia, traslada esa lógica a `useEffect`, divide el componente en porciones seguras para el servidor y exclusivas del cliente, o reemplaza la dependencia en tiempo de renderizado con datos serializados deterministas provenientes del servidor.

Aquí hay un wrapper simple exclusivo para el cliente para contenido sensible a la hidratación:

```tsx
const ClientOnly: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [isClient, setIsClient] = useState(false)

  useEffect(() => {
    setIsClient(true)
  }, [])

  if (!isClient) return null
  return <>{children}</>
}
```

Esta sección cubrió la depuración de problemas de hidratación en frameworks de renderizado del lado del servidor, centrándose en herramientas específicas de Next.js como la configuración de React Strict Mode, reportes de error mejorados y detección integrada de errores de hidratación.

---

## Estrategias para depurar aplicaciones React

La depuración efectiva es crucial para mantener aplicaciones de React y garantizar un rendimiento óptimo. React 19 proporciona capacidades de depuración mejoradas que, combinadas con las herramientas y estrategias adecuadas, permiten a los desarrolladores identificar y resolver problemas rápidamente a lo largo de todo el ciclo de vida de la aplicación.

### Uso de React DevTools para la inspección y depuración de componentes

React DevTools debe tratarse como la interfaz principal para comprender el comportamiento en tiempo de ejecución más que como una ayuda de depuración secundaria. La pestaña **Components** te permite inspeccionar props, estado y hooks en tiempo real, lo que hace posible rastrear cómo fluyen los datos a través del árbol y dónde se originan las inconsistencias. El **Profiler** complementa esto mostrando cuándo se renderizan los componentes, por qué se renderizaron y qué tan costosa fue cada actualización. Juntas, estas herramientas brindan una imagen completa tanto de la corrección como del rendimiento, reduciendo la necesidad de registros ad hoc y haciendo que la depuración sea más determinista y repetible.

### Perfilado de rendimiento: identificación de componentes lentos

React DevTools Profiler es una herramienta poderosa para identificar cuellos de botella de rendimiento en tus aplicaciones React. Las funciones concurrentes de React 19 hacen que el perfilado de rendimiento sea aún más importante y revelador.

El perfilado de rendimiento en React 19 debe centrarse en identificar el trabajo innecesario en lugar de simplemente medir el tiempo de renderizado. Con el Profiler, puedes analizar las fases de confirmación (*commit phases*) para determinar qué componentes se renderizaron, qué desencadenó el renderizado y si el cambio de UI resultante justificó el costo. Los componentes costosos que se renderizan con frecuencia sin cambios significativos en la salida son los principales candidatos para la optimización, pero estas decisiones deben guiarse por datos de perfilado reales y no por suposiciones. Los gráficos de llama (*flame graphs*) y las vistas clasificadas de componentes proporcionan una forma rápida de identificar puntos críticos (*hotspots*), permitiéndote priorizar optimizaciones que tengan un impacto medible en la experiencia del usuario.

### Análisis de renderizados y re-renderizados para la optimización del rendimiento

Analizar los renderizados en el React moderno requiere más que contar con qué frecuencia se actualiza un componente. La pregunta importante es si un renderizado fue necesario y si su costo es proporcional al cambio visible para el usuario que produjo. React DevTools Profiler debe ser la primera herramienta para este trabajo porque muestra qué componentes se renderizaron, por qué lo hicieron y qué tan costoso fue cada commit. Los hooks de depuración personalizados pueden seguir siendo útiles durante el desarrollo, pero deben complementar, no reemplazar, al Profiler. En React 19, esto es particularmente importante porque el renderizado concurrente, las optimizaciones automáticas y la memoización asistida por el compilador cambian el perfil de rendimiento de las aplicaciones de formas que no siempre son visibles únicamente a través del registro manual en consola:

```tsx
const useRenderDebugger = (componentName: string, props: any) => {
  const prevProps = useRef(props);

  useEffect(() => {
    if (process.env.NODE_ENV === 'development') {
      console.log(`${componentName} rendered`);
      if (prevProps.current) {
        const changedProps = Object.keys(props).filter(
          key => props[key] !== prevProps.current[key]
        );
        if (changedProps.length) {
          console.log(`Props changed: ${changedProps.join(', ')}`);
        }
      }
      prevProps.current = props;
    }
  });
};
```

En cuanto a la monitorización del rendimiento, implementa una monitorización ligera que rastree métricas clave como tiempos de renderizado, frecuencias de montaje/desmontaje de componentes y capacidad de respuesta a la interacción del usuario.

Las estrategias de optimización deben aplicarse como intervenciones específicas basadas en datos de perfilado en lugar de patrones predeterminados. Los cálculos costosos pueden memoizarse cuando sus entradas son estables y su costo es significativo, mientras que los componentes que se re-renderizan con frecuencia con props estables pueden beneficiarse de `React.memo`. Los controladores de eventos solo deben memoizarse cuando se pasan a componentes hijos profundamente optimizados que dependen de la igualdad referencial. En muchos casos, decisiones arquitectónicas como dividir componentes, reducir la superficie de props o diferir el trabajo no crítico brindan mayores ganancias de rendimiento que las microoptimizaciones. El énfasis debe estar en mejoras medibles en lugar de mejores prácticas teóricas.

Esta sección cubrió la optimización del rendimiento y las estrategias de depuración en producción para aplicaciones de React 19.

---

## Mejores prácticas y consideraciones de producto

En sistemas de producción, el manejo de errores debe diseñarse como una capacidad operativa, no solo como una salvaguarda de renderizado. Eso significa colocar Error Boundaries lo suficientemente cerca de las funciones inestables para limitar el radio de impacto (*blast radius*), clasificar los errores según su recuperabilidad e impacto en el usuario y dirigir todos los fallos a un pipeline de monitoreo centralizado con suficiente contexto para respaldar el diagnóstico. Los mecanismos de recuperación deben ser deliberados y no genéricos: reintentar fallos de red transitorios, actualizar código obsoleto para fallos de carga de chunks y evitar reintentar operaciones no idempotentes a menos que el backend lo admita explícitamente. El objetivo no es simplemente prevenir caídas, sino hacer que los fallos sean observables, aislados y procesables sin degradar la experiencia general del usuario.

### Uso de `captureOwnerStack()` para diagnósticos más enriquecidos en desarrollo

Los Error Boundaries de React ya proporcionan información útil a través del objeto de error y el stack de componentes. El stack de componentes muestra dónde ocurrió el error en el árbol renderizado, lo que suele ser suficiente para identificar el componente que falló. Sin embargo, en aplicaciones grandes, especialmente aquellas construidas a partir de muchos componentes pequeños compuestos, saber dónde se creó un componente puede ser tan importante como saber dónde falló.

La API `captureOwnerStack()` de React ayuda con este problema durante el desarrollo. Captura información del stack de propietarios (*owner stack*), brindando a los desarrolladores contexto adicional sobre la cadena de propiedad del componente que condujo al subárbol fallido. Esto puede facilitar la depuración cuando la salida renderizada está a varias capas de distancia del componente que realmente configuró o creó al hijo que falló.

Esto es particularmente útil en aplicaciones con sistemas de diseño extensos, interfaces de dashboard y layouts profundamente anidados. Por ejemplo, un error puede aparecer dentro de una celda de tabla reutilizable, un gráfico o un campo de formulario, pero el verdadero problema puede provenir del componente padre de la funcionalidad que pasó props no válidas. El stack de propietarios puede ayudar a conectar esos dos puntos con mayor claridad.

Debido a que `captureOwnerStack()` está pensado para diagnósticos de desarrollo, no debe tratarse como una dependencia de telemetría de producción. El registro en producción debe seguir basándose en información estable como el mensaje de error, el stack de componentes, la ruta actual, la acción del usuario, el ID de solicitud y la versión de la aplicación. Una buena estrategia es enriquecer los logs de desarrollo con datos del stack de propietarios cuando estén disponibles, mientras se mantienen los reportes de error de producción basados en información disponible de manera consistente en todos los entornos.

En la práctica, `captureOwnerStack()` debe verse como una mejora de depuración y no como un reemplazo de los reportes de error convencionales. Ofrece a los desarrolladores un modelo mental más claro de cómo se ensambló la interfaz fallida, lo que puede reducir el tiempo necesario para rastrear errores a través de jerarquías complejas de componentes.

Un patrón de uso básico consiste en capturar el stack de propietarios cuando React reporta un error y luego registrarlo junto al stack regular de componentes:

```tsx
import { captureOwnerStack } from "react"
import { createRoot } from "react-dom/client"
import App from "./App"

const rootElement = document.getElementById("root")

createRoot(rootElement, {
  onCaughtError(error, errorInfo) {
    const ownerStack = process.env.NODE_ENV !== "production" ? captureOwnerStack() : null

    console.group("Caught React error")
    console.error(error)
    console.log("Component stack:")
    console.log(errorInfo.componentStack)
    if (ownerStack) {
      console.log("Owner stack:")
      console.log(ownerStack)
    }
    console.groupEnd()
  }
}).render(<App />)
```

### Manejo de errores a nivel de segmento de ruta con `error.tsx`

En el App Router de Next.js, el manejo de errores no se limita a colocar manualmente Error Boundaries de React en todo el árbol de componentes. Next.js proporciona una convención basada en el sistema de archivos a través de `error.tsx`, que actúa como un Error Boundary para un segmento de ruta específico. Cuando ocurre un error no capturado dentro de ese segmento, Next.js renderiza el fallback `error.tsx` coincidente más cercano en lugar de bloquear toda la aplicación.

Esta convención fomenta un enfoque más arquitectónico para la recuperación de errores. En lugar de tener un único fallback global para cada fallo, cada segmento de ruta puede definir su propia experiencia de recuperación. Una página de dashboard puede mostrar un mensaje de error específico del dashboard. Una página de configuración puede ofrecer una acción de reintento sin afectar el resto de la aplicación. Una sección de administración anidada puede fallar independientemente del contenedor principal de la aplicación.

El archivo `error.tsx` debe ser un Client Component porque necesita admitir un comportamiento de recuperación interactivo. Por ejemplo, suele recibir una función `reset()` que permite al usuario reintentar el renderizado del segmento. Esto es importante porque muchos fallos son temporales: una petición de red puede fallar, una respuesta del servidor puede estar incompleta o una transición de estado en el cliente puede producir un error recuperable.

Un archivo `error.tsx` básico se ve así:

```tsx
"use client"

import { useEffect } from "react"

type ErrorPageProps = {
  error: Error & { digest?: string }
  reset: () => void
};

export default function ErrorPage({ error, reset }: ErrorPageProps) {
  useEffect(() => {
    console.error("Route segment error:", error)
  }, [error])

  return (
    <main className="flex min-h-[400px] flex-col items-center justify-center gap-4 px-6 text-center">
      <div>
        <h2 className="text-2xl font-semibold">
          Something went wrong
        </h2>
        <p className="mt-2 max-w-md text-sm text-gray-600">
          This section could not be loaded. You can try again without leaving the current page.
        </p>
      </div>
      <button
        type="button"
        onClick={reset}
        className="rounded-md bg-black px-4 py-2 text-sm font-medium text-white"
      >
        Try again
      </button>
    </main>
  )
}
```

Un `error.tsx` bien diseñado debe ser específico, útil y sobrio. Debe explicar que algo salió mal en esa sección, proporcionar una acción de recuperación clara y evitar exponer detalles internos de implementación a los usuarios finales. Al mismo tiempo, debe registrar suficiente contexto para que los desarrolladores diagnostiquen el fallo.

Esto hace que `error.tsx` sea ideal para fallos recuperables a nivel de ruta. Protege el layout circundante, preserva la mayor parte de la aplicación posible y le da a cada segmento de ruta control sobre su propio estado de fallo.

### Manejo de fallos a nivel de aplicación con `global-error.tsx`

Mientras que `error.tsx` maneja fallos dentro de segmentos de ruta, `global-error.tsx` está diseñado para una categoría de fallo más severa. Actúa como el fallback final cuando un error escapa a la estructura normal de boundaries a nivel de segmento, incluyendo errores que ocurren en el layout raíz (*root layout*).

Esta distinción es importante. Un error a nivel de segmento generalmente debe ser manejado por `error.tsx`, porque el resto de la estructura de la aplicación aún puede ser segura de renderizar. Por el contrario, `global-error.tsx` se usa cuando el propio árbol del layout normal ya no es confiable. En ese escenario, Next.js reemplaza el layout raíz con la interfaz de error global.

Debido a que `global-error.tsx` reemplaza el layout raíz, debe definir sus propios elementos `html` y `body`. Esto lo hace diferente de los componentes normales de página o layout. Debe ser mínimo, resiliente e independiente de dependencias frágiles de la aplicación. El objetivo es renderizar un respaldo seguro incluso cuando la estructura principal de la aplicación haya fallado.

Un `global-error.tsx` sólido debe evitar depender de proveedores complejos, componentes de UI profundamente anidados o estado de aplicación que pueda haber contribuido al colapso. Debe presentar un mensaje claro, proporcionar una forma segura de recargar o regresar a una ruta estable y registrar el error para diagnósticos. Este no es el lugar para una recuperación altamente personalizada a nivel de funcionalidad; es la última línea de defensa.

Aquí hay un ejemplo básico de cómo implementar `global-error.tsx`:

```tsx
"use client"

import { useEffect } from "react"

type GlobalErrorProps = {
  error: Error & { digest?: string }
  reset: () => void
}

export default function GlobalError({ error, reset }: GlobalErrorProps) {
  useEffect(() => {
    console.error("Application-level error:", error)
  }, [error])

  return (
    <html lang="en">
      <body>
        <main style={{ padding: "2rem", textAlign: "center" }}>
          <h1>Something went wrong</h1>
          <p>
            The application could not recover from an unexpected error. You can try loading it again.
          </p>
          {error.digest ? (
            <p>
              <small>Error reference: {error.digest}</small>
            </p>
          ) : null}
          <button type="button" onClick={reset}>
            Try again
          </button>
        </main>
      </body>
    </html>
  )
}
```

Usado correctamente, `global-error.tsx` le da a la aplicación un límite frente a fallos catastróficos. Evita que los usuarios vean una pantalla en blanco cuando el layout raíz falla, mientras mantiene la mayoría de los errores rutinarios delegados a boundaries `error.tsx` más precisos y cercanos al punto de fallo.

---

## Resumen

El manejo avanzado de errores en React 19 se entiende mejor como un sistema en capas más que como una única característica. Los Error Boundaries aíslan los fallos de renderizado y protegen la experiencia del usuario a nivel de componente, sección o página, mientras que `onCaughtError`, `onUncaughtError` y `onRecoverableError` brindan visibilidad a nivel de raíz sobre cómo se propagan los fallos a través de la aplicación. `captureOwnerStack()` de React agrega diagnósticos de desarrollo más ricos al ayudar a los desarrolladores a comprender no solo dónde ocurrió un error, sino qué componente creó el subárbol fallido. Cuando se combinan con una depuración disciplinada de la hidratación, recuperación a nivel de ruta en frameworks como Next.js y un monitoreo de nivel de producción, estos patrones forman la base de una arquitectura de frontend resiliente.

Las técnicas de este capítulo establecen los patrones esenciales requeridos para flujos de trabajo de depuración y manejo de errores listos para producción. Al clasificar los fallos, reportarlos con un contexto significativo, aislar subárboles inestables con Error Boundaries y usar React DevTools Profiler para investigar el comportamiento del renderizado, los desarrolladores pueden construir aplicaciones que fallen de manera más elegante y sean más fáciles de diagnosticar bajo condiciones del mundo real. En aplicaciones con App Router de Next.js, `error.tsx` extiende este modelo proporcionando recuperación a nivel de segmento, permitiendo que rutas individuales o secciones anidadas fallen sin colapsar la estructura completa de la aplicación. Para fallos más severos, `global-error.tsx` proporciona un fallback final a nivel de aplicación cuando el layout raíz o la estructura de renderizado de nivel superior ya no son confiables.

En conjunto, estos patrones ayudan a los equipos a diseñar los estados de fallo con tanta deliberación como los estados exitosos. El valor radica no solo en prevenir pantallas en blanco, sino en hacer que los fallos sean medibles, recuperables y más fáciles de priorizar. Los Error Boundaries a nivel de componente, los diagnósticos del stack de propietarios exclusivos para desarrollo, la recuperación de segmentos de ruta con `error.tsx` y el manejo de respaldos catastróficos con `global-error.tsx` abordan cada uno una capa diferente de fallo. Utilizados en conjunto, crean una estrategia de manejo de errores más predecible y mantenible para aplicaciones modernas de React y Next.js.

Mientras que un manejo de errores robusto garantiza que tu aplicación pueda recuperarse elegantemente de los fallos, una gestión de estado efectiva determina qué tan eficientemente funciona tu aplicación en primer lugar. El [Capítulo 4](https://subscription.packtpub.com/book/web-development/9781806108251/4) explora las capacidades avanzadas de gestión de estado de React 19, incluyendo el hook `use` para operaciones asíncronas y técnicas de optimización de rendimiento con `useDeferredValue`. Aprenderás cuándo confiar en el estado local, cuándo introducir soluciones externas como Redux Toolkit o Zustand, y cómo diseñar patrones de estado que escalen con la complejidad de tu aplicación. Estas técnicas, combinadas con las estrategias de manejo de errores cubiertas en este capítulo, te ayudarán a construir aplicaciones que sean tanto resilientes como de alto rendimiento.
