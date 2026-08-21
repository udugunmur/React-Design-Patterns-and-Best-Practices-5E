# Capítulo 4: Técnicas Avanzadas de Gestión de Estado

La gestión del estado sigue siendo una preocupación fundamental en las aplicaciones React, especialmente a medida que las interfaces se vuelven más complejas y el estado se distribuye entre componentes, servidores, formularios y fuentes de datos externas. React 19 introduce nuevas APIs y refinamientos que influyen en cómo los desarrolladores coordinan estos aspectos, incluyendo `use()` para leer Promesas con Suspense y acceder al Contexto, Actions para manejar mutaciones asíncronas, y patrones mejorados para estados de UI optimistas y pendientes. Este capítulo examina estas capacidades junto con técnicas de optimización del rendimiento y soluciones externas de gestión de estado, con un enfoque en elegir la estrategia adecuada para cada tipo de estado.

Las aplicaciones modernas de React a menudo tratan con múltiples tipos de estado: estado local de componentes, estado compartido entre componentes, estado del servidor proveniente de APIs y estado global de la aplicación. Cada tipo requiere diferentes estrategias y herramientas. Las nuevas características de React 19, combinadas con librerías externas maduras, proporcionan a los desarrolladores un conjunto integral de herramientas para gestionar el estado de manera eficiente y con alto rendimiento.

La clave para una gestión de estado eficaz radica en comprender cuándo utilizar cada enfoque. El estado local con `useState` funciona perfectamente para datos simples a nivel de componente, mientras que las aplicaciones complejas pueden beneficiarse de las actualizaciones predecibles de estado de Redux Toolkit o del enfoque minimalista de Zustand. El nuevo hook `use` de React 19 cierra la brecha entre la gestión de estado síncrona y asíncrona, haciendo que la obtención de datos (*data fetching*) y las actualizaciones de estado sean más intuitivas que nunca.

A lo largo de este capítulo, exploraremos ejemplos prácticos utilizando TypeScript, asegurando que la seguridad de tipos (*type safety*) y la experiencia del desarrollador sigan siendo primordiales. Cubriremos técnicas de optimización de rendimiento, mejores prácticas para la arquitectura de estado y patrones del mundo real que escalan con el crecimiento de tu aplicación.

El capítulo cubre los siguientes temas:

- Gestión de estado con `use` para operaciones asíncronas
- Optimización del rendimiento de renderizado con `useDeferredValue`
- Gestión de limpieza y efectos secundarios con callbacks de ref (*ref callbacks*)
- Comprensión de Redux Toolkit: beneficios y casos de uso
- Context API vs. Redux: cuándo usar cada uno
- Zustand: gestión de estado ligera para aplicaciones modernas
- Mejores prácticas para gestionar el estado global y local

---

## Gestión de estado con `use` para operaciones asíncronas

React 19 introduce el revolucionario hook `use`, que cambia fundamentalmente la forma en que manejamos las operaciones asíncronas en componentes de React. A diferencia de los hooks tradicionales, `use` se puede llamar condicionalmente y funciona a la perfección con los límites de Suspense, haciendo que la gestión del estado asíncrono sea más intuitiva y potente.

### Comprensión del hook `use`

El hook `use` acepta una Promesa o un Contexto y devuelve el valor resuelto. Cuando se utiliza con Promesas, se integra perfectamente con el mecanismo de Suspense de React, permitiendo estados de carga elegantes y límites de error (*error boundaries*).

```tsx
import { Suspense, use } from 'react';

interface User {
  id: number;
  name: string;
  email: string;
}

interface UserProfileProps {
  userPromise: Promise<User>;
}

async function fetchUser(userId: number): Promise<User> {
  const response = await fetch(`/api/user/${userId}`);

  if (!response.ok) {
    throw new Error(`Failed to fetch user: ${response.status}`);
  }

  return response.json() as Promise<User>;
}

// Create the Promise once instead of recreating it on every render.
const userPromise = fetchUser(1);

function UserProfile({ userProfilePromise }: UserProfileProps) {
  const user = use(userPromise);

  return (
    <div className="user-profile">
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
}

export default function App() {
  return (
    <Suspense fallback={<div>Loading user...</div>}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  );
}
```

El hook `use()` te permite leer una Promesa o un Contexto directamente durante el renderizado. Cuando llamas a `use(promise)`, si la promesa está pendiente, React suspende el componente y muestra el fallback de `<Suspense>` más cercano; si se rechaza, el error se propaga al Error Boundary más cercano; si se resuelve, `use()` devuelve el valor. A diferencia de los hooks tradicionales, `use()` se puede llamar condicionalmente (aún únicamente durante el renderizado), lo que hace que las lecturas asíncronas se sientan como variables comunes y corrientes y se integren limpiamente con Suspense tanto en el cliente como en el servidor.

### Patrones avanzados de estado asíncrono

En interfaces de usuario complejas, los datos rara vez llegan todos a la vez. Una publicación necesita su autor; los comentarios son opcionales hasta que el lector los solicite. El hook `use()` de React te permite modelar esta realidad directamente en el renderizado: lees valores asíncronos como si ya estuvieran allí, y React maneja la espera a través de Suspense. Estudiemos un patrón de dos etapas: peticiones dependientes y peticiones condicionales.

#### Parte I: Primero el Post y luego el Autor: construyendo una cadena de dependencias

**Idea:** Obtén primero el Post. Una vez que se resuelva, extrae `authorId` y obtén el Autor. Cada paso puede suspenderse de forma independiente, y Suspense mantiene la interfaz de usuario elegante:

```tsx
import { Suspense, use } from "react";

interface Post {
  id: number;
  title: string;
  content: string;
  authorId: number;
}

interface Author {
  id: number;
  name: string;
  avatar: string;
}

const postCache = new Map<number, Promise<Post>>();
const authorCache = new Map<number, Promise<Author>>();

async function requestJson<T>(url: string): Promise<T> {
  const response = await fetch(url);

  if (!response.ok) {
    throw new Error(`Request failed with status ${response.status}`);
  }

  return response.json() as Promise<T>;
}

function getPost(postId: number): Promise<Post> {
  let promise = postCache.get(postId);

  if (!promise) {
    promise = requestJson<Post>(`/api/posts/${postId}`);
    postCache.set(postId, promise);
  }

  return promise;
}

function getAuthor(authorId: number): Promise<Author> {
  let promise = authorCache.get(authorId);

  if (!promise) {
    promise = requestJson<Author>(`/api/users/${authorId}`);
    authorCache.set(authorId, promise);
  }

  return promise;
}

function AuthorInfo({ authorId }: { authorId: number }) {
  const author = use(getAuthor(authorId));

  return (
    <div className="author-info">
      <img src={author.avatar} alt={author.name} />
      <span>By {author.name}</span>
    </div>
  );
}

function PostDetail({ postId }: { postId: number }) {
  const post = use(getPost(postId));

  return (
    <article className="post-detail">
      <header>
        <h1>{post.title}</h1>

        <Suspense fallback={<div>Loading author...</div>}>
          <AuthorInfo authorId={post.authorId} />
        </Suspense>
      </header>

      <div className="post-content">{post.content}</div>
    </article>
  );
}

export function PostPage({ postId }: { postId: number }) {
  return (
    <Suspense fallback={<div>Loading post...</div>}>
      <PostDetail postId={postId} />
    </Suspense>
  );
}
```

##### Qué observar:

- `use(getPost(postId))` lee una Promesa estable y en caché. Si la Promesa está pendiente, React suspende el subárbol actual y muestra el fallback de Suspense más cercano.
- Después de que el post se resuelve, su `authorId` determina qué recurso de autor leer. `use(getAuthor(post.authorId))` puede suspenderse de forma independiente. Cuando se envuelve en un límite de Suspense anidado, solo la sección del autor muestra su fallback; sin ese límite anidado, todo el subárbol de `PostDetail` se suspende nuevamente.
- Cuando `postId` cambia, React lee la Promesa asociada con la nueva clave de caché. Sin embargo, Suspense no cancela automáticamente la solicitud de red anterior. La capa de almacenamiento en caché o de obtención de datos debe manejar la cancelación, deduplicación, invalidación y expulsión. Se puede utilizar un `AbortController` para cancelar solicitudes que ya no son necesarias.

##### Conclusiones de diseño:

