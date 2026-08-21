# Capítulo 5: Antipatrones y Mejores Prácticas en React

La flexibilidad y el poder de React vienen acompañados de la responsabilidad de comprender sus principios fundamentales. Los antipatrones son prácticas de programación comunes que inicialmente parecen ser soluciones efectivas, pero que en última instancia conducen a un código problemático que es difícil de mantener, depurar o escalar. En el desarrollo con React, estos antipatrones a menudo se manifiestan como cuellos de botella en el rendimiento, re-renderizados innecesarios, fugas de memoria y un código del que resulta cada vez más difícil razonar a medida que las aplicaciones crecen.

La distinción entre un antipatrón y una mejor práctica a menudo radica en comprender el algoritmo de reconciliación de React, el ciclo de vida de los componentes y el principio fundamental del flujo de datos unidireccional. Lo que podría funcionar en aplicaciones de JavaScript tradicionales puede volverse problemático en el paradigma declarativo de React. Este capítulo explora los antipatrones más frecuentes en el desarrollo con React, explicando no solo qué evitar, sino por qué estos patrones son problemáticos y cómo implementar mejores alternativas.

Comprender estos antipatrones es crucial por varias razones:

- **Prevenir problemas de rendimiento a escala:** Detectar patrones que parecen funcionar bien localmente pero colapsan con usuarios y datos reales.
- **Mejorar la mantenibilidad:** Establecer patrones claros y consistentes que los equipos puedan leer, probar y modificar con confianza.
- **Alinearse con la filosofía de React:** Escribir código que funcione en armonía con las partes internas de React para que puedas adoptar nuevas funciones y optimizaciones más fácilmente.

En este capítulo se cubrirán los siguientes temas:

- Antipatrones comunes de estado y props
- Uso excesivo del estado global cuando el estado local es suficiente
- Errores en el manejo de keys y listas
- Manejo de eventos y problemas de rendimiento
- Estructura de componentes y re-renderizados

---

## Antipatrones comunes de estado y props

Incluso los desarrolladores experimentados caen en trampas al gestionar la relación entre props y estado. Exploremos los antipatrones más comunes y aprendamos cómo evitarlos.

### Inicialización del estado mediante props en lugar de `useState`

Uno de los antipatrones más comunes consiste en inicializar el estado del componente directamente a partir de las props sin considerar adecuadamente cómo gestiona React las actualizaciones de los componentes. Este patrón a menudo surge cuando los desarrolladores intentan crear componentes controlados pero inadvertidamente crean problemas de sincronización entre props y estado.

Considera esta implementación problemática donde el estado se inicializa incorrectamente desde las props:

```tsx
// Anti-pattern: Directly using props to initialize state
interface UserProfileProps {
  initialName: string;
  initialEmail: string;
  onSave: (name: string, email: string) => void;
}

const UserProfileBad = ({
  initialName,
  initialEmail,
  onSave
}: UserProfileProps) => {
  // This creates a synchronization problem
  const [name, setName] = useState(initialName)
  const [email, setEmail] = useState(initialEmail)
 
  // The state won't update when props change
  // This component becomes "stuck" with the initial values
 
  return (
    <form onSubmit={(e) => {
      e.preventDefault()
      onSave(name, email)
    }}>
      <input
        value={name}
        onChange={(e) => setName(e.target.value)}
      />
      <input
        value={email}
        onChange={(e) => setEmail(e.target.value)}
      />
      <button type="submit">Save</button>
    </form>
  )
}
```

El fallo se hace visible cuando el padre actualiza las props después de que el componente se ha montado. Por ejemplo, si el padre obtiene nuevos datos de usuario y actualiza `initialName`, el campo de entrada continuará mostrando el valor anterior porque el estado solo se inicializó una vez. Esto crea un error sutil pero crítico: la UI parece interactiva, pero ahora está desconectada de la fuente de la verdad. Los usuarios pueden editar datos obsoletos, enviar valores incorrectos o perder actualizaciones por completo. La causa raíz es que `useState` no reacciona a los cambios de props después de la inicialización, lo que hace que este patrón sea intrínsecamente inseguro a menos que la sincronización se maneje explícitamente.

Aquí está el enfoque corregido utilizando patrones adecuados para manejar los cambios de props:

```tsx
// Best practice: Properly handling prop changes
interface UserProfileProps {
  name: string;
  email: string;
  onNameChange: (name: string) => void;
  onEmailChange: (email: string) => void;
  onSave: () => void;
}

const UserProfileGood = ({
  name,
  email,
  onNameChange,
  onEmailChange,
  onSave
}: UserProfileProps) => {
  // Fully controlled component - state managed by parent
  return (
    <form onSubmit={(e) => {
      e.preventDefault()
      onSave()
    }}>
      <input
        value={name}
        onChange={(e) => onNameChange(e.target.value)}
      />
      <input
        value={email}
        onChange={(e) => onEmailChange(e.target.value)}
      />
      <button type="submit">Save</button>
    </form>
  )
}

// Alternative: Using useEffect to sync with props when needed
const UserProfileWithSync = ({ defaultName, defaultEmail, onSave, resetTrigger }: {
  defaultName: string;
  defaultEmail: string;
  onSave: (name: string, email: string) => void;
  resetTrigger?: number;
}) => {
  const [name, setName] = useState(defaultName)
  const [email, setEmail] = useState(defaultEmail)
 
  // Explicitly handle prop changes when needed
  useEffect(() => {
    if (resetTrigger !== undefined) {
      setName(defaultName)
      setEmail(defaultEmail)
    }
  }, [resetTrigger, defaultName, defaultEmail])
 
  return (
    <form onSubmit={(e) => {
      e.preventDefault()
      onSave(name, email)
    }}>
      <input value={name} onChange={(e) => setName(e.target.value)} />
      <input value={email} onChange={(e) => setEmail(e.target.value)} />
      <button type="submit">Save</button>
    </form>
  )
}
```

La diferencia clave en los enfoques corregidos radica en cómo se manejan la propiedad del estado y la sincronización. En la versión totalmente controlada, el componente ya no es el propietario del estado, eliminando cualquier posibilidad de divergencia entre las props y el estado interno. En la versión semicontrolada, `resetTrigger` actúa como una señal explícita del padre para resincronizar el estado local con las props entrantes. Esto evita el acoplamiento implícito y hace que las actualizaciones sean predecibles: las ediciones locales permanecen estables durante la interacción del usuario y solo se restablecen cuando el padre lo solicita explícitamente. Sin este disparador explícito, sincronizar a ciegas las props con el estado sobrescribiría la entrada del usuario y crearía una mala experiencia de usuario.

---

## Uso excesivo del estado global cuando el estado local es suficiente

