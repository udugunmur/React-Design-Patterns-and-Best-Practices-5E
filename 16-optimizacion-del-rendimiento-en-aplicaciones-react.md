# Capítulo 16: Optimización del Rendimiento en Aplicaciones React

La optimización del rendimiento en React opera en dos frentes: el rendimiento real (qué tan rápido suceden las cosas) y el rendimiento percibido (qué tan rápido se sienten las cosas). Una solicitud de red que tarda 500 ms es lenta, pero si los usuarios ven una retroalimentación inmediata, un estado de carga o una actualización optimista, se siente receptiva. La arquitectura de React 19 admite ambos a través del renderizado concurrente (*concurrent rendering*), que permite a React interrumpir y priorizar el trabajo, y las transiciones (*transitions*), que te permiten marcar las actualizaciones como no urgentes para que la interfaz de usuario siga respondiendo durante cálculos pesados.

Este capítulo se basa en los conceptos de renderizado de capítulos anteriores y los conecta con resultados medibles. Nos centraremos en Core Web Vitals: Largest Contentful Paint (LCP) para el rendimiento de carga, Interaction to Next Paint (INP) para la capacidad de respuesta y Cumulative Layout Shift (CLS) para la estabilidad visual, porque estas son las métricas que reflejan la experiencia real del usuario e impactan en el posicionamiento en los motores de búsqueda. Más allá de las métricas, exploraremos los mecanismos que proporciona React: memorización para omitir trabajo innecesario, división de código (*code splitting*) para reducir el tamaño inicial del bundle y Suspense para coordinar los estados de carga. El objetivo no es optimizarlo todo, sino identificar lo que realmente importa y medir si tus cambios marcan una diferencia.

En este capítulo, cubriremos los siguientes temas:

- Comprensión de los procesos de renderizado y reconciliación de React
- Técnicas de optimización de rendimiento para aplicaciones React
- Herramientas y librerías para la monitorización del rendimiento
- Rendimiento de hidratación
- Mejoras avanzadas de rendimiento en React 19

---

## Comprensión de los procesos de renderizado y reconciliación de React

Antes de que podamos optimizar de manera efectiva, debemos entender qué hace que las operaciones del DOM sean costosas. Una sola actualización del DOM es económica. Lo que es costoso es el *layout thrashing* (sacudida del diseño), leer y escribir repetidamente en el DOM de una manera que obliga al navegador a recalcular el diseño varias veces. Cuando lees una propiedad de diseño (como `offsetHeight`), el navegador debe vaciar cualquier cambio de estilo pendiente para darte un valor exacto. Si luego escribes en el DOM y vuelves a leer, has forzado dos cálculos de diseño en lugar de uno. El proceso de reconciliación de React ayuda agrupando las actualizaciones por lotes (*batching*): en lugar de actualizar el DOM después de cada cambio de estado, React recopila los cambios, los compara con el estado actual y los aplica en una sola pasada. Esto minimiza los reflujos (*reflows*) y evita lecturas y escrituras intercaladas. Pero React no puede arreglarlo todo; si tus componentes activan lecturas de diseño durante el renderizado o los efectos, aún puedes causar *layout thrashing* dentro de un solo commit de React.

### Cómo funciona el árbol Fiber de React

Cuando escribes JSX, no estás creando elementos del DOM, estás creando elementos de React, objetos JavaScript ligeros que describen cómo debería verse la interfaz de usuario. Un componente que devuelve `<div className="card">Hello</div>` en realidad devuelve un objeto como `{ type: 'div', props: { className: 'card', children: 'Hello' } }`. React mantiene estos elementos en una estructura llamada árbol Fiber, que rastrea el estado, las props y la posición de cada componente en la jerarquía de la interfaz de usuario.

Cuando el estado cambia, React construye un nuevo árbol de elementos y lo compara con el árbol Fiber actual, un proceso llamado reconciliación (*reconciliation*). React recorre ambos árboles, identifica qué cambió y calcula el conjunto mínimo de operaciones del DOM necesarias. Esta comparación ocurre en JavaScript, lo cual es rápido; las actualizaciones reales del DOM se agrupan en lotes y se aplican juntas al final.

La agrupación por lotes (*batching*) es clave para la eficiencia de React. Las múltiples actualizaciones de estado que ocurren en el mismo manejador de eventos o efecto se procesan en una sola pasada. Si llamas a `setCount(1)`, `setName('Carlos')` y `setActive(true)` en el mismo manejador de clics, React no vuelve a renderizar tres veces; las agrupa en un solo renderizado, una reconciliación y un commit al DOM. Esto reduce los cálculos de diseño y mantiene la interfaz de usuario receptiva:

```tsx
import { useState } from 'react';

interface User {
  name: string;
  email: string;
  preferences: { theme: string; notifications: boolean };
}

const UserProfile = () => {
  const [user, setUser] = useState<User>({
    name: 'Carlos Santana',
    email: 'carlos@example.com',
    preferences: { theme: 'dark', notifications: true }
  });

  const updatePreferences = () => {
    // These updates are batched automatically since React +18
    setUser(prev => ({ ...prev, preferences: { ...prev.preferences, theme: 'light' } }));
    setUser(prev => ({ ...prev, preferences: { ...prev.preferences, notifications: false } }));
    // Only one re-render occurs despite two state updates
  };

  return (
    <div className="max-w-md mx-auto p-6 bg-white rounded-lg shadow-lg">
      <h2 className="text-2xl font-bold mb-4">{user.name}</h2>
      <p className="text-gray-600 mb-2">{user.email}</p>
      <div className="mt-4 p-4 bg-gray-50 rounded">
        <p className="text-sm">Theme: {user.preferences.theme}</p>
        <p className="text-sm">Notifications: {user.preferences.notifications ? 'On' : 'Off'}</p>
      </div>
      <button
        onClick={updatePreferences}
        className="mt-4 px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700"
      >
        Update Preferences
      </button>
    </div>
  );
};
```

En este ejemplo, las dos actualizaciones de estado dentro de `updatePreferences` se agrupan automáticamente en un lote, lo que da como resultado un solo re-renderizado. Este es el *automatic batching*, una funcionalidad que se convirtió en predeterminada en React 18 y se aplica a las actualizaciones dentro de manejadores de eventos, promesas, temporizadores e incluso eventos nativos.

### Comprensión de la reconciliación y la arquitectura Fiber

El proceso de actualización de React involucra dos fases distintas manejadas por sistemas separados:
1. El **Reconciliador** (*Reconciler*) determina qué cambió comparando los árboles de elementos anteriores y siguientes.
2. El **Renderizador** (*Renderer*, ReactDOM para navegadores, React Native para móviles) aplica esos cambios a la plataforma real. Esta separación es la razón por la que React puede dirigirse a múltiples plataformas: la lógica de reconciliación sigue siendo la misma, solo cambia el renderizador.

El algoritmo de reconciliación utiliza heurísticas para lograr una complejidad $O(n)$ en lugar del $O(n^3)$ que requeriría una comparación de árbol completa. Estos son atajos, no análisis sofisticados:

- **React solo compara elementos en el mismo nivel del árbol.** Si mueves un componente de un padre a otro, React no detecta el movimiento; destruye la instancia anterior y crea una nueva. Cualquier estado local, valor de entrada de formulario, posición de desplazamiento o foco se pierde. Esta es una limitación en torno a la cual debes diseñar, no una característica.
- **Si el tipo de un elemento cambia**, digamos, de `<div>` a `<section>`, o de `<Input>` a `<TextArea>`, React destruye todo el subárbol y lo reconstruye. Esta es una operación destructiva. Los componentes dentro de ese subárbol se desmontan, su estado desaparece y los efectos se limpian. React asume que diferentes tipos producen diferentes árboles, lo cual suele ser cierto, pero pagas el costo cuando sucede.
- **Al renderizar listas, React usa la prop `key` para rastrear la identidad a través de los renderizados.** Sin claves estables, React no puede saber si un elemento se movió, se agregó o se eliminó; recurre a la comparación basada en índices, lo que provoca desmontajes y remontajes innecesarios cuando cambia el orden de la lista.

Así es como se ven las claves estables en la práctica: cada tarea usa su `id` único en lugar del índice del arreglo. Esto importa porque cuando los elementos se reordenan, agregan o eliminan, React usa claves para rastrear qué elementos del DOM corresponden a qué datos. Si usas índices de arreglos como claves, React ve la posición, no la identidad: eliminar el primer elemento hace que React piense que todos los elementos subsiguientes cambiaron porque sus índices se desplazaron. Con IDs estables, React identifica correctamente que solo se eliminó un elemento y deja los demás intactos, preservando su estado del DOM, valores de entrada y foco:

```tsx
interface Task {
  id: string
  title: string
  completed: boolean
  priority: 'low' | 'medium' | 'high'
}
interface TaskListProps {
  tasks: Task[]
  onToggle: (id: string) => void
}

const TaskList = ({ tasks, onToggle }: TaskListProps) => {
  const getPriorityColor = (priority: Task['priority']) => {
    const colors = {
      low: 'bg-green-100 text-green-800',
      medium: 'bg-yellow-100 text-yellow-800',
      high: 'bg-red-100 text-red-800'
    }
    return colors[priority]
  }

  return (
    <ul className="space-y-2">
      {tasks.map((task) => (
        <li
          key={task.id}
          className="flex items-center justify-between p-3 bg-white rounded-lg shadow"
        >
          <div className="flex items-center gap-3">
            <input
              type="checkbox"
              checked={task.completed}
              onChange={() => onToggle(task.id)}
              className="h-4 w-4 rounded border-gray-300"
            />
     <span className={task.completed ? 'line-through text-gray-400' : ''}>
              {task.title}
            </span>
          </div>
          <span className={`px-2 py-1 rounded text-sm ${getPriorityColor(task.priority)}`}>
            {task.priority}
          </span>
        </li>
      ))}
    </ul>
  )
}
```

La prop `key` en esta lista de tareas no es solo una formalidad; es fundamental para el rendimiento. Cuando cambia el estado de completado de una tarea, React puede identificar exactamente qué elemento del DOM corresponde a esa tarea y actualizar solo ese elemento, en lugar de recrear toda la lista.

### Cuándo y por qué React vuelve a renderizar componentes