La dependencia entre los recursos se expresa directamente en el orden de renderizado: primero se lee el post, seguido de su autor. Esto elimina el estado de carga manual y la secuenciación de solicitudes basada en efectos, reduciendo problemas comunes de estado obsoleto. Sin embargo, `use()` y Suspense no eliminan todas las posibles condiciones de carrera ni gestionan la cancelación de solicitudes; esas responsabilidades permanecen en el framework, la caché o la capa de datos compatible con Suspense.

#### Parte II: Comentarios bajo demanda: peticiones condicionales sin código repetitivo

En muchas interfaces del mundo real, no todos los datos deben obtenerse de forma anticipada. Los datos secundarios, como comentarios, registros de actividad o elementos relacionados, a menudo solo se necesitan cuando el usuario los solicita explícitamente. Obtener estos datos por adelantado incrementa el costo de red y puede degradar el rendimiento del renderizado inicial. El desafío consiste en obtener estos datos solo cuando sea necesario, sin introducir una complejidad adicional en el estado de carga ni una lógica de efectos condicionales. Aquí es donde el hook `use()` permite un patrón más limpio: obtención condicional de datos directamente en el renderizado, aprovechando Suspense para la orquestación:

```tsx
interface Comment {
  id: number;
  content: string;
  authorId: number;
  postId: number
}

function PostDetail({ postId }: { postId: number }) {
  // ... post + author from Part I
  const [showComments, setShowComments] = useState(false);

  const commentsPromise = useCallback(
    () =>
      showComments
        ? fetch(`/api/posts/${postId}/comments`).then(
            res => res.json() as Promise<Comment[]>
          )
        : Promise.resolve([]), // 3. Instantly resolved when hidden → no suspend
    [postId, showComments]
  );

  const comments = use(commentsPromise());

  return (
    <article className="post-detail">
      {/* ... header + content */}
      <button onClick={() => setShowComments(v => !v)}>
        {showComments ? "Hide Comments" : "Show Comments"}
      </button>

      {showComments && (
        // (Recommended) Isolate loading with a nested Suspense:
        // <Suspense fallback={<div>Loading comments...</div>}>
        <div className="comments">
          {comments.map(c => (
            <div key={c.id} className="comment">{c.content}</div>
          ))}
        </div>
        // </Suspense>
      )}
    </article>
  );
}
```

##### Qué observar:

Los comentarios solo deben leerse después de que el usuario elija mostrarlos. Sin embargo, la Promesa pasada a `use()` debe provenir de una caché o de una capa de datos compatible con Suspense para que se reutilice la misma instancia de Promesa entre los intentos de renderizado. Llamar a `fetch()` a través de un callback memoizado no satisface este requisito porque cada invocación crea una nueva Promesa.

Para aislar el estado de carga, traslada la lectura de comentarios a un componente secundario y coloca ese componente dentro de un límite de Suspense anidado. El boundary puede entonces mostrar un fallback específico para los comentarios mientras mantiene visibles el post y el autor ya renderizados.

##### Conclusiones de diseño:

Suspense puede expresar el límite de carga de forma declarativa, pero no proporciona una arquitectura completa de obtención de datos. Una implementación de producción aún necesita un framework, caché o librería de datos para gestionar la identidad estable de las promesas, deduplicación de solicitudes, cancelación, invalidación, reintentos, errores y expulsión de caché.

Obtener comentarios bajo demanda puede reducir solicitudes innecesarias, pero no basta con devolver una Promesa resuelta recién creada mientras los comentarios están ocultos y una Promesa de red recién creada mientras están visibles. La rama inactiva debe evitar por completo renderizar el componente de lectura de datos, mientras que la rama activa debe leer un recurso estable gestionado fuera del componente de renderizado.

Esta estructura modela la intención del usuario con claridad: renderizar el post, resolver su autor y montar el recurso de comentarios solo cuando se solicite. React coordina los fallbacks de Suspense, mientras que la capa de datos sigue siendo responsable del ciclo de vida y la corrección de las solicitudes subyacentes.

### Manejo de errores con `use`

Los Error Boundaries son componentes de clase especiales que capturan errores de JavaScript lanzados durante el renderizado, en métodos del ciclo de vida de los hijos y en constructores de su árbol descendiente. Cuando se captura un error, renderizan una interfaz de usuario de respaldo en lugar de dejar que toda la aplicación colapse. No capturan errores en controladores de eventos, código asíncrono fuera del renderizado (por ejemplo, `setTimeout`) o código exclusivo del servidor. En React 19, cuando utilizas `use(promise)` y esa promesa es rechazada, el rechazo emerge como un error en tiempo de renderizado, por lo que es capturado por el Error Boundary más cercano.

El manejo de errores con el hook `use` se integra perfectamente con los Error Boundaries de React:

```tsx
import { use, Suspense, Component, ReactNode } from 'react';

interface ErrorBoundaryState {
  hasError: boolean;
  error?: Error;
}

class AsyncErrorBoundary extends Component<
  { children: ReactNode },
  ErrorBoundaryState
> {
  constructor(props: { children: ReactNode }) {
    super(props);
    this.state = { hasError: false };
  }
  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }
 
  render() {
    if (this.state.hasError) {
      return (
        <div className="error-boundary">
          <h2>Something went wrong!</h2>
          <p>{this.state.error?.message}</p>
        </div>
      );
    }
    return this.props.children;
  }
}

function DataComponent({ dataPromise }: { dataPromise: Promise<any> }) {
  const data = use(dataPromise);
 
  return (
    <div>
      <pre>{JSON.stringify(data, null, 2)}</pre>
    </div>
  );
}

function App() {
  const dataPromise = fetch('/api/data').then(res => {
    if (!res.ok) throw new Error('Failed to fetch data');
    return res.json();
  });
 
  return (
    <AsyncErrorBoundary>
      <Suspense fallback={<div>Loading...</div>}>
        <DataComponent dataPromise={dataPromise} />
      </Suspense>
    </AsyncErrorBoundary>
  );
}
```

### Creación de hooks asíncronos personalizados con `use`

Aunque `use()` simplifica las lecturas asíncronas dentro de los componentes, repetir la misma creación de promesas, manejo de errores y lógica condicional en múltiples componentes se vuelve rápidamente difícil de mantener. En aplicaciones más grandes, necesitas una abstracción reutilizable que estandarice cómo se obtienen, almacenan en caché y manejan los datos asíncronos. El desafío es encapsular esta lógica sin reintroducir la complejidad que `use()` fue diseñado para eliminar. Un hook personalizado construido sobre `use()` proporciona esta abstracción, permitiéndote centralizar el comportamiento asíncrono mientras mantienes el código de los componentes declarativo:

#### 1. Define la interfaz de tu hook
Comienza definiendo las opciones que aceptará tu hook personalizado. Esto hace que tu hook sea flexible y reutilizable:

```ts
import { use, useCallback, useMemo } from 'react';

interface UseAsyncDataOptions<T> {
  enabled?: boolean;           // Conditional fetching
  refetchInterval?: number;    // Auto-refresh capability
  onSuccess?: (data: T) => void;  // Success callback
  onError?: (error: Error) => void;  // Error handling
}
```

Esta interfaz define las opciones de configuración que los consumidores de tu hook pueden proporcionar.

#### 2. Crea la Promesa con `useMemo`
Utiliza `useMemo` para crear una promesa estable que solo cambie cuando las dependencias cambien. Esta es la clave para hacer que `use` funcione eficazmente:

```ts
function useAsyncData<T>(
  fetchFn: () => Promise<T>,
  dependencies: any[] = [],
  options: UseAsyncDataOptions<T> = {}
) {
  const { enabled = true, onSuccess, onError } = options;

  const promise = useMemo(() => {
    // Conditional fetching - return resolved promise if disabled
    if (!enabled) return Promise.resolve(null as T);

    // Execute fetch and attach callbacks
    return fetchFn()
      .then(data => {
        onSuccess?.(data);
        return data;
      })
      .catch(error => {
        onError?.(error);
        throw error;  // Re-throw so Suspense/ErrorBoundary can catch
      });
  }, [...dependencies, enabled]);

  // ... rest of hook
}
```

Algunos puntos clave:

- `useMemo` garantiza que la promesa solo se vuelva a crear cuando cambien las dependencias.
- La promesa está memoizada, por lo que `use` no provocará re-renderizados innecesarios.
- Los callbacks se ejecutan dentro de la cadena de promesas.
- Los errores se relanzan para que los límites de Suspense puedan capturarlos.