También es importante distinguir el Contexto nativo de React de las librerías de estado basadas en selectores como Redux y Zustand. Con el Contexto nativo, cualquier cambio en el valor del proveedor puede hacer que todos los componentes consumidores se vuelvan a renderizar, incluso si un componente solo usa una pequeña parte de ese valor. Redux y Zustand evitan este problema mediante el uso de selectores: los componentes se suscriben únicamente a la porción específica de estado que leen, y solo se re-renderizan cuando ese valor seleccionado cambia. Esto significa que la advertencia de re-renderizado amplio de Context no se aplica a Redux o Zustand de la misma manera, asumiendo que los selectores se utilicen correctamente y los valores seleccionados permanezcan referencialmente estables.

Aquí hay un ejemplo concreto del antipatrón en acción: una funcionalidad simple y totalmente local (alternar un modal) se eleva a un contexto global, forzando re-renderizados en toda la aplicación y acoplando componentes no relacionados:

```tsx
// Anti-pattern: Using global state for local UI state
interface GlobalUIState {
  isModalOpen: boolean;
  selectedTab: string;
  searchQuery: string;
  formData: Record<string, any>;
}

// Unnecessary global context for UI state
const UIContext = React.createContext<{
  state: GlobalUIState;
  setState: React.Dispatch<React.SetStateAction<GlobalUIState>>;
} | null>(null)

const BadModalComponent = () => {
  const context = useContext(UIContext)
  if (!context) throw new Error('UIContext required')
 
  const { state, setState } = context
 
  // This causes every component using UIContext to re-render
  // when the modal state changes
  return (
    <div>
      <button onClick={() =>
        setState(prev => ({ ...prev, isModalOpen: true }))
      }>
        Open Modal
      </button>
      {state.isModalOpen && (
        <div className="modal">
          <button onClick={() =>
            setState(prev => ({ ...prev, isModalOpen: false }))
          }>
            Close
          </button>
        </div>
      )}
    </div>
  )
}
```

El problema aquí es que el estado del modal, que solo es relevante para el componente modal y su padre inmediato, se está elevando al estado global. Esto causa re-renderizados innecesarios en toda la aplicación cada vez que el modal se abre o se cierra. Además, complica las pruebas ya que ahora necesitas proporcionar el contexto en tus tests.

Aquí está el enfoque mejorado utilizando estado local:

```tsx
// Best practice: Keep local state local
const GoodModalComponent = () => {
  const [isOpen, setIsOpen] = useState(false)
 
  return (
    <div>
      <button onClick={() => setIsOpen(true)}>
        Open Modal
      </button>
      {isOpen && (
        <Modal onClose={() => setIsOpen(false)} />
      )}
    </div>
  )
}

interface ModalProps {
  onClose: () => void
}

const Modal = ({ onClose }: ModalProps) => {
  // Modal can manage its own internal state
  const [formData, setFormData] = useState({ name: '', email: '' })
 
  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault()
    // Process form data
    console.log('Submitted:', formData)
    onClose()
  }
 
 
  return (
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        <form onSubmit={handleSubmit}>
          <input
            value={formData.name}
            onChange={(e) =>
              setFormData(prev => ({ ...prev, name: e.target.value }))
            }
            placeholder="Name"
          />
          <input
            value={formData.email}
            onChange={(e) =>
              setFormData(prev => ({ ...prev, email: e.target.value }))
            }
            placeholder="Email"
          />
          <button type="submit">Submit</button>
          <button type="button" onClick={onClose}>Cancel</button>
        </form>
      </div>
    </div>
  )
}
```

La versión mejorada mantiene el estado donde realmente se necesita. El estado abierto/cerrado del modal es gestionado por su componente padre, y el estado del formulario interno del modal se gestiona dentro del propio modal. Esto crea límites claros de responsabilidad y hace que los componentes sean más reutilizables y comprobables.

### Mutar el estado directamente en lugar de usar `setState`

La mutación directa del estado es uno de los antipatrones más peligrosos en React porque viola el principio fundamental de inmutabilidad de React. Cuando mutas el estado directamente, el algoritmo de reconciliación de React no puede detectar los cambios, lo que lleva a componentes que no se re-renderizan cuando deberían, creando errores que son difíciles de rastrear:

```tsx
// Anti-pattern: Directly mutating state
interface Todo {
  id: number;
  text: string;
  completed: boolean;
}

const TodoListBad = () => {
  const [todos, setTodos] = useState<Todo[]>([
    { id: 1, text: 'Learn React', completed: false },
    { id: 2, text: 'Build an app', completed: false }
  ])
 
  const toggleTodoBad = (id: number) => {
    // This mutates the existing array and objects
    const todo = todos.find(t => t.id === id)
    if (todo) {
      todo.completed = !todo.completed // Direct mutation!
      setTodos(todos) // Same reference, no re-render
    }
  }
 
  const addTodoBad = (text: string) => {
    // This mutates the existing array
    todos.push({ id: Date.now(), text, completed: false })
    setTodos(todos) // Same reference, no re-render
  }
 
  return (
    <div>
      {todos.map(todo => (
        <div key={todo.id}>
          <input
            type="checkbox"
            checked={todo.completed}
            onChange={() => toggleTodoBad(todo.id)}
          />
          <span>{todo.text}</span>
        </div>
      ))}
    </div>
  )
}
```

Las mutaciones en este código no activarán re-renderizados porque React utiliza la comparación `Object.is()` para determinar si el estado ha cambiado. Dado que estamos pasando la misma referencia de arreglo a `setTodos`, React asume que nada ha cambiado. Esto da como resultado que la UI no se actualice a pesar de que los datos subyacentes sí cambiaron.

Aquí está el enfoque correcto utilizando actualizaciones inmutables:

```tsx
// Best practice: Immutable state updates
const TodoListGood = () => {
  const [todos, setTodos] = useState<Todo[]>([
    { id: 1, text: 'Learn React', completed: false },
    { id: 2, text: 'Build an app', completed: false }
  ])
 
  const toggleTodo = (id: number) => {
    // Create a new array with updated objects
    setTodos(prevTodos =>
      prevTodos.map(todo =>
        todo.id === id
          ? { ...todo, completed: !todo.completed } // New object
          : todo
      )
    )
  }
 
  const addTodo = (text: string) => {
    // Create a new array with the new item
    setTodos(prevTodos => [
      ...prevTodos,
      { id: Date.now(), text, completed: false }
    ])
  }
 
  const removeTodo = (id: number) => {
    // Create a new array without the removed item
    setTodos(prevTodos => prevTodos.filter(todo => todo.id !== id));
  }
 
  const updateTodoText = (id: number, newText: string) => {
    // Complex updates remain immutable
    setTodos(prevTodos =>
      prevTodos.map(todo =>
        todo.id === id
          ? { ...todo, text: newText }
          : todo
      )
    )
  }
 
  return (
    <div>
      {todos.map(todo => (
        <TodoItem
          key={todo.id}
          todo={todo}
          onToggle={() => toggleTodo(todo.id)}
          onRemove={() => removeTodo(todo.id)}
          onUpdateText={(text) => updateTodoText(todo.id, text)}
        />
      ))}
      <AddTodoForm onAdd={addTodo} />
    </div>
  )
}
```

