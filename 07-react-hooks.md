# Capítulo 7: React Hooks

Los React Hooks han revolucionado la forma en que escribimos aplicaciones en React, permitiéndonos usar componentes funcionales en lugar de componentes de clase y haciendo que la programación sea más rápida y eficiente. Desde su introducción en React 16.8, los Hooks se han convertido en una parte esencial del desarrollo con React y han mejorado enormemente el rendimiento de nuestras aplicaciones. Con los Hooks, podemos gestionar el estado, manejar efectos secundarios y reutilizar código de una manera más concisa y legible. En este capítulo, exploraremos los diferentes tipos de Hooks y cómo utilizarlos para mejorar nuestras aplicaciones de React.

En este capítulo, cubriremos los siguientes temas:

- Los nuevos React Hooks y cómo utilizarlos
- Las reglas de los Hooks
- Cómo migrar un componente de clase a React Hooks
- Comprensión del ciclo de vida del componente con Hooks y efectos
- Cómo obtener datos (*data fetching*) con Hooks
- Cómo memorizar componentes, valores y funciones con `memo`, `useMemo` y `useCallback`
- Cómo implementar `useReducer`
- El nuevo React Compiler: optimización y memoización automática
- Hooks modernos en React 19

---

## Requisitos técnicos

Para completar este capítulo, necesitarás lo siguiente:

- Node.js 24
- Cursor o VSCode

---

## Introducción a los React Hooks

Los React Hooks, introducidos en React 16.8, cambiaron fundamentalmente la forma en que estructuramos las aplicaciones de React al permitir lógica con estado en componentes funcionales. Si bien la mayoría de los desarrolladores ya están familiarizados con Hooks fundamentales como `useState` y `useEffect`, el desarrollo moderno de React, especialmente con React 19, se extiende mucho más allá de estos conceptos básicos. En este capítulo, no solo revisaremos los Hooks fundamentales a través de ejemplos prácticos, sino que también los conectaremos con patrones más avanzados y APIs más nuevas que reflejan mejor cómo se usan los Hooks en las aplicaciones de producción actuales.

### Sin cambios disruptivos (*No breaking changes*)

En el contexto del desarrollo de React, existe la idea errónea común de que la introducción de los React Hooks ha dejado obsoletos los componentes de clase. Sin embargo, esto no es cierto, ya que no hay planes de eliminar las clases de React. La API de Hooks no reemplaza tu comprensión de los conceptos de React, sino que ofrece un enfoque más simplificado para trabajar con esos conceptos, como props, estado, contexto, referencias (*refs*) y ciclo de vida, con los que ya estás familiarizado.

Aunque las siguientes secciones revisitan los Hooks centrales, el objetivo no es volver a aprender la sintaxis, sino establecer un modelo mental compartido sobre el cual construiremos. Comprender cómo estos Hooks se asignan al ciclo de vida del componente, el comportamiento de renderizado y las características de rendimiento es esencial antes de introducir Hooks y patrones más avanzados más adelante en el capítulo.

### Uso del State Hook

En el código antiguo de React usábamos `this.setState` para usar el estado del componente. Ahora usamos el hook `useState` para hacer esto.

Primero, necesitas extraer el Hook `useState` de React:

```typescript
import { useState } from 'react'
```

Desde React 17, el objeto `React` ya no es necesario para renderizar código JSX.

Luego debes declarar el estado que deseas usar definiendo el estado y la función de actualización (*setter*) para este estado específico:

```tsx
  const Counter = () => {
    const [counter, setCounter] = useState<number>(0)
  }
```

Como puedes ver, estamos declarando el estado `counter` con el actualizador `setCounter`, especificamos que solo aceptaremos números y, finalmente, establecemos el valor inicial en cero.

Para probar nuestro estado, necesitamos crear un método que sea activado por el evento `onClick`:

```tsx
  type Operation = 'add' | 'subtract'
  const Counter = () => {
    const [counter, setCounter] = useState<number>(0)
    const handleCounter = (operation: Operation) => {
      if (operation === 'add') {
        return setCounter(counter + 1)
      }
   
      setCounter(counter - 1)
    }
  }
```

Finalmente, podemos renderizar el estado `counter` y algunos botones para aumentar o disminuir el estado del contador:

```tsx
  return (
    <p>
      Counter: {counter} <br />
      <button onClick={() => handleCounter('add')}>+ Add</button>
      <button onClick={() => handleCounter('subtract')}>- Subtract</button>
    </p>
  )
```

Si haces clic en el botón `+ Add` una vez, deberías ver 1 para Counter.

Y si haces clic en el botón `- Subtract` dos veces, entonces deberías ver -1 para Counter.

Como puedes ver, el Hook `useState` cambia las reglas del juego en React y hace que sea muy fácil manejar el estado en un componente funcional.