#### 3. Utiliza el hook `use` para desenvolver la Promesa
Aquí es donde ocurre la magia: `use` desenvuelve la promesa y se integra con el sistema de Suspense de React:

```ts
function useAsyncData<T>(
  fetchFn: () => Promise<T>,
  dependencies: any[] = [],
  options: UseAsyncDataOptions<T> = {}
) {
  const { enabled = true, onSuccess, onError } = options;

  const promise = useMemo(() => {
    if (!enabled) return Promise.resolve(null as T);
    return fetchFn()
      .then(data => { onSuccess?.(data); return data; })
      .catch(error => { onError?.(error); throw error; });
  }, [...dependencies, enabled]);

  // use() suspends the component until the promise resolves
  const data = use(promise);

  // ... rest of hook
}
```

---

## Optimización del rendimiento de renderizado con `useDeferredValue`

La optimización del rendimiento es crucial en las aplicaciones React modernas, especialmente cuando se trata de cálculos costosos o grandes conjuntos de datos. El hook `useDeferredValue` de React 19 proporciona un mecanismo potente para diferir actualizaciones no urgentes, permitiendo que las actualizaciones de alta prioridad se rendericen de inmediato mientras se mantiene la interfaz de usuario con capacidad de respuesta.

### Comprensión de `useDeferredValue`

El hook `useDeferredValue` acepta un valor y devuelve una versión diferida de ese valor. Este valor diferido puede retrasarse con respecto al valor real durante actualizaciones urgentes, lo que permite a React priorizar renderizados más importantes:

```tsx
import { useDeferredValue, useState, useMemo } from 'react';

interface SearchResultsProps {
  query: string;
  results: string[];
}

function SearchResults({ query, results }: SearchResultsProps) {
  const deferredQuery = useDeferredValue(query);
  const deferredResults = useDeferredValue(results);
 
  const filteredResults = useMemo(() => {
    return deferredResults.filter(result =>
      result.toLowerCase().includes(deferredQuery.toLowerCase())
    );
  }, [deferredQuery, deferredResults]);
 
  return (
    <div className="search-results">
      <p>Results for: {deferredQuery}</p>
      <ul>
        {filteredResults.map((result, index) => (
          <li key={index}>{result}</li>
        ))}
      </ul>
    </div>
  );
}

function SearchApp() {
  const [query, setQuery] = useState('');
  const [results] = useState(() =>
    Array.from({ length: 10000 }, (_, i) => `Result ${i + 1}`)
  );
 
  return (
    <div>
      <input
        type="text"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search..."
      />
      <SearchResults query={query} results={results} />
    </div>
  );
}
```

En este componente `SearchResults`, `useDeferredValue` divide las actualizaciones importantes del trabajo lento al permitir que el campo de entrada se actualice de inmediato con la última consulta, mientras que `deferredQuery` y `deferredResults` se actualizan después de que la entrada termina de actualizarse. Esto significa que cuando un usuario escribe, la entrada se mantiene fluida y rápida mientras React maneja el filtrado lento de 10,000 resultados en segundo plano usando valores diferidos ligeramente anteriores. La combinación de `useDeferredValue` y `useMemo` es importante: `useDeferredValue` controla cuándo ocurre el trabajo lento (esperando hasta después de que terminen las actualizaciones importantes), mientras que `useMemo` se asegura de que el filtrado solo se vuelva a ejecutar cuando los valores diferidos realmente cambien. Sin esto, cada pulsación de tecla desencadenaría el filtrado de los 10,000 elementos de inmediato, lo que podría congelar la interfaz y hacer que la escritura se sienta lenta.

### Patrones avanzados de rendimiento

Aquí hay un ejemplo más complejo que muestra cómo usar `useDeferredValue` con cálculos costosos y visualización de datos:

```tsx
import { useDeferredValue, useState, useMemo } from 'react';

interface DataPoint {
  id: number;
  value: number;
  category: string;
}

interface ChartProps {
  data: DataPoint[];
  filterCategory: string;
}

function ExpensiveChart({ data, filterCategory }: ChartProps) {
  const deferredData = useDeferredValue(data);
  const deferredFilter = useDeferredValue(filterCategory);
 
  const processedData = useMemo(() => {
    console.log('Processing data...');
   
    if (!deferredFilter) return deferredData;
   
    return deferredData.filter(point =>
      point.category.toLowerCase().includes(deferredFilter.toLowerCase())
    );
  }, [deferredData, deferredFilter]);
 
  return (
    <div className="chart">
      <p className="text-gray-600">Showing {processedData.length} results</p>
     
      <div className="space-y-2">
        {processedData.slice(0, 100).map(point => (
          <div key={point.id} className="flex justify-between p-2 bg-gray-50">
            <span>{point.category}</span>
            <span className="font-semibold">{point.value.toFixed(2)}</span>
          </div>
        ))}
      </div>
    </div>
  );
}

function DataVisualization() {
  const [filterCategory, setFilterCategory] = useState('');
 
  const data = useMemo(() => {
    return Array.from({ length: 50000 }, (_, i) => ({
      id: i,
      value: Math.random() * 1000,
      category: `Category ${Math.floor(Math.random() * 10)}`
    }));
  }, []);
 
  return (
    <div className="p-4">
      <input
        type="text"
        value={filterCategory}
        onChange={(e) => setFilterCategory(e.target.value)}
        placeholder="Filter by category..."
        className="w-full p-2 border rounded mb-4"
      />
     
      <ExpensiveChart data={data} filterCategory={filterCategory} />
    </div>
  );
}
```

En el componente `ExpensiveChart`, `useDeferredValue` mantiene la entrada del filtro receptiva mientras maneja el filtrado costoso de 50,000 puntos de datos. Cuando un usuario escribe en la entrada del filtro, el campo de entrada se actualiza de inmediato con el valor más reciente de `filterCategory`, mientras que `deferredFilter` y `deferredData` se actualizan después de que la entrada termina de actualizarse. Esto significa que la experiencia de escritura se mantiene fluida mientras React filtra a través del gran conjunto de datos en segundo plano usando valores diferidos ligeramente anteriores. El hook `useMemo` se asegura de que el filtrado solo se vuelva a ejecutar cuando los valores diferidos realmente cambien, no en cada renderizado. Sin esta optimización, cada pulsación de tecla filtraría inmediatamente los 50,000 elementos, lo que podría congelar el campo de entrada y hacer que la escritura se sienta con retraso. Con los valores diferidos, la entrada permanece receptiva mientras que los resultados se actualizan poco después de que dejas de escribir.

### Combinación de `useDeferredValue` con `useTransition`

`useTransition` se utiliza para marcar las escrituras de estado que desencadenan un trabajo de UI pesado como no urgentes y para controlar la UI pendiente. Y `useDeferredValue` se utiliza cuando un valor cambia con frecuencia pero sus consumidores son costosos; pasa el valor diferido hacia abajo y calcula derivaciones pesadas de `useMemo` a partir de él.

Para un control aún más granular sobre las prioridades de actualización, puedes combinar `useDeferredValue` con `useTransition`:

```tsx
import { useDeferredValue, useTransition, useState, useMemo } from 'react';

interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
  description: string;
}

function ProductList({ products }: { products: Product[] }) {
  const [searchTerm, setSearchTerm] = useState('');
  const [selectedCategory, setSelectedCategory] = useState('all');
  const [isPending, startTransition] = useTransition();
 
  const deferredSearchTerm = useDeferredValue(searchTerm);
  const deferredSelectedCategory = useDeferredValue(selectedCategory);
 
  const filteredProducts = useMemo(() => {
    let filtered = products;
   
    if (deferredSearchTerm) {
      filtered = filtered.filter(product =>
        product.name.toLowerCase().includes(deferredSearchTerm.toLowerCase()) ||
   product.description.toLowerCase().includes(deferredSearchTerm.toLowerCase())
      );
    }
   
    if (deferredSelectedCategory !== 'all') {
      filtered = filtered.filter(product =>
        product.category === deferredSelectedCategory
      );
    }
   
    return filtered;
  }, [products, deferredSearchTerm, deferredSelectedCategory]);
 
  const handleSearchChange = (value: string) => {
    setSearchTerm(value);
    startTransition(() => {
      // This update is marked as non-urgent
      // React will prioritize other updates over this
    });
  };
 
  const handleCategoryChange = (category: string) => {
    setSelectedCategory(category);
    startTransition(() => {
      // Non-urgent category filter update
    });
  };
 
  return (
    <div className="product-list">
      <div className="filters">
        <input
          type="text"
          value={searchTerm}
          onChange={(e) => handleSearchChange(e.target.value)}
          placeholder="Search products..."
        />
        <select
          value={selectedCategory}
          onChange={(e) => handleCategoryChange(e.target.value)}
        >
          <option value="all">All Categories</option>
          <option value="electronics">Electronics</option>
          <option value="clothing">Clothing</option>
          <option value="books">Books</option>
        </select>
      </div>
     
{isPending&&<div className="loading">Filtering products...</div>}
     
      <div className="products">
        {filteredProducts.map(product => (
          <div key={product.id} className="product-card">
            <h3>{product.name}</h3>
            <p className="price">${product.price}</p>
            <p className="category">{product.category}</p>
            <p className="description">{product.description}</p>
          </div>
        ))}
      </div>
    </div>
  );
}
```