La versión corregida crea nuevos arreglos y objetos para cada actualización de estado, asegurando que React pueda detectar cambios y activar re-renderizados adecuadamente. Este patrón se extiende a objetos y arreglos anidados, donde debes crear nuevas referencias en cada nivel que cambie.

---

## Errores en el manejo de keys y listas

Las keys juegan un papel crítico en el proceso de reconciliación de React, ayudando al framework a actualizar listas de manera eficiente al rastrear qué elementos han cambiado, se han agregado o se han eliminado. Sin embargo, los errores relacionados con las keys se encuentran entre los antipatrones más comunes en el desarrollo con React, lo que a menudo genera errores sutiles que solo aparecen durante interacciones específicas del usuario. Estos problemas pueden provocar una preservación incorrecta del estado, bajo rendimiento y un comportamiento impredecible del componente. En esta sección, examinaremos los errores más frecuentes en el manejo de keys y listas y aprenderemos cómo evitarlos.

### Uso de índices como keys en listas (por qué es una mala idea)

Usar índices de arreglo como keys en listas dinámicas es uno de los antipatrones más comunes de React. Aunque puede parecer conveniente y aparenta funcionar en casos simples, puede provocar errores sutiles, preservación incorrecta del estado del componente y problemas de rendimiento cuando la lista cambia:

```tsx
// Anti-pattern: Using index as key in dynamic lists
interface ListItem {
  id: string;
  name: string;
  description: string;
}

const DynamicListBad = ({ items }: { items: ListItem[] }) => {
  const [expandedItems, setExpandedItems] = useState<Set<number>>(new Set())
 
  const toggleExpanded = (index: number) => {
    setExpandedItems(prev => {
      const newSet = new Set(prev)
      if (newSet.has(index)) {
        newSet.delete(index)
      } else {
        newSet.add(index)
      }
      return newSet
    })
  }
 
  return (
    <ul>
      {items.map((item, index) => (
        // Using index as key causes problems
        <li key={index}>
          <div onClick={() => toggleExpanded(index)}>
            <h3>{item.name}</h3>
            {expandedItems.has(index) && (
              <p>{item.description}</p>
            )}
          </div>
          <input type="text" placeholder="Add note..." />
        </li>
      ))}
    </ul>
  )
}
```

El problema con las keys basadas en índices se hace evidente cuando se modifica la lista. Considera un escenario donde un usuario escribe en el segundo campo de entrada y luego elimina el primer elemento. Debido a que React usa el índice como identidad, el segundo elemento ahora pasa al índice 0 y React reutiliza el nodo DOM de manera incorrecta. Como resultado, el valor de entrada parece saltar a otro elemento. Esto no es solo un problema teórico; conduce directamente a un estado de UI corrupto donde la entrada del usuario se asocia con los datos incorrectos. El problema no es la lista en sí, sino la identidad inestable causada por las keys basadas en índices:

- **El estado se desalinea:** Eliminar un elemento puede desplazar el estado preservado (por ejemplo, el valor de un input de la fila eliminada aparece en la siguiente fila).
- **Las animaciones/transiciones se rompen:** React no puede saber qué elementos se agregaron/eliminaron/reordenaron, por lo que las animaciones de entrada/salida/movimiento fallan.
- **Las optimizaciones de rendimiento se degradan:** Sin una identidad estable, React no puede calcular actualizaciones mínimas del DOM de manera eficiente.

Aquí hay una demostración del problema y la solución correcta:

```tsx
// Best practice: Using stable, unique IDs as keys
const DynamicListGood = ({ items }: { items: ListItem[] }) => {
  // Use item IDs instead of indexes for tracking expanded state
  const [expandedItems, setExpandedItems] = useState<Set<string>>(new Set())
  const [itemNotes, setItemNotes] = useState<Record<string, string>>({})
 
  const toggleExpanded = (itemId: string) => {
    setExpandedItems(prev => {
      const newSet = new Set(prev)
      if (newSet.has(itemId)) {
        newSet.delete(itemId)
      } else {
        newSet.add(itemId)
      }
      return newSet
    })
  }
 
  const updateNote = (itemId: string, note: string) => {
    setItemNotes(prev => ({
      ...prev,
      [itemId]: note
    }))
  }
 
  return (
    <ul>
      {items.map((item) => (
        // Using stable ID as key
        <li key={item.id}>
          <div onClick={() => toggleExpanded(item.id)}>
            <h3>{item.name}</h3>
            {expandedItems.has(item.id) && (
              <p>{item.description}</p>
            )}
          </div>
          <input
            type="text"
            placeholder="Add note..."
            value={itemNotes[item.id] || ''}
            onChange={(e) => updateNote(item.id, e.target.value)}
          />
        </li>
      ))}
    </ul>
  )
}

// Example of generating stable keys when items don't have IDs
interface RawDataItem {
  name: string;
  value: number;
}

const ListWithGeneratedKeys = ({ data }: { data: RawDataItem[] }) => {
  // Generate stable IDs once when data changes
  const itemsWithIds = useMemo(() =>
    data.map(d => ({
      ...item,
      id: `${d.name}-${d.value}-${Math.random().toString(36).substr(2, 9)}`
    })),
    [data]
  )
 
  return (
    <ul>
      {itemsWithIds.map(item => (
        <li key={item.id}>
          {item.name}: {item.value}
        </li>
      ))}
    </ul>
  )
}
```

Evitar keys basadas en índices no se trata solo de corrección; protege la experiencia de usuario y el rendimiento a medida que la lista evoluciona. Prefiere IDs de elementos estables y únicos para que React pueda preservar el estado local, ejecutar animaciones fluidas y calcular actualizaciones mínimas del DOM. Cuando los datos carezcan de IDs, deriva una clave determinista (por ejemplo, a partir de un ID del backend o un hash estable de campos inmutables). Reserva las keys por índice únicamente para listas verdaderamente estáticas que nunca se reordenan.

### Mejores prácticas para keys estables y únicas

Una estrategia de keys confiable se basa fundamentalmente en preservar la identidad a través de los renderizados. React no compara elementos por posición, sino por key, lo que significa que las keys determinan si una instancia de componente se reutiliza o se reemplaza. Cuando las keys son inestables, React no puede asignar correctamente el estado anterior a nuevos elementos, lo que genera errores como entradas perdidas, animaciones rotas o actualizaciones innecesarias del DOM. Las keys estables y únicas garantizan que cada instancia de componente mantenga su identidad incluso cuando la lista circundante cambie, lo cual es esencial tanto para la corrección como para el rendimiento.