Comprender cuándo React vuelve a renderizar es crucial para la optimización. Un componente se vuelve a renderizar en cuatro escenarios:
1. Cuando cambia su estado.
2. Cuando su padre se vuelve a renderizar.
3. Cuando cambia un contexto que consume.
4. Cuando un hook personalizado que utiliza desencadena una actualización.

Observa que el "cambio de props" no está en esta lista. Por defecto, React no compara las props en absoluto. Cuando un padre se vuelve a renderizar, todos los hijos se vuelven a renderizar incondicionalmente, independientemente de si sus props cambiaron.

Esto puede parecer un desperdicio, pero es una elección de diseño deliberada. React prioriza un modelo mental simple sobre el seguimiento complejo de dependencias. En lugar de analizar qué props cambiaron realmente y qué componentes necesitan actualizaciones verdaderamente, React vuelve a renderizar todo el subárbol y confía en que el proceso de reconciliación minimizará las mutaciones reales del DOM. Para la mayoría de las aplicaciones, esto funciona bien; JavaScript es rápido y la arquitectura Fiber puede interrumpir el trabajo si llegan actualizaciones de mayor prioridad.

React proporciona `memo()` para optar por la comparación de props, pero recurre a él con cuidado. Envolver todo en `memo` perjudica la legibilidad del código y agrega sobrecarga: React ahora debe comparar todas las props en cada renderizado del padre. Para los componentes que reciben objetos complejos o funciones como props, esa comparación se ejecuta en cada renderizado y el componente se vuelve a renderizar de todos modos a menos que también estabilices esas props con `useMemo` y `useCallback`. A veces, el costo de la comparación excede el costo de simplemente volver a renderizar. Primero crea perfiles (*profile first*), luego optimiza los componentes donde los re-renderizados innecesarios causen problemas medibles.

Aquí hay un ejemplo donde `memo()` tiene sentido: el contador de actualización del panel se actualiza con frecuencia, pero las tarjetas de métricas no necesitan volver a renderizarse ya que sus datos no han cambiado:

```tsx
import { useState, memo } from 'react'

interface MetricCardProps {
  title: string;
  value: number;
  change: number;
  icon: string;
}

const MetricCard = memo(({ title, value, change, icon }: MetricCardProps) => {
  console.log(`Rendering: ${title}`)
 
  return (
    <div className="p-6 bg-white rounded-lg shadow-md">
      <div className="flex items-center justify-between mb-2">
        <span className="text-gray-600 text-sm font-medium">{title}</span>
        <span className="text-2xl">{icon}</span>
      </div>
      <div className="text-3xl font-bold text-gray-900">{value.toLocaleString()}</div>
      <div className={`text-sm mt-2 ${change >= 0 ? 'text-green-600' : 'text-red-600'}`}>
        {change >= 0 ? '↑' : '↓'} {Math.abs(change)}% from last month
      </div>
    </div>
  )
})

const metrics = [
  { title: 'Total Users', value: 12543, change: 12.5, icon: '👥' },
  { title: 'Revenue', value: 54320, change: 8.3, icon: '💰' },
  { title: 'Active Sessions', value: 1834, change: -3.2, icon: '📊' }

]
const Dashboard = () => {
  const [refreshCount, setRefreshCount] = useState(0)
  const [selectedMetric, setSelectedMetric] = useState<string | null>(null)

  return (
    <div className="max-w-6xl mx-auto p-6">
      <div className="flex justify-between items-center mb-6">
        <h1 className="text-3xl font-bold">Dashboard</h1>
        <button
          onClick={() => setRefreshCount(c => c + 1)}
          className="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700"
        >
          Refresh {refreshCount}
        </button>
      </div>
      <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
        {metrics.map((metric) => (
          <MetricCard key={metric.title} {...metric} />
        ))}
      </div>
    </div>
  )
}
```

El componente `MetricCard` está envuelto en `memo()`, lo que le indica a React que omita el re-renderizado si las props no han cambiado. Cuando haces clic en el botón de actualización, el componente `Dashboard` se vuelve a renderizar por completo: ejecuta el cuerpo de su función y crea nuevos objetos de elementos de React para cada `MetricCard`. Sin embargo, `memo()` intercepta esto: React compara las nuevas props con las anteriores y, si son iguales, reutiliza el resultado de renderizado almacenado en caché en lugar de ejecutar el cuerpo de la función `MetricCard`.

Un detalle crítico: `memo()` utiliza una comparación superficial (*shallow comparison*). Comprueba si `prevProps.title === nextProps.title`, `prevProps.value === nextProps.value`, y así sucesivamente. Esto funciona aquí porque todas las props son primitivas (cadenas y números). Si pasas un objeto o arreglo como prop, por ejemplo, `data={{ value: 100 }}`, se crea un nuevo objeto en cada renderizado del padre y la comparación superficial ve referencias diferentes incluso cuando el contenido es idéntico. La memorización se rompe y el componente se vuelve a renderizar de todos modos. Para manejar props complejas, necesitarías estabilizar las referencias con `useMemo()` o proporcionar una función de comparación personalizada como segundo argumento de `memo()`.

---

## Técnicas de optimización de rendimiento para aplicaciones React

Ahora que entendemos cómo funciona React internamente, exploremos técnicas prácticas para optimizar nuestras aplicaciones. La optimización del rendimiento debe basarse en datos: primero mide, identifica cuellos de botella, aplica soluciones específicas y luego vuelve a medir para verificar la mejora. Sin mediciones, estás adivinando y a menudo optimizando código que no era un problema mientras ignoras los cuellos de botella reales.

Comienza con React DevTools Profiler, que te muestra exactamente qué componentes se renderizaron, cuánto tiempo tardó cada renderizado y qué desencadenó la actualización. La pestaña Rendimiento (*Performance*) de Chrome DevTools captura una imagen más amplia: ejecución de JavaScript, cálculos de diseño, operaciones de pintura, y ayuda a identificar si tu problema es específico de React o a nivel de navegador. Para la monitorización en producción, Lighthouse y Web Vitals te brindan las métricas de Core Web Vitals (LCP, INP, CLS) que reflejan la experiencia real del usuario.

### Cómo evitar actualizaciones de estado y re-renderizados innecesarios

Uno de los errores de rendimiento más comunes en las aplicaciones de React es crear nuevas referencias de objetos innecesariamente. La comparación de igualdad de JavaScript para objetos y arreglos se basa en la referencia, no en el valor. Esto significa que `{a: 1}` nunca es igual a `{a: 1}`; son objetos diferentes, aunque sus contenidos sean idénticos.

Esto se convierte en un problema cuando pasas objetos o funciones como props. Si creas un nuevo objeto o función en cada renderizado, las herramientas de memorización de React no pueden ayudarte porque las props parecen diferentes cada vez.

Históricamente, los desarrolladores resolvían esto con `useMemo()` y `useCallback()` para estabilizar las referencias manualmente. React Compiler (anteriormente llamado React Forget) cambia esto; analiza tu código en tiempo de compilación e inserta automáticamente la memorización donde sea beneficiosa. Si tu proyecto utiliza React 19 con el compilador habilitado, no necesitas pensar en la mayoría de estos problemas de estabilidad de referencias; el compilador los maneja.

Dicho esto, comprender por qué importa la igualdad de referencias sigue siendo valioso. El compilador no es magia; sigue las mismas reglas que tú seguirías. Saber cuándo y por qué las referencias rompen la memorización te ayuda a depurar re-renderizados inesperados, comprender la salida del compilador y trabajar en bases de código que aún no lo han adoptado. Los patrones que siguen se aplican tanto si estás escribiendo memorización manual como revisando lo que generó el compilador:

```tsx
import { useState, useMemo, useCallback, memo } from 'react'

interface Product {
  id: string
  name: string
  price: number
  category: string
}

interface ProductListProps {
  products: Product[]
  onProductClick: (product: Product) => void
}

// memo() prevents re-renders when products and onProductClick references are stable
const ProductList = memo(({ products, onProductClick }: ProductListProps) => {
  console.log('ProductList rendered')

  return (
    <div className="grid grid-cols-2 gap-4">
      {products.map(product => (
        <div
          key={product.id}
          onClick={() => onProductClick(product)}
          className="p-4 bg-white rounded-lg shadow cursor-pointer"
        >
          <h3 className="font-medium">{product.name}</h3>
          <p className="text-gray-500">{product.category}</p>
          <p className="text-blue-600 font-bold">${product.price}</p>
        </div>
      ))}
    </div>
  )
})

const allProducts: Product[] = [
  { id: '1', name: 'Laptop Pro', price: 1299, category: 'Electronics' },
  { id: '2', name: 'Wireless Mouse', price: 29, category: 'Electronics' },
  { id: '3', name: 'Office Chair', price: 299, category: 'Furniture' }
]

const ProductStore = () => {
  const [category, setCategory] = useState('all')
  const [search, setSearch] = useState('')

  // useMemo: only recompute filtered list when category or search changes
  // Without this, filtering runs on every render even if inputs haven't changed
  const filteredProducts = useMemo(() => {
    return allProducts.filter(product => {
      const matchesCategory = category === 'all' || product.category === category
      const matchesSearch = product.name.toLowerCase().includes(search.toLowerCase())
      return matchesCategory && matchesSearch
    })
  }, [category, search])

  // useCallback: stable function reference across renders
  // Without this, memo() on ProductList is useless—new function = new props
  const handleProductClick = useCallback((product: Product) => {
    console.log('Clicked:', product.name)
  }, [])

  return (
    <div className="p-6">
      <div className="flex gap-4 mb-6">
        <input
          type="text"
          placeholder="Search..."
          value={search}
          onChange={(e) => setSearch(e.target.value)}
          className="flex-1 px-4 py-2 border rounded-lg"
        />
        <select
          value={category}
          onChange={(e) => setCategory(e.target.value)}
          className="px-4 py-2 border rounded-lg"
        >
          <option value="all">All</option>
          <option value="Electronics">Electronics</option>
          <option value="Furniture">Furniture</option>
        </select>
      </div>

      {/* Both props are now stable references, so memo() actually works */}
      <ProductList products={filteredProducts} onProductClick={handleProductClick} />
    </div>
  )
}
```