El hook `useTransition` te permite marcar actualizaciones de estado como no urgentes envolviéndolas en `startTransition`, lo que le indica a React que estas actualizaciones pueden interrumpirse si llegan actualizaciones más importantes. Devuelve una bandera `isPending` que se vuelve verdadera mientras ocurre la transición, permitiéndote mostrar indicadores de carga. Para un control granular sobre las prioridades de actualización, puedes combinar ambos hooks: usa `useTransition` para marcar qué actualizaciones de estado no son urgentes y `useDeferredValue` para diferir cálculos costosos basados en esos valores, dándote control tanto sobre cuándo ocurren las actualizaciones de estado como sobre cuándo se ejecuta el trabajo costoso.

En resumen, `useDeferredValue` mantiene las interacciones receptivas al permitir que los consumidores costosos se retrasen ligeramente, reduciendo re-renderizados innecesarios sin recurrir a temporizadores. Con la volatilidad de valores bajo control, ahora pasamos de gestionar cuándo se ejecutan los cálculos a gestionar qué elementos tocamos, introduciendo los callbacks de ref para un acceso imperativo y preciso a los nodos del DOM y a los ciclos de vida de los componentes en la siguiente sección.

---

## Gestión de limpieza y efectos secundarios con callbacks de ref

Hasta ahora, nos hemos centrado en gestionar el estado y los flujos de datos asíncronos de forma declarativa. Sin embargo, no todo el comportamiento con estado encaja perfectamente en el modelo de renderizado de React. Algunos escenarios, como la integración con APIs del navegador, librerías de terceros u observadores del DOM, requieren un control preciso sobre la configuración y la limpieza vinculadas a nodos reales del DOM. Los patrones tradicionales con `useEffect` pueden introducir re-renderizados innecesarios o complejidad en el ciclo de vida. Los callbacks de ref proporcionan una alternativa más determinista al permitirte gestionar recursos en el momento exacto en que un nodo del DOM se adjunta o se desvincula, convirtiéndose en una extensión natural de la gestión avanzada del estado y del ciclo de vida.

Los callbacks de ref son funciones que reciben el elemento DOM o la instancia del componente como argumento. Se les llama cuando el componente se monta y se desmonta, proporcionando oportunidades ideales para operaciones de configuración y limpieza:

```tsx
import { useCallback, useEffect, useRef } from 'react';

interface ResizeObserverHookReturn {
  ref: (element: HTMLElement | null) => void;
  dimensions: { width: number; height: number } | null;
}

function useResizeObserver(): ResizeObserverHookReturn {
  const [dimensions, setDimensions] = useState<{ width: number; height: number } | null>(null);
  const resizeObserverRef = useRef<ResizeObserver | null>(null);
 
  const ref = useCallback((element: HTMLElement | null) => {
    // Cleanup previous observer
    if (resizeObserverRef.current) {
      resizeObserverRef.current.disconnect();
    }
   
    if (element) {
      // Create new observer
      resizeObserverRef.current = new ResizeObserver(([entry]) => {
        const { width, height } = entry.contentRect;
        setDimensions({ width, height });
      });
     
      resizeObserverRef.current.observe(element);
     
      // Set initial dimensions
      const { width, height } = element.getBoundingClientRect();
      setDimensions({ width, height });
    } else {
      // Element is being unmounted
      resizeObserverRef.current = null;
      setDimensions(null);
    }
  }, []);
 
  return { ref, dimensions };
}

function ResizableComponent() {
  const { ref, dimensions } = useResizeObserver();
 
  return (
    <div
      ref={ref}
      style={{
        resize: 'both',
        overflow: 'hidden',
        border: '1px solid #ccc',
        minWidth: '200px',
        minHeight: '100px'
      }}
    >
      <h3>Resizable Component</h3>
      {dimensions && (
        <p>Size: {dimensions.width}x{dimensions.height}</p>
      )}
    </div>
  );
}
```

El hook personalizado `useResizeObserver` rastrea las dimensiones de un elemento utilizando la API `ResizeObserver` del navegador. Utiliza un patrón de callback ref donde la función `ref` adjunta un `ResizeObserver` a cualquier elemento que se le pase. Cuando se monta un elemento, el hook crea un observador que actualiza el estado de `dimensions` cada vez que el elemento cambia de tamaño. La instancia del observador se almacena en `resizeObserverRef` para persistir a través de los renderizados y permitir la limpieza; cuando se pasa un nuevo elemento o el componente se desmonta, el observador anterior se desconecta. El componente `ResizableComponent` simplemente pasa este ref a su `div`, y el hook rastrea y devuelve automáticamente el ancho y alto actuales.

Los callbacks de ref complementan el tema general de este capítulo al abordar una clase de estado que no es puramente declarativa: los ciclos de vida de recursos vinculados a elementos reales del DOM. Mientras que hooks como `use()` y `useDeferredValue` optimizan el flujo de datos y el comportamiento de renderizado, los callbacks de ref te brindan un control preciso sobre integraciones imperativas. En conjunto, estos patrones ofrecen un modelo mental más completo para gestionar tanto los aspectos declarativos como imperativos de las aplicaciones modernas de React.

---

## Comprensión de Redux Toolkit: beneficios y casos de uso

Redux Toolkit (RTK) es la forma oficial y con todo incluido (*batteries-included*) de usar Redux que soluciona los puntos débiles: exceso de código repetitivo, actualizaciones inmutables frágiles y convenciones dispersas, mientras preserva aquello en lo que los equipos confían: una única fuente de verdad, transiciones predecibles, depuración con viajes en el tiempo en DevTools y soporte de TypeScript de primer nivel. Al estandarizar los *slices*, la configuración del store y los middlewares, RTK te permite expresar la intención de manera concisa (`createSlice`, reducers potenciados por Immer), manejar flujos asíncronos sin complicaciones (`createAsyncThunk`) y escalar la obtención de datos con el almacenamiento en caché, deduplicación e invalidación integrados de RTK Query. El resultado son menos errores, una estructura más clara, mejor rendimiento y una incorporación más fluida en bases de código grandes.

RTK brilla cuando el estado abarca múltiples componentes o rutas, autenticación y permisos, banderas de funciones (*feature flags*), interfaces de usuario globales (toasts/modales), flujos de trabajo de múltiples pasos y entidades normalizadas compartidas entre vistas (usuarios, productos, comentarios). Es especialmente valioso para aplicaciones conectadas a red donde la consistencia y la trazabilidad importan: RTK Query coordina las interacciones con el servidor, admite actualizaciones optimistas, sondeo (*polling*) y hooks del ciclo de vida de la caché, y coexiste limpiamente con otras herramientas (RSC, TanStack Query o estado local de componentes) para que puedas adoptarlo de forma incremental. En resumen, RTK te brinda una vía pragmática hacia un estado global predecible y mantenible, sin la sobrecarga que hacía que el Redux clásico se sintiera pesado.

### Configuración de RTK

Primero, establezcamos un store de Redux correctamente tipado con RTK. Comenzaremos con la configuración básica:

```ts
import { configureStore, createSlice, PayloadAction } from '@reduxjs/toolkit';
import { TypedUseSelectorHook, useDispatch, useSelector } from 'react-redux';

interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user' | 'guest';
}

interface AuthState {
  user: User | null;
  isAuthenticated: boolean;
  loading: boolean;
  error: string | null;
}

const initialState: AuthState = {
  user: null,
  isAuthenticated: false,
  loading: false,
  error: null,
};
```