Habiendo apreciado cómo el Hook `useState` revoluciona la gestión del estado en componentes funcionales dentro de React, ahora estamos listos para profundizar en los matices de los Hooks. La siguiente sección discutirá las Reglas de los Hooks esenciales que rigen su uso en las aplicaciones de React.

---

## Reglas de los Hooks

Los React Hooks son básicamente funciones de JavaScript, pero hay dos reglas que debes seguir para utilizarlos. React proporciona un plugin de linter para hacer cumplir esas reglas por ti, que puedes instalar ejecutando el siguiente comando:

```bash
npm install --save-dev eslint-plugin-react-hooks
```

Veamos estas dos reglas.

### Regla 1: llama a los Hooks solo en el nivel superior

Para garantizar el funcionamiento adecuado de los React Hooks, es importante evitar llamarlos dentro de bucles, condiciones o funciones anidadas. En su lugar, se recomienda utilizar siempre los Hooks en el nivel superior de tu función de React. Esta práctica garantiza que los Hooks se llamen en el mismo orden cada vez que se renderiza un componente, lo que permite a React preservar correctamente el estado de los Hooks entre múltiples llamadas a `useState` y `useEffect`. Seguir esta regla te ayudará a escribir código más eficiente y mantenible con React Hooks.

### Regla 2: llama a los Hooks solo desde funciones de React

Para garantizar que toda la lógica con estado en un componente sea claramente visible desde su código fuente, evita llamar a los Hooks desde funciones regulares de JavaScript. En su lugar, usa Hooks en componentes funcionales de React o Hooks personalizados (*custom Hooks*). Al seguir esta práctica, puedes asegurarte de que toda la lógica con estado esté centralizada y sea fácilmente comprensible.

---

## Migración de un componente de clase a React Hooks

Transformemos un código que actualmente usa componentes de clase y que también utiliza algunos métodos del ciclo de vida. En este ejemplo, obtenemos los *issues* de un repositorio de GitHub y los listamos.

Para este ejemplo, necesitarás instalar `axios` para realizar la petición:

```bash
npm install axios
```

Esta es la versión con componente de clase:

```tsx
  import axios from 'axios'
  import { Component } from 'react'

  type Issue = {
    number: number
    title: string
    state: string
  }
  type Props = {}
  type State = { issues: Issue[] }

  class Issues extends Component<Props, State> {
    constructor(props: Props) {
      super(props)

      this.state = {
        issues: []
      }
    }

    componentDidMount() {
      axios.get('https://api.github.com/repos/ContentPI/ContentPI/issues')
        .then((response: any) => {
          this.setState({
            issues: response.data
          })
        })
    }

    render() {
      const { issues = [] } = this.state

      return (
        <>
          <h1>ContentPI Issues</h1>

          {issues.map((issue: Issue) => (
            <p key={issue.title}>
              <strong>#{issue.number}</strong>{' '}
              <a
                href={`https://github.com/ContentPI/ContentPI/issues/${issue.number}`}
                target="_blank"
              >
                {issue.title}
              </a>{' '}
              {issue.state}
            </p>
          ))}
        </>
      )
    }
  }

  export default Issues
```

Ahora, transformemos nuestro código para que sea un componente funcional usando React Hooks. Lo primero que debemos hacer es importar algunas funciones y tipos de React:

```tsx
  import { FC, useState, useEffect } from 'react'
   import axios from 'axios'
```

Ahora podemos eliminar los tipos `Props` y `State` que creamos anteriormente y dejar solo el tipo `Issue`:

```typescript
type Issue = {
  number: number
  title: string
  state: string
}
```

Después de esto, puedes cambiar la definición de la clase para usar un componente funcional:

```tsx
 const Issues: FC = () => {...}
```

El tipo `FC` se utiliza para definir un componente funcional (*Functional Component*) en React. Si necesitas pasar algunas props al componente, puedes pasarlas de esta forma:

```tsx
type Props = {
  propX: string
  propY: number
  propZ: boolean
}