Las keys deben ser:
- **Estables:** Permanecer iguales entre renderizados para el mismo elemento.
- **Únicas:** Ningún par de elementos debe compartir la misma key.
- **Predecibles:** Los mismos datos siempre deben generar la misma key.

Para poner esto en práctica, aquí se muestra cómo aplicar keys estables en múltiples niveles: usando un ID único para elementos de nivel superior, una clave compuesta para variantes anidadas e IDs generados para campos de formulario creados por el usuario, de modo que la identidad permanezca consistente a medida que los datos cambian:

```tsx
interface Product {
  sku: string;
  name: string;
  variants?: Array<{
    color: string;
    size: string;
    stock: number;
  }>
}

const ProductList = ({ products }: { products: Product[] }) => {
  return (
    <div>
      {products.map(product => (
        // Primary key from unique identifier
        <div key={product.sku} className="product">
          <h2>{product.name}</h2>
          {product.variants && (
            <div className="variants">
              {product.variants.map(variant => (
                // Composite key for nested items
                <div key={`${product.sku}-${variant.color}-${variant.size}`}>
                  <span>{variant.color} / {variant.size}</span>
                  <span>Stock: {variant.stock}</span>
                </div>
              ))}
            </div>
          )}
        </div>
      ))}
    </div>
  )
}

// Handling dynamic forms with proper keys
interface FormField {
  id: string;
  type: 'text' | 'number' | 'select';
  label: string;
  value: any;
}

const DynamicForm = () => {
  const [fields, setFields] = useState<FormField[]>([])
 
  const addField = (type: FormField['type']) => {
    // Generate unique ID at creation time
    const newField: FormField = {
      id: `field-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
      type,
      label: `Field ${fields.length + 1}`,
      value: ''
    }
    setFields(prev => [...prev, newField])
  }
  const removeField = (fieldId: string) => {
    setFields(prev => prev.filter(field => field.id !== fieldId))
  }
 
  const updateField = (fieldId: string, value: any) => {
    setFields(prev =>
      prev.map(field =>
        field.id === fieldId ? { ...field, value } : field
      )
    )
  }
 
  return (
    <form>
      {fields.map(field => (
        <div key={field.id} className="form-field">
          <label>{field.label}</label>
          {field.type === 'select' ? (
            <select
              value={field.value}
              onChange={(e) => updateField(field.id, e.target.value)}
            >
              <option value="">Select...</option>
            </select>
          ) : (
            <input
              type={field.type}
              value={field.value}
              onChange={(e) => updateField(field.id, e.target.value)}
            />
          )}
          <button type="button" onClick={() => removeField(field.id)}>
            Remove
          </button>
        </div>
      ))}
      <button type="button" onClick={() => addField('text')}>
        Add Text Field
      </button>
    </form>
  )
}
```

Al trabajar con keys, recuerda que solo necesitan ser únicas entre elementos hermanos, no globalmente únicas en toda la aplicación. Esto significa que puedes reutilizar las mismas keys en diferentes partes de tu árbol de componentes sin problemas. Sin embargo, dentro de una sola lista, cada key debe ser única para garantizar que React pueda rastrear los cambios adecuadamente.

---

## Manejo de eventos y problemas de rendimiento

El manejo de eventos es fundamental para las aplicaciones interactivas de React, pero la forma en que definimos y adjuntamos los controladores de eventos afecta directamente el rendimiento. Crear nuevas instancias de funciones en cada renderizado, un error común al usar controladores en línea en JSX, causa re-renderizados innecesarios en componentes secundarios y rompe las optimizaciones de memoización. Estos problemas de rendimiento a menudo pasan desapercibidos con conjuntos de datos pequeños, pero se vuelven significativos a medida que las aplicaciones escalan para manejar listas más grandes e interacciones más complejas. En esta sección, examinaremos los antipatrones comunes de manejo de eventos y aprenderemos técnicas para mantener tanto la claridad del código como un rendimiento óptimo.

### Evitar la recreación innecesaria de funciones en JSX

Crear nuevas instancias de función en cada renderizado es un antipatrón de rendimiento común que puede provocar re-renderizados innecesarios en componentes hijos y sobrecarga de memoria. Esto sucede cuando los desarrolladores definen controladores de eventos en línea dentro de JSX, lo que provoca que se creen nuevas referencias de función en cada ciclo de renderizado:

```tsx
// Anti-pattern: Creating new functions on every render
interface ExpensiveChildProps {
  onClick: (id: string) => void
  data: string
}

const ExpensiveChild = React.memo<ExpensiveChildProps>(({ onClick, data }) => {
  console.log('ExpensiveChild rendered')
  // Imagine this component has expensive computations
  const processedData = useMemo(() => {
    // Expensive processing
    return data.split('').reverse().join('')
  }, [data])
 
  return (
    <div onClick={() => onClick(data)}>
      {processedData}
    </div>
  )
})
const ParentComponentBad = () => {
  const [items, setItems] = useState(['item1', 'item2', 'item3'])
  const [selectedItem, setSelectedItem] = useState<string>('')
 
  const handleClick = (id: string) => {
    setSelectedItem(id)
  }
 
  return (
    <div>
      {items.map(item => (
        <ExpensiveChild
          key={item}
          data={item}
          // Creating new function on every render breaks memoization
          onClick={(id) => handleClick(id)}
        />
      ))}
    </div>
  )
}
```

A pesar de que `ExpensiveChild` está envuelto con `React.memo`, se volverá a renderizar en cada renderizado del padre porque la prop `onClick` recibe una nueva referencia de función cada vez. Esto anula el propósito de la memoización y puede causar problemas significativos de rendimiento en listas con muchos elementos o componentes hijos complejos.

### Creación innecesaria de funciones anónimas en JSX

En proyectos de React 19 que utilizan React Compiler, muchos controladores y valores derivados se pueden memoizar automáticamente por el compilador, lo que reduce la necesidad de agregar `useCallback` o `useMemo` en todas partes de manera predeterminada. La memoización manual sigue siendo útil en casos específicos, especialmente cuando se trabaja con código no compilado, APIs de terceros que dependen de referencias estables o cálculos costosos que necesitan control explícito. Sin embargo, en código compilado de React, `useCallback` debe tratarse como una herramienta de optimización en lugar de un requisito rutinario:

```tsx
// Best practice: Properly memoized event handlers
const ParentComponentGood = () => {
  const [items, setItems] = useState(['item1', 'item2', 'item3'])
  const [selectedItem, setSelectedItem] = useState<string>('')
 
  // Memoize the event handler
  const handleClick = useCallback((id: string) => {
    setSelectedItem(id)
  }, []) // Empty deps array since setState is stable
 
  return (
    <div>
      {items.map(item => (
        <OptimizedChild
          key={item}
          data={item}
          onSelect={handleClick}
        />
      ))}
    </div>
  )
}

// Even better: Let child components handle their own events
interface OptimizedChildProps {
  data: string
  onSelect: (id: string) => void
}