Ahora creamos el slice de autenticación con reducers:

```ts
const authSlice = createSlice({
  name: 'auth',
  initialState,
  reducers: {
    // Sets loading state to true when login process begins
    loginStart: (state) => {
      state.loading = true;
      state.error = null;
    },
    // Updates state with user data when login succeeds
    loginSuccess: (state, action: PayloadAction<User>) => {
      state.loading = false;
      state.isAuthenticated = true;
      state.user = action.payload;
      state.error = null;
    },
    // Sets error message when login fails
    loginFailure: (state, action: PayloadAction<string>) => {
      state.loading = false;
      state.isAuthenticated = false;
      state.user = null;
      state.error = action.payload;
    },
    // Clears user data and authentication state on logout
    logout: (state) => {
      state.user = null;
      state.isAuthenticated = false;
      state.error = null;
    },
  },
});

export const { loginStart, loginSuccess, loginFailure, logout } = authSlice.actions;

export const store = configureStore({
  reducer: {
    auth: authSlice.reducer,
  },
});

// Type exports
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

// Typed hooks
export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

`createSlice` de RTK genera automáticamente creadores de acciones (*action creators*) y tipos de acción, eliminando código repetitivo. Immer trabaja entre bastidores, permitiéndonos escribir lógica de mutación que en realidad es inmutable.

### Operaciones asíncronas con `createAsyncThunk`

`createAsyncThunk` condensa toda la ceremonia del Redux asíncrono en una sola unidad declarativa: escribes un único creador de carga útil asíncrono y RTK genera las tres acciones del ciclo de vida (`pending`/`fulfilled`/`rejected`), conecta sus tipos y gestiona tus banderas de carga/error a través de `extraReducers`. No más tipos y creadores de acciones escritos a mano, bloques `try/catch` dispersos ni conmutadores `isLoading` ad hoc; las transiciones de estado se vuelven predecibles, definidas centralmente y fáciles de probar. Debido a que el ciclo de vida está estandarizado, tu equipo obtiene patrones consistentes para errores, reintentos y retroalimentación de la UI, y las líneas de tiempo de DevTools se mantienen legibles (cada paso asíncrono es una acción por la que puedes viajar en el tiempo).

Más allá de reducir el código repetitivo, también resuelve problemas complejos integrados: cancelación mediante `thunkAPI.signal`, deduplicación/protección con `condition`, acceso tipado a `getState`/`dispatch`, y un `unwrap()` simple para una ergonomía similar a promesas en los componentes. El resultado son menos errores en casos límite, una intención más clara y un desarrollo de funciones más rápido, especialmente a escala, donde docenas de thunks se benefician del mismo contrato de ciclo de vida.

`createAsyncThunk` de RTK simplifica el manejo de operaciones asíncronas generando automáticamente acciones pendientes, cumplidas y rechazadas para cualquier función asíncrona, eliminando el código repetitivo manual de tipos de acción, bloques `try/catch` y gestión del estado de carga. Creemos un thunk para obtener productos:

```ts
import { createAsyncThunk, createSlice } from '@reduxjs/toolkit';

interface Product {
  id: string;
  name: string;
  price: number;
  inventory: number;
}

interface ProductState {
  items: Product[];
  loading: boolean;
  error: string | null;
}

// Async thunk for fetching products
export const fetchProducts = createAsyncThunk<Product[]>(
  'products/fetch',
  async () => {
    const response = await fetch('/api/products');
    if (!response.ok) {
      throw new Error('Failed to fetch products');
    }
    return response.json();
  }
);
```

Ahora creemos el slice de productos que maneja estas acciones asíncronas:

```ts
const productsSlice = createSlice({
  name: 'products',
  initialState: {
    items: [],
    loading: false,
    error: null,
  } as ProductState,
  reducers: {
    // Updates the inventory quantity for a specific product
    updateInventory: (state, action: PayloadAction<{
      productId: string;
      quantity: number;
    }>) => {
      const product = state.items.find(p => p.id === action.payload.productId);
      if (product) {
        product.inventory = action.payload.quantity;
      }
    },
  },
  extraReducers: (builder) => {
    builder
      // Sets loading state when product fetch starts
      .addCase(fetchProducts.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      // Updates state with fetched products when request succeeds
      .addCase(fetchProducts.fulfilled, (state, action) => {
        state.loading = false;
        state.items = action.payload;
      })
      // Sets error message when product fetch fails
      .addCase(fetchProducts.rejected, (state, action) => {
        state.loading = false;
        state.error = action.error.message || 'Failed to fetch';
      });
  },
});

export const { updateInventory } = productsSlice.actions;
export default productsSlice.reducer;
```

El thunk asíncrono maneja el ciclo de vida completo, despachando automáticamente las acciones pendientes, cumplidas y rechazadas.

### RTK Query para la obtención de datos

RTK Query convierte la obtención y el almacenamiento en caché en un flujo declarativo y estandarizado: defines endpoints y genera hooks tipados que manejan la deduplicación de solicitudes, caché, re-obtención al enfocar/reconectar, sondeo e invalidación de caché; sin banderas de carga hechas a mano, cableado de efectos ni memoización ad hoc. La invalidación basada en etiquetas (*tags*) mantiene sincronizadas las vistas derivadas después de las mutaciones, las actualizaciones optimistas son de primer nivel y los eventos de DevTools se mantienen legibles ya que cada paso es una acción. El efecto neto es menos código, menos errores en casos límite y ciclos de vida de datos predecibles en toda tu aplicación.

Recurre a RTK Query cuando tengas estado del servidor que se lea en múltiples lugares, necesite caché a través de páginas o se beneficie de reglas consistentes de re-obtención (listas + detalles, paginación, datos vinculados a autenticación, dashboards). Es especialmente útil a escala; los equipos obtienen los mismos patrones para errores, reintentos, actualización en segundo plano e invalidación sin reinventarlos por funcionalidad. Si tu estado es puramente local o ya está completamente administrado por una capa de renderizado/caché del servidor, puede que no lo necesites en todas partes, pero es un excelente valor predeterminado para la obtención de datos en el cliente en aplicaciones Redux.

Ahora, creemos un slice de API simple:

```ts
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

interface Post {
  id: string;
  title: string;
  content: string;
  authorId: string;
}

export const postsApi = createApi({
  reducerPath: 'postsApi',
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  tagTypes: ['Post'],
  endpoints: (builder) => ({
    getPosts: builder.query<Post[], void>({
      query: () => 'posts',
      providesTags: ['Post'],
    }),
    getPostById: builder.query<Post, string>({
      query: (id) => `posts/${id}`,
      providesTags: (result, error, id) => [{ type: 'Post', id }],
    }),
    createPost: builder.mutation<Post, Partial<Post>>({
      query: (post) => ({
        url: 'posts',
        method: 'POST',
        body: post,
      }),
      invalidatesTags: ['Post'],
    }),
  }),
});

export const {
  useGetPostsQuery,
  useGetPostByIdQuery,
  useCreatePostMutation
} = postsApi;
```

En resumen, Redux Toolkit y RTK Query te ofrecen un camino consistente y con poco código repetitivo desde acciones locales hasta datos en red, con ciclos de vida estandarizados, almacenamiento en caché y depuración predecible. Con esa base establecida, la pregunta natural es el alcance: ¿cuándo vale la pena la sobrecarga del estado global de Redux y cuándo es suficiente un Context ligero? Comparemos la Context API y Redux para que puedas elegir la herramienta adecuada para cada segmento de estado.

---

## Context API vs. Redux: cuándo usar cada uno

Aquí trazaremos una línea clara entre la Context API y Redux centrándonos en lo que cada uno está diseñado para resolver. Context sobresale en aliviar la perforación de props (*prop drilling*) y en compartir valores estables y de baja volatilidad (tema, configuración regional, usuario actual) dentro de un subárbol. Redux, especialmente con Redux Toolkit y RTK Query, se orienta a un estado global predecible y ciclos de vida de datos del servidor, acciones, depuración con viajes en el tiempo, almacenamiento en caché, invalidación y coordinación entre funcionalidades.

### Patrones modernos de Context API

Esta sección pone los patrones en práctica: construiremos un Módulo de Contexto con un hook personalizado `useTheme()`, dividiremos el contexto en estado de solo lectura y acciones de solo escritura para reducir re-renderizados, memoizaremos el valor provisto con `useMemo` y expondremos actualizadores estables a través de `useCallback` (tanto un `toggleMode()` como un `setMode(mode)` explícito). También admitiremos proveedores con ámbito/anidados para reemplazos por ruta o por componente, agregaremos una protección segura para SSR que lance un error si falta el proveedor, y conectaremos el tema a variables CSS/Tailwind (usando `class="dark"` o `data-theme`) para que los estilos se actualicen al instante. Finalmente, cubriremos la persistencia (sincronización con `localStorage` y `prefers-color-scheme`) y consejos de rendimiento (mantener pequeña la forma del contexto, derivar valores fuera de los consumidores) para mantener los re-renderizados al mínimo.

Estos son los patrones que cubriremos en esta sección:

- **Patrón de módulo de contexto:** Hook personalizado `useTheme()` que encapsula la lógica del contexto.
- **Memoización de valores:** Uso de `useMemo` para evitar recrear el valor del contexto.
- **Actualizadores estables:** Uso de `useCallback` para la función `toggleMode`.
- **Protecciones seguras para SSR:** Lanzar un error cuando se use `useTheme` fuera de `ThemeProvider`.
- **Patrón de seguridad de tipos:** Interfaces de TypeScript para la configuración del tema y el valor del contexto.

Si bien la API de Contexto central no ha cambiado desde React 16.3, las actualizaciones recientes de React han mejorado su rendimiento y capacidades. El procesamiento por lotes automático (*automatic batching*) de React 18 agrupa múltiples actualizaciones de Contexto en un solo re-renderizado, reduciendo renderizados innecesarios. El hook `use()` de React 19 permite la lectura condicional de Contexto (dentro de sentencias `if` o bucles), lo cual no era posible con `useContext`. Context también se integra mejor con funciones concurrentes como Suspense y streaming SSR, haciéndolo más adecuado para aplicaciones complejas.

```tsx
import React, { createContext, useContext, useState, useCallback, useMemo } from 'react';

interface ThemeColors {
  primary: string;
  secondary: string;
  background: string;
  text: string;
}

interface ThemeContextValue {
  colors: ThemeColors;
  mode: 'light' | 'dark';
  setMode: (mode: 'light' | 'dark') => void;
  toggleMode: () => void;
}

const themes = {
  light: {
    primary: '#3b82f6',
    secondary: '#8b5cf6',
    background: '#ffffff',
    text: '#1f2937',
  },
  dark: {
    primary: '#60a5fa',
    secondary: '#a78bfa',
    background: '#1f2937',
    text: '#f3f4f6',
  },
};
```

Ahora creemos el contexto y el proveedor:

```tsx
const ThemeContext = createContext<ThemeContextValue | undefined>(undefined);

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [mode, setMode] = useState<'light' | 'dark'>('light');
 
  const toggleMode = useCallback(() => {
    setMode(prev => prev === 'light' ? 'dark' : 'light');
  }, []);
  const value = useMemo(() => ({
    colors: themes[mode],
    mode,
    setMode,
    toggleMode,
  }), [mode, toggleMode]);
 
  return (
    <ThemeContext.Provider value={value}>
      {children}
    </ThemeContext.Provider>
  );
}