const Issues: FC<Props> = () => {...}
```

Lo siguiente que debemos hacer es reemplazar nuestro constructor y nuestra definición de estado utilizando el Hook `useState`:

```tsx
// The useState hook replace the this.setState method
const [issues, setIssues] = useState<Issue[]>([])
```

Hemos usado antes el método del ciclo de vida llamado `componentDidMount`, que se ejecuta cuando el componente se monta y se ejecutará solo una vez. El nuevo React Hook, llamado `useEffect`, ahora manejará todos los métodos del ciclo de vida usando diferentes sintaxis para cada uno, pero por ahora, veamos cómo podemos obtener el mismo efecto de `componentDidMount` en nuestro nuevo componente funcional:

```tsx
// When we use the useEffect hook with an empty array [] on the
// dependencies (second parameter)
// this represents the componentDidMount method (will be executed when the
// component is mounted).
useEffect(() => {
  axios
     .get('https://api.github.com/repos/ContentPI/ContentPI/issues')
     .then((response: any) => {
      // Here we update directly our issue state
      setIssues(response.data)
    })
}, [])
```

Y finalmente, renderizamos nuestro código JSX:

```tsx
  return (
     <>
       <h1>ContentPI Issues</h1>
 {issues.map((issue: Issue) => (
     <p key={issue.title}>
       <strong>#{issue.number}</strong> {' '}
        <a
         href={`https://github.com/ContentPI/ContentPI/issues/${issue.number}`}     
              target="_blank">
         {issue.title}
       </a> {' '}
        {issue.state}
     </p>
   ))}
    </>
)
```

Como puedes ver, los nuevos Hooks nos ayudan a simplificar nuestro código y a que tenga más sentido. Además, redujimos nuestro código en 10 líneas (el código del componente de clase tiene 53 líneas y el componente funcional tiene 43 líneas).

---

## Comprensión de los efectos en React

En esta sección, aprenderemos la diferencia entre los métodos del ciclo de vida de los componentes que usamos en los componentes de clase y los nuevos efectos de React. Aunque hayas leído en otros lugares que son lo mismo con una sintaxis diferente, esto no es correcto.

### Comprensión de `useEffect`

Cuando trabajas con `useEffect`, necesitas pensar en efectos. Si quieres realizar el método equivalente a `componentDidMount` usando `useEffect`, puedes hacer lo siguiente:

```tsx
useEffect(() => {
  // Here you perform your side effect
}, [])
```

El primer parámetro es el callback del efecto que deseas ejecutar, y el segundo parámetro es el arreglo de dependencias. Si pasas un arreglo vacío (`[]`) en las dependencias, el estado y las props tendrán sus valores iniciales originales.

Sin embargo, es importante mencionar que aunque este es el equivalente más cercano a `componentDidMount`, no tiene el mismo comportamiento. A diferencia de `componentDidMount` y `componentDidUpdate`, la función que pasamos a `useEffect` se dispara después del diseño (*layout*) y el pintado (*paint*), durante un evento diferido. Esto normalmente funciona para muchos efectos secundarios comunes, como configurar suscripciones y controladores de eventos, porque la mayoría de los tipos de trabajo no deberían bloquear al navegador para que actualice la pantalla.

Sin embargo, no todos los efectos deben diferirse hasta después de que el navegador pinte. Si un efecto necesita leer información de diseño o mutar sincrónicamente el DOM antes de que el usuario vea el resultado, usar `useEffect` puede causar un parpadeo visible (*flicker*). React proporciona `useLayoutEffect` para este caso. Utiliza la misma API que `useEffect`, pero su sincronización es diferente: `useLayoutEffect` se ejecuta sincrónicamente después de que React ha actualizado el DOM y antes de que el navegador pinte, mientras que `useEffect` se ejecuta después del pintado. Debido a que `useLayoutEffect` puede bloquear el renderizado, debe reservarse para mediciones de diseño y actualizaciones del DOM que deben ocurrir antes de que se muestre el fotograma.

### Disparar un efecto condicionalmente

Si necesitas disparar un efecto condicionalmente, debes agregar una dependencia al arreglo de dependencias; de lo contrario, ejecutarás el efecto varias veces y esto puede causar un bucle infinito. Si pasas un arreglo de dependencias, el Hook `useEffect` solo se ejecutará si una de esas dependencias cambia:

```tsx
  useEffect(() => {
  // When you pass an array of dependencies the useEffect hook will only
  // run if one of the dependencies changes.
  }, [dependencyA, dependencyB])
```

Si comprendes cómo funcionan los métodos del ciclo de vida de las clases de React, básicamente `useEffect` se comporta de la misma manera que `componentDidMount`, `componentDidUpdate` y `componentWillUnmount` combinados.

---

## Comprensión de `useCallback`, `useMemo` y `memo`

Para comprender la diferencia entre `useCallback`, `useMemo` y `memo`, realizaremos un ejemplo de lista de tareas pendientes (*to-do list*). Puedes crear una aplicación básica utilizando `create-vite` y TypeScript como plantilla:

```bash
create-vite todo --template react-ts
```

Inmediatamente después, puedes eliminar todos los archivos adicionales (`App.css`, `App.test.ts`, `index.css`, `logo.svg`, etc.). Solo necesitas conservar el archivo `App.tsx`, que contendrá el siguiente código:

```tsx
import { FC, useState, useEffect, useMemo, useCallback, ChangeEvent } from 'react'
     import List, { Todo } from './List'

const initialTodos: Todo[] = [
  { id: 1, task: 'Go shopping' },
  { id: 2, task: 'Pay the electricity bill'}
]