El hook `useMemo` almacena en caché el resultado de la operación de filtrado, recalculando solo cuando cambia `category` o `search`. Esto no evita que `ProductList` se vuelva a renderizar cuando escribes; si el término de búsqueda cambia, `useMemo` recalcula el arreglo filtrado, devuelve una nueva referencia y `ProductList` se vuelve a renderizar con los nuevos resultados. Ese es el comportamiento correcto. Lo que `useMemo` evita es ejecutar la lógica de filtrado cuando cambia un estado no relacionado. Si este componente tuviera otro estado, digamos, la alternancia de un modal abierto/cerrado, el filtro normalmente se recalcularía en cada renderizado aunque sus entradas no hayan cambiado. Con `useMemo`, React omite el cálculo y devuelve el arreglo almacenado en caché.

El hook `useCallback` sirve para un propósito diferente: mantiene estable la referencia de la función a través de los renderizados. Sin él, `memo()` en `ProductList` sería inútil; se crea una nueva función en cada renderizado, `memo()` ve props diferentes y el componente se vuelve a renderizar de todos modos. Juntos, `useMemo` y `useCallback` aseguran que `ProductList` solo se vuelva a renderizar cuando los datos reales cambian, no cuando se actualizan partes no relacionadas del padre.

### Gestión eficiente de grandes conjuntos de datos con virtualización

Cuando se trabaja con listas grandes (cientos o miles de elementos), renderizar cada elemento a la vez puede paralizar la aplicación. La solución es la virtualización: renderizar solo los elementos que son visibles actualmente en la ventana gráfica (*viewport*).

La virtualización reduce drásticamente el número de nodos DOM en tu aplicación. En lugar de renderizar 10,000 elementos de lista, es posible que solo renderices 20, los que el usuario realmente puede ver. A medida que el usuario se desplaza, la librería calcula qué elementos deberían ser visibles y renderiza solo esos, desmontando los elementos que salen de la vista. La mayoría de las librerías de virtualización también admiten una configuración de sobreexploración (*overscan*), renderizando algunos elementos adicionales por encima y por debajo del área visible para que el contenido esté listo antes de que el usuario se desplace hacia él, lo que reduce el parpadeo en el desplazamiento rápido:

```tsx
import { useState, useCallback, useMemo, useRef, useEffect } from 'react'

// Generic type T allows this component to work with any data type
// VirtualList<Message> renders messages, VirtualList<Product> renders products

interface VirtualListProps<T> {
  items: T[]
  itemHeight: number
  containerHeight: number
  overscan?: number
  // renderItem receives the typed item, ensuring type safety in the callback
  renderItem: (item: T, index: number) => React.ReactNode
}

// The <T> after function name makes this a generic function component
// TypeScript infers T from the items array you pass in
function VirtualList<T>({ 
  items,
  itemHeight,
  containerHeight,
  overscan = 3,
  renderItem
}: VirtualListProps<T>) {
  const [scrollTop, setScrollTop] = useState(0)
  const rafId = useRef<number | null>(null)

  // Cancel any pending animation frame on unmount
  // Prevents "setState on unmounted component" warnings
  useEffect(() => {
    return () => {
      if (rafId.current) {
        cancelAnimationFrame(rafId.current)
      }
    }
  }, [])

  // Throttle scroll updates to animation frames
  // Without this, scroll fires 60-120 times/second, each triggering a state update
  const handleScroll = useCallback((e: React.UIEvent<HTMLDivElement>) => {
    const target = e.currentTarget
    if (rafId.current) cancelAnimationFrame(rafId.current)
    rafId.current = requestAnimationFrame(() => setScrollTop(target.scrollTop))
  }, [])

  const { visibleItems, offsetY, totalHeight } = useMemo(() => {
    const start = Math.max(0, Math.floor(scrollTop / itemHeight) - overscan)
    const end = Math.min(items.length, Math.ceil((scrollTop + containerHeight) / itemHeight) + overscan)
   
    return {
      visibleItems: items.slice(start, end).map((item, i) => ({ item, index: start + i })), 
      offsetY: start * itemHeight,
      totalHeight: items.length * itemHeight
    }
  }, [items, itemHeight, containerHeight, scrollTop, overscan])

  return (
    <div onScroll={handleScroll} style={{ height: containerHeight, overflow: 'auto' }}>
      <div style={{ height: totalHeight, position: 'relative' }}>
        <div style={{ transform: `translateY(${offsetY}px)` }}>
          {visibleItems.map(({ item, index }) => (
            <div key={index} style={{ height: itemHeight }}>
              {renderItem(item, index)}
            </div>
          ))}
        </div>
      </div>
    </div>
  )
}

interface Message {
  id: string
  sender: string
  content: string
}

const MessageList: React.FC = () => {
  const messages = useMemo<Message[]>(() =>
    Array.from({ length: 10000 }, (_, i) => ({
      id: `msg-${i}`,
      sender: `User ${i % 50}`,
      content: `Message content ${i} - Sample text for virtualization demo.`
    })), [])

  const renderMessage = useCallback((message: Message) => (
    <div className="p-3 border-b border-gray-100">
      <span className="font-medium">{message.sender}</span>
      <p className="text-gray-600">{message.content}</p>
    </div>
  ), [])

  return (
    <div className="max-w-2xl mx-auto p-4">
      <h2 className="text-xl font-bold mb-4">Messages (10,000 items)</h2>
      <VirtualList
        items={messages}
        itemHeight={72}
        containerHeight={400}
        overscan={5}
        renderItem={renderMessage}
      />
    </div>
  )
}
```

La limitación de `requestAnimationFrame` asegura que las actualizaciones de desplazamiento se alineen con el ciclo de repintado del navegador, como máximo una actualización de estado por fotograma. La prop `overscan` renderiza elementos adicionales más allá del área visible para evitar destellos en blanco durante el desplazamiento rápido. Para uso en producción, librerías como `@tanstack/react-virtual` manejan casos extremos adicionales como elementos de altura variable y desplazamiento horizontal.

### Carga diferida de componentes y división de código con React Suspense

Tu aplicación no necesita cargar todo a la vez. La división de código (*code splitting*) divide tu bundle de JavaScript en fragmentos más pequeños que se cargan bajo demanda. Antes de implementar la división manual de código, debes saber que los frameworks modernos como Next.js, Remix y React Router v7 manejan la división basada en rutas automáticamente de forma gratuita.

La carga diferida (*lazy loading*) manual tiene sentido para componentes que son genuinamente pesados (más de 100 KB después de la minificación), se renderizan condicionalmente y no se necesitan en la carga inicial de la página. Los ejemplos comunes incluyen editores de texto enriquecido, librerías de gráficos o visores de PDF. Para componentes más pequeños, la sobrecarga de una solicitud de red adicional puede superar los ahorros.

El patrón básico utiliza `lazy()` con `Suspense`, pero este enfoque ingenuo tiene problemas:

```tsx
import { lazy, Suspense, useState } from 'react'

// Components load only when rendered—but this creates a network waterfall
const HeavyChart = lazy(() => import('./HeavyChart'))
const HeavyDataTable = lazy(() => import('./HeavyDataTable'))

const Analytics = () => {
  const [activeTab, setActiveTab] = useState<'overview' | 'chart' | 'data'>('overview')

  return (
    <div className="p-6">
      <div className="flex gap-4 mb-6">
        <button onClick={() => setActiveTab('overview')}>Overview</button>
        <button onClick={() => setActiveTab('chart')}>Charts</button>
        <button onClick={() => setActiveTab('data')}>Data</button>
      </div>

      {/* Problem: switching tabs unmounts components, losing all state */}
      {/* Problem: spinner shows even for fast loads, causing flash */}
      <Suspense fallback={<div>Loading...</div>}>
{activeTab==='chart'&&<HeavyChart />}
{activeTab==='data'&&<HeavyDataTable />}
      </Suspense>
    </div>
  )
}
```

Un mejor enfoque realiza la precarga (*prefetch*) al pasar el cursor para que el contenido esté listo cuando se haga clic:

```tsx
import { lazy, Suspense, useState, useTransition } from 'react'

const HeavyChart = lazy(() => import('./HeavyChart'))
const HeavyDataTable = lazy(() => import('./HeavyDataTable'))

// Prefetch functions—call these on hover
const prefetchChart = () => import('./HeavyChart')
const prefetchDataTable = () => import('./HeavyDataTable')

const Analytics = () => {
  const [activeTab, setActiveTab] = useState<'overview' | 'chart' | 'data'>('overview')
  // useTransition prevents the UI from showing fallback for fast loads
  const [isPending, startTransition] = useTransition()

  const handleTabChange = (tab: typeof activeTab) => {
    startTransition(() => setActiveTab(tab))
  }

  return (
    <div className="p-6">
      <div className="flex gap-4 mb-6">
        <button onClick={() => handleTabChange('overview')}>
          Overview
        </button>
        <button
          onClick={() => handleTabChange('chart')}
          onMouseEnter={prefetchChart}
        >
          Charts
        </button>
        <button
          onClick={() => handleTabChange('data')}
          onMouseEnter={prefetchDataTable}
        >
          Data
        </button>
      </div>

      <div className={isPending ? 'opacity-50' : ''}>
        <Suspense fallback={<div>Loading...</div>}>
{activeTab==='overview'&&<div>Welcome to Analytics</div>}
{activeTab==='chart'&&<HeavyChart />}
{activeTab==='data'&&<HeavyDataTable />}
        </Suspense>
      </div>
    </div>
  )
}
```

#### El nuevo componente `<Activity>`

El enfoque de precarga todavía tiene una limitación: cambiar de pestaña desmonta el componente anterior, perdiendo cualquier estado interno como entradas de formulario, posición de desplazamiento o configuraciones de zoom de gráficos. React 19 introduce el componente `<Activity>`, que mantiene los componentes montados pero ocultos, preservando su estado:

```tsx
import { lazy, Suspense, useState, useTransition, Activity } from 'react'

const HeavyChart = lazy(() => import('./HeavyChart'))
const HeavyDataTable = lazy(() => import('./HeavyDataTable'))
const prefetchChart = () => import('./HeavyChart')
const prefetchDataTable = () => import('./HeavyDataTable')

const Analytics = () => {
  const [activeTab, setActiveTab] = useState<'overview' | 'chart' | 'data'>('overview')
  const [isPending, startTransition] = useTransition()

  const handleTabChange = (tab: typeof activeTab) => {
    startTransition(() => setActiveTab(tab))
  }

  return (
    <div className="p-6">
      <div className="flex gap-4 mb-6">
        <button onClick={() => handleTabChange('overview')}>Overview</button>
        <button onClick={() => handleTabChange('chart')} onMouseEnter={prefetchChart}>
          Charts
        </button>
        <button onClick={() => handleTabChange('data')} onMouseEnter={prefetchDataTable}>
          Data
        </button>
      </div>
      <div className={isPending ? 'opacity-50' : ''}>
        <Suspense fallback={<div>Loading...</div>}>
          {/* Activity keeps components mounted but hidden when mode="hidden" */}
          {/* All state is preserved: form inputs, scroll position, selections */}
          <Activity mode={activeTab === 'overview' ? 'visible' : 'hidden'}>
            <div>Welcome to Analytics</div>
          </Activity>
          <Activity mode={activeTab === 'chart' ? 'visible' : 'hidden'}>
            <HeavyChart />
          </Activity>
          <Activity mode={activeTab === 'data' ? 'visible' : 'hidden'}>
            <HeavyDataTable />
          </Activity>
        </Suspense>
      </div>
    </div>
  )
}
```

Con `<Activity>`, cuando cambias de la pestaña de gráficos a la de datos, el componente de gráficos permanece montado pero entra en un estado oculto. Esto no es solo CSS `display: none`; React trata los componentes ocultos de manera diferente a nivel del programador (*scheduler*). Los efectos se posponen hasta que el componente vuelve a ser visible, por lo que un componente oculto no realizará solicitudes de red ni configurará suscripciones mientras esté fuera de la pantalla. La prioridad de actualización se reduce, lo que significa que React desprioriza los re-renderizados para el contenido oculto en favor de la interfaz visible. El estado del componente (nivel de zoom, puntos de datos seleccionados, entradas de formulario) permanece intacto, pero React evita desperdiciar recursos en contenido que el usuario no puede ver. Cuando regresas, el componente se reactiva instantáneamente sin volver a montarse ni volver a obtener datos. Su estado permanece intacto. La desventaja es un mayor uso de memoria, ya que todas las pestañas visitadas permanecen en la memoria, pero para componentes interactivos complejos, el estado preservado proporciona una experiencia de usuario significativamente mejor.

### Optimización del manejo de eventos y listeners

Los manejadores de eventos pueden ser una fuente oculta de problemas de rendimiento, especialmente cuando se trata de eventos de alta frecuencia como desplazamientos (*scrolling*), cambios de tamaño (*resizing*) o movimientos del ratón. Cada vez que se activan estos eventos, se ejecuta tu manejador, y si ese manejador es costoso, notarás un comportamiento entrecortado y no receptivo.

El *debouncing* (antirrebote) y el *throttling* (regulación) son dos técnicas para controlar con qué frecuencia se ejecuta una función:
- El **debouncing** espera hasta que los eventos dejen de activarse antes de ejecutar el manejador.
- El **throttling** asegura que el manejador se ejecute como máximo una vez por período de tiempo especificado.

Veamos el debouncing y el throttling por separado. El debouncing retrasa la ejecución hasta que la actividad se detiene, ideal para entradas de búsqueda donde deseas el valor final, no las pulsaciones intermedias:

```tsx
import { useState, useEffect } from 'react'

function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value)

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value)
    }, delay)

    // Cancel and restart if value changes before delay completes
    return () => clearTimeout(handler)
  }, [value, delay])

  return debouncedValue
}

function SearchInput() {
  const [searchTerm, setSearchTerm] = useState('')
  const debouncedSearch = useDebounce(searchTerm, 500)

  useEffect(() => {
    if (debouncedSearch) {
      console.log('Searching for:', debouncedSearch)
      // API call goes here
    }
  }, [debouncedSearch])

  return (
    <input
      value={searchTerm}
      onChange={(e) => setSearchTerm(e.target.value)}
      placeholder="Type to search..."
    />
  )
}
```

Para la búsqueda específicamente, React 18+ ofrece `useDeferredValue` como una alternativa integrada. Logra resultados similares sin temporizadores manuales; React difiere automáticamente las actualizaciones durante interacciones urgentes:

```tsx
import { useState, useDeferredValue, useMemo } from 'react'

function SearchWithDeferred() {
  const [searchTerm, setSearchTerm] = useState('')
  // React defers this value during rapid typing
  const deferredSearch = useDeferredValue(searchTerm)
 
  // Expensive filtering only runs with deferred value
  const results = useMemo(() => {
    return items.filter(item =>
      item.name.toLowerCase().includes(deferredSearch.toLowerCase())
    )
  }, [deferredSearch])

  return (
    <div>
      <input
        value={searchTerm}
        onChange={(e) => setSearchTerm(e.target.value)}
      />
      {/* Show stale indicator while deferred value catches up */}
      <div style={{ opacity: searchTerm !== deferredSearch ? 0.5 : 1 }}>
        {results.map(item => <div key={item.id}>{item.name}</div>)}
      </div>
    </div>
  )
}
```

El throttling limita la tasa de ejecución, ideal para eventos continuos como el desplazamiento o el movimiento del ratón donde necesitas actualizaciones pero no 60 por segundo:

```tsx
import { useRef, useCallback, useEffect } from 'react'

// This implementation includes "trailing edge" -
// ensures the final value is captured even if events stop
function useThrottle<T extends (...args: any[]) => any>(
  callback: T,
  delay: number
) {
  const lastRun = useRef(Date.now())
  const timeoutRef = useRef<NodeJS.Timeout | null>(null)
  const lastArgs = useRef<Parameters<T> | null>(null)

  // Cleanup on unmount
  useEffect(() => {
    return () => {
      if (timeoutRef.current) clearTimeout(timeoutRef.current)
    }
  }, [])

  return useCallback((...args: Parameters<T>) => {
    const now = Date.now()
    lastArgs.current = args

    if (now - lastRun.current >= delay) {
      // Enough time passed - run immediately
      callback(...args)
      lastRun.current = now
    } else {
      // Schedule trailing edge call to capture final value
      if (timeoutRef.current) clearTimeout(timeoutRef.current)
      timeoutRef.current = setTimeout(() => {
        if (lastArgs.current) {
          callback(...lastArgs.current)
          lastRun.current = Date.now()
        }
      }, delay - (now - lastRun.current))
    }
  }, [callback, delay])
}

function MouseTracker() {
  const [position, setPosition] = useState({ x: 0, y: 0 })

  const handleMouseMove = useThrottle((e: React.MouseEvent) => {
    setPosition({ x: e.clientX, y: e.clientY })
  }, 100)

  return (
    <div onMouseMove={handleMouseMove} className="h-64 bg-gray-100">
      Position: ({position.x}, {position.y})
    </div>
  )
}
```

La implementación de throttle incluye un flanco final (*trailing edge*). Cuando los eventos se detienen, se dispara una última vez con el último valor capturado. Sin esto, el movimiento rápido del ratón que se detiene repentinamente perdería la posición final. El debouncing es para escenarios de "esperar hasta que termine"; el throttling es para "ejecutar regularmente pero no demasiado a menudo".

---

## Herramientas y librerías para la monitorización del rendimiento

La optimización sin medición es una conjetura. React proporciona varias herramientas para identificar cuellos de botella en el rendimiento, y comprender cómo utilizarlas eficazmente es esencial para crear aplicaciones rápidas.

### Uso de React Profiler para identificar cuellos de botella

La API de React Profiler te permite medir cuánto tiempo tardan los componentes en renderizarse y por qué se vuelven a renderizar. Esta información es invaluable al buscar problemas de rendimiento en aplicaciones complejas:

```tsx
import { Profiler, ProfilerOnRenderCallback, useState } from 'react'

interface ProfilerData {
  id: string
  phase: 'mount' | 'update'
  actualDuration: number
  baseDuration: number
  startTime: number
  commitTime: number
}

// Simulates an expensive computation to demonstrate profiling
// In real apps, this might be complex data transformations or renders
const ExpensiveComponent = ({ iterations }: { iterations: number }) => {
  const result = Array.from({ length: iterations }, (_, i) => {
    return Array.from({ length: 100 }, (_, j) => i * j).reduce((a, b) => a + b, 0)
  })

  return (
    <div className="p-4 bg-gray-100 rounded">
      <p>Performed {iterations} expensive calculations</p>
      <p>Result sum: {result.reduce((a, b) => a + b, 0)}</p>
    </div>
  )
}

const PerformanceMonitor = () => {
  const [iterations, setIterations] = useState(100)
  const [profileData, setProfileData] = useState<ProfilerData[]>([])

  // Profiler callback fires after every commit (render to DOM)
  // Use this data to identify slow components
  const onRenderCallback: ProfilerOnRenderCallback = (
    id,              // The "id" prop of the Profiler tree
    phase,           // "mount" (first render) or "update" (re-render)
    actualDuration,  // Time spent rendering this update (ms)
    baseDuration,    // Estimated time to render without memoization (ms)
    startTime,       // When React began rendering this update
    commitTime       // When React committed this update
  ) => {
    const data: ProfilerData = {
      id, phase, actualDuration, baseDuration, startTime, commitTime
    }
    // Keep only the last 10 renders to avoid memory bloat
    setProfileData(prev => [...prev.slice(-9), data])
  }

  return (
    <div className="p-6 max-w-2xl mx-auto">
      <h1 className="text-2xl font-bold mb-4">React Profiler Demo</h1>
     
      <div className="mb-6">
        <label className="block mb-2">
          Complexity: {iterations} iterations
        </label>
        <input
          type="range"
          min="100"
          max="5000"
          value={iterations}
          onChange={(e) => setIterations(Number(e.target.value))}
          className="w-full"
        />
      </div>

      {/* Profiler wraps components you want to measure */}
      {/* It has no visual output—purely for measurement */}
      <Profiler id="expensive-component" onRender={onRenderCallback}>
        <ExpensiveComponent iterations={iterations} />
      </Profiler>

      <div className="mt-6">
        <h2 className="font-semibold mb-2">Render Performance (last 10 renders)</h2>
        <div className="space-y-1">
          {profileData.map((data, index) => (
            <div key={index} className="flex justify-between text-sm">
              <span>{data.phase}</span>
              {/* Red if over 16ms (below 60 FPS threshold) */}
              <span className={`font-mono ${
                data.actualDuration > 16 ? 'text-red-600' : 'text-green-600'
              }`}>
                {data.actualDuration.toFixed(2)}ms
              </span>
            </div>
          ))}
        </div>
        {/* 60 FPS = 16.67ms per frame budget */}
        {/* If render exceeds this, the UI may feel sluggish */}
        <p className="text-xs text-gray-500 mt-2">
          Target: <16ms per render for 60 FPS
        </p>
      </div>
    </div>
  )
}
```