export function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  return context;
}
```

Este patrón demuestra cómo Context puede gestionar eficientemente el estado global con una optimización adecuada mediante `useMemo` y `useCallback`.

### Optimización del rendimiento con múltiples contextos

Para evitar re-renderizados innecesarios, podemos dividir los contextos según su responsabilidad:

```tsx
const UserContext = createContext<User | null>(null);
const UserActionsContext = createContext<{
  login: (user: User) => void;
  logout: () => void;
} | null>(null);

function UserProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
 
  const actions = useMemo(() => ({
    login: (user: User) => setUser(user),
    logout: () => setUser(null),
  }), []);
  return (
    <UserContext.Provider value={user}>
      <UserActionsContext.Provider value={actions}>
        {children}
      </UserActionsContext.Provider>
    </UserContext.Provider>
  );
}

// Hooks for consuming contexts
export function useUser() {
  const user = useContext(UserContext);
  return user;
}

export function useUserActions() {
  const actions = useContext(UserActionsContext);
  if (!actions) {
    throw new Error('useUserActions must be used within UserProvider');
  }
  return actions;
}
```

Al separar el estado de las acciones, los componentes que solo necesitan despachar acciones no se volverán a renderizar cuando cambie el estado del usuario.

### Cuándo elegir Redux vs. Context

La decisión entre Context API y Redux es menos una cuestión de preferencia y más sobre la complejidad del sistema y sus restricciones. Context funciona bien cuando el estado es relativamente estático, tiene baja frecuencia de actualización y se consume dentro de un subárbol limitado. Sin embargo, a medida que las aplicaciones crecen, problemas como re-renderizados innecesarios, falta de herramientas de depuración y dificultad para coordinar actualizaciones entre funciones se vuelven más pronunciados. Redux, particularmente con Redux Toolkit, aborda estos desafíos introduciendo transiciones de estado estructuradas, lógica centralizada y herramientas para desarrolladores. La elección debe guiarse por cuán predecible, observable y escalable debe ser tu estado.

#### Redux sobresale en escenarios que requieren:
- Lógica de estado compleja con muchas acciones
- Capacidades de depuración con viajes en el tiempo
- Middleware para efectos secundarios
- Actualizaciones de estado predecibles en equipos grandes
- Integración con dev tools para depuración

#### Context API es ideal para:
- Gestión de temas
- Estado de autenticación del usuario
- Ajustes de localización
- Aplicaciones pequeñas a medianas
- Prevención de prop drilling en árboles de componentes

La decisión a menudo se reduce al tamaño y la complejidad de la aplicación. Context API se ha vuelto lo suficientemente potente para muchos casos de uso, pero las funciones de Redux Toolkit lo hacen invaluable para aplicaciones a gran escala.

Para redondear esto, existe un camino intermedio pragmático antes de saltar a Redux completo: **Zustand**. Si Context comienza a crujir (granularidad de selectores, coordinación entre slices o estado derivado) pero Redux se siente pesado, Zustand te ofrece suscripciones basadas en selectores y stores que evitan re-renderizados por defecto, mínimo código repetitivo, inferencia de TypeScript de primer nivel y middlewares fáciles (`persist`, `immer`, `devtools`) cuando los necesitas. Es ideal para una complejidad media: stores ubicables por dominio, sincronización opcional con servidor/caché en Next.js y una ruta de migración suave: comienza en pequeño junto a tu Context existente y luego traslada la lógica compartida a un store enfocado de Zustand a medida que crezcan los requerimientos.

---

## Zustand: gestión de estado ligera para aplicaciones modernas

Zustand ofrece un término medio entre la Context API y Redux, proporcionando una solución de gestión de estado simple pero potente. Combina lo mejor de ambos mundos con mínimo código repetitivo y un excelente soporte de TypeScript.

### Comenzando con Zustand

Creemos una tienda de carrito de compras con Zustand:

```ts
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';

interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

interface CartStore {
  items: CartItem[];
  total: number;
  addItem: (item: Omit<CartItem, 'quantity'>) => void;
  removeItem: (id: string) => void;
  updateQuantity: (id: string, quantity: number) => void;
  clearCart: () => void;
}