const App: FC = () => {
  const [todoList, setTodoList] = useState<Todo[]>(initialTodos)
  const [task, setTask] = useState<string>('')

  useEffect(() => {
    console.log('Rendering <App />')
  })

  const handleCreate = () => {
     const newTodo = {
      id: Date.now(),
      task
     }
  
    // Pushing the new todo to the list
    setTodoList([...todoList, newTodo])
  
    // Resetting input value
   setTask('')
  }

  return (
    <>
      <input
         type="text"
        value={task}
        onChange={(e: ChangeEvent<HTMLInputElement>) => setTask(e.target.value)}
     />

     <button onClick={handleCreate}>Create</button>

      <List todoList={todoList} />
   </>
  )
}

export default App
```

Básicamente, estamos definiendo algunas tareas iniciales y creando el estado `todoList`, que pasaremos al componente de lista. Luego debes crear el archivo `List.tsx` con el siguiente código:

```tsx
import { FC, useEffect } from 'react'
import Task from './Task'

export type Todo = {
  id: number
  task: string
}

interface Props {
  todoList: Todo[]
}

const List: FC<Props> = ({ todoList }) => {
  useEffect(() => {
    // This effect is executed every new render
    console.log('Rendering <List />')
  })
  return (
     <ul>
      {todoList.map((todo: Todo) => (
         <Task key={todo.id} id={todo.id} task={todo.task} />
      ))}
   </ul>
  )
}

export default List
```

Como puedes ver, estamos renderizando cada tarea del arreglo `todoList` usando el componente `Task` y pasamos `task` como prop. También agregué un Hook `useEffect` para ver cuántos renderizados estamos realizando.

Finalmente, creamos nuestro archivo `Task.tsx` con el siguiente código:

```tsx
import { FC, useEffect } from 'react'

interface Props {
  id: number
  task: string
}

const Task: FC<Props> = ({ task }) => {
  useEffect(() => {
   console.log('Rendering <Task />', task)
  })

  return (
   <li>{task}</li>
  )
}

export default Task
```

Cuando renderizamos nuestra lista de tareas pendientes, por defecto, realizamos dos renderizados del componente `Task`, un renderizado para `List` y el otro para el componente `App`.

Ahora, si intentamos escribir una nueva tarea en el input, podemos ver que por cada letra que escribimos, volveremos a ver todos esos renderizados. Con solo escribir "Go", tenemos dos nuevos lotes de renderizados, por lo que podemos determinar que este componente no tiene un buen rendimiento, y aquí es donde `memo` puede ayudarnos a mejorar el rendimiento.

### Memoización de un componente con `memo`

El componente de orden superior (*Higher-Order Component* o HOC) `memo` es similar a `PureComponent` de una clase de React porque realiza una comparación superficial (*shallow comparison*) de las props. Si intentamos renderizar un componente con las mismas props todo el tiempo, el componente se renderizará solo una vez y se memorizará. La única forma de volver a renderizar el componente es cuando una prop cambia su valor.

Para solucionar nuestros componentes y evitar los múltiples renderizados cuando escribimos en el input, necesitamos envolver nuestros componentes en el HOC `memo`.

El primer componente que arreglaremos es nuestro componente `List`, y solo necesitas importar `memo` y envolver el componente en `export default`:

```tsx
import { FC, useEffect, memo } from 'react'
...
export default memo(List)
```

Luego debes hacer lo mismo con el componente `Task`:

```tsx
import { FC, useEffect, memo } from 'react'
...