El Profiler realiza un seguimiento de la duración de cada renderizado, lo que te ayuda a identificar los componentes que tardan demasiado.

Si bien 16 milisegundos es el presupuesto total de fotograma a 60 FPS, el renderizado de React es solo una parte de lo que sucede en un fotograma. Después de que React aplica los cambios, el navegador aún necesita tiempo para los cálculos de diseño, la pintura y la composición. En la práctica, apunta a tiempos de renderizado de React por debajo de 10 ms para dejar margen para el trabajo del navegador. Los renderizados constantemente superiores a 10 ms justifican una investigación; o bien el componente está haciendo demasiado trabajo, o bien se está volviendo a renderizar con más frecuencia de la necesaria.

### Monitorización del rendimiento con Lighthouse y Web Vitals

Mientras que el Profiler de React se centra en el rendimiento a nivel de componente, Web Vitals mide la experiencia del usuario de toda tu aplicación. Las Core Web Vitals son tres métricas que se correlacionan directamente con cómo los usuarios perciben la velocidad y la capacidad de respuesta de tu aplicación:

- **LCP** mide el rendimiento de carga: cuánto tiempo tarda en renderizarse el elemento de contenido visible más grande. Objetivo: menos de 2.5 segundos.
- **INP** mide la capacidad de respuesta: la latencia de todas las interacciones del usuario a lo largo del ciclo de vida de la página, informada como la peor interacción (en el percentil 98). Objetivo: menos de 200 ms. INP reemplazó a First Input Delay (FID) en marzo de 2024 porque FID solo medía la primera interacción, pasando por alto respuestas lentas más adelante en la sesión.
- **CLS** mide la estabilidad visual: cuánto cambia inesperadamente el diseño de la página durante la carga. Objetivo: menos de 0.1. Un CLS alto a menudo proviene de imágenes sin dimensiones, contenido inyectado dinámicamente o fuentes web que provocan reflujo de texto.

La librería `web-vitals` maneja la complejidad de medir estas métricas correctamente:

```typescript
// First install: npm install web-vitals

import { useEffect } from 'react'
import { onLCP, onINP, onCLS, type Metric } from 'web-vitals'

// Report metrics to your analytics service
function sendToAnalytics(metric: Metric) {
  // Example: send to Google Analytics
  // gtag('event', metric.name, {
  //   value: Math.round(metric.value),
  //   metric_id: metric.id,
  //   metric_rating: metric.rating,
  // })

  // Example: send to custom endpoint
  fetch('/api/analytics', {
    method: 'POST',
    body: JSON.stringify({
      name: metric.name,
      value: metric.value,
      rating: metric.rating, // 'good' | 'needs-improvement' | 'poor'
      navigationType: metric.navigationType,
    }),
    headers: { 'Content-Type': 'application/json' },
  })
}

// Call this once at app initialization, not inside a component
export function initWebVitals() {
  // LCP: Largest Contentful Paint - measures loading performance
  // Reports when the largest image or text block becomes visible
  onLCP(sendToAnalytics)

  // INP: Interaction to Next Paint - measures responsiveness
  // Reports the worst interaction latency during the session
  // Replaced FID in March 2024
  onINP(sendToAnalytics)

  // CLS: Cumulative Layout Shift - measures visual stability
  // Reports how much the layout shifts unexpectedly
  onCLS(sendToAnalytics)
}

// If you need to trigger measurement in a component (e.g., for SPAs)
export function useWebVitals() {
  useEffect(() => {
    // For SPAs, you may want to report on route changes
    // The library handles this automatically with reportAllChanges option
    onLCP(sendToAnalytics, { reportAllChanges: true })
    onINP(sendToAnalytics, { reportAllChanges: true })
    onCLS(sendToAnalytics, { reportAllChanges: true })
  }, [])
}

// In your app entry point (e.g., main.tsx or _app.tsx)
import { initWebVitals } from './webVitals'

// Initialize once, outside React's render cycle
initWebVitals()

// Or in Next.js, use the built-in reportWebVitals:
// export function reportWebVitals(metric: Metric) {
//   sendToAnalytics(metric)
// }
```

LCP y CLS pueden cambiar a lo largo del ciclo de vida de la página, e INP rastrea todas las interacciones para informar la peor. El campo `rating` te indica si el valor es bueno, necesita mejorar o es deficiente según los umbrales de Google. Para las aplicaciones de Next.js, el framework proporciona una exportación integrada `reportWebVitals` que simplifica esto aún más.

### Análisis de actualizaciones de componentes

A veces, los componentes se vuelven a renderizar y no estás seguro de por qué. La librería `why-did-you-render` parchea React para notificarte sobre re-renderizados potencialmente innecesarios, ayudándote a rastrear props que están cambiando inesperadamente:

```tsx
import { useState, useCallback, useMemo, memo } from 'react'

interface User {
  id: number
  name: string
  email: string
}

interface UserCardProps {
  user: User
  onSelect: (id: number) => void
  settings: { showEmail: boolean; theme: string }
}

const UserCard = memo(({ user, onSelect, settings }: UserCardProps) => {
  console.log(`UserCard rendered for ${user.name}`)

  return (
    <div className="p-4 bg-white rounded-lg shadow">
      <h3 className="font-medium">{user.name}</h3>
      {settings.showEmail && (
        <p className="text-gray-600 text-sm">{user.email}</p>
      )}
      <button
        onClick={() => onSelect(user.id)}
        className="mt-3 px-4 py-2 bg-blue-600 text-white rounded text-sm"
      >
        Select
      </button>
    </div>
  )
})

const users: User[] = [
  { id: 1, name: 'Alice Johnson', email: 'alice@example.com' },
  { id: 2, name: 'Bob Smith', email: 'bob@example.com' },
  { id: 3, name: 'Carol Williams', email: 'carol@example.com' }
]

const UserList = () => {
  const [selectedId, setSelectedId] = useState<number | null>(null)

  //  Problem: new function reference on every render breaks memo()
  // const handleSelect = (id: number) => setSelectedId(id)

  //  Fix: useCallback keeps the same reference across renders
  const handleSelect = useCallback((id: number) => {
    setSelectedId(id)
  }, [])

  //  Problem: new object reference on every render breaks memo()
  // const settings = { showEmail: true, theme: 'light' }

  //  Fix: useMemo keeps the same reference when values don't change
  const settings = useMemo(() => ({
    showEmail: true,
    theme: 'light'
  }), [])

  return (
    <div className="p-6">
      <h1 className="text-2xl font-bold mb-4">User Directory</h1>
      <div className="grid gap-4">
        {users.map(user => (
          <UserCard
            key={user.id}
            user={user}
            onSelect={handleSelect}
            settings={settings}
          />
        ))}
      </div>
      {selectedId && (
        <p className="mt-4 text-gray-600">Selected user ID: {selectedId}</p>
      )}
    </div>
  )
}
```

Para identificar estos problemas tú mismo, usa React DevTools Profiler. Abre DevTools, ve a la pestaña Profiler, haz clic en grabar, interactúa con tu aplicación y luego detén la grabación. El gráfico de llamas (*flame graph*) muestra qué componentes se renderizaron y por qué. Haz clic en cualquier componente para ver "¿Por qué se renderizó esto?"; te indica si las props cambiaron, el estado cambió o el padre se volvió a renderizar. Para los componentes envueltos en `memo()`, muestra específicamente qué props tenían nuevas referencias incluso si los valores eran idénticos. Esto está integrado en React DevTools, no se necesitan librerías adicionales.

### Optimización del tamaño del bundle

Cada kilobyte que envías a los usuarios les cuesta tiempo y ancho de banda. La optimización del tamaño del paquete no se trata de obsesionarse con los bytes, se trata de respetar los recursos de tus usuarios y garantizar que tu aplicación se cargue rápidamente, especialmente en conexiones más lentas o dispositivos menos potentes. El JavaScript que envías debe analizarse, compilarse y ejecutarse antes de que los usuarios puedan interactuar con tu aplicación, lo que hace que el tamaño del paquete sea un factor directo en el tiempo hasta la interactividad (*time-to-interactive*).

#### Análisis de tu paquete

Antes de que puedas optimizar, necesitas entender qué hay realmente en tu paquete. Herramientas como `webpack-bundle-analyzer` o el analizador integrado de Next.js revelan verdades sorprendentes sobre tus dependencias. Esa librería ligera que instalaste podría estar trayendo megabytes de código que nunca usas:

```typescript
// next.config.ts - Enable bundle analysis in Next.js
import type { NextConfig } from 'next'
import bundleAnalyzer from '@next/bundle-analyzer'

const withBundleAnalyzer = bundleAnalyzer({
  enabled: process.env.ANALYZE === 'true'
})

const config: NextConfig = {
  // Your existing config
}

export default withBundleAnalyzer(config)

// Run with: ANALYZE=true npm run build
```

#### Tree shaking e importaciones nombradas

Los empaquetadores modernos pueden eliminar el código no utilizado mediante *tree shaking*, pero solo si les das la oportunidad. La diferencia entre importar una librería completa en comparación con funciones específicas puede ser dramática:

```typescript
// Bad: Imports entire lodash library (~70KB minified)
import _ from 'lodash'
const sorted = _.sortBy(users, 'name')

// Good: Imports only what you need (~2KB)
import sortBy from 'lodash/sortBy'
const sorted = sortBy(users, 'name')

// Better: Use native methods when possible (0KB added)
const sorted = [...users].sort((a, b) => a.name.localeCompare(b.name))
```

---

## Rendimiento de hidratación

La hidratación es el proceso mediante el cual React toma el control del HTML renderizado en el servidor y lo hace interactivo. Durante la hidratación, React reconstruye su representación interna del árbol de componentes, adjunta listeners de eventos y configura la gestión de estado, todo mientras se asegura de que el DOM existente coincida con lo que React habría renderizado. Un rendimiento deficiente de la hidratación puede hacer que tu aplicación no responda durante los primeros segundos críticos después de la carga.