const OptimizedChild = React.memo<OptimizedChildProps>(({ data, onSelect }) => {
  // Handle event binding inside the component
  const handleClick = useCallback(() => {
    onSelect(data)
  }, [data, onSelect])
 
  const processedData = useMemo(() => {
    return data.split('').reverse().join('')
  }, [data])
 
  return (
    <div onClick={handleClick}>
      {processedData}
    </div>
  )
})

// Alternative pattern using data attributes
const ListWithDataAttributes = () => {
  const [selectedId, setSelectedId] = useState<string>('')
 
  // Single handler for all items
  const handleItemClick = useCallback((e: React.MouseEvent<HTMLDivElement>) => {
    const id = e.currentTarget.dataset.itemId
    if (id) {
      setSelectedId(id)
    }
  }, [])
 
  const items = ['item1', 'item2', 'item3']
 
  return (
    <div>
      {items.map(item => (
        <div
          key={item}
          data-item-id={item}
          onClick={handleItemClick}
          className={selectedId === item ? 'selected' : ''}
        >
          {item}
        </div>
      ))}
    </div>
  )
}
```

La mejora crítica en estas soluciones es la estabilidad de las referencias de función. Al evitar la creación de funciones en línea, React puede comparar correctamente las props entre renderizados y evitar actualizaciones innecesarias en componentes memoizados. En la primera solución, `useCallback` garantiza que la identidad del controlador permanezca estable. En la segunda, mover el controlador dentro del hijo aísla los re-renderizados solo al componente afectado. Estos cambios mejoran directamente el rendimiento del renderizado sin alterar el comportamiento de la aplicación.

### Manejo incorrecto de efectos secundarios dentro de componentes

Cada problema en este ejemplo se debe a la ejecución de efectos secundarios durante el renderizado en lugar de dentro de límites de ciclo de vida controlados. La llamada a `fetch` se activa en cada renderizado, creando un bucle infinito de actualizaciones de estado. La manipulación directa del DOM se ejecuta repetidamente sin coordinación, y el escuchador de eventos se adjunta en cada renderizado sin limpieza, provocando fugas de memoria. Estos problemas no son independientes; todos resultan de violar la regla de que el renderizado debe permanecer puro y los efectos secundarios deben aislarse y gestionarse explícitamente:

```tsx
// Anti-pattern: Side effects in component body
const BadSideEffects = ({ userId }: { userId: string }) => {
  const [userData, setUserData] = useState(null)
 
  // Wrong: Fetch in component body causes issues
  fetch(`/api/user/${userId}`)
    .then(res => res.json())
    .then(data => setUserData(data)) // Infinite loop!
 
  // Wrong: Direct DOM manipulation
  document.title = `User: ${userId}` // Runs on every render
 
  // Wrong: Subscription without cleanup
  const handleResize = () => console.log(window.innerWidth)
  window.addEventListener('resize', handleResize) // Memory leak!
 
  return <div>{userData?.name}</div>
};
```

El enfoque correcto gestiona adecuadamente los efectos secundarios utilizando `useEffect` con dependencias apropiadas y limpieza:

```tsx
// Best practice: Proper side effect management
interface UserData {
  id: string;
  name: string;
  email: string;
}