export default memo(Task)
```

Ahora, cuando intentamos escribir "Go" nuevamente en el input, solo obtenemos el primer lote de renderizados la primera vez, y luego, cuando escribimos "Go", solo obtenemos dos renderizados más del componente `App`, lo cual es totalmente normal porque el estado `task` (valor del input) que estamos cambiando es en realidad parte del componente `App`.

También podemos ver cuántos renderizados estamos realizando cuando creamos una nueva tarea haciendo clic en el botón `Create`: solo vemos un renderizado del componente `Task`, un renderizado de `List` y un renderizado del componente `App`. Hemos mejorado mucho el rendimiento y solo realizamos el renderizado exacto necesario.

¿Deberíamos agregar `memo` a todos nuestros componentes por defecto?

La razón para no hacerlo es el rendimiento: no es una buena idea agregar `memo` a todos nuestros componentes a menos que sea totalmente necesario; de lo contrario, el proceso de comparaciones superficiales y memorización tendrá un rendimiento inferior en comparación con no usarlo.

Una regla práctica: normalmente, cuando tenemos componentes pequeños o lógica básica, no necesitamos esto a menos que estés trabajando con grandes volúmenes de datos de alguna API o tu componente necesite realizar muchos renderizados (normalmente listas enormes), o cuando notes que tu aplicación se vuelve lenta.

### Memoización de un valor con `useMemo`

Supongamos que ahora queremos implementar una funcionalidad de búsqueda en nuestra lista de tareas. Lo primero que debemos hacer es agregar un nuevo estado llamado `term` al componente `App`:

```tsx
const [term, setTerm] = useState('')
```

Luego debemos crear una función llamada `handleSearch`:

```tsx
const handleSearch = () => {
  setTerm(task)
}
```

Justo antes del retorno, crearemos `filteredTodoList`, que filtrará las tareas según el término de búsqueda, y agregaremos un console log allí para ver cuántas veces se está ejecutando:

```tsx
const filteredTodoList = todoList.filter((todo: Todo) => {
  console.log('Filtering...')
  return todo.task.toLowerCase().includes(term.toLowerCase())
})
```

Finalmente, agregamos un nuevo botón al lado del botón `Create`:

```tsx
<button onClick={handleSearch}>Search</button>
```

Por cada letra que escribes en el input, obtendrás llamadas de filtrado y un renderizado de `App`, lo cual genera problemas de rendimiento con conjuntos de datos grandes.

El Hook `useMemo` es la solución en esta situación: movemos nuestro filtro dentro de `useMemo`. Veamos la sintaxis:

```tsx
const filteredTodoList = useMemo(() => SomeProcessHere, [])
```

El Hook `useMemo` memorizará el resultado (valor) de una función y tendrá algunas dependencias a las que escuchar. Veamos cómo podemos implementarlo:

```tsx
const filteredTodoList = useMemo(() => todoList.filter((todo: Todo) => {
  console.log('Filtering...')
  return todo.task.toLowerCase().includes(term.toLowerCase())
}), [])
```

Si intentas hacer clic en el botón Search, no se filtrará porque faltan las dependencias (advertencia `react-hooks/exhaustive-deps`). Debes agregar las dependencias `term` y `todoList` al arreglo:

```tsx
const filteredTodoList = useMemo(() => todoList.filter((todo: Todo) => {
  console.log('Filtering...')
  return todo.task.toLowerCase().includes(term.toLowerCase())
}), [term, todoList])
```

### Memoización de una definición de función con `useCallback`

Ahora agregaremos una función para eliminar tareas para aprender cómo funciona `useCallback`. Lo primero que debemos hacer es crear una nueva función llamada `handleDelete` en nuestro componente `App`:

```tsx
const handleDelete = (taskId: number) => {
  const newTodoList = todoList.filter((todo: Todo) => todo.id !== taskId)
  setTodoList(newTodoList)
}
```

Y luego pasamos esta función al componente `List` como prop:

```tsx
<List todoList={filteredTodoList} handleDelete={handleDelete} />
```

Luego, en nuestro componente `List`, agregamos la prop a la interfaz `Props`:

```tsx
interface Props {
  todoList: Todo[]
  handleDelete: any
}
```

A continuación, la extraemos de las props y la pasamos al componente `Task`:

```tsx
const List: FC<Props> = ({ todoList, handleDelete }) => {
  useEffect(() => {
    // This effect is executed every new render
    console.log('Rendering <List />')
  })

  return (
    <ul>
       {todoList.map((todo: Todo) => (
         <Task
          key={todo.id}
          id={todo.id}
          task={todo.task}
          handleDelete={handleDelete}
          />
      ))}
    </ul>
  )
}
```

En el componente `Task`, creamos un botón que ejecutará `handleDelete` en el `onClick`:

```tsx
interface Props {
  id: number
  task: string
  handleDelete: any
}

const Task: FC<Props> = ({ id, task, handleDelete }) => {
  useEffect(() => {
    console.log('Rendering <Task />', task)
  })

  return (
    <li>{task} <button onClick={() => handleDelete(id)}>X</button></li>
  )
}
```

El problema ahora es que nuestra función `handleDelete` se pasa a dos componentes (de `App` a `List` y luego a `Task`), y esta función se regenera cada vez que tenemos un nuevo re-renderizado (por ejemplo, cada vez que escribimos algo en el input).

El Hook `useCallback` soluciona esto: en lugar de memorizar el valor resultante de una función (como hace `useMemo`), memoriza la **definición de la función**:

```tsx
 const handleDelete = useCallback(() => SomeFunctionDefinition, [])
```

Nuestra función `handleDelete` debería quedar así:

```tsx
const handleDelete = useCallback((taskId: number) => {
  const newTodoList = todoList.filter((todo: Todo) => todo.id !== taskId)
  setTodoList(newTodoList)
}, [todoList])
```

### Memoización de una función pasada como argumento en un efecto

Hay un caso especial en el que necesitaremos usar el Hook `useCallback`, y es cuando pasamos una función como argumento en un Hook `useEffect`. Por ejemplo, en nuestro componente `App`, creemos un nuevo bloque `useEffect`:

```tsx
const printTodoList = () => {
  console.log('Changing todoList')
}