### Comprensión de los costos de hidratación

La hidratación no es gratuita. React debe recorrer todo tu árbol de componentes, ejecutar cada función de componente y reconciliarse con el DOM existente. Cuantos más componentes tengas, más tiempo tomará esto. Los usuarios ven tu contenido pero no pueden interactuar con él (los botones no responden, los formularios no funcionan) hasta que se completa la hidratación:

```tsx
// components/HydrationMonitor.tsx - Measure hydration time
'use client'

import { useEffect, useState } from 'react'

interface HydrationMetrics {
  hydrationTime: number;
  componentCount: number;
}

export const HydrationMonitor = ({ children }: { children: React.ReactNode }) => {
  const [metrics, setMetrics] = useState<HydrationMetrics | null>(null)

  useEffect(() => {
    // This runs after hydration completes
    const hydrationEnd = performance.now()
   
    // Get the navigation timing
    const navEntry = performance.getEntriesByType('navigation')[0] as PerformanceNavigationTiming
    const hydrationStart = navEntry?.domContentLoadedEventStart ?? 0
   
    // Count hydrated components (approximate via DOM query)
    const componentCount = document.querySelectorAll('[data-hydrated]').length

    setMetrics({
      hydrationTime: Math.round(hydrationEnd - hydrationStart),
      componentCount,
    })

    // Report to analytics
    if (process.env.NODE_ENV === 'production') {
      // sendToAnalytics('hydration', { hydrationTime, componentCount })
    }
  }, [])

  if (process.env.NODE_ENV === 'development' && metrics) {
    console.log(`Hydration completed in ${metrics.hydrationTime}ms`)
  }

  return <>{children}</>
}
```

### Hidratación selectiva con Suspense

La hidratación selectiva de React permite que diferentes partes de tu página se hidraten de forma independiente. Los componentes envueltos en límites de Suspense pueden hidratarse en paralelo, y React prioriza la hidratación de los componentes con los que el usuario intenta interactuar:

```tsx
// app/page.tsx - Selective hydration with lazy loading
import { Suspense, lazy } from 'react'

// Static content - ships as Server Components, no hydration needed
import { Header } from './Header'
import { HeroSection } from './HeroSection'

// Interactive components - lazy loaded for selective hydration
// Without lazy(), these would bundle together and hydrate simultaneously
const SearchBar = lazy(() => import('./SearchBar'))
const ProductCarousel = lazy(() => import('./ProductCarousel'))
const NewsletterForm = lazy(() => import('./NewsletterForm'))

export default function HomePage() {
  return (
    <>
      <Header />
      <HeroSection />
      {/* Critical interaction - hydrates first when user interacts */}
      <Suspense fallback={<SearchBarSkeleton />}>
        <SearchBar />
      </Suspense>

      {/* Below the fold - hydrates when scrolled into view or idle */}
      <Suspense fallback={<CarouselSkeleton />}>
        <ProductCarousel />
      </Suspense>

      {/* Footer content - lowest priority */}
      <Suspense fallback={<FormSkeleton />}>
        <NewsletterForm />
      </Suspense>
    </>
  )
}
function SearchBarSkeleton() {
  return <div className="h-12 bg-gray-200 rounded animate-pulse" />
}

function CarouselSkeleton() {
  return <div className="h-64 bg-gray-200 rounded animate-pulse" />
}

function FormSkeleton() {
  return <div className="h-32 bg-gray-200 rounded animate-pulse" />
}
```

### Cómo evitar desajustes de hidratación (*Hydration mismatches*)

Los desajustes de hidratación ocurren cuando el HTML renderizado en el servidor no coincide con lo que React espera en el cliente. Esto hace que React descarte el HTML del servidor y vuelva a renderizar desde cero, anulando los beneficios de SSR y, a veces, provocando parpadeos visibles:

```tsx
// Bad: Causes hydration mismatch
const BadTimestamp = () => {
  // Different value on server vs client!
  return <span>{new Date().toLocaleTimeString()}</span>
}

// Bad: Random values differ between server and client
const BadRandomGreeting = () => {
  const greetings = ['Hello', 'Hi', 'Welcome']
  const greeting = greetings[Math.floor(Math.random() * greetings.length)]
  return <h1>{greeting}</h1>
}

// Good: Consistent initial render, update on client
const GoodTimestamp = () => {
  const [time, setTime] = useState<string | null>(null)
 
  useEffect(() => {
    setTime(new Date().toLocaleTimeString())
    const interval = setInterval(() => {
      setTime(new Date().toLocaleTimeString())
    }, 1000)
    return () => clearInterval(interval)
  }, [])
 
  // Render nothing or placeholder during SSR/hydration
  if (!time) return <span className="text-gray-400">--:--:--</span>
  return <span>{time}</span>
}

// Good: Use suppressHydrationWarning for intentional differences
const GoodCurrentYear = () => {
  return (
    <span suppressHydrationWarning>
      © {new Date().getFullYear()} Company Name
    </span>
  )
}
```

### Optimización de la complejidad de los componentes para la hidratación

Los componentes complejos con anidamiento profundo y muchos hijos ralentizan la hidratación. Aplanar la estructura de tus componentes y reducir los componentes envolventes innecesarios mejora el rendimiento de la hidratación:

```tsx
// Bad: Deeply nested structure
const BadProductCard = ({ product }: { product: Product }) => (
  <div>
    <div>
      <div>
        <div>
          <img src={product.image} alt={product.name} />
        </div>
      </div>
      <div>
        <div>
          <h3>{product.name}</h3>
        </div>
        <div>
          <span>{product.price}</span>
        </div>
      </div>
    </div>
  </div>
)

// Good: Flat structure with semantic HTML
const GoodProductCard = ({ product }: { product: Product }) => (
  <article className="flex gap-4 p-4 bg-white rounded-lg shadow">
    <img
      src={product.image}
      alt={product.name}
      className="w-24 h-24 object-cover rounded"
    />
    <div>
      <h3 className="font-semibold text-lg">{product.name}</h3>
      <span className="text-xl font-bold text-blue-600">${product.price}</span>
    </div>
  </article>
)
```

### Arquitectura de islas para una hidratación mínima

La arquitectura de islas lleva la hidratación selectiva a su extremo: solo los componentes interactivos se hidratan, mientras que el contenido estático permanece como HTML sin procesar. Esto reduce drásticamente el tiempo de ejecución de JavaScript en páginas con contenido principalmente estático:

```tsx
// components/Island.tsx - Client-side island wrapper
'use client'

import { useState, useEffect, ReactNode } from 'react'

interface IslandProps {
  children: ReactNode;
  load: 'eager' | 'lazy' | 'visible';
}

export const Island = ({ children, load }: IslandProps) => {
  const [isActive, setIsActive] = useState(load === 'eager')

  useEffect(() => {
    if (load === 'lazy') {
      const id = requestIdleCallback(() => setIsActive(true))
      return () => cancelIdleCallback(id)
    }
  }, [load])

  // For 'visible', use IntersectionObserver (omitted for brevity)
  if (!isActive) {
    // Return static placeholder that matches server HTML
    return <div data-island-placeholder="true">{children}</div>
  }

  return <>{children}</>
}

// Page with islands
export default function BlogPost() {
  return (
    <article>
      {/* Static content - no JavaScript needed */}
      <h1>Blog Post Title</h1>
      <div className="prose">{/* Article content */}</div>
     
      {/* Interactive island - hydrates eagerly */}
      <Island load="eager">
        <LikeButton postId="123" />
      </Island>
     
      {/* Interactive island - hydrates when visible */}
      <Island load="visible">
        <CommentSection postId="123" />
      </Island>
    </article>
  )
}
```

El rendimiento de la hidratación a menudo se pasa por alto porque los efectos son sutiles: una breve falta de respuesta, un ligero retraso antes de que funcionen las interacciones. Pero estos momentos definen la primera impresión del usuario sobre tu aplicación. Al comprender los costos de hidratación y aplicar estas estrategias de optimización, aseguras que los beneficios de velocidad del renderizado en el servidor realmente se traduzcan en una experiencia más rápida y receptiva para tus usuarios.

### Prerenderizado parcial (PPR)

Next.js 15 introduce el Prerenderizado Parcial (*Partial Pre-rendering*, PPR), que lleva esto más allá al combinar el renderizado estático y dinámico en una sola solicitud.

> [!NOTE]
> PPR es una tecnología experimental que no se recomienda para producción. Es posible que encuentres algunos problemas de experiencia del desarrollador (DX), especialmente en bases de código más grandes.

Con PPR, la estructura estática de tu página (encabezado, diseño, contenido estático) se prerenderiza en el momento de la compilación y se sirve instantáneamente desde la CDN, mientras que las partes dinámicas (contenido específico del usuario, datos en tiempo real) se transmiten por streaming posteriormente. Esto significa que los usuarios ven una página de aspecto completo de inmediato, con elementos interactivos que se hidratan a medida que llega su código:

```tsx
// app/page.tsx - Partial Pre-rendering
import { Suspense } from 'react'
import { Header } from './Header'           // Static - pre-rendered
import { ProductGrid } from './ProductGrid' // Static - pre-rendered

// Dynamic components stream in after initial render
import { UserGreeting } from './UserGreeting'
import { RecommendedProducts } from './RecommendedProducts'

export default function HomePage() {
  return (
    <>
      {/* Static shell - served instantly from CDN */}
      <Header />
     
      {/* Dynamic - streams in after authentication check */}
      <Suspense fallback={<div className="h-8 bg-gray-200 animate-pulse" />}>
        <UserGreeting />
      </Suspense>
     
      {/* Static product grid - pre-rendered */}
      <ProductGrid />
     
      {/* Dynamic recommendations - streams based on user history */}
      <Suspense fallback={<RecommendationsSkeleton />}>
        <RecommendedProducts />
      </Suspense>
    </>
  )
}
```

Para habilitar PPR necesitas actualizar tu archivo `next.config.ts`:

```typescript
// next.config.ts - Enable PPR
import type { NextConfig } from 'next'

const config: NextConfig = {
  experimental: {
    ppr: true
  }
}

export default config
```