const ProperSideEffects = ({ userId }: { userId: string }) => {
  const [userData, setUserData] = useState<UserData | null>(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<string | null>(null)
 
  // Properly managed data fetching
  useEffect(() => {
    let cancelled = false
   
    const fetchUserData = async () => {
      setLoading(true)
      setError(null)
     
      try {
        const response = await fetch(`/api/user/${userId}`)
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`)
        }
        const data = await response.json()
   
        // Check if component is still mounted
        if (!cancelled) {
          setUserData(data)
        }
      } catch (err) {
        if (!cancelled) {
          setError(err instanceof Error ? err.message : 'Unknown error')
        }
      } finally {
        if (!cancelled) {
          setLoading(false)
        }
      }
    }
    fetchUserData()
   
    // Cleanup function
    return () => {
      cancelled = true
    };
  }, [userId])
  // Properly managed DOM updates
  useEffect(() => {
    const previousTitle = document.title
    document.title = userData ? `User: ${userData.name}` : 'Loading...'
    // Restore previous title on unmount
    return () => {
      document.title = previousTitle
    };
  }, [userData])
  // Properly managed event listeners
  useEffect(() => {
    const handleResize = () => {
      console.log('Window resized:', window.innerWidth)
    }
    window.addEventListener('resize', handleResize)
    // Cleanup
    return () => {
      window.removeEventListener('resize', handleResize)
    }
  }, []) // Empty deps array - only subscribe once
 
  if (loading) return <div>Loading...</div>
  if (error) return <div>Error: {error}</div>
 
  return (
    <div>
      <h2>{userData?.name}</h2>
      <p>{userData?.email}</p>
    </div>
  );
};
```

Los efectos secundarios pertenecen a efectos bien delimitados o controladores de eventos con ciclos de vida claros: realiza la obtención de datos dentro de `useEffect` con cancelación, refleja el estado del DOM (como `document.title`) en efectos con limpieza y adjunta suscripciones una sola vez con la desconexión adecuada. Cuando los patrones se repitan, extrae un hook personalizado para que tus componentes se mantengan puros y predecibles.

En React 19, la obtención de datos generalmente debe gestionarse a través de patrones compatibles con Suspense, como `use()` con promesas o los mecanismos de carga de datos proporcionados por frameworks como Next.js. Estos enfoques se integran mejor con el renderizado, los estados de carga y los límites de error que coordinar manualmente el estado de las solicitudes dentro de `useEffect`. Cuando una solicitud basada en efectos sigue siendo necesaria, prefiere `AbortController` sobre una simple bandera `isMounted` o de cancelación. Una bandera solo evita que el componente use un resultado obsoleto después de que la solicitud se completa, mientras que `AbortController` puede cancelar la solicitud subyacente en sí, evitando trabajo innecesario de red y procesamiento.

Con los efectos bajo control, veamos cómo la estructura de los componentes y, especialmente, la propagación de props (*prop spreading*), pueden desencadenar silenciosamente re-renderizados innecesarios, atributos inválidos e incluso problemas de seguridad.

---

## Estructura de componentes y re-renderizados

Propagar props directamente sobre elementos del DOM es un antipatrón peligroso que puede provocar vulnerabilidades de seguridad, atributos DOM no válidos y problemas de rendimiento. Este patrón suele surgir cuando los desarrolladores intentan crear componentes contenedores flexibles pero involuntariamente exponen sus aplicaciones a riesgos:

```tsx
// Anti-pattern: Spreading all props onto DOM elements
interface ButtonWrapperBadProps {
  onClick: () => void;
  className?: string;
  dangerousUserInput?: string;
  [key: string]: any; // Allowing arbitrary props
}

const ButtonWrapperBad: React.FC<ButtonWrapperBadProps> = (props) => {
  // This spreads ALL props, including non-DOM attributes
  return <button {...props} />
}

// Usage that demonstrates the problems
const DangerousUsage = () => {
  const userControlledProps = {
    onClick: () => console.log('clicked'),
    className: 'primary-button',
    dangerouslySetInnerHTML: { __html: '<script>alert("XSS")</script>' },
    customData: { internal: 'data' }, // Non-DOM prop
    'data-testid': 'button',
    style: { backgroundColor: 'red' }
  }
 
  return (
    // This could lead to XSS vulnerabilities and React warnings
    <ButtonWrapperBad {...userControlledProps}>
      Click me
    </ButtonWrapperBad>
  )
}
```

La preocupación de seguridad aquí no es la operación de propagación de objetos en sí. Propagar props no crea inherentemente una vulnerabilidad XSS; el riesgo aparece cuando se renderiza contenido no confiable o no desinfectado a través de APIs como `dangerouslySetInnerHTML`, especialmente mediante la propiedad `__html`. Usar una lista de permitidos (*allowlist*) de props aceptadas sigue siendo una buena práctica defensiva porque reduce la superficie para el reenvío accidental de props, pero no resuelve el problema subyacente por sí sola. Cualquier valor pasado a `dangerouslySetInnerHTML` debe desinfectarse o generarse a partir de una fuente confiable antes del renderizado. Aquí está el enfoque seguro y de alto rendimiento:

```tsx
// Best practice: Explicitly handle props
interface ButtonWrapperGoodProps {
  onClick?: () => void;
  className?: string;
  disabled?: boolean;
  type?: 'button' | 'submit' | 'reset';
  children: React.ReactNode;
  variant?: 'primary' | 'secondary' | 'danger';
  size?: 'small' | 'medium' | 'large';
  'data-testid'?: string;
  'aria-label'?: string;
}
const ButtonWrapperGood = ({
  onClick,
  className,
  disabled,
  type = 'button',
  children,
  variant = 'primary',
  size = 'medium',
  'data-testid': dataTestId,
  'aria-label': ariaLabel
}: ButtonWrapperGoodProps) => {
  // Explicitly construct className
  const buttonClassName = [
    'btn',
    `btn-${variant}`,
    `btn-${size}`,
    className
  ].filter(Boolean).join(' ')
  // Only pass valid, explicitly defined DOM props
  return (
    <button
      onClick={onClick}
      className={buttonClassName}
      disabled={disabled}
      type={type}
      data-testid={dataTestId}
      aria-label={ariaLabel}
    >
      {children}
    </button>
  )
}
```

El principio clave aquí es el manejo explícito de props. Al definir exactamente qué props acepta nuestro componente y cómo se pasan al elemento DOM, prevenimos vulnerabilidades de seguridad y mantenemos una API de componente clara. El componente solo reenvía las props que están definidas explícitamente en la interfaz, evitando que propiedades peligrosas como `dangerouslySetInnerHTML` se pasen accidentalmente:

```tsx
// Alternative: Using prop destructuring with validation
interface SafeInputProps {
  value: string;
  onChange: (value: string) => void;
  placeholder?: string;
  type?: 'text' | 'email' | 'password' | 'number';
  required?: boolean;
  className?: string;
  disabled?: boolean;
  maxLength?: number;
  'data-testid'?: string;
}

const SafeInput = ({
  value,
  onChange,
  type = 'text',
  placeholder,
  required,
  className,
  disabled,
  maxLength,
  'data-testid': dataTestId
}: SafeInputProps) => {
  // Create a safe props object with only allowed attributes
  const safeProps = {
    value,
    onChange: (e: React.ChangeEvent<HTMLInputElement>) => onChange(e.target.value),
    type,
    placeholder,
    required,
    className,
    disabled,
    maxLength,
    'data-testid': dataTestId
  }
 
  // Remove undefined values for cleaner DOM
  const cleanProps = Object.entries(safeProps).reduce((acc, [key, val]) => {
    if (val !== undefined) {
      acc[key as keyof typeof safeProps] = val
    }
    return acc
  }, {} as Partial<typeof safeProps>)
 
  return <input {...cleanProps} />
}
```

Este enfoque demuestra cómo manejar de forma segura las props de entrada mientras se mantiene la seguridad de tipos. El componente enumera explícitamente todas las props permitidas y las maneja individualmente, evitando que cualquier prop inesperada o maliciosa llegue al elemento DOM.

```tsx
// Advanced: Type-safe prop forwarding with forwardRef
interface TypeSafeButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'danger';
  size?: 'small' | 'medium' | 'large';
  loading?: boolean;
}

const TypeSafeButton = React.forwardRef<HTMLButtonElement, TypeSafeButtonProps>(
  ({
    variant = 'primary',
    size = 'medium',
    loading = false,
    className,
    children,
    disabled,
    onClick,
    ...restProps  // These are now type-safe HTML button attributes
  }, ref) => {
    // Separate custom props from DOM props
    const buttonClassName = [
      'btn',
      `btn-${variant}`,
      `btn-${size}`,
      loading && 'btn-loading',
      className
    ].filter(Boolean).join(' ')
   
    // Prevent click when loading
    const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
      if (loading) {
        e.preventDefault()
        return
      }
      onClick?.(e)
    }
   
    return (
      <button
        ref={ref}
        className={buttonClassName}
        disabled={disabled || loading}
        onClick={handleClick}
        {...restProps}  // Safe because TypeScript ensures only valid props
      >
{loading&&<span className="spinner" />}
        {children}
      </button>
    );
  }
);

TypeSafeButton.displayName = 'TypeSafeButton'
```

Cuando necesitas más flexibilidad, extender los tipos de atributos HTML integrados de React proporciona seguridad de tipos al tiempo que permite el paso de props válidas del DOM. Este patrón utiliza el sistema de tipos de TypeScript para garantizar que solo se propaguen atributos HTML válidos, combinando flexibilidad con seguridad. Las props personalizadas (`variant`, `size`, `loading`) se extraen y manejan por separado, mientras que los atributos estándar de botones HTML se pasan de manera segura.

```tsx
// Security-focused wrapper component
interface SecureWrapperProps {
  as?: 'div' | 'section' | 'article' | 'main';
  children: React.ReactNode;
  className?: string;
  id?: string;
  role?: string;
  'aria-label'?: string;
  'data-testid'?: string;
  onClick?: (e: React.MouseEvent) => void;
}

const SecureWrapper = ({
  as: Component = 'div',
  children,
  className,
  id,
  role,
  'aria-label': ariaLabel,
  'data-testid': dataTestId,
  onClick
}: SecureWrapperProps) => {
  // Create props object with only safe attributes
  const safeProps = {
    className,
    id,
    role,
    'aria-label': ariaLabel,
    'data-testid': dataTestId,
    onClick
  }
 
  // Remove undefined values
  const cleanedProps = Object.entries(safeProps).reduce((acc, [key, value]) => {
    if (value !== undefined) {
      (acc as any)[key] = value
    }
    return acc
  }, {})
 
  return <Component {...cleanedProps}>{children}</Component>
}

const ProperUsageExample = () => {
  return (
    <div>
      <ButtonWrapperGood
        onClick={() => console.log('Safe click')}
        variant="primary"
        size="large"
        data-testid="safe-button"
        aria-label="Submit form"
      >
        Safe Button
      </ButtonWrapperGood>
     
      <SafeInput
        value=""
        onChange={(val) => console.log('Input changed:', val)}
        placeholder="Enter email"
        type="email"
        required
        data-testid="email-input"
      />
     
      <TypeSafeButton
        variant="danger"
        loading={false}
        onClick={() => console.log('Type-safe click')}
      >
        Delete
      </TypeSafeButton>
    </div>
  )
}
```

El componente `SecureWrapper` demuestra cómo crear un contenedor flexible manteniendo la seguridad. Al definir explícitamente las props permitidas y utilizar un enfoque de lista blanca, nos aseguramos de que solo los atributos seguros lleguen al DOM. Este patrón es particularmente útil al crear componentes contenedores reutilizables que necesitan admitir diferentes elementos HTML mientras mantienen la seguridad y la tipificación estricta.

### Uso excesivo de Context API que conduce a renderizados innecesarios

Context API parece la solución perfecta para el prop drilling, hasta que tu aplicación comienza a ralentizarse. La atractiva simplicidad de arrojar todo dentro de un único contexto crea una pesadilla de rendimiento donde cambiar un tema desencadena re-renderizados del perfil de usuario.

Considera este patrón común en el que caen los desarrolladores:

```tsx
interface AppContextType {
  user: User | null;
  theme: 'light' | 'dark';
  notifications: Notification[];
  isLoading: boolean;
}

const AppContext = createContext<AppContextType | undefined>(undefined)

const AppProvider = ({ children }: { children: React.ReactNode }) => {
  const [user, setUser] = useState<User | null>(null)
  const [theme, setTheme] = useState<'light' | 'dark'>('light')
  const [notifications, setNotifications] = useState<Notification[]>([])
  const [isLoading, setIsLoading] = useState(false)

  // Every render creates a new object reference
  const value = {
    user, theme, notifications, isLoading,
    setUser, setTheme, setNotifications, setIsLoading
  }

  return <AppContext.Provider value={value}>{children}</AppContext.Provider>
}

const UserProfile = () => {
  const context = useContext(AppContext)
  return <div>{context?.user?.name}</div> // Re-renders on theme changes
}
```

React 19 también agrega `use(Context)` como una forma más flexible de leer valores de contexto, pero no hace que el Contexto nativo esté basado en selectores por sí mismo. Para el estado que cambia con frecuencia o se comparte entre muchos componentes, considera un patrón de store externo con `useSyncExternalStore` y selectores, o una librería basada en selectores construida sobre la misma idea. Esto permite que los consumidores se suscriban a la porción específica que necesitan, reduciendo re-renderizados innecesarios cuando cambian partes no relacionadas del estado.

La solución requiere un diseño de contexto disciplinado: dividir responsabilidades y estabilizar referencias:

```tsx
// Focused contexts prevent cross-contamination
const UserContext = createContext<UserContextType | undefined>(undefined)
const ThemeContext = createContext<ThemeContextType | undefined>(undefined)

const UserProvider = ({ children }: { children: React.ReactNode }) => {
  const [user, setUser] = useState<User | null>(null)
 
  // Stable reference prevents unnecessary re-renders
  const value = useMemo(() => ({ user, setUser }), [user])
 
  return <UserContext.Provider value={value}>{children}</UserContext.Provider>
}

const ThemeProvider = ({ children }: { children: React.ReactNode }) => {
  const [theme, setTheme] = useState<'light' | 'dark'>('light')
  const value = useMemo(() => ({ theme, setTheme }), [theme])
 
  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>
}

const UserProfile = React.memo(() => {
  const { user } = useContext(UserContext)!
  return <div>{user?.name}</div> // Only cares about user changes
})
```

Ahora tu perfil de usuario ignora los cambios de tema por completo, y las actualizaciones de tema no desencadenan un procesamiento costoso de datos de usuario. El Contexto se convierte en un bisturí en lugar de un mazo.

### La ceguera ante la memoización

Los desarrolladores de React a menudo tratan cada renderizado como si fuera gratuito, recalculando operaciones costosas y recreando funciones indiscriminadamente. Este enfoque ingenuo transforma interacciones fluidas en experiencias entrecortadas, especialmente con transformaciones de datos complejas.

Aquí hay una lista de productos típica que recalcula todo en cada pulsación de tecla:

```tsx
const ProductList = ({ products }: { products: Product[] }) => {
  const [searchTerm, setSearchTerm] = useState('')
  const [sortBy, setSortBy] = useState<'name' | 'price'>('name')

  // Expensive work on every render - even unrelated state changes
  const processedProducts = products
    .filter(product => product.name.toLowerCase().includes(searchTerm.toLowerCase()))
    .sort((a, b) => {
      if (sortBy === 'name') return a.name.localeCompare(b.name)
      return a.price - b.price
    })
    .map(product => ({
      ...product,
      formattedPrice: new Intl.NumberFormat('en-US', {
        style: 'currency',
        currency: 'USD'
      }).format(product.price)
    }))

  // New function instance every render
  const handleProductClick = (productId: number) => {
    console.log(`Clicked product ${productId}`)
  }

  return (
    <div>
      <input
        value={searchTerm}
        onChange={(e) => setSearchTerm(e.target.value)}
        placeholder="Search products..."
      />
      {processedProducts.map(product => (
        <ProductCard
          key={product.id}
          product={product}
          onClick={handleProductClick} // Forces ProductCard re-render
        />
      ))}
    </div>
  );
};
```

El problema de rendimiento aquí no es solo el costo computacional, sino la repetición innecesaria de ese costo. Cada renderizado recalcula el filtrado, la ordenación y el formateo, incluso cuando ocurren cambios de estado no relacionados. Además, el controlador de eventos recreado fuerza a todos los componentes hijos a volver a renderizarse, anulando cualquier beneficio de memoización. Esto crea un efecto compuesto donde tanto el cálculo como el trabajo de renderizado aumentan juntos. El objetivo de la optimización no es eliminar el trabajo por completo, sino garantizar que las operaciones costosas solo se ejecuten cuando sus entradas realmente cambien.

La memoización estratégica transforma este componente lento en una interfaz responsiva:

```tsx
const ProductList = ({ products }: { products: Product[] }) => {
  const [searchTerm, setSearchTerm] = useState('')
  const [sortBy, setSortBy] = useState<'name' | 'price'>('name')

  // Expensive computation only runs when inputs actually change
  const processedProducts = useMemo(() => {
    return products
      .filter(product =>
        product.name.toLowerCase().includes(searchTerm.toLowerCase())
      )
      .sort((a, b) => {
        if (sortBy === 'name') return a.name.localeCompare(b.name)
        return a.price - b.price
      })
      .map(product => ({
        ...product,
        formattedPrice: new Intl.NumberFormat('en-US', {
          style: 'currency',
          currency: 'USD'
        }).format(product.price)
      }))
  }, [products, searchTerm, sortBy])
  // Stable function reference prevents child re-renders
  const handleProductClick = useCallback((productId: number) => {
    console.log(`Clicked product ${productId}`)
  }, [])

  const handleSearchChange = useCallback((e: React.ChangeEvent<HTMLInputElement>) => {
    setSearchTerm(e.target.value)
  }, [])

  return (
    <div>
      <input
        value={searchTerm}
        onChange={handleSearchChange}
        placeholder="Search products..."
      />
      {processedProducts.map(product => (
        <ProductCard
          key={product.id}
          product={product}
          onClick={handleProductClick} // Stable reference
        />
      ))}
    </div>
  )
}

const ProductCard = React.memo<{
  product: Product & { formattedPrice: string };
  onClick: (id: number) => void;
}>(({ product, onClick }) => {
  return (
    <div onClick={() => onClick(product.id)}>
      <h3>{product.name}</h3>
      <p>{product.formattedPrice}</p>
    </div>
  )
})
```

### La falacia del scroll infinito

Renderizar listas masivas parece sencillo: simplemente itera sobre el arreglo con `map` y deja que React se encargue. Este enfoque funciona de maravilla para docenas de elementos, pero se vuelve catastrófico con miles. El hilo principal del navegador se satura con la manipulación del DOM mientras los usuarios observan pantallas en blanco.

El enfoque ingenuo al que recurren los desarrolladores:

```tsx
const LargeList = ({ items }: { items: ListItem[] }) => {
  return (
    <div style={{ height: '400px', overflow: 'auto' }}>
      {items.map(item => (
        <div key={item.id} style={{ padding: '16px', borderBottom: '1px solid #eee' }}>
          <h3>{item.title}</h3>
          <p>{item.description}</p>
          <small>{item.timestamp.toLocaleString()}</small>
        </div>
      ))}
    </div>
  )
}
```

Con 10,000 elementos, React crea 10,000 nodos DOM por adelantado. El renderizado inicial bloquea el hilo principal, el desplazamiento se vuelve entrecortado y el uso de memoria se dispara. Los usuarios solo pueden ver 5 o 6 elementos a la vez y, sin embargo, estás renderizando miles.

El desplazamiento virtual (*virtual scrolling*) resuelve esto elegantemente renderizando solo los elementos visibles más un pequeño búfer:

```tsx
const ITEM_HEIGHT = 100
const CONTAINER_HEIGHT = 400
const VISIBLE_ITEMS = Math.ceil(CONTAINER_HEIGHT / ITEM_HEIGHT)
const BUFFER_SIZE = 5
const VirtualizedList = ({ items }: { items: ListItem[] }) => {
  const [scrollTop, setScrollTop] = useState(0)

  const visibleRange = useMemo(() => {
    const startIndex = Math.floor(scrollTop / ITEM_HEIGHT)
    const endIndex = Math.min(
      items.length - 1,
      startIndex + VISIBLE_ITEMS + BUFFER_SIZE
    )
    const visibleStartIndex = Math.max(0, startIndex - BUFFER_SIZE)
   
    return { startIndex: visibleStartIndex, endIndex }
  }, [scrollTop, items.length])

  const visibleItems = useMemo(() => {
    return items.slice(visibleRange.startIndex, visibleRange.endIndex + 1)
  }, [items, visibleRange])

  const handleScroll = useCallback((e: React.UIEvent<HTMLDivElement>) => {
    setScrollTop(e.currentTarget.scrollTop)
  }, [])

  const totalHeight = items.length * ITEM_HEIGHT
  const offsetY = visibleRange.startIndex * ITEM_HEIGHT

  return (
    <div
      style={{ height: CONTAINER_HEIGHT, overflow: 'auto' }}
      onScroll={handleScroll}
    >
      <div style={{ height: totalHeight, position: 'relative' }}>
        <div style={{ transform: `translateY(${offsetY}px)` }}>
          {visibleItems.map((item, index) => {
            const actualIndex = visibleRange.startIndex + index;
            return (
              <ListItemComponent
                key={item.id}
                item={item}
                style={{ height: ITEM_HEIGHT }}
                index={actualIndex}
              />
            );
          })}
        </div>
      </div>
    </div>
  )
}

const ListItemComponent = React.memo<{
  item: ListItem;
  style: React.CSSProperties;
  index: number;
}>(({ item, style, index }) => {
  return (
    <div style={{
      ...style,
      padding: '16px',
      borderBottom: '1px solid #eee',
      display: 'flex',
      flexDirection: 'column',
      justifyContent: 'center'
    }}>
      <h3 style={{ margin: '0 0 8px 0' }}>{item.title}</h3>
      <p style={{ margin: '0 0 4px 0' }}>{item.description}</p>
      <small>#{index} - {item.timestamp.toLocaleString()}</small>
    </div>
  )
})
```

Esta implementación de desplazamiento virtual renderiza quizás 15 elementos en lugar de 10,000. El desplazamiento se mantiene increíblemente fluido, el uso de memoria permanece constante y el renderizado inicial ocurre de manera instantánea. El usuario no experimenta ninguna diferencia funcional mientras que el rendimiento mejora en órdenes de magnitud.

---

## Resumen

Los antipatrones explorados en este capítulo comparten un tema común: rompen las suposiciones fundamentales de React sobre identidad, inmutabilidad y flujo predecible de datos. Ya sea inicializar el estado a partir de props, usar keys inestables, mutar el estado directamente o recrear funciones innecesariamente, cada problema introduce inconsistencias sutiles entre lo que React espera y cómo se comporta la aplicación. Comprender por qué fallan estos patrones es más importante que memorizar las correcciones, ya que te permite reconocer y evitar problemas similares en diferentes contextos.

Las mejores prácticas presentadas: elevación adecuada del estado, actualizaciones inmutables, keys estables, estrategias de memoización, limpieza de efectos y uso reflexivo del Contexto, forman la base de aplicaciones escalables en React. Dominar estos patrones requiere comprender los principios subyacentes de React: la importancia de la inmutabilidad, la programación declarativa y el pensamiento en flujo de datos. Al evitar estos antipatrones y adoptar las mejores prácticas establecidas, los desarrolladores pueden crear aplicaciones que no solo sean funcionales sino también de alto rendimiento, fáciles de mantener y listas para escalar a medida que evolucionan los requisitos.