const useCartStore = create<CartStore>()(
  devtools(
    persist(
      (set) => ({
        items: [],
        total: 0,
        addItem: (item) => set((state) => {
          const existingItem = state.items.find(i => i.id === item.id);
         
          if (existingItem) {
            return {
              items: state.items.map(i =>
                i.id === item.id
                  ? { ...i, quantity: i.quantity + 1 }
                  : i
              ),
              total: state.total + item.price,
            };
          }
         
          return {
            items: [...state.items, { ...item, quantity: 1 }],
            total: state.total + item.price,
          };
        }),
        removeItem: (id) => set((state) => {
          const item = state.items.find(i => i.id === id);
          if (!item) return state;
         
          return {
            items: state.items.filter(i => i.id !== id),
            total: state.total - (item.price * item.quantity),
          };
        }),
        updateQuantity: (id, quantity) => set((state) => {
          if (quantity <= 0) {
            return state;
          }
         
          const item = state.items.find(i => i.id === id);
          if (!item) return state;
         
          const quantityDiff = quantity - item.quantity;
         
          return {
            items: state.items.map(i =>
              i.id === id ? { ...i, quantity } : i
            ),
            total: state.total + (item.price * quantityDiff),
          };
        }),
        clearCart: () => set({ items: [], total: 0 }),
      }),
      {
        name: 'cart-storage',
      }
    )
  )
);
```

Este ejemplo demuestra las características centrales de Zustand: `create()` inicializa el store con el estado y las acciones en un solo objeto, `set()` actualiza el estado de forma inmutable y activa automáticamente los re-renderizados en los componentes suscritos, el middleware `devtools()` se integra con Redux DevTools para depuración, y el middleware `persist()` guarda el carrito en `localStorage`. Los componentes consumen el store con `const { items, addItem } = useCartStore()`, y Zustand maneja la gestión de suscripciones; los componentes solo se re-renderizan cuando cambian sus slices de estado específicos.

### Patrones avanzados de Zustand

Exploremos patrones más avanzados con acciones asíncronas y selectores:

```ts
import { create } from 'zustand';
import { subscribeWithSelector } from 'zustand/middleware';

interface User {
  id: string;
  name: string;
  email: string;
}

interface AuthStore {
  user: User | null;
  isLoading: boolean;
  error: string | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  checkAuth: () => Promise<void>;
}

const useAuthStore = create<AuthStore>()(
  subscribeWithSelector((set, get) => ({
    user: null,
    isLoading: false,
    error: null,
    login: async (email, password) => {
      set({ isLoading: true, error: null });
     
      try {
        const response = await fetch('/api/auth/login', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ email, password }),
        });
       
        if (!response.ok) {
          throw new Error('Invalid credentials');
        }
       
        const user = await response.json();
        set({ user, isLoading: false });
      } catch (error) {
        set({
          error: error instanceof Error ? error.message : 'Login failed',
          isLoading: false,
        });
      }
    },
    logout: () => {
      set({ user: null });
      fetch('/api/auth/logout', { method: 'POST' });
    },
    checkAuth: async () => {
      try {
        const response = await fetch('/api/auth/me');
        if (response.ok) {
          const user = await response.json();
          set({ user });
        }
      } catch {
        // Silent fail - user not authenticated
      }
    },
  }))
);

// Selectors for fine-grained subscriptions
export const useUser = () => useAuthStore((state) => state.user);
export const useIsAuthenticated = () => useAuthStore((state) => !!state.user);
export const useAuthActions = () => useAuthStore((state) => ({
  login: state.login,
  logout: state.logout,
  checkAuth: state.checkAuth,
}));
```

Este código muestra prácticas avanzadas de Zustand: suscripciones basadas en selectores a través de `subscribeWithSelector` y pequeños hooks (`useUser`, `useIsAuthenticated`, `useAuthActions`) para evitar re-renderizados innecesarios; una separación limpia entre estado y acciones para que los componentes puedan leer datos o llamar a comandos sin acoplamiento; efectos asíncronos encapsulados (`login`, `logout`, `checkAuth`) que gestionan los ciclos de vida de obtención en el store; una máquina de estados ligera utilizando `user`/`isLoading`/`error`; actualizaciones parciales con `set` para limitar notificaciones; normalización de errores para mensajes amigables en la UI; rehidratación de sesión con `checkAuth` al cargar; selectores derivados (convirtiendo `user` en booleano) para dependencias estables y mínimas; y selección de acciones estable para que los consumidores de solo acciones no se vuelvan a renderizar, manteniendo el store listo para middlewares como `persist`, `immer` o `devtools` más adelante.

Los selectores permiten que los componentes se suscriban únicamente a partes específicas del estado, evitando re-renderizados innecesarios.

### Combinación de stores y slices de estado

Zustand facilita la combinación de múltiples stores:

```ts
interface AppStore {
  cart: CartStore;
  auth: AuthStore;
  ui: UIStore;
}

interface UIStore {
  sidebarOpen: boolean;
  theme: 'light' | 'dark';
  toggleSidebar: () => void;
  setTheme: (theme: 'light' | 'dark') => void;
}

const useUIStore = create<UIStore>((set) => ({
  sidebarOpen: false,
  theme: 'light',
  toggleSidebar: () => set((state) => ({
    sidebarOpen: !state.sidebarOpen
  })),
  setTheme: (theme) => set({ theme }),
}));

// Combine stores using a custom hook
function useStore() {
  const cart = useCartStore();
  const auth = useAuthStore();
  const ui = useUIStore();
 
  return { cart, auth, ui };
}
```

Este patrón proporciona una forma limpia de organizar y acceder a diferentes partes del estado de tu aplicación.

Al componer stores enfocados de Zustand (`cart`, `auth`, `ui`) y exponerlos a través de una fachada delgada `useStore()`, mantienes los dominios independientes, permites suscripciones dirigidas por selectores y preservas la capacidad de prueba y el margen de escalabilidad (se pueden agregar middlewares como `persist` o `devtools` por slice sin acoplamiento); este diseño modular también hace que la promoción/degradación de estado sea trivial: comienza en local, elévalo a un slice solo cuando múltiples funcionalidades lo necesiten, y prefiere selectores derivados antes que almacenar valores computados, preparándote perfectamente para la siguiente sección, comenzando con el principio de localidad del estado: mantén el estado lo más cerca posible de donde se utiliza y elévalo únicamente cuando el acceso compartido sea verdaderamente necesario.

---

## Mejores prácticas para gestionar el estado global y local

Elegir entre estado local y global no es una decisión binaria sino una compensación de diseño que depende del alcance, la propiedad y la reutilización. Muchos problemas de rendimiento y mantenibilidad surgen de promover el estado demasiado pronto o de mantenerlo local durante demasiado tiempo. Antes de aplicar mejores prácticas, es importante comprender el problema subyacente: quién es el propietario de este estado, cuántos componentes dependen de él y con qué frecuencia cambia. Al razonar sobre estos factores, puedes tomar decisiones deliberadas sobre dónde debe residir el estado en lugar de depender de reglas rígidas.

### Principio de localidad del estado

Mantén el estado lo más cerca posible de donde se usa. Solo eleva el estado cuando múltiples componentes necesiten acceso:

```tsx
// Good: Local state for form inputs
function ContactForm() {
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    message: ''
  });
 
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    await sendMessage(formData);
    setFormData({ name: '', email: '', message: '' });
  };
 
  return (
    <form onSubmit={handleSubmit}>
      <input
        value={formData.name}
        onChange={(e) => setFormData({
          ...formData,
          name: e.target.value
        })}
      />
      {/* Other inputs */}
    </form>
  );
}

// Good: Elevated state when needed by siblings
function ProductPage() {
  const [selectedVariant, setSelectedVariant] = useState<string>('default');
 
  return (
    <>
      <ProductImages variant={selectedVariant} />
      <ProductDetails
        variant={selectedVariant}
        onVariantChange={setSelectedVariant}
      />
      <ProductReviews variant={selectedVariant} />
    </>
  );
}
```

Mantener el estado cerca de sus consumidores reduce el alcance de renderizado, el acoplamiento y la carga cognitiva. `ContactForm` mantiene el estado transitorio de las entradas dentro del componente, por lo que solo el formulario se vuelve a renderizar con las pulsaciones de teclas y nada más depende de esa estructura. En `ProductPage`, elevar `selectedVariant` solo un nivel permite la coordinación entre componentes hermanos (`Images`/`Details`/`Reviews`) sin globalizarlo, logrando una propiedad clara, un radio de impacto mínimo y una fácil comprobabilidad. El patrón: comienza en local; eleva solo cuando dos o más hermanos o rutas lo necesiten.

### Manejo del estado derivado

Evita almacenar estado derivado. Calcúlalo bajo demanda o utiliza memoización:

```ts
// Bad: Storing derived state
interface BadCartState {
  items: CartItem[];
  total: number; // Derived from items
  itemCount: number; // Derived from items
  hasDiscount: boolean; // Derived from total
}

// Good: Computing derived values
function useCart() {
  const items = useCartStore(state => state.items);
 
  const derivedValues = useMemo(() => {
    const total = items.reduce((sum, item) =>
      sum + item.price * item.quantity, 0
    );
   
    return {
      total,
      itemCount: items.length,
      hasDiscount: total > 100,
      isEmpty: items.length === 0,
    };
  }, [items]);
 
  return { items, ...derivedValues };
}