useEffect(() => {
  printTodoList()
}, [todoList])
```

Si agregamos `todoList` al console log:

```tsx
const printTodoList = () => {
  console.log('Changing todoList', todoList)
}
```

Si estás utilizando Visual Studio Code, obtendrás una advertencia que te pedirá que agregues la función `printTodoList` a las dependencias:

```tsx
useEffect(() => {
  printTodoList()
}, [todoList, printTodoList])
```

Pero después de hacer eso, obtenemos una advertencia de `useCallback` porque estamos manipulando un estado, por lo que debemos envolver la función con `useCallback`:

```tsx
const printTodoList = useCallback(() => {
  console.log('Changing todoList', todoList)
}, [todoList])
```

### Resumen rápido de optimización

- **`memo`:**
  - Memoriza un componente.
  - Vuelve a renderizar solo cuando las props cambian.
  - Evita re-renderizados innecesarios.
- **`useMemo`:**
  - Memoriza un valor calculado.
  - Para propiedades calculadas y procesos pesados.
- **`useCallback`:**
  - Memoriza la definición de una función para evitar redefinirla en cada renderizado.
  - Úsalo siempre que se pase una función como argumento de un efecto o por props a un componente memoizado.

---

## Comprensión del Hook `useReducer`

Si tienes experiencia usando Redux (`react-redux`) con componentes de clase, comprenderás cómo funciona `useReducer`. Los conceptos son básicamente los mismos: acciones (*actions*), reductores (*reducers*), despacho (*dispatch*), almacén (*store*) y estado (*state*). La principal diferencia es que `react-redux` proporciona middleware y envoltorios como `thunk`, `sagas` y muchos más, mientras que `useReducer` solo te brinda un método `dispatch` que puedes usar para despachar objetos planos como acciones. Además, `useReducer` no tiene un store global por defecto; su ámbito está limitado al componente y sus hijos.

Creemos una aplicación básica de Notas para entender cómo funciona `useReducer`:

```bash
create-vite reducer --template react-ts
```

En el componente `App.tsx`:

```tsx
import Notes from './Notes'

function App() {
  return (
     <Notes />
  )
}

export default App
```

Ahora, en nuestro componente `Notes.tsx`, importamos `useReducer` y `useState`:

```tsx
import { useReducer, useState, ChangeEvent } from 'react'
```

Definimos los tipos de TypeScript para nuestro objeto `Note`, la acción y los tipos de acción:

```tsx
type Note = {
  id: number
  note: string
}

type Action = {
  type: string
  payload?: any
}

type ActionTypes = {
  ADD: 'ADD'
  UPDATE: 'UPDATE'
  DELETE: 'DELETE'
}

const actionType: ActionTypes = {
  ADD: 'ADD',
  DELETE: 'DELETE',
  UPDATE: 'UPDATE'
}
```

Creamos `initialNotes` (también conocido como `initialState`):

```tsx
const initialNotes: Note[] = [
  {
    id: 1,
    note: 'Note 1'
  },
  {
    id: 2,
    note: 'Note 2'
  }
]
```

Definimos la función reductora:

```tsx
const reducer = (state: Note[], action: Action) => {
  switch (action.type) {
    case actionType.ADD:
      return [...state, action.payload]

    case actionType.DELETE:
      return state.filter(note => note.id !== action.payload)
  
    case actionType.UPDATE:
      const updatedNote = action.payload
      return state.map((n: Note) => n.id === updatedNote.id ? updatedNote : n)
  
   default:
       return state
  }
}
```

En el componente, obtenemos `notes` y el método `dispatch` del Hook `useReducer`:

```tsx
const Notes = () => {
  const [notes, dispatch] = useReducer(reducer, initialNotes)
  const [note, setNote] = useState<string>('')
  ...
}
```

Creamos la función `handleSubmit`:

```tsx
import type { FormEvent } from 'react'

const handleSubmit = (e: FormEvent<HTMLFormElement>) => {
  e.preventDefault()

  const newNote = {
    id: Date.now(),
    note
  }

  dispatch({ type: actionType.ADD, payload: newNote })
}
```

Y renderizamos la lista de notas junto con las acciones de eliminar y actualizar:

```tsx
return (
  <div>
    <h2>Notes</h2>

    <ul>
       {notes.map((n: Note) => (
         <li key={n.id}>
               {n.note} {' '}
          
            <button onClick={() => dispatch({ type: actionType.DELETE, payload: n.id })}>
           X
         </button>

         <button
                 onClick={() => dispatch({ type: actionType.UPDATE, payload: {...n, note} })}
         >
           Update
         </button>
         </li>
      ))}
    </ul>
    
    <form onSubmit={handleSubmit}>
      <input
            placeholder="New note"
            value={note}
            onChange={e => setNote(e.target.value)}
      />
    </form>
  </div>
)