PPR reduce el alcance de la hidratación porque las partes estáticas no necesitan hidratación en absoluto; son HTML plano. Solo los componentes dinámicos e interactivos requieren JavaScript e hidratación, y se cargan en orden de prioridad según su posición en el árbol de Suspense. Esto es diferente de la SSR tradicional donde toda la página se hidrata como una unidad; con PPR, obtienes lo mejor de la generación estática (carga instantánea, almacenamiento en caché de CDN) y el renderizado dinámico (personalización, datos en tiempo real) en la misma página.

---

## Mejoras avanzadas de rendimiento en React 19

React introduce varias características potentes que amplían los límites de lo que es posible con las aplicaciones web. Estas funcionalidades no son solo mejoras incrementales; representan cambios fundamentales en la forma en que pensamos sobre la arquitectura de aplicaciones React.

### Aprovechamiento del renderizado concurrente para una interfaz de usuario más receptiva

El renderizado concurrente (*concurrent rendering*) le permite a React trabajar en múltiples versiones de la interfaz de usuario simultáneamente. Esto puede sonar complejo, pero el resultado es sencillo: tu aplicación sigue respondiendo incluso durante actualizaciones costosas. React puede pausar, reanudar o abandonar el trabajo según sea necesario, priorizando las actualizaciones urgentes como la entrada del usuario sobre las actualizaciones menos críticas como las animaciones.

La belleza del renderizado concurrente es que no necesitas hacer nada especial para habilitarlo; está disponible automáticamente en React 18 y posteriores. Sin embargo, puedes aprovechar APIs específicas para sacar el máximo provecho de él:

```tsx
import { useState, useTransition, useMemo } from 'react'

interface Product {
  id: number
  name: string
  category: string
  price: number
}

// Generate products once, outside the component
const products: Product[] = Array.from({ length: 5000 }, (_, i) => ({
  id: i,
  name: `Product ${i}`,
  category: ['Electronics', 'Clothing', 'Home', 'Sports'][i % 4],
  price: Math.floor(Math.random() * 1000) + 10
}))

const SearchableProductCatalog = () => {
  // Two separate states: one for input, one for filtering
  const [inputValue, setInputValue] = useState('')
  const [filterQuery, setFilterQuery] = useState('')
  const [isPending, startTransition] = useTransition()

  // Only recalculate when filterQuery changes
  const filteredProducts = useMemo(() => {
    return products.filter(product =>
      product.name.toLowerCase().includes(filterQuery.toLowerCase()) ||
      product.category.toLowerCase().includes(filterQuery.toLowerCase())
    )
  }, [filterQuery])

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value
   
    // Urgent: update input immediately so typing feels instant
    setInputValue(value)
   
    // Non-urgent: update filter with low priority
    // React can interrupt this if user keeps typing
    startTransition(() => {
      setFilterQuery(value)
    })
  }

  return (
    <div className="p-6 max-w-4xl mx-auto">
      <div className="mb-6">
        <div className="relative">
          <input
            type="text"
            value={inputValue}
            onChange={handleChange}
            placeholder="Search 5,000 products..."
            className="w-full px-4 py-3 border rounded-lg"
          />
          {/* isPending is true while the transition is processing */}
          {isPending && (
            <div className="absolute right-3 top-1/2 -translate-y-1/2">
              <div className="w-5 h-5 border-2 border-blue-600 border-t-transparent rounded-full animate-spin" />
            </div>
          )}
        </div>
        <p className="text-sm text-gray-500 mt-2">
          Found {filteredProducts.length} products
        </p>
      </div>

      {/* Dim the list while transition is pending */}
      <div className={`grid grid-cols-2 gap-4 ${isPending ? 'opacity-50' : ''}`}>
        {filteredProducts.slice(0, 50).map(product => (
          <div key={product.id} className="p-4 bg-white rounded-lg shadow">
            <h3 className="font-medium">{product.name}</h3>
            <p className="text-gray-600 text-sm">{product.category}</p>
            <p className="text-blue-600 font-bold">${product.price}</p>
          </div>
        ))}
      </div>
    </div>
  )
}
```

El hook `useTransition` marca la actualización del filtro como no urgente, lo que permite a React mantener la entrada receptiva incluso mientras filtra 5,000 productos. La clave es separar el estado: `inputValue` se actualiza inmediatamente en cada pulsación de tecla para que la escritura se sienta instantánea, mientras que `filterQuery` se actualiza dentro de `startTransition` para que React pueda interrumpir y reiniciar el filtrado costoso si el usuario sigue escribiendo. El booleano `isPending` te permite mostrar retroalimentación visual (aquí un spinner y una lista atenuada) mientras se procesa la transición. Este patrón funciona porque React trata las dos actualizaciones de estado de manera diferente: la urgente se aplica de inmediato y la no urgente se puede interrumpir y posponer.

### Uso de `startTransition` para actualizaciones de interfaz no urgentes

La API `startTransition` es tu forma de decirle a React: "Esta actualización es importante, pero no es urgente". React completará esta actualización cuando tenga tiempo, pero no permitirá que interfiera con actualizaciones más urgentes como la entrada del usuario o las animaciones.

Esto es útil para actualizaciones que desencadenan un renderizado costoso pero que no están directamente vinculadas a la interacción del usuario, como actualizar una visualización de datos grande o recalcular diseños complejos:

```tsx
import { useState, useTransition } from 'react'

interface Tab {
  id: string
  title: string
  content: string
}

// Simulates a heavy component that takes time to render
// In real apps, this might be a complex chart, data grid, or form
const HeavyTabContent = ({ content }: { content: string }) => {
  const items = Array.from({ length: 1000 }, (_, i) => `${content} - Item ${i}`)

  return (
    <div className="space-y-2">
      {items.map((item, index) => (
        <div key={`content-${index}`} className="p-2 bg-gray-50 rounded">
          {item}
        </div>
      ))}
    </div>
  )
}

// Tabs defined outside component to avoid recreation on each render
const tabs: Tab[] = [
  { id: 'tab1', title: 'Dashboard', content: 'Dashboard Data' },
  { id: 'tab2', title: 'Analytics', content: 'Analytics Data' },
  { id: 'tab3', title: 'Reports', content: 'Report Data' },
  { id: 'tab4', title: 'Settings', content: 'Settings Data' }
]

const TabNavigator = () => {
  const [activeTab, setActiveTab] = useState('tab1')
  // isPending: true while transition is processing
  // startTransition: wraps state updates to mark them as non-urgent
  const [isPending, startTransition] = useTransition()

  const handleTabChange = (tabId: string) => {
    // Without startTransition, clicking a tab would freeze the UI
    // until HeavyTabContent finishes rendering 1,000 items
    // With startTransition, the tab click feels instant—React keeps
    // the old content visible while preparing the new content
    startTransition(() => {
      setActiveTab(tabId)
    })
  }

  const currentTab = tabs.find(tab => tab.id === activeTab)!

  return (
    <div className="p-6">
      <div className="flex border-b">
        {tabs.map(tab => (
          <button
            key={tab.id}
            onClick={() => handleTabChange(tab.id)}
            // Tab button updates immediately even though content is pending
            className={`px-6 py-3 font-medium transition-colors relative ${
              activeTab === tab.id
                ? 'border-b-2 border-blue-600 text-blue-600'
                : 'text-gray-600 hover:text-gray-900'
            }`}
          >
            {tab.title}
            {/* Show loading indicator on the clicked tab while transitioning */}
            {isPending && activeTab !== tab.id && (
              <span className="absolute top-1 right-1 w-2 h-2 bg-blue-600 rounded-full animate-pulse" />
            )}
          </button>
        ))}
      </div>

      {/* Dim content while new tab is loading */}
      <div className={`mt-6 ${isPending ? 'opacity-50' : ''}`}>
        <h2 className="text-xl font-bold mb-4">{currentTab.title}</h2>
        <div className="max-h-96 overflow-y-auto">
          <HeavyTabContent content={currentTab.content} />
        </div>
      </div>
    </div>
  )
}
```

Observa que el contenido de la pestaña se atenúa durante la transición (a través de `isPending`), manteniendo visible el contenido antiguo mientras se prepara el nuevo contenido. Sin embargo, el botón de la pestaña en sí no se resalta de inmediato, porque `activeTab` es el único estado y está dentro de `startTransition`; la selección visual no se actualiza hasta que se completa el renderizado pesado.

### Streaming y Suspense para cargas de página más rápidas

El renderizado en el servidor siempre ha tenido como objetivo llevar el contenido a los usuarios más rápido, pero el SSR tradicional tenía una limitación significativa: el servidor tenía que terminar de renderizar toda la página antes de enviar algo al cliente. El SSR con streaming cambia esto al enviar HTML en fragmentos a medida que están listos.

Es importante distinguir entre dos usos diferentes de Suspense:
1. **La división de código del lado del cliente** utiliza `lazy()` para dividir los paquetes de JavaScript. El servidor envía HTML de respaldo y luego el cliente descarga y ejecuta los archivos JS. Esto reduce el tamaño inicial del paquete pero no es streaming; el HTML está completo cuando llega.
2. **El SSR con streaming** requiere RSC o soporte de framework (Next.js App Router, Remix). El servidor envía la estructura de la página inmediatamente como HTML y luego transmite fragmentos adicionales de HTML a medida que se completan las operaciones asíncronas. No se necesita JavaScript del lado del cliente para el contenido inicial.

Aquí está el SSR con streaming real con Next.js App Router:

```tsx
// app/product/[id]/page.tsx
import { Suspense } from 'react'
import { getProduct, getReviews, getRelatedProducts } from '@/lib/api'

interface Review { id: string; author: string; content: string }
interface Product { id: string; name: string; price: number; description: string; category: string }

// Async Server Component - runs on server, streams HTML when data resolves
async function Reviews({ productId }: { productId: string }) {
  const reviews = await getReviews(productId)
  return (
    <section className="mt-8">
      <h2 className="text-xl font-bold mb-4">Reviews</h2>
      {reviews.map((review: Review) => (
        <div key={review.id} className="p-4 border-b">
          <p className="font-medium">{review.author}</p>
          <p className="text-gray-600">{review.content}</p>
        </div>
      ))}
    </section>
  )
}

async function RelatedProducts({ category }: { category: string }) {
  const products = await getRelatedProducts(category)
  return (
    <section className="mt-8">
      <h2 className="text-xl font-bold mb-4">Related Products</h2>
      <div className="grid grid-cols-3 gap-4">
        {products.map((product: Product) => (
          <div key={product.id} className="p-4 bg-white rounded-lg shadow">
            <h3 className="font-medium">{product.name}</h3>
            <p className="text-blue-600">${product.price}</p>
          </div>
        ))}
      </div>
    </section>
  )
}

function ReviewsSkeleton() {
  return (
    <section className="mt-8">
      <div className="h-6 w-32 bg-gray-200 rounded animate-pulse mb-4" />
      {[1, 2, 3].map(i => <div key={i} className="h-20 bg-gray-100 rounded animate-pulse mb-2" />)}
    </section>
  )
}

function RelatedSkeleton() {
  return (
    <section className="mt-8">
      <div className="h-6 w-40 bg-gray-200 rounded animate-pulse mb-4" />
      <div className="grid grid-cols-3 gap-4">
        {[1, 2, 3].map(i => <div key={i} className="h-32 bg-gray-100 rounded animate-pulse" />)}
      </div>
    </section>
  )
}

// Next.js 15: params is now a Promise
export default async function ProductPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  const product = await getProduct(id)

  return (
    <div className="max-w-4xl mx-auto p-6">
      {/* Sent immediately - blocks initial HTML */}
      <div className="bg-white rounded-lg shadow p-6">
        <h1 className="text-3xl font-bold">{product.name}</h1>
        <p className="text-2xl text-blue-600 mt-2">${product.price}</p>
        <p className="text-gray-600 mt-4">{product.description}</p>
        <button className="mt-6 px-6 py-3 bg-blue-600 text-white rounded-lg">Add to Cart</button>
      </div>

      {/* Stream independently as data resolves - no client JS needed */}
      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews productId={id} />
      </Suspense>
      <Suspense fallback={<RelatedSkeleton />}>
        <RelatedProducts category={product.category} />
      </Suspense>
    </div>
  )
}
```

Esto es RSC con streaming, fundamentalmente diferente de `lazy()` del lado del cliente. Los componentes `Reviews` y `RelatedProducts` son funciones asíncronas que se ejecutan enteramente en el servidor. Cuando entra una solicitud, Next.js envía los detalles del producto y el HTML del esqueleto inmediatamente. A medida que se completa la obtención de datos de cada componente asíncrono, el servidor transmite un fragmento de HTML que reemplaza el esqueleto, sin descargas de JavaScript ni hidratación para estos componentes. El navegador recibe HTML listo para mostrar a través de una sola conexión HTTP. Ten en cuenta que en Next.js 15, `params` es una Promesa y debe ser esperada (`await`).

### Mejora del rendimiento de la obtención de datos con RSC

Los React Server Components (RSC) representan un cambio de paradigma en la forma en que pensamos sobre las aplicaciones de React. Los Server Components se ejecutan exclusivamente en el servidor, lo que les permite acceder directamente a bases de datos, sistemas de archivos y otros recursos exclusivos del servidor. El árbol de componentes que generan se serializa y se envía al cliente, pero el código del componente en sí nunca se envía al navegador.

Esto tiene profundas implicaciones para el rendimiento. Puedes obtener datos directamente en tus componentes sin solicitudes en cascada (*waterfall requests*) en el lado del cliente. Puedes usar librerías exclusivas del servidor sin aumentar el tamaño de tu bundle. Y puedes calcular operaciones costosas en el servidor donde la CPU y la memoria son abundantes:

```tsx
// UserProfile.tsx - This runs only on the server (notice does not have "use client")
import { Suspense } from 'react'
import db from './database'

// Only the fields safe to expose to the client
// The database entity might have password, role, tokens, etc.
interface UserDTO {
  id: string
  name: string
  email: string
  posts: number
  joined: string
}

// Data Access Layer: controls what data leaves the server
// Never return raw database entities to components
async function getUserProfile(userId: string): Promise<UserDTO> {
  const user = await db.users.findById(userId)
 
  // Explicitly pick safe fields—everything else stays on the server
  // Raw entity might include: passwordHash, resetToken, role, isAdmin, etc.
  return {
    id: user.id,
    name: user.name,
    email: user.email,
    posts: user.posts,
    joined: user.joined
  }
}

async function UserProfile({ userId }: { userId: string }) {
  // Fetch through the data access layer, not directly from db
  const user = await getUserProfile(userId)

  return (
    <div className="bg-white rounded-lg shadow p-6">
      <div className="flex items-center gap-4">
        <div className="w-16 h-16 bg-blue-600 rounded-full flex items-center justify-center text-white text-xl font-bold">
          {user.name.charAt(0)}
        </div>
        <div>
          <h1 className="text-2xl font-bold">{user.name}</h1>
          <p className="text-gray-600">{user.email}</p>
        </div>
      </div>

      <div className="grid grid-cols-2 gap-4 mt-6">
        <div className="p-4 bg-gray-50 rounded">
          <p className="text-sm text-gray-500">Posts</p>
          <p className="text-xl font-bold">{user.posts}</p>
        </div>
        <div className="p-4 bg-gray-50 rounded">
          <p className="text-sm text-gray-500">Member Since</p>
          <p className="text-xl font-bold">{user.joined}</p>
        </div>
      </div>
    </div>
  )
}

function ProfileSkeleton() {
  return (
    <div className="bg-white rounded-lg shadow p-6 animate-pulse">
      <div className="flex items-center gap-4">
        <div className="w-16 h-16 bg-gray-200 rounded-full" />
        <div>
          <div className="h-6 w-32 bg-gray-200 rounded" />
          <div className="h-4 w-48 bg-gray-200 rounded mt-2" />
        </div>
      </div>
    </div>
  )
}

export default function ProfilePage({ userId }: { userId: string }) {
  return (
    <div className="max-w-2xl mx-auto p-6">
      <Suspense fallback={<ProfileSkeleton />}>
        <UserProfile userId={userId} />
      </Suspense>
    </div>
  )
}
```

Los Server Components tienen acceso directo a la base de datos, lo que facilita la exposición accidental de datos confidenciales. Nunca devuelvas entidades de base de datos sin procesar; utiliza siempre un Objeto de Transferencia de Datos (*DTO*) que enumere explícitamente los campos seguros para el cliente. La función `getUserProfile` actúa como una capa de acceso a datos: obtiene la entidad completa pero devuelve solo los campos públicos. Este patrón protege contra la fuga de campos confidenciales como hashes de contraseñas, tokens de autenticación o indicadores internos que puedan existir en el modelo de base de datos.

La optimización del rendimiento en React opera en dos niveles que requieren enfoques diferentes:
- Las **optimizaciones del lado del cliente** (memorización, virtualización, división de código) a menudo se pueden aplicar de forma incremental. Cuando notas una interacción lenta, la analizas, identificas el cuello de botella y aplicas una solución específica. El Profiler de React DevTools te muestra exactamente qué componentes se renderizaron y por qué, lo que facilita encontrar re-renderizados innecesarios o cálculos costosos.
- Los **Server Components** son diferentes. Son una decisión arquitectónica, no una optimización que se añade más tarde. Los límites entre los componentes del servidor y del cliente afectan toda tu estrategia de obtención de datos, tus patrones de composición de componentes y cómo fluye el estado a través de tu aplicación. Si diseñas mal estos límites al principio, es posible que necesites reescrituras significativas para solucionarlos, no mejoras incrementales. Para nuevos proyectos que utilizan Next.js App Router o frameworks similares, piensa bien en tu división servidor/cliente desde el principio. Ten en cuenta que React DevTools estándar no puede perfilar los tiempos de renderizado de Server Components ya que eso sucede en el servidor; necesitarás herramientas de registro y monitorización del lado del servidor en su lugar.

Las técnicas de este capítulo (renderizado concurrente, límites de Suspense, SSR con streaming) te brindan herramientas para crear aplicaciones receptivas. Pero las herramientas no son soluciones. La verdadera habilidad radica en saber cuándo se aplica cada técnica y reconocer que un código más simple suele ser un código más rápido.

---

## Resumen

React 19 representa un cambio fundamental en la forma en que pensamos sobre el rendimiento. Los patrones de optimización manual que dominaron el desarrollo de React durante años (envolver componentes en `memo()`, estabilizar referencias con `useMemo` y `useCallback`) están siendo automatizados por el compilador de React. En lugar de que los desarrolladores realicen un seguimiento manual de qué valores necesitan memorización, el compilador analiza tu código en el momento de la compilación e inserta optimizaciones automáticamente. Esto no significa que comprender estos conceptos sea inútil; significa que puedes escribir código sencillo y dejar que las herramientas se encarguen de las partes tediosas.

Quizás lo más crítico es que hemos examinado la hidratación, el proceso a menudo invisible que une el HTML renderizado en el servidor y la interactividad del lado del cliente, y aprendimos estrategias para minimizar su impacto en la experiencia del usuario. Estas técnicas funcionan juntas sinérgicamente: un paquete más pequeño se hidrata más rápido, la hidratación selectiva con límites de Suspense permite el SSR con streaming y la división de código reduce tanto el tamaño de descarga inicial como el alcance de la hidratación. También aprendimos que la optimización eficaz comienza con la medición, utilizando herramientas como React Profiler, Lighthouse y Web Vitals para identificar cuellos de botella reales antes de aplicar soluciones.

A medida que continúas creando aplicaciones de React, mantén la mentalidad de que el rendimiento es una funcionalidad, no una ocurrencia tardía. Los usuarios perciben la velocidad de manera holística: una carga inicial rápida no significa nada si las interacciones se sienten lentas, y las actualizaciones instantáneas se ven socavadas por animaciones entrecortadas. Las características avanzadas que hemos explorado, particularmente el renderizado concurrente y los Server Components, representan la evolución de React hacia aplicaciones que se sienten nativas en su capacidad de respuesta. Comienza con un código limpio y fácil de mantener, mide el rendimiento real de tu aplicación a través de Core Web Vitals y métricas centradas en el usuario, y aplica optimizaciones estratégicamente donde más importen para tu aplicación y audiencia específicas. Recuerda que la mejor optimización suele ser el código que no escribes: simplificar la arquitectura de tus componentes y el flujo de datos puede generar mayores ganancias de rendimiento que cualquier técnica de memorización. Con el conocimiento de este capítulo, ahora estás equipado para crear aplicaciones de React que no solo funcionen bien, sino que se sientan excepcionales.

En el próximo capítulo, el final, veremos cómo desplegar una aplicación de Next.js en producción, en Vercel.