// For complex computations, use selector patterns
const selectCartSummary = (state: CartState) => {
  const subtotal = state.items.reduce(
    (sum, item) => sum + item.price * item.quantity,
    0
  );
  const tax = subtotal * 0.08;
  const shipping = subtotal > 50 ? 0 : 10;
 
  return {
    subtotal,
    tax,
    shipping,
    total: subtotal + tax + shipping,
  };
};
```

Almacenar derivaciones (como `total`, `itemCount`) crea desincronización (*drift*) y condiciones de carrera porque debes mantener sincronizadas dos fuentes de datos. El mal ejemplo muestra ese riesgo. El buen `useCart` calcula los totales en un único `useMemo` indexado por `items` para que el recálculo sea determinista y económico. El selector `selectCartSummary` lleva esto aún más lejos: las derivaciones residen en selectores (o getters computados), no en el store, brindándote funciones puras, pruebas sencillas y suscripciones de grano fino (solo los consumidores del resumen se vuelven a renderizar).

### Estrategias de persistencia del estado

Implementa persistencia selectiva para una mejor experiencia de usuario:

```ts
// Selective persistence with Zustand
interface AppSettings {
  theme: 'light' | 'dark';
  language: string;
  notifications: boolean;
}

interface SessionData {
  lastVisited: string;
  cartId: string;
  tempPreferences: Record<string, any>;
}

const useSettingsStore = create<AppSettings>()(
  persist(
    (set) => ({
      theme: 'light',
      language: 'en',
      notifications: true,
    }),
    {
      name: 'app-settings',
      // Persist only specific fields
      partialize: (state) => ({
        theme: state.theme,
        language: state.language,
      }),
    }
  )
);

// Session storage for temporary data
const useSessionStore = create<SessionData>()(
  persist(
    (set) => ({
      lastVisited: '/',
      cartId: '',
      tempPreferences: {},
    }),
    {
      name: 'session-data',
      storage: createJSONStorage(() => sessionStorage),
    }
  )
);
```

La persistencia selectiva mejora la experiencia de usuario (recordar tema/idioma) sin filtrar datos confidenciales o volátiles. En `useSettingsStore`, `persist` y `partialize` colocan en lista blanca únicamente el tema y el idioma, evitando el almacenamiento accidental de campos ruidosos o privados (principio de menor persistencia). `useSessionStore` utiliza intencionalmente `sessionStorage` para datos efímeros (se borra al cerrar la pestaña), evitando artefactos de larga duración.

Encripta o evita persistir secretos, versiona las claves (`name`) y prefiere valores predeterminados seguros para la rehidratación para evitar saltos visuales de diseño (*layout jank*) en el primer renderizado.

### Pruebas de la gestión del estado

Escribe una lógica de estado comprobable separando la lógica de negocio de la interfaz de usuario:

```ts
// Testable state logic
export const cartReducer = {
  addItem: (state: CartState, item: Product): CartState => {
    const existingItem = state.items.find(i => i.id === item.id);
   
    if (existingItem) {
      return {
        ...state,
        items: state.items.map(i =>
          i.id === item.id
            ? { ...i, quantity: i.quantity + 1 }
            : i
        ),
      };
    }
    return {
      ...state,
      items: [...state.items, { ...item, quantity: 1 }],
    };
  },
  removeItem: (state: CartState, itemId: string): CartState => ({
    ...state,
    items: state.items.filter(item => item.id !== itemId),
  }),
};

// Test the logic
describe('cartReducer', () => {
  it('should add new item to cart', () => {
    const initialState: CartState = { items: [] };
    const product = { id: '1', name: 'Test', price: 10 };
    const newState = cartReducer.addItem(initialState, product);
    expect(newState.items).toHaveLength(1);
    expect(newState.items[0]).toEqual({ ...product, quantity: 1 });
  });
 
  it('should increment quantity for existing item', () => {
    const initialState: CartState = {
      items: [{ id: '1', name: 'Test', price: 10, quantity: 1 }],
    };
    const product = { id: '1', name: 'Test', price: 10 };
    const newState = cartReducer.addItem(initialState, product);
    expect(newState.items[0].quantity).toBe(2);
  });
});
```

Separar la lógica de negocio de la UI hace que la corrección sea medible. `cartReducer` es un módulo puro: dado `(state, actionLikePayload) → newState`. Las pruebas demuestran comportamientos idempotentes (`add`, `increment`) sin montar React. Los beneficios son pruebas rápidas, deterministas, refactorizaciones sencillas (intercambiar Zustand/Redux/Context) mientras se preserva el comportamiento, y un rastro de auditoría claro para casos límite complejos (por ejemplo, cambios de precios, límites de inventario) que puedes codificar como especificaciones adicionales del reducer.

### Estrategias de migración

A medida que las aplicaciones evolucionan, es común superar un enfoque inicial de gestión de estado. Por ejemplo, un proyecto puede comenzar con Context por simplicidad, pero luego requerir suscripciones más granulares, mejor rendimiento o herramientas para desarrolladores mejoradas. Migrar de un enfoque a otro, como de Context a Zustand o Redux Toolkit, puede ser riesgoso si se hace abruptamente. El objetivo es realizar la transición de forma incremental manteniendo la aplicación estable. Los patrones de adaptadores proporcionan una ruta de migración segura al permitir que ambos sistemas coexistan temporalmente, asegurando que los componentes existentes sigan funcionando mientras que los nuevos adoptan la arquitectura actualizada:

```tsx
// Adapter for migrating from Context to Zustand
interface LegacyContextValue {
  user: User | null;
  setUser: (user: User | null) => void;
}

// Bridge hook during migration
function useLegacyAuth(): LegacyContextValue {
  const user = useAuthStore(state => state.user);
  const setUser = useAuthStore(state => state.setUser);
 
  return { user, setUser };
}

// Gradual migration approach
function AuthProvider({ children }: { children: React.ReactNode }) {
  const legacyValue = useLegacyAuth();
 
  return (
    <LegacyAuthContext.Provider value={legacyValue}>
      {children}
    </LegacyAuthContext.Provider>
  );
}
```

Los adaptadores te permiten evolucionar la arquitectura sin una fecha límite de cambio total (*flag day*). `useLegacyAuth()` conecta Zustand con el Context anterior a través de la misma estructura `{ user, setUser }`. Envolver el árbol con `LegacyAuthContext.Provider` permite que los consumidores existentes sigan funcionando mientras que el código nuevo utiliza el store. Esto permite migraciones progresivas mediante el patrón *strangler*: mueve las funciones slice por slice, mantén estables los contratos públicos y elimina el adaptador una vez completada la adopción. Una ventaja adicional es que puedes ejecutar ambos sistemas durante despliegues canary o pruebas A/B.

---

## Resumen

La gestión del estado en el React moderno no consiste en elegir una sola herramienta, sino en comprender la naturaleza del estado con el que estás trabajando. El estado local de la UI, los datos asíncronos del servidor, el estado compartido de la aplicación y los cálculos sensibles al rendimiento imponen restricciones diferentes. A lo largo de este capítulo, exploramos cómo las primitivas de React 19 como `use()` y `useDeferredValue` abordan desafíos específicos, y cómo herramientas externas como Redux Toolkit y Zustand extienden estas capacidades cuando las aplicaciones escalan. La clave no es la herramienta en sí, sino el razonamiento detrás de cuándo y por qué usarla.

Recuerda estos principios clave:

- Mantén el estado local cuando sea posible.
- Optimiza el rendimiento con memoización y suscripciones selectivas.
- Organiza el estado por funcionalidad, no por tipo.
- Prueba tu lógica de estado independientemente de los componentes de la interfaz de usuario.
- Elige la estrategia de persistencia adecuada para tus datos.

A medida que construyas aplicaciones más complejas, descubrirás que combinar estas técnicas crea soluciones de gestión de estado potentes y mantenibles que escalan con las necesidades de tu aplicación.

A continuación, cambiaremos de perspectiva: en lugar de ver qué hacer, identificaremos los errores que deterioran silenciosamente el rendimiento y la mantenibilidad. Al reconocer estos antipatrones de React y las alternativas de mejores prácticas, convertirás la intuición adquirida con esfuerzo en hábitos repetibles y amigables para el equipo.