export default Notes
```

---

## El nuevo React Compiler: optimización y memoización automática

El React Compiler cambia fundamentalmente la forma en que pensamos sobre Hooks como `useMemo`, `useCallback` y `memo`. Tradicionalmente, los desarrolladores tenían que decidir manualmente cuándo aplicar estas optimizaciones, lo que a menudo conducía a un uso excesivo o a oportunidades perdidas. Con el compilador, muchas de estas decisiones se automatizan, lo que permite a los desarrolladores centrarse en la corrección y la claridad mientras el sistema de compilación se encarga de las optimizaciones de rendimiento.

### Introducción al React Compiler

El React Compiler representa un cambio de paradigma en la optimización de las aplicaciones de React. Creado por el equipo de React en Meta, este compilador opera a nivel de compilación, transformando tus componentes y hooks de React en versiones altamente optimizadas sin requerir ningún cambio en tu código fuente.

El compilador funciona analizando los gráficos de dependencias de tus componentes y hooks, identificando qué valores y funciones deben ser estables entre renderizados e insertando automáticamente la memoización adecuada. Puede detectar cuándo las props de un componente no han cambiado, cuándo las dependencias de una función callback permanecen estables o cuándo no es necesario repetir un cálculo costoso.

Considera este componente típico de React:

```tsx
interface UserCardProps {
  user: {
    id: string;
    name: string;
    email: string;
    avatar: string;
  };
  onEdit: (userId: string) => void;
  onDelete: (userId: string) => void;
}

const UserCard = ({ user, onEdit, onDelete }: UserCardProps) => {
  const formatUserName = (name: string) => {
    return name.split(' ').map(part =>
      part.charAt(0).toUpperCase() + part.slice(1).toLowerCase()
    ).join(' ')
  }

  const handleEdit = () => {
    onEdit(user.id)
  }

  const handleDelete = () => {
    onDelete(user.id)
  }

  const initials = user.name.split(' ')
    .map(n => n[0])
    .join('')
    .toUpperCase()

  return (
    <div className="bg-white rounded-lg shadow-md p-6 max-w-sm">
      <div className="flex items-center space-x-4">
        {user.avatar ? (
          <img
            src={user.avatar}
            alt={user.name}
            className="w-12 h-12 rounded-full object-cover"
          />
        ) : (
          <div className="w-12 h-12 rounded-full bg-blue-500 flex items-center justify-center text-white font-bold">
            {initials}
          </div>
        )}
        <div className="flex-1">
          <h3 className="text-lg font-semibold text-gray-900">
            {formatUserName(user.name)}
          </h3>
          <p className="text-gray-600">{user.email}</p>
        </div>
      </div>
      <div className="mt-4 flex space-x-2">
        <button  
          onClick={handleEdit}
          className="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600 transition-colors"
        >
          Edit
        </button>
        <button
          onClick={handleDelete}
          className="px-4 py-2 bg-red-500 text-white rounded hover:bg-red-600 transition-colors"
        >
          Delete
        </button>
      </div>
    </div>
  )
}
```

En el desarrollo tradicional de React, podrías optimizar manualmente este componente envolviéndolo con `React.memo`, memorizando la función `formatUserName` con `useCallback` o memorizando las iniciales calculadas con `useMemo`. El React Compiler elimina este trabajo manual al identificar automáticamente estas oportunidades de optimización y aplicarlas durante el proceso de compilación.

### Cómo automatiza el compilador `useMemo` y `useCallback`

El compilador utiliza análisis estático avanzado para determinar exactamente cuándo la memoización evitará re-renderizados innecesarios o recálculos costosos:

```tsx
interface ProductListProps {
  products: Product[];
  categories: Category[];
  filters: FilterConfig;
  onFilterChange: (filters: FilterConfig) => void;
  onProductSelect: (productId: string) => void;
}

const ProductList = ({
  products,
  categories,
  filters,
  onFilterChange,
  onProductSelect
}: ProductListProps) => {
  const filteredProducts = products.filter(product => {
    if (filters.category && product.categoryId !== filters.category) {
      return false
    }
    if (filters.minPrice && product.price < filters.minPrice) {
      return false
    }
    if (filters.maxPrice && product.price > filters.maxPrice) {
      return false
    }
    if (filters.searchTerm) {
      const searchLower = filters.searchTerm.toLowerCase();
      return product.name.toLowerCase().includes(searchLower) ||
             product.description.toLowerCase().includes(searchLower)
    }
    return true
  })

  const groupedProducts = filteredProducts.reduce((groups, product) => {
    const category = categories.find(cat => cat.id === product.categoryId)
    const categoryName = category?.name || 'Uncategorized'
   
    if (!groups[categoryName]) {
      groups[categoryName] = []
    }
    groups[categoryName].push(product)
    return groups
  }, {} as Record<string, Product[]>)

  const handleProductClick = (productId: string) => {
    onProductSelect(productId)
  }

  const handleFilterReset = () => {
    onFilterChange({})
  }

  return (
    <div className="space-y-6">
      <div className="flex justify-between items-center">
        <h2 className="text-2xl font-bold text-gray-900">
          Products ({filteredProducts.length})
        </h2>
        <button
          onClick={handleFilterReset}
          className="px-4 py-2 text-sm bg-gray-200 text-gray-700 rounded hover:bg-gray-300"
        >
          Reset Filters
        </button>
      </div>
     
      {Object.entries(groupedProducts).map(([categoryName, categoryProducts]) => (
        <div key={categoryName} className="space-y-4">
          <h3 className="text-lg font-semibold text-gray-800 border-b pb-2">
            {categoryName}
          </h3>
          <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
            {categoryProducts.map(product => (
              <ProductCard
                key={product.id}
                product={product}
                onSelect={handleProductClick}
              />
            ))}
          </div>
        </div>
      ))}
    </div>
  )
}
```

El React Compiler analiza este componente y optimiza automáticamente varios aspectos: reconoce que `filteredProducts` depende solo de `products` y `filters`, haciendo que sea un candidato ideal para la memoización. El cálculo costoso de `groupedProducts`, que depende de `filteredProducts` y `categories`, también se memoiza automáticamente. Las funciones `handleProductClick` y `handleFilterReset` se estabilizan ya que se pasan a componentes secundarios.

### Uso del compilador con Next.js

Para comenzar con el React Compiler en tu proyecto de Next.js, primero instala el plugin de Babel:

```bash
npm install babel-plugin-react-compiler
```

Luego, habilita el compilador en tu configuración de Next.js agregando la opción experimental `reactCompiler` a tu archivo `next.config.ts`:

```typescript
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  experimental: {
    reactCompiler: true
  }
}

export default nextConfig
```

---

## Hooks modernos en React 19

React 19 introduce un nuevo conjunto de Hooks que se centran en mejorar la experiencia del usuario, la concurrencia y las interacciones con el servidor.

### `useTransition`: gestión de actualizaciones no bloqueantes

El Hook `useTransition` te permite marcar actualizaciones de estado como no urgentes, permitiendo que React mantenga la interfaz interactiva mientras realiza actualizaciones pesadas en segundo plano:

```tsx
const [isPending, startTransition] = useTransition()

const handleSearch = (value: string) => {
  startTransition(() => {
    setSearchTerm(value)
  })
}
```

### `useDeferredValue`: aplazamiento de cálculos costosos

`useDeferredValue` te permite retrasar la actualización de un valor para evitar re-renderizados innecesarios durante actualizaciones rápidas (como escribir en un input):

```tsx
const deferredSearch = useDeferredValue(searchTerm)
const filteredList = useMemo(() => {
  return items.filter(item => item.includes(deferredSearch))
}, [deferredSearch])
```

### `useOptimistic`: actualizaciones optimistas de la interfaz

El Hook `useOptimistic` te permite reflejar inmediatamente los cambios en la UI antes de que el servidor confirme la actualización:

```tsx
const [optimisticTodos, addOptimisticTodo] = useOptimistic(todos)
const handleAdd = async (todo) => {
  addOptimisticTodo(todo)
  await saveTodo(todo)
}
```

### `useActionState`: gestión de acciones asíncronas de formularios

`useActionState` simplifica el manejo del envío asíncrono de formularios y Server Actions:

```tsx
const [state, submitAction] = useActionState(async (prevState, formData) => {
  const result = await submitForm(formData)
  return result
})
```

### `useEffectEvent`: controladores de eventos estables dentro de efectos

`useEffectEvent` ayuda a evitar cierres obsoletos (*stale closures*) cuando se trabaja con controladores de eventos dentro de efectos sin requerir re-suscripciones innecesarias:

```tsx
const onChange = useEffectEvent((value) => {
  console.log(value)
})

useEffect(() => {
  window.addEventListener('resize', () => onChange(window.innerWidth))
}, [])
```

### `useImperativeHandle`: APIs imperativas controladas

Permite exponer métodos imperativos controlados desde componentes a través de referencias:

```tsx
useImperativeHandle(ref, () => ({
  focus: () => inputRef.current?.focus()
}))
```

### Cuándo usar los Hooks modernos

- Usa `useTransition` y `useDeferredValue` para rendimiento y capacidad de respuesta.
- Usa `useOptimistic` y `useActionState` para flujos de trabajo asíncronos y formularios.
- Usa `useEffectEvent` para lógica estable de efectos secundarios.
- Usa `useImperativeHandle` para interacciones imperativas controladas.

---

## Resumen

En este capítulo, exploramos cómo los React Hooks evolucionaron de una simple alternativa a los componentes de clase a un sistema integral para administrar el estado, los efectos y el rendimiento. Mientras que los Hooks fundamentales como `useState` y `useEffect` siguen siendo esenciales, el desarrollo moderno en React se apoya cada vez más en nuevos Hooks como `useTransition`, `useOptimistic` y `useDeferredValue` para manejar la concurrencia y los flujos asíncronos de manera más efectiva. Junto con herramientas como el React Compiler, estos patrones permiten a los desarrolladores crear aplicaciones que son más sencillas de razonar y más eficientes por defecto.

A continuación, nos sumergiremos en **React Router 7**, cubriendo loaders/actions para la obtención de datos, APIs de formularios, respuestas diferidas/en streaming, integración SSR y consejos prácticos de migración desde v6.
