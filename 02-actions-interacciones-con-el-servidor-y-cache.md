# Capítulo 2: Actions, Interacciones con el Servidor y Caché

Las aplicaciones web modernas exigen interacciones fluidas con el servidor y experiencias de usuario altamente interactivas. React 19 introduce las **Actions**, un enfoque revolucionario que cambia fundamentalmente la forma en que manejamos las operaciones del lado del servidor, los envíos de formularios y las mutaciones de datos. Este capítulo explora cómo las Actions simplifican las interacciones con el servidor manteniendo un rendimiento óptimo a través de estrategias inteligentes de almacenamiento en caché.

El capítulo cubre los siguientes temas:

- Comprensión de las Actions en React 19
- Manejo de envíos de formularios con `use server`
- Actualizaciones optimistas de la interfaz con `useOptimistic`
- Gestión del estado del formulario con `useFormStatus` y `useActionState`
- Mejora del rendimiento con estrategias de caché
- ¿Qué hay de nuevo en Next.js 16?

---

## Comprensión de las Actions en React 19

Las Actions representan un cambio de paradigma en el enfoque de React hacia las interacciones del lado del servidor. En lugar de gestionar complejas llamadas a la API desde el lado del cliente, las Actions proporcionan una forma declarativa de manejar operaciones del servidor directamente dentro de tus componentes de React.

### Por qué las Actions mejoran las interacciones con el servidor

Las aplicaciones tradicionales de React requieren que los desarrolladores orquesten múltiples partes móviles para las interacciones con el servidor: gestionar estados de carga, manejar errores, coordinar llamadas a la API y actualizar el estado de la interfaz de usuario. Esta complejidad a menudo conduce a una lógica dispersa y experiencias de usuario inconsistentes.

Las Actions consolidan estas responsabilidades en un único patrón cohesivo. Cuando defines una Action, React gestiona automáticamente los estados pendientes, los errores y las actualizaciones optimistas a través de hooks integrados como `useTransition`, `useActionState` y `useOptimistic`. Esto reduce el código repetitivo (*boilerplate*) y garantiza un comportamiento consistente en toda tu aplicación.

Considera la diferencia de complejidad al manejar el envío de un formulario simple. Los enfoques tradicionales requieren hooks `useState` para los estados de carga, lógica de manejo de errores, llamadas `fetch` y actualizaciones manuales de estado. Las Actions encapsulan todas estas preocupaciones, permitiéndote concentrarte en la lógica de negocio en lugar de la infraestructura.

Las Actions también se integran estrechamente con el modelo de renderizado concurrente de React. React mantiene la interfaz de usuario interactiva mientras una Action está en curso y realiza transiciones de estado sin bloquear la entrada del usuario. Al combinarse con un framework como Next.js, optimizaciones adicionales como la deduplicación de solicitudes, el almacenamiento en caché del lado del servidor y la revalidación automática ocurren a nivel del framework, mejorando aún más el rendimiento sin esfuerzo adicional del desarrollador.

### Cómo las Actions reducen la complejidad de las APIs del lado del cliente

La gestión de APIs del lado del cliente ha sido históricamente uno de los aspectos más propensos a errores en el desarrollo con React. Los desarrolladores deben manejar fallos de red, condiciones de carrera (*race conditions*), datos obsoletos y estados de carga en numerosos componentes. Las Actions eliminan gran parte de esta complejidad al trasladar la lógica del servidor al servidor, donde corresponde.

Cuando utilizas Actions, el servidor se encarga directamente de la validación de datos, la lógica de negocio y las operaciones de base de datos. El cliente recibe respuestas limpias y validadas sin necesidad de gestionar estados intermedios o escenarios de error complejos. Esta separación de responsabilidades conduce a aplicaciones más mantenibles y confiables.

Las Actions también eliminan la necesidad de controladores de rutas de API personalizados (*custom API route handlers*) en muchos escenarios. En lugar de crear archivos de endpoints separados, puedes definir funciones de servidor directamente junto a tus componentes. Esta coubicación (*co-location*) mejora la experiencia del desarrollador y reduce la sobrecarga cognitiva de gestionar múltiples archivos para funcionalidades relacionadas.

El manejo de errores proporcionado por las Actions es particularmente valioso. En lugar de escribir lógica personalizada de manejo de errores y reintentos para cada llamada a la API, las Actions ofrecen patrones consistentes de manejo de errores, expuestos a través de hooks como `useActionState`, que funcionan en toda tu aplicación.

### Cómo funcionan las Actions con REST y GraphQL

Las Actions no son un reemplazo de REST o GraphQL; más bien, sirven como un puente entre tus componentes de React y tu infraestructura de backend. Cuando llamas a una Action desde el cliente, React no realiza una petición tradicional de REST o GraphQL. En su lugar, ejecuta de forma segura una llamada a procedimiento remoto (*Remote Procedure Call - RPC*) hacia el servidor mediante una petición POST especializada. Sin embargo, dentro de la función de la Action en el servidor, aún puedes realizar peticiones a tus APIs existentes de REST o GraphQL. Lo que proporcionan las Actions es una forma estandarizada de desencadenar lógica del lado del servidor y manejar mutaciones, envíos de formularios y sus estados asociados (pendiente, éxito, error) directamente dentro del modelo de componentes de React:

```tsx
// Actions still call your REST API
async function createUser(formData: FormData) {
  'use server'
 
  const response = await fetch('https://api.example.com/users', {
    method: 'POST',
    body: JSON.stringify({
      name: formData.get('name'),
      email: formData.get('email'),
    }),
    headers: { 'Content-Type': 'application/json' },
  })
 
  if (!response.ok) throw new Error('Failed to create user')
  return response.json()
}

// Or your GraphQL API
async function createUserGraphQL(formData: FormData) {
  'use server'
 
  const response = await fetch('/graphql', {
    method: 'POST',
    body: JSON.stringify({
      query: `mutation CreateUser($input: UserInput!) {
        createUser(input: $input) { id name email }
      }`,
      variables: {
        input: {
          name: formData.get('name'),
          email: formData.get('email'),
        },
      },
    }),
    headers: { 'Content-Type': 'application/json' },
  })
 
  return response.json()
}
```

Lo que simplifican las Actions es la parte de React: no necesitas estados de carga, estados de error ni controladores de éxito independientes dispersos por tus componentes. El hook `useActionState` gestiona esto automáticamente. Pero tu capa de API, ya sea REST, GraphQL u otra cosa, permanece sin cambios. Las Actions tratan sobre cómo maneja React las mutaciones, no sobre reemplazar tu infraestructura de obtención de datos.

### Uso de `use server` para interacciones directas con el servidor

La directiva `use server` marca las funciones que deben ejecutarse en el servidor en lugar del cliente. Esta sencilla anotación transforma funciones regulares de JavaScript en Server Actions que se pueden invocar directamente desde tus componentes de React:

```tsx
'use server'

import { db } from '@/lib/db'

export async function createUser(formData: FormData): Promise<{ success: boolean; user: any }> {
  const name = formData.get('name') as string | null
  const email = formData.get('email') as string | null
  // Server-side validation
  if (!name || !email) {
    throw new Error('Name and email are required')
  }
  // Database operation
  const user = await db.user.create({
    data: { name, email }
  })
  return { success: true, user }
}
```

Las Server Actions tienen acceso a todos los recursos del lado del servidor: bases de datos, sistemas de archivos, variables de entorno y APIs externas. Esto elimina la necesidad de rutas de API intermedias en muchos escenarios. Puedes interactuar con tu base de datos directamente desde la Action, aplicar lógica de negocio y devolver los resultados al cliente.

La seguridad de tipos que ofrecen las Actions es particularmente potente. TypeScript puede inferir los tipos de los parámetros y del valor de retorno de tus funciones de servidor, proporcionando seguridad en tiempo de compilación para tus interacciones con el servidor. Esto elimina categorías enteras de errores en tiempo de ejecución causados por discrepancias en los contratos de la API.

### Manejo de mutaciones y envíos de formularios sin rutas de API

El manejo tradicional de formularios en Next.js requiere la creación de rutas de API, la gestión del análisis de solicitudes y la coordinación entre el código del cliente y del servidor. Las Actions eliminan esta complejidad al permitir llamadas a funciones directas desde los elementos del formulario:

```tsx
// components/UserForm.tsx
'use client'

import { createUser } from '../actions/users'

export default function UserForm() {
  return (
    <form action={createUser}>
      <input name="name" placeholder="Name" required />
      <input name="email" type="email" placeholder="Email" required />
      <button type="submit">Create User</button>
    </form>
  )
}
```

El componente `UserForm` renderiza un formulario HTML estándar con dos campos de entrada para nombre y correo electrónico. En lugar de especificar un endpoint URL en el atributo `action` del formulario, pasa directamente la Server Action `createUser`. Cuando un usuario envía el formulario, React recopila automáticamente los datos del formulario (usando los atributos `name` de los inputs) y los pasa a `createUser` como un objeto `FormData`. No se necesitan manejadores de eventos manuales, llamadas a `preventDefault()` ni peticiones `fetch` explícitas. La Server Action se ejecuta en el servidor, procesa los datos del formulario, realiza las operaciones de base de datos necesarias y puede devolver resultados o desencadenar una revalidación. A lo largo de este ciclo, React gestiona los estados pendientes y preserva la mejora progresiva, por lo que el formulario sigue siendo funcional incluso antes de que se haya cargado el JavaScript del lado del cliente.

Las Actions admiten la invocación programática, lo que te permite activar operaciones del servidor desde controladores de eventos u otra lógica del componente. Esta flexibilidad habilita tanto interacciones declarativas basadas en formularios como control programático imperativo según lo requiera tu aplicación.

Hemos explorado cómo las Actions transforman las interacciones con el servidor en React 19 al consolidar los estados de carga, el manejo de errores y las llamadas a la API en un patrón declarativo. La directiva `use server` permite llamadas a funciones de servidor directas y con seguridad de tipos sin rutas de API intermedias, eliminando gran parte de la complejidad asociada con los enfoques tradicionales de REST y GraphQL.

---

## Manejo de envíos de formularios con `use server`

Los envíos de formularios representan una de las interacciones con el servidor más comunes en las aplicaciones web. Las Actions transforman el manejo de formularios de un proceso complejo y propenso a errores en un patrón declarativo y directo.

### Envío de datos al servidor mediante Actions

Los datos de los formularios fluyen de forma natural del cliente al servidor a través de las Actions. La API nativa `FormData` del navegador se integra perfectamente con las funciones del servidor, eliminando la necesidad de serialización manual de datos o gestión compleja de estado:

```tsx
'use server'

import { revalidatePath } from 'next/cache'
import { db } from '@/lib/db'
import { auth } from '@/lib/auth' // Your auth solution (NextAuth, Clerk, etc.)

export async function updateProfile(
  formData: FormData
): Promise<{ success: boolean; profile: any }> {
  // Get userId from authenticated session, NOT from form data
  // Trusting client-provided userId lets any user update any profile
  const session = await auth()
 
  if (!session?.user?.id) {
    throw new Error('Unauthorized')
  }

  const userId = session.user.id
  const bio = formData.get('bio')

  const preferences = {
    newsletter: formData.get('newsletter') === 'on',
    notifications: formData.get('notifications') === 'on',
  }

  const updatedProfile = await db.profile.update({
    where: { userId },
    data: {
      bio: bio as string | null,
      preferences,
    },
  })

  revalidatePath('/profile')

  return { success: true, profile: updatedProfile }
}
```

La interfaz `FormData` proporciona una forma estandarizada de acceder a los valores de los campos del formulario, incluyendo soporte para subida de archivos, casillas de verificación y selecciones múltiples. Las Server Actions pueden procesar estos datos directamente sin bibliotecas adicionales de análisis o validación.

La subida de archivos se vuelve particularmente sencilla con las Actions. El objeto `FormData` contiene objetos `File` que se pueden procesar directamente en el servidor, eliminando la necesidad de análisis complejos de multipart o servicios externos de subida en muchos escenarios.

### Manejo de validaciones y errores en el servidor

La validación del lado del servidor dentro de las Actions proporciona un enfoque seguro y centralizado para la validación de datos. A diferencia de la validación del lado del cliente, la validación en el servidor no puede ser omitida por usuarios malintencionados, lo que garantiza la integridad y seguridad de los datos:

```tsx
'use server'

import { z } from 'zod'
import { db } from '@/lib/db'

const userSchema = z.object({
  name: z.string().min(2).max(50),
  email: z.string().email(),
  age: z.number().min(13).max(120),
})

// Infer the type from the schema
type UserInput = z.infer<typeof userSchema>

// Define what the database returns
interface User extends UserInput {
  id: string
  createdAt: Date
}

type CreateUserResult =
  | { success: true; user: User }
  | { success: false; errors: z.typeToFlattenedError<UserInput> }

export async function createUser(formData: FormData):Promise<CreateUserResult> {
  const raw = {
    name: formData.get('name'),
    email: formData.get('email'),
    age: Number(formData.get('age')),
  }

  const result = userSchema.safeParse(raw)

  if (!result.success) {
    return { success: false, errors: result.error.flatten() }
  }

  const user = await db.user.create({ data: result.data })

  return { success: true, user }
}
```

El manejo de errores en las Actions tiene dos caminos. Los errores recuperables, como los fallos de validación o las violaciones de reglas de negocio, se devuelven mejor como parte del resultado de la Action para que puedas renderizar comentarios en línea junto al campo de formulario correspondiente. Los errores inesperados que lanzas (`throw`) dentro de una Server Action se serializan y se muestran al cliente, donde pueden capturarse a través de `useActionState` o, en Next.js, escalarse al segmento de ruta `error.tsx` más cercano. Esto te brinda flexibilidad en cómo presentas los errores a los usuarios mientras mantienes un patrón consistente en toda la aplicación.

Las bibliotecas de validación como Zod se integran de forma natural con las Actions, ofreciendo tanto validación en tiempo de ejecución como inferencia de tipos de TypeScript. Esta combinación asegura que tus datos se validen de forma consistente y que tus tipos de TypeScript se mantengan precisos en toda la aplicación.

### Retorno de respuestas y actualización de la interfaz de usuario

Las Actions pueden devolver cualquier dato serializable, lo que te permite proporcionar una retroalimentación enriquecida a tus componentes. Los datos devueltos activan automáticamente re-renderizados en React, asegurando que tu UI se mantenga sincronizada con el estado del servidor:

```tsx
'use server'

import { revalidatePath } from 'next/cache'
import { db } from '@/lib/db'
import { auth } from '@/lib/auth'

export async function toggleFavorite(
  postId: string
): Promise<{ success: boolean; isFavorited: boolean; favoriteCount: number }> {
  // Get user from session, not from client
  const session = await auth()
 
  if (!session?.user?.id) {
    throw new Error('Unauthorized')
  }

  const userId = session.user.id

  // Check if user already favorited this post
  const existingFavorite = await db.favorite.findUnique({
    where: {
      userId_postId: { userId, postId },
    },
  })

  if (existingFavorite) {
    // Remove favorite
    await db.favorite.delete({
      where: { id: existingFavorite.id },
    })
  } else {
    // Add favorite
    await db.favorite.create({
      data: { userId, postId },
    })
  }

  // Get updated count
  const favoriteCount = await db.favorite.count({
    where: { postId },
  })

  revalidatePath('/posts')

  return {
    success: true,
    isFavorited: !existingFavorite,
    favoriteCount,
  }
}
```

La función `revalidatePath` asegura que los datos almacenados en caché se actualicen después de las mutaciones, manteniendo la consistencia en toda tu aplicación. Esta invalidación automática de caché elimina la compleja gestión de caché requerida con los enfoques de API tradicionales.

Los datos de respuesta pueden incluir valores calculados, estado derivado o cualquier otra información que tus componentes necesiten para actualizar su presentación. Esto elimina llamadas adicionales a la API y reduce la complejidad de la gestión del estado en el cliente.

Hemos examinado cómo las Actions simplifican los envíos de formularios a través de la integración nativa con `FormData`, eliminando la serialización manual y la gestión compleja del estado. Has visto cómo implementar una validación segura en el servidor con Zod, manejar errores a través de los patrones de límites de React y devolver respuestas enriquecidas que activan automáticamente actualizaciones en la interfaz de usuario. La función `revalidatePath` asegura la consistencia de la caché sin la lógica compleja de invalidación que exigían los enfoques tradicionales.

---

## Actualizaciones optimistas de la interfaz con `useOptimistic`

Las actualizaciones optimistas de la interfaz de usuario brindan retroalimentación inmediata a los usuarios al actualizar la interfaz antes de la confirmación del servidor. Este patrón mejora significativamente el rendimiento percibido y crea experiencias de usuario mucho más reactivas.

### Comprensión de la UI optimista y por qué es importante

Los usuarios esperan una retroalimentación inmediata cuando interactúan con aplicaciones web. Las interacciones tradicionales con el servidor introducen una latencia que puede hacer que las aplicaciones se sientan lentas, incluso cuando el tiempo de respuesta real del servidor es razonable. La UI optimista aborda esto actualizando la interfaz de inmediato y luego reconciliando con la respuesta del servidor.

El impacto psicológico de la UI optimista es significativo. Los usuarios perciben las aplicaciones como más rápidas y reactivas cuando sus acciones producen una respuesta visual instantánea. Esta mejor percepción conduce a una mayor satisfacción y compromiso del usuario.

Sin embargo, la UI optimista introduce complejidad en torno al manejo de errores y la reconciliación de estados. Cuando las operaciones del servidor fallan, los cambios optimistas deben revertirse y se debe informar a los usuarios sobre el fallo. El hook `useOptimistic` de React 19 proporciona un enfoque estructurado para gestionar estos escenarios.

### Implementación de `useOptimistic` para retroalimentación instantánea

El hook `useOptimistic` gestiona las actualizaciones optimistas de estado proporcionando un valor de estado y una función de actualización. El hook maneja automáticamente la transición del estado optimista al estado confirmado cuando se completan las operaciones del servidor:

```tsx
'use client'

import { useOptimistic } from 'react'
import { addComment } from '../actions/comments'

type Comment = {
  id: string
  content: string
  author: string
  createdAt: string
  pending?: boolean
}

type CommentListProps = {
  comments: Comment[]
  currentUser: { name: string }
}

export default function CommentList({ comments, currentUser}:CommentListProps) {
  const [optimisticComments, addOptimisticComment] = useOptimistic<Comment[], Comment>(
    comments,
    (state, newComment) => [...state, newComment]
  )

  async function handleAddComment(formData: FormData) {
    const content = formData.get('content') as string

    if (!content) return

    const optimisticComment: Comment = {
      id: `temp-${Date.now()}`,
      content,
      author: currentUser.name,
      createdAt: new Date().toISOString(),
      pending: true
    }

    addOptimisticComment(optimisticComment)

    try {
      await addComment(formData)
    } catch (error) {
      console.error('Failed to add comment:', error)
    }
  }
  return (
    <div>
      {optimisticComments.map(comment => (
        <Comment
          key={comment.id}
          comment={comment}
          isPending={comment.pending ?? false}
        />
      ))}
      <form action={handleAddComment}>
        <textarea name="content" placeholder="Add a comment..." />
        <button type="submit">Post Comment</button>
      </form>
    </div>
  )
}
```

El componente `CommentList` demuestra cómo `useOptimistic` ofrece retroalimentación instantánea para operaciones asíncronas. El hook recibe el arreglo actual de comentarios y una función de actualización que agrega nuevos comentarios. Cuando un usuario envía un comentario, `handleAddComment` crea de inmediato un objeto de comentario temporal con un ID generado y lo marca como pendiente. Este comentario optimista se agrega instantáneamente a la interfaz mediante `addOptimisticComment`, proporcionando una retroalimentación visual inmediata sin esperar al servidor. La función luego llama a la Server Action `addComment` de forma asíncrona. Una vez que la Server Action se completa, el estado optimista se descarta y el componente se re-renderiza con la prop de comentarios actualizada proveniente del padre, generalmente actualizada a través de `revalidatePath` o un mecanismo de revalidación similar. Si la acción falla, la actualización optimista también se descarta, por lo que la interfaz de usuario vuelve a la prop de comentarios original, correspondiendo a la aplicación mostrar un mensaje de error al usuario.

### Reversión de cambios ante fallos en el servidor

El manejo elegante de fallos es crucial para las implementaciones de UI optimista. Los usuarios necesitan información clara cuando sus actualizaciones optimistas no pueden ser confirmadas por el servidor. El hook `useOptimistic` de React proporciona una reversión automática, pero las aplicaciones también deben proporcionar mensajes de error comprensibles para el usuario:

```tsx
import { useOptimistic, useState, useTransition } from 'react'

interface UseOptimisticActionResult<T, U> {
  optimisticState: T
  isPending: boolean
  error: string | null
  execute: (optimisticValue: U) => Promise<void>
  retry: () => Promise<void>
}

export function useOptimisticAction<T, U>(
  initialState: T,
  updateFn: (state: T, value: U) => T,
  serverAction: (value: U) => Promise<void>
): UseOptimisticActionResult<T, U> {
  const [optimisticState, addOptimistic] = useOptimistic(initialState, updateFn)
  const [error, setError] = useState<string | null>(null)
  const [isPending, startTransition] = useTransition()
  const [lastValue, setLastValue] = useState<U | null>(null)

  const execute = async (optimisticValue: U) => {
    setError(null)
    setLastValue(optimisticValue)

    startTransition(async () => {
      addOptimistic(optimisticValue)

      try {
        await serverAction(optimisticValue)
      } catch (err) {
        setError(err instanceof Error ? err.message : 'Something went wrong')
      }
    })
  }

  const retry = async () => {
    if (lastValue !== null) {
      await execute(lastValue)
    }
  }

  return { optimisticState, isPending, error, execute, retry }
}
```

Los mensajes de error deben ser contextuales y orientados a la acción. En lugar de notificaciones genéricas, proporciona detalles específicos sobre lo que salió mal y cómo el usuario puede resolverlo. Esto podría traducirse en un botón de reintento junto al elemento fallido, un mensaje de validación en línea sobre el campo correspondiente o una sugerencia de una acción alternativa.

El momento en que se presentan las notificaciones de error también importa. Mostrar un error en el instante en que ocurre puede ser desconcertante, mientras que una notificación retrasada podría confundir a un usuario que ya ha avanzado a otra sección. Las transiciones sutiles, un breve resalte en el elemento afectado o un mensaje flotante (*toast*) que permanezca el tiempo suficiente para ser percibido sin interrumpir el flujo suelen ser el equilibrio adecuado.

Hemos explorado cómo `useOptimistic` habilita una retroalimentación instantánea mostrando inmediatamente actualizaciones temporales mientras una Server Action se ejecuta en segundo plano. Cuando la acción finaliza, el estado optimista se descarta y la interfaz es impulsada por los datos actualizados del componente padre. Combinar esto con un manejo explícito de errores, como ofrecer una opción de reintento o revertir al último valor correcto conocido, garantiza que los usuarios reciban retroalimentación clara y procesable ante operaciones fallidas.

---

## Gestión del estado del formulario con `useFormStatus` y `useActionState`

Los formularios complejos requieren una gestión de estado sofisticada para manejar flujos de trabajo de varios pasos, retroalimentación de validación e interacciones de usuario. React 19 proporciona hooks especializados para gestionar el estado relacionado con formularios en conjunto con las Actions.

### Uso de `useFormStatus` para retroalimentación de carga y envío

El hook `useFormStatus` proporciona información en tiempo real sobre el estado de envío del formulario, permitiéndote crear interfaces reactivas que ofrecen a los usuarios información clara sobre las operaciones en curso:

```tsx
'use client'

import { useFormStatus } from 'react-dom'
import { updateProfile } from '../actions/profile'
import Spinner from './Spinner'

function SubmitButton(): JSX.Element {
  const { pending } = useFormStatus()

  return (
    <button
      type="submit"
      disabled={pending}
      className={`inline-flex items-center px-4 py-2 rounded ${
        pending ? 'opacity-50 cursor-not-allowed' : ''
      }`}
    >
      {pending ? (
        <>
          <Spinner className="w-4 h-4 mr-2" />
          Saving...
        </>
      ) : (
        'Save Changes'
      )}
    </button>
  )
}

export default function ProfileForm(): JSX.Element {
  return (
    <form action={updateProfile}>
      <input name="name" placeholder="Full Name" />
      <input name="email" type="email" placeholder="Email" />
      <textarea name="bio" placeholder="Bio" />
      <SubmitButton />
    </form>
  )
}
```

El hook `useFormStatus` debe ser llamado desde un componente que sea hijo directo o indirecto de un elemento `<form>`. Este alcance garantiza que la información de estado esté asociada con precisión al envío del formulario correspondiente.

Los estados de carga deben modelarse según el límite de ejecución de la acción. El envío de un formulario es atómico: se ejecuta una única mutación y el estado `pending` permanece en `true` hasta que concluye. No existe un procesamiento intrínseco por sección o por campo dentro de un único envío. Lograr cargas a nivel de campo o sección requiere dividir la interfaz en múltiples formularios o acciones independientes, lo que representa un enfoque arquitectónico diferente.

### Gestión del estado de formularios de varios pasos con `useActionState`

Los formularios de varios pasos requieren coordinación entre múltiples Actions y una gestión de estado persistente a través de los diferentes pasos. El hook `useActionState` proporciona un enfoque estructurado para gestionar estos flujos de trabajo complejos:

```tsx
'use client'

import React, { useActionState } from 'react'
import { validateStep, submitCompleteForm } from '../actions/formSteps'
import PersonalInfoStep from './PersonalInfoStep'
import AddressStep from './AddressStep'
import ReviewStep from './ReviewStep'
import SuccessMessage from './SuccessMessage'
import FormNavigation from './FormNavigation'

type State = { step: number; data: Record<string, any>; errors: Record<string, string> }
const initial: State = { step: 1, data: {}, errors: {} }

const fdToObj = (fd: FormData) =>
  Object.fromEntries([...fd.entries()].map(([k, v]) => [k, v instanceof File ? v : String(v)]))

async function action(prev: State, fd: FormData): Promise<State> {
  const payload = fdToObj(fd)
  const data = { ...prev.data, ...payload }
  const v = await validateStep(prev.step, data)
  if (!v.success) return { ...prev, data, errors: v.errors }
  if (prev.step === 3) {
    await submitCompleteForm(data)
    return { step: 4, data, errors: {} }
  }
  return { step: prev.step + 1, data, errors: {} }
}
export default function MultiStepForm() {
  const [s, formAction, pending] = useActionState<State, FormData>(action, initial)
  const [step, setStep] = React.useState(1)
  React.useEffect(() => setStep(s.step), [s.step])

  return (
    <form action={formAction}>
      {Object.entries(s.data).map(([k, v]) =>
        v instanceof File || v === null ? null : <input key={k} type="hidden" name={k} value={String(v)} />
      )}

{step===1&&<PersonalInfoStep errors={s.errors} />}
{step===2&&<AddressStep errors={s.errors} />}
{step===3&&<ReviewStep data={s.data} />}
{step===4&&<SuccessMessage />}

      {step < 4 && (
        <FormNavigation
          step={step}
          isPending={pending}
          onPrevious={() => setStep((x) => Math.max(1, x - 1))}
        />
      )}
    </form>
  )
}
```

El componente `MultiStepForm` gestiona un proceso de formulario de cuatro pasos utilizando `useActionState` para coordinar las transiciones de estado. La función `action` sirve como manejador de la acción, recibiendo el estado anterior y los nuevos datos del formulario en cada envío. Valida los datos del paso actual mediante la Server Action `validateStep` y, si la validación es exitosa, fusiona los datos del paso en el acumulado del formulario. Para los pasos 1 y 2, avanza al siguiente paso; en el paso 3, llama a `submitCompleteForm` con los datos completos y transiciona a un estado de éxito (paso 4). El componente renderiza condicionalmente el componente del paso apropiado según `state.step`, muestra los errores de validación y proporciona controles de navegación que permiten a los usuarios retroceder sin perder los datos introducidos previamente. El valor `pending` de `useActionState` rastrea si la acción se está procesando actualmente, permitiendo indicadores de carga en todo el formulario.

El hook `useActionState` puede simplificar significativamente la gestión del estado de formularios en distintos envíos, exponiendo los estados pendiente y de error. Sin embargo, no elimina por completo la necesidad de `useState` del lado del cliente en flujos de varios pasos. La navegación (como retroceder entre pasos), el estado exclusivo de la UI local (paneles abiertos/cerrados, gestión de foco, notificaciones emergentes temporales) y la experiencia de usuario específica de cada paso a menudo siguen requiriendo estado del lado del cliente o estructuras adicionales junto al estado de la acción.

La persistencia del estado entre sesiones del navegador se puede incorporar guardando el borrador del formulario en `localStorage` o `sessionStorage`. En la práctica, esto generalmente se conecta a través de efectos del cliente: restaurar el borrador después de la hidratación y luego mantener el almacenamiento sincronizado cada vez que la acción genera un nuevo estado estable. Se debe tener cuidado de no leer desde el almacenamiento durante el renderizado; de lo contrario, pueden producirse discrepancias en la hidratación (*hydration mismatches*).

### Manejo de reinicio de formularios y recuperación de errores

La funcionalidad de reinicio de formularios y los mecanismos de recuperación de errores son esenciales para una experiencia de usuario amigable. Los usuarios deben poder limpiar formularios, recuperarse de errores y reiniciar procesos sin perder el contexto:

```tsx
import { useActionState, useRef, useState } from 'react'

type WithLast = { lastFormData?: FormData }

export function useFormWithReset<T extends object>(
  initialState: T & WithLast,
  actionFn: (prev: T & WithLast, fd: FormData) => Promise<T&WithLast>
) {
  const formRef = useRef<HTMLFormElement | null>(null)
  const [resetKey, setResetKey] = useState(0)

  const wrappedAction = async (prev: T & WithLast, fd: FormData) => {
    const next = await actionFn(prev, fd)
    return { ...next, lastFormData: fd }
  }

  const [state, formAction, isPending] = useActionState(wrappedAction, initialState)

  const resetForm = () => setResetKey((x) => x + 1)

  const retryLastAction = () => {
    // Re-submit with the same current DOM values (or you can rehydrate fields from state)
    // Storing FormData is still useful if you want to re-apply it to inputs.
    formRef.current?.requestSubmit()
  }

  return {
    state,
    formAction,
    isPending,
    resetForm,
    retryLastAction,
    resetKey,
    formRef
  }
}
```

El hook personalizado `useFormWithReset` extiende `useActionState` con capacidades de reinicio y reintento para un manejo de formularios más robusto. Envuelve la gestión de estado estándar de la acción y agrega un contador `resetKey` que fuerza el remontaje del formulario cuando se incrementa. Modificar la clave (`key`) le da al formulario un estado completamente limpio al desmontar y volver a montar todo el árbol de componentes con sus valores iniciales. El hook también expone una referencia `formRef`, que el consumidor asocia al elemento `<form>` junto con `resetKey` como la clave de React, por ejemplo: `<form key={resetKey} ref={formRef} action={formAction}>`. Sin esa conexión, los asistentes de reinicio y reintento no tienen sobre qué actuar.

Para el reintento, el hook almacena en el estado el último `FormData` enviado, manteniendo disponibles los valores emitidos previamente. La función `resetForm` incrementa la clave para provocar el remontaje, mientras que `retryLastAction` invoca `formRef.current?.requestSubmit()` para reenviar el formulario. Debido a que `requestSubmit` envía los valores que están actualmente en el DOM, el reintento funciona mejor inmediatamente después de un fallo, antes de que el usuario modifique los campos. Si necesitas garantizar que se reenvíe exactamente la misma carga útil anterior independientemente del estado de los campos, un enfoque más sólido es capturar una instantánea de objeto plano mediante `Object.fromEntries(fd)` y reproducirla llamando a la acción programáticamente. Este patrón es particularmente útil para formularios que experimentan fallos intermitentes o que requieren una recuperación sencilla frente a errores transitorios.

El autoguardado de formularios puede reducir aún más el riesgo de pérdida de datos durante sesiones prolongadas. Un enfoque común es suscribirse al estado producido por `useActionState` (o a los eventos `onChange` del formulario, con *debounce*) y escribir un borrador en `localStorage` después de cada actualización estable. Al montar el componente, restaura el borrador dentro de un efecto del cliente en lugar de durante el renderizado, para mantener consistente la hidratación.

Hemos explorado cómo `useActionState` gestiona flujos de trabajo complejos de formularios, desde formularios de varios pasos hasta mecanismos de recuperación de errores. El hook proporciona un control integral sobre el estado del formulario, habilitando características como navegación por pasos, manejo de errores de validación y funcionalidad de reintento a través de wrappers personalizados como `useFormWithReset`. Con una recuperación contextual de errores y patrones de autoguardado implementados, tus formularios se vuelven resilientes y accesibles. A continuación, examinemos cómo las estrategias de almacenamiento en caché pueden optimizar aún más el rendimiento y reducir las solicitudes innecesarias al servidor.

---

## Mejora del rendimiento con estrategias de caché

La optimización del rendimiento mediante un almacenamiento en caché inteligente es una piedra angular de las aplicaciones web modernas. En la práctica, el almacenamiento en caché es una preocupación por capas: React proporciona primitivas que influyen en el renderizado y el uso de datos, mientras que frameworks como Next.js implementan la caché y exponen APIs como `revalidatePath` y `revalidateTag` para invalidar y actualizar datos. Ser explícito sobre esta distinción es importante; la mayoría de los mecanismos de almacenamiento en caché con los que interactúan los desarrolladores en aplicaciones React son comportamientos a nivel de framework y no APIs de React propiamente dichas.

### Introducción al almacenamiento en caché en React 19

React 19 introduce sofisticados mecanismos de almacenamiento en caché que operan en múltiples niveles: componente, servidor y red. Estas capas de caché colaboran para minimizar cómputos redundantes, reducir peticiones de red y mejorar el rendimiento percibido.

Los frameworks modernos como Next.js buscan hacer que el almacenamiento en caché sea ergonómico, pero ya no es completamente transparente. Las versiones anteriores tendían a un almacenamiento en caché automático agresivo, lo que a menudo creaba confusión sobre cuándo se actualizarían los datos. Las versiones recientes se han movido hacia un modelo más explícito y de inclusión voluntaria (*opt-in*), donde los desarrolladores deben definir intencionalmente qué debe almacenarse en caché y cuándo ocurre la revalidación. El resultado es una mayor previsibilidad y un modelo mental más claro, incluso si requiere una configuración más deliberada.

Comprender la invalidación de caché es crucial para mantener la consistencia de los datos. React proporciona tanto invalidación automática basada en dependencias como invalidación manual a través de funciones como `revalidatePath` y `revalidateTag`. Este enfoque dual asegura que los datos en caché permanezcan actualizados mientras se maximizan los beneficios de rendimiento.

### Uso del almacenamiento en caché integrado de React para Server Components

Los Server Components pueden beneficiarse enormemente del almacenamiento en caché porque se ejecutan en un entorno donde el almacenamiento es amplio y la coordinación es centralizada. Sin embargo, React en sí mismo no almacena en caché automáticamente los renderizados ni las dependencias de datos. En su lugar, proporciona primitivas de bajo nivel como `cache()` que los desarrolladores deben aplicar manualmente. Los comportamientos de nivel superior, como la reutilización implícita de resultados o directivas como `use cache`, son características del framework y no APIs de React:

```tsx
// app/components/ProductList.tsx

import { db } from '@/lib/db'
import { Product, Review } from '@prisma/client'
import ProductCard from './ProductCard'

type ProductWithReviews = Product & {
  reviews: Review[]
}

type ProductListProps = {
  category: string
}

export default async function ProductList({ category }: ProductListProps) {
  const products: ProductWithReviews[] = await db.product.findMany({
    where: { category },
    include: { reviews: true },
  })

  return (
    <div>
      {products.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  )
}
```

```tsx
// app/data/getProductData.ts

'use server'

import { db } from '@/lib/db'
import { Product, Review, Variant } from '@prisma/client'

type ProductWithDetails = Product & {
  reviews: Review[]
  variants: Variant[]
}

export async function getProductData(productId: string): Promise<ProductWithDetails | null> {
  'use cache'

  const product = await db.product.findUnique({
    where: { id: productId },
    include: {
      reviews: true,
      variants: true,
    },
  })

  return product
}
```

Las claves de caché se generan automáticamente a partir de los argumentos pasados a una función en caché y, en frameworks como Next.js, a partir de cualquier valor capturado en el ámbito léxico circundante. Esto garantiza que los datos en caché se asocien con precisión al contexto específico en el que se generaron, evitando la contaminación de la caché y asegurando la coherencia.

El control manual de la caché permite una optimización del rendimiento muy precisa. A través de APIs del framework como `cacheLife` y `cacheTag` en Next.js, puedes especificar la duración de la caché, las condiciones de invalidación y las etiquetas de caché para crear estrategias sofisticadas que se adapten a los requisitos particulares de tu aplicación.

### Integración de Actions con React Query y SWR

Aunque las Server Actions pueden mutar datos en el servidor, librerías como React Query y SWR mantienen sus propias cachés en el lado del cliente. React en sí mismo no sincroniza automáticamente esas cachés con los resultados de las acciones. Al combinar estos sistemas, debes decidir explícitamente qué capa tiene la autoridad y actualizar las cachés del cliente tras el éxito de la acción.

En la práctica, la integración es directa: deja que la Server Action realice la mutación y luego invalida o actualiza manualmente la caché de React Query/SWR usando claves consistentes. No asumas que React invalidará la caché automáticamente; la coordinación ocurre en tu capa de caché del cliente (React Query/SWR) y, si corresponde, en la caché del framework (por ejemplo, revalidación de rutas o peticiones `fetch`).

Veamos primero la versión con **React Query**:

```tsx
'use client'

import { useMutation, useQueryClient } from '@tanstack/react-query'
import { updateUser } from '../actions/users'

type UserProfileProps = { userId: string }

export default function UserProfile({ userId }: UserProfileProps) {
  const queryClient = useQueryClient()

  const mutation = useMutation({
    mutationFn: updateUser,
    onSuccess: (result: { user: any }) => {
      // Patch cache immediately
      queryClient.setQueryData(['user', userId], result.user)
      // Then revalidate in background (optional)
      queryClient.invalidateQueries({ queryKey: ['user', userId] })
    },
    onError: (err) => {
      console.error('Update failed:', err)
    }
  })

  return (
    <form action={(fd) => mutation.mutate(fd)}>
      <input name="name" placeholder="Name" />
      <input name="email" type="email" placeholder="Email" />

      <button type="submit" disabled={mutation.isPending}>
        {mutation.isPending ? 'Updating...' : 'Update Profile'}
      </button>
    </form>
  )
}
```

Ahora veamos la versión con **SWR**:

```tsx
'use client'

import useSWR, { mutate } from 'swr'
import { updateUser } from '../actions/users'

type UserProfileProps = { userId: string }

export default function UserProfile({ userId }: UserProfileProps) {
  const key = ['user', userId] as const
  const { data } = useSWR(key)

  const handleSubmit = async (fd: FormData) => {
    const result: { user: any } = await updateUser(fd)

    // Patch SWR cache immediately (no refetch)
    await mutate(key, result.user, { revalidate: false })

    // If you prefer to refetch instead of patch:
    // await mutate(key)
  }

  return (
    <form action={handleSubmit}>
      <input name="name" placeholder="Name" defaultValue={data?.name ?? ''} />
      <input name="email" type="email" placeholder="Email" defaultValue={data?.email ?? ''} />

      <button type="submit">Update Profile</button>
    </form>
  )
}
```

La conclusión fundamental es que SWR no sabrá automáticamente que una Server Action ha modificado datos. La caché debe actualizarse explícitamente, ya sea mutándola con el valor devuelto para una actualización instantánea de la UI o desencadenando una petición de revalidación. Elegir entre actualizar directamente o volver a solicitar depende de tus garantías de consistencia, la tolerancia a la latencia y si el servidor puede aplicar transformaciones adicionales que el cliente deba recibir.

La clave para una integración exitosa es asegurarse de que las estrategias de invalidación de caché estén coordinadas entre la capa de almacenamiento en caché del framework y las bibliotecas externas. Utiliza claves de caché e iniciadores de invalidación consistentes para mantener la integridad de los datos en todas las capas. Por ejemplo, cuando una Server Action llama a `revalidateTag('posts', 'max')` en Next.js, la consulta del lado del cliente correspondiente en SWR o React Query debe tener una clave que permita invalidarla en el mismo paso lógico, normalmente desde el componente que activó la acción.

Las estrategias de reobtención en segundo plano (*background refetching*) deben alinearse entre los diferentes sistemas de almacenamiento en caché. Configura React Query o SWR para que sus ventanas de revalidación y la reobtención por foco o intervalo no entren en conflicto con las señales de invalidación del framework, lo que provocaría solicitudes redundantes o parpadeos entre datos obsoletos y actualizados.

### Evitar peticiones innecesarias e inconsistencias de estado

La consistencia de la caché entre diferentes componentes y sesiones de usuario requiere especial atención a los patrones de invalidación y a las dependencias de datos. Los estados de caché inconsistentes pueden generar experiencias de usuario confusas y problemas difíciles de depurar:

```tsx
'use server'

import { revalidatePath, revalidateTag } from 'next/cache'
import { db } from '@/lib/db'
import type { Post } from '@prisma/client'

export async function updatePost(
  postId: string,
  formData: FormData
): Promise<{ success: true; post: Post }> {
  const title = formData.get('title') as string | null
  const content = formData.get('content') as string | null

  const updatedPost = await db.post.update({
    where: { id: postId },
    data: {
      title,
      content,
      updatedAt: new Date(),
    },
  })

  revalidateTag(`post-${postId}`, 'max')
  revalidateTag('posts-list', 'max')
  revalidatePath('/posts')
  revalidatePath(`/posts/${postId}`)

  return { success: true, post: updatedPost }
}
```

La invalidación de caché basada en etiquetas proporciona un control granular sobre qué datos en caché deben actualizarse tras las mutaciones. Al etiquetar entradas de caché relacionadas, puedes asegurar que todos los datos relevantes se actualicen de manera consistente sin invalidar en exceso datos no relacionados.

Las actualizaciones optimistas deben coordinarse cuidadosamente con los sistemas de almacenamiento en caché para evitar inconsistencias. Cuando una Server Action exitosa confirma una actualización optimista, todas las entradas de caché relacionadas deben invalidarse para que la interfaz converja en el estado autorizado del servidor. Cuando una actualización optimista falla, el propio hook descarta el valor temporal y el estado base permanece sin cambios, por lo que la invalidación de la caché suele ser innecesaria; lo que importa en ese caso es mostrar el error al usuario.

### Caché del lado del servidor vs. caché del lado del cliente: cuándo usar cada una

La elección entre el almacenamiento en caché en el servidor y en el cliente depende de las características de los datos, los patrones de los usuarios y los requisitos de rendimiento. Cada enfoque tiene ventajas particulares y casos de uso apropiados.

El **almacenamiento en caché en el servidor** es ideal para datos costosos de calcular, compartidos entre múltiples usuarios o que requieren coordinación entre diferentes partes de la aplicación. Los resultados de consultas a bases de datos, las respuestas de APIs externas y las agregaciones calculadas son candidatos ideales para la caché del servidor.

El **almacenamiento en caché en el cliente** es óptimo para datos específicos del usuario, información a la que se accede con frecuencia y datos que se benefician de una disponibilidad inmediata. Las preferencias del usuario, los elementos vistos recientemente y el contenido personalizado funcionan muy bien con estrategias de caché del lado del cliente:

```tsx
'use cache'

import { db } from '@/lib/db'

type DateRange = {
  start: Date
  end: Date
}

type ProductAnalytics = {
  _sum: { total: number | null }
  _count: { id: number }
}

export async function getProductAnalytics(dateRange: DateRange): Promise<ProductAnalytics> {
  const analytics = await db.order.aggregate({
    where: {
      createdAt: {
        gte: dateRange.start,
        lte: dateRange.end,
      },
    },
    _sum: { total: true },
    _count: { id: true },
  })

  return analytics
}
```

Las estrategias de almacenamiento en caché híbridas combinan enfoques del lado del servidor y del cliente. Un patrón común es almacenar en caché del servidor la carga útil inicial, hidratarla en el cliente y luego permitir que la caché del cliente maneje las lecturas y mutaciones posteriores. Esto mantiene baja la latencia del primer renderizado mientras evita viajes de ida y vuelta repetidos para las interacciones que siguen.

El precalentamiento de caché (*cache warming*) llena las cachés con datos que probablemente se solicitarán pronto, antes de que el primer usuario los pida. Es muy útil para las cachés del lado del servidor donde los patrones de tráfico son predecibles y el cómputo subyacente es costoso. Los enfoques comunes incluyen la prerenderización programada de rutas de alto tráfico, tareas de revalidación en segundo plano, la precarga de datos inmediatamente después de mutaciones y el precalentamiento de consultas críticas durante el despliegue o arranque. El objetivo es apartar el trabajo de la ruta de la solicitud, reduciendo los picos de latencia bajo carga.

El almacenamiento en caché en el ecosistema de React y Next.js ha pasado de ser un comportamiento implícito gestionado por el framework a un modelo explícito y de inclusión voluntaria (*opt-in*). Utilizados en conjunto, las Server Actions, `useOptimistic` y directivas como `use cache` cubren las tres etapas principales de una mutación: enviar el cambio, reflejarlo en la interfaz de usuario y mantener consistentes las lecturas almacenadas en caché a posteriori. Comprender qué capa es responsable de cada función es lo que hace que la combinación funcione en la práctica.

---

## ¿Qué hay de nuevo en Next.js 16?

Los lanzamientos importantes de un framework son puntos de inflexión donde las decisiones arquitectónicas emergen de forma práctica. Next.js 16 representa un cambio significativo hacia una semántica de caché más clara, flujos de datos más predecibles y una mejor coordinación entre el renderizado en servidor y la interactividad en cliente. En lugar de añadir funciones superficiales, el lanzamiento se centra en afinar el modelo de ejecución para que los desarrolladores puedan razonar con mayor precisión sobre cuándo ocurre el trabajo, qué se almacena en caché y cómo se propagan las actualizaciones. El resultado no es novedad vacía, sino mayor control, menos sorpresas y sistemas que escalan con menos complejidad accidental.

### El nuevo valor predeterminado: Turbopack toma el escenario

Durante años, webpack ha sido el empaquetador predeterminado en el ecosistema de Next.js, pero los tiempos de compilación tienden a aumentar a medida que crece el proyecto. Next.js 16 aborda esto convirtiendo a **Turbopack** en el empaquetador predeterminado para todos los proyectos nuevos. El cambio no es estético; afecta tanto a la velocidad de iteración local como al tiempo de compilación en producción.

Según los puntos de referencia publicados por Vercel, las compilaciones de producción son de 2 a 5 veces más rápidas, y Fast Refresh (el mecanismo que actualiza la interfaz de usuario al editar archivos de código) es de 5 a 10 veces más rápido. En el día a día, esto significa ciclos de retroalimentación más cortos durante el desarrollo y menos tiempo de espera para que finalicen las compilaciones de CI.

Las cifras de adopción respaldan la transición. Cuando Next.js 16 entró en fase beta, más del 50% de las sesiones de desarrollo y alrededor del 20% de las compilaciones de producción en Next.js 15.3+ ya se ejecutaban en Turbopack. Para proyectos nuevos, no se requiere configuración: ejecutar `next dev` o `next build` utiliza Turbopack directamente.

Para proyectos con configuraciones personalizadas de webpack que no se pueden migrar de inmediato, puedes volver a utilizar webpack con flags explícitos:

```bash
next dev --webpack
next build --webpack
```

Turbopack también admite el almacenamiento en caché en el sistema de archivos (*filesystem caching*), lo que reduce aún más los tiempos de compilación. Cuando el servidor de desarrollo se reinicia, Turbopack carga los artefactos compilados previamente desde el disco en lugar de recompilar desde cero. Esto es especialmente útil en monorrepositorios grandes, donde los tiempos de arranque en frío pueden dominar el ciclo de desarrollo. A partir de Next.js 16.1, el almacenamiento en caché del sistema de archivos para `next dev` es estable y está habilitado de forma predeterminada; no se necesita configuración. El almacenamiento en caché del sistema de archivos para `next build` sigue siendo opcional y se puede habilitar mediante la opción `experimental.turbopackFileSystemCacheForBuild` en `next.config.ts`.

Para habilitar el almacenamiento en caché en el sistema de archivos, puedes agregar una línea a tu configuración:

```ts
const nextConfig = {
  turbopackFileSystemCacheForDev: true
}

export default nextConfig
```

Los equipos internos de Vercel han estado usando esta función en sus repositorios más grandes, y las ganancias de productividad han sido lo suficientemente sustanciales como para ponerla a disposición de toda la comunidad.

### Enrutamiento y navegación más inteligentes

Algunos de los cambios más impactantes en Next.js 16 se encuentran en la capa de enrutamiento. La navegación y la precarga (*prefetching*) se han rediseñado para reducir el trabajo redundante sin necesidad de modificar el código de la aplicación.

Considera un panel de control donde la mayoría de las páginas comparten la misma barra lateral y diseño de encabezado. En versiones anteriores de Next.js, el navegador almacenaba en caché el bundle del layout tras la primera descarga, por lo que no se volvía a descargar desde cero en cada navegación. Sin embargo, precargar cada enlace aún provocaba trabajo redundante. El payload de RSC del layout compartido se resolvía y se solicitaba por cada enlace. Next.js 16 deduplica los layouts compartidos en el momento de la precarga, por lo que el layout se obtiene una sola vez y se reutiliza en todos los enlaces. En una página con muchos enlaces que apuntan a rutas bajo el mismo layout, esto convierte una obtención de layout por enlace en una única obtención compartida, y la transferencia total por precarga se reduce en consecuencia.

La precarga también se ha vuelto más selectiva respecto a qué obtiene y cuándo. Ahora utiliza precarga incremental (*incremental prefetching*), que solo descarga las partes de una página que aún no están en la caché. Cuando un enlace sale de la ventana gráfica (*viewport*), Next.js cancela su solicitud de precarga pendiente, para que no se gaste ancho de banda en rutas que ya no es probable que el usuario visite. Cuando el enlace vuelve a ser visible, la precarga se reanuda, y pasar el cursor sobre un enlace activa una precarga más completa.

Hay un equilibrio que vale la pena entender. Dependiendo de cómo los usuarios se desplacen y pasen el cursor, es posible que veas una distribución diferente de solicitudes de precarga en la pestaña de red en comparación con versiones anteriores, pero la cantidad total de datos transferidos es generalmente menor debido a la deduplicación de layouts y la obtención incremental. Este es el comportamiento predeterminado en Next.js 16, y se puede observar directamente en el panel de red de la mayoría de las aplicaciones después de la actualización.

### La evolución del almacenamiento en caché

El almacenamiento en caché sigue siendo uno de los aspectos más potentes y, en ocasiones, más confusos de los frameworks web modernos. Next.js 16 aborda esto introduciendo un vocabulario más explícito para las operaciones de caché, brindándote un control preciso sobre diferentes escenarios.

Los cambios se centran en tres funciones: `revalidateTag()`, que ahora requiere una configuración más explícita; `updateTag()`, una nueva incorporación para actualizaciones inmediatas de caché; y `refresh()`, para actualizar datos no almacenados en caché.

#### Comprensión de `revalidateTag()`

La función `revalidateTag()` ahora requiere un perfil de vida de caché (*cache life profile*) como segundo argumento. Esto no es burocracia, es claridad. El perfil determina qué tan obsoletos pueden estar los datos antes de la revalidación, habilitando el patrón *stale-while-revalidate* que hace que las aplicaciones se sientan rápidas incluso mientras se actualizan:

```ts
// app/actions/blog.ts
'use server';
import { revalidateTag } from 'next/cache';

export async function publishBlogPost(postData: {
  title: string;
  content: string;
  slug: string;
}) {
  // Save the blog post to your database
  await db.posts.create(postData);

  // Revalidate the blog posts list
  // Using 'max' profile for long-lived content with background revalidation
  revalidateTag('blog-posts', 'max');

  return { success: true };
}
export async function updateNewsArticle(articleId: string, data: any) {
  await db.articles.update(articleId, data);

  // News updates more frequently - use 'hours' profile
  revalidateTag('news-feed', 'hours');

  return { success: true };
}
```

Los nombres de los perfiles describen claramente su propósito. El perfil `max` funciona para contenido que cambia con poca frecuencia: publicaciones de blog, documentación, páginas de marketing. El perfil `hours` se adapta a contenido que se actualiza regularmente pero no constantemente: canales de noticias, listados de productos, contenido social. También puedes definir perfiles personalizados en tu configuración o pasar un objeto en línea con un tiempo de revalidación específico.

Cuando los usuarios solicitan contenido etiquetado dentro de la ventana de obsolescencia del perfil, reciben la respuesta en caché de inmediato mientras Next.js revalida la entrada en segundo plano. La siguiente solicitud servida después de que finalice la revalidación devuelve los datos actualizados. Una vez que la entrada pasa su límite de expiración, se trata como completamente expirada en lugar de obsoleta, y la siguiente solicitud esperará datos nuevos antes de responder. Esta es una forma acotada de consistencia eventual, útil cuando la baja latencia importa más que leer el valor absoluto más reciente.

#### La inmediatez de `updateTag()`

La consistencia eventual no siempre es aceptable. Cuando un usuario actualiza su perfil, cambia su configuración o envía un formulario, se espera que la interfaz de usuario refleje el nuevo estado en el siguiente renderizado, no después de un ciclo de revalidación en segundo plano. `updateTag()` aborda este caso. Es una API exclusiva para Server Actions que proporciona semántica de lectura de tus propias escrituras (*read-your-writes*) al expirar la entrada de caché etiquetada y leer datos nuevos dentro del mismo ciclo de vida de la solicitud:

```ts
// app/actions/user.ts
'use server';

import { updateTag } from 'next/cache';

interface UserProfile {
  displayName: string;
  bio: string;
  avatarUrl: string;
  theme: 'light' | 'dark';
}

export async function updateUserProfile(
  userId: string,
  profile: UserProfile
) {
  // Update the user's profile in the database
  await db.users.update(userId, profile);

  // Expire cache and refresh immediately
  // User sees their changes right away
  updateTag(`user-${userId}`);

  return { success: true, profile };
}

export async function toggleNotificationSetting(
  userId: string,
  settingKey: string,
  enabled: boolean
) {
  await db.users.updateSetting(userId, settingKey, enabled);

  // Immediate cache update for settings
  updateTag(`user-settings-${userId}`);

  return { success: true };
}
```

La distinción entre `revalidateTag()` y `updateTag()` refleja dos patrones de actualización diferentes. Usa `revalidateTag()` para cambios de contenido que pueden tolerar un breve retraso, como publicar artículos, actualizar precios de productos o moderar comentarios. En estos casos, la siguiente solicitud sirve datos en caché mientras la revalidación ocurre en segundo plano. Usa `updateTag()` para cambios iniciados por el usuario donde la retroalimentación inmediata es esencial, como actualizaciones de perfil, cambios de preferencias o modificaciones en el carrito de compras. Aquí, la etiqueta se expira y se leen datos frescos dentro de la misma solicitud, por lo que el usuario ve el efecto de su acción en el siguiente renderizado en lugar de en una visita posterior.

#### Actualización sin caché (`refresh()`)

La tercera pieza de este rompecabezas de almacenamiento en caché es `refresh()`. A diferencia de `revalidateTag()` y `updateTag()`, no invalida ninguna entrada de caché. En su lugar, desencadena una actualización del enrutador del lado del cliente desde dentro de una Server Action, lo que hace que la ruta actual se vuelva a renderizar y que los datos no almacenados en caché en esa ruta se vuelvan a solicitar. Esto es útil cuando una mutación modifica datos que se muestran en la página pero que nunca se almacenaron en caché en primer lugar, como contadores de paneles en vivo, insignias de notificación o valores por solicitud derivados de `cookies()` o `headers()`. Ten en cuenta que llamar a `refresh()` también borra toda la caché de precarga del lado del cliente, por lo que los enlaces actualmente en el viewport se volverán a precargar; esto vale la pena recordarlo si planeas llamarlo con frecuencia:

```ts
// app/actions/notifications.ts
'use server';

import { refresh } from 'next/cache';

export async function markNotificationAsRead(notificationId: string) {
  // Mark the notification as read
  await db.notifications.update(notificationId, { read: true });

  // Refresh the unread count shown in the navigation
  // This data isn't cached, but it's displayed on the current page
  refresh();

  return { success: true };
}

export async function updateDashboardMetrics(userId: string) {
  await db.analytics.processUserActivity(userId);

  // Refresh live dashboard metrics without touching cached content
  refresh();

  return { success: true };
}
```

Además de estos comportamientos, un detalle útil sobre `refresh()` es que no reinicia ningún estado de React en el cliente. Los inputs de formularios, los menús abiertos, la posición del scroll y otros elementos transitorios de la UI permanecen exactamente como los dejó el usuario. Solo las partes renderizadas en el servidor de la ruta actual se vuelven a renderizar con datos nuevos.

### Creación de adaptadores para despliegues personalizados

Para los equipos que despliegan Next.js en infraestructura personalizada o plataformas distintas a Vercel, la API de Adaptadores de Compilación (**Build Adapters API**) proporciona un punto de integración de primer nivel. Introducida como versión alfa en Next.js 16.0 y estabilizada en 16.2, ofrece a las plataformas de despliegue una descripción tipada y versionada de la salida de compilación y una interfaz definida para conectarse al proceso de construcción.

La API surgió del RFC de Adaptadores de Compilación y de un grupo de trabajo que incluía a ingenieros de Vercel, Netlify, Cloudflare, AWS Amplify, Google Cloud y OpenNext. El objetivo era sustituir la situación anterior, en la que las plataformas que no eran de Vercel tenían que integrarse con rutas de código internas no documentadas, por un contrato compartido al que todas las plataformas, incluida Vercel, se dirigen de la misma manera.

Para usar un adaptador, apunta `adapterPath` a tu módulo de adaptador en `next.config.js`. A partir de Next.js 16.2, `adapterPath` es una opción estable de nivel superior en lugar de un flag experimental:

```ts
// next.config.ts
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    adapterPath: require.resolve('./my-adapter.js'),
  },
};

module.exports = nextConfig;
```

Para las plataformas de despliegue que desean un uso sin configuración, se puede configurar la variable de entorno `NEXT_ADAPTER_PATH`, de modo que la propia aplicación no necesite configurar el adaptador explícitamente.

Un adaptador es un módulo que exporta un objeto que implementa la interfaz `NextAdapter`. Los dos hooks principales son `modifyConfig`, que se ejecuta antes de la compilación y puede ajustar la configuración de Next.js, y `onBuildComplete`, que se ejecuta después de la compilación y recibe la salida de construcción para que la plataforma la procese:

```js
// my-adapter.js
module.exports = function createAdapter() {
  return {
    name: 'my-custom-adapter',

    async modifyConfig(config) {
      // Adjust Next.js configuration before the build runs.
      return {
        ...config,
        // Your platform-specific modifications
      }
    },

    async onBuildComplete(buildOutput) {
      // Consume the build output: routes, prerenders, static assets,
      // runtime targets, dependencies, caching rules, routing decisions.
      // Transform into the shape your platform expects.
    },
  }
}
```

El hook `onBuildComplete` recibe una descripción tipada de la aplicación, incluidas rutas, prerenderizados, recursos estáticos, objetivos de ejecución, dependencias, reglas de almacenamiento en caché y decisiones de enrutamiento. Esta descripción es el contrato estable al que apuntan los adaptadores, y se espera que cada nueva función de Next.js se documente contra ella. Una suite de pruebas compartida valida el comportamiento del adaptador para que la corrección se mantenga bajo el mismo estándar en todas las plataformas. Para una implementación de referencia completa, el paquete oficial `@vercel/adapter-next` es de código abierto y muestra cómo una plataforma mapea esta salida en su propia infraestructura.

### Todo asíncrono (*Async everything*)

Uno de los cambios disruptivos más generalizados en Next.js 16 es el paso al acceso asíncrono para los parámetros de ruta y los metadatos de las solicitudes. Las props `params` y `searchParams` ahora se pasan como promesas, y APIs como `cookies()`, `headers()` y `draftMode()` también devuelven promesas. Todas ellas deben ser esperadas con `await` antes de poder leer sus valores subyacentes. La razón de fondo es que estos valores dependen de la solicitud entrante y no del estado en tiempo de compilación o a nivel de módulo. Hacer que el acceso sea explícitamente asíncrono permite a Next.js posponer su resolución hasta el punto del renderizado donde realmente se necesitan, lo que permite que el streaming y el prerenderizado parcial envíen primero una estructura estática al cliente y transmitan las partes específicas de la solicitud a medida que estén disponibles:

```tsx
// app/products/[id]/page.tsx
interface ProductPageProps {
  params: Promise<{ id: string }>;
  searchParams: Promise<{ variant?: string }>;
}

export default async function ProductPage({
  params,
  searchParams,
}: ProductPageProps) {
  // Await params and searchParams
  const { id } = await params;
  const { variant } = await searchParams;

  const product = await fetch(`https://api.example.com/products/${id}`)
    .then((res) => res.json());

  return (
    <div className="max-w-4xl mx-auto p-6">
      <div className="grid grid-cols-1 md:grid-cols-2 gap-8">
        <div className="aspect-square bg-gray-100 rounded-lg overflow-hidden">
          <img
            src={product.images[0]}
            alt={product.name}
            className="w-full h-full object-cover"
          />
        </div>

        <div>
          <h1 className="text-3xl font-bold text-gray-900 mb-4">
            {product.name}
          </h1>
          <p className="text-2xl font-bold text-blue-600 mb-6">
            ${product.price}
          </p>
          <p className="text-gray-600 leading-relaxed mb-8">
            {product.description}
          </p>
          {variant && (
            <p className="text-sm text-gray-500">
              Selected variant: {variant}
            </p>
          )}
        </div>
      </div>
    </div>
  );
}
```

Las Server Actions siguen el mismo patrón:

```ts
// app/actions/cart.ts
'use server';

import { cookies } from 'next/headers';

export async function addToCart(productId: string, quantity: number) {
  // Await cookies access
  const cookieStore = await cookies();
  const userId = cookieStore.get('user_id')?.value;

  if (!userId) {
    return { error: 'Not authenticated' };
  }

  await db.cart.add({
    userId,
    productId,
    quantity,
  });

  return { success: true };
}

export async function getCartCount() {
  const cookieStore = await cookies();
  const userId = cookieStore.get('user_id')?.value;

  if (!userId) return 0;

  const count = await db.cart.getItemCount(userId);
  return count;
}
```

Los tipos de TypeScript reflejan la firma asíncrona directamente, por lo que intentar leer un campo de forma síncrona produce un error en tiempo de compilación que señala el `await` faltante. Para las bases de código existentes, Next.js proporciona una ruta de migración automatizada mediante `npx @next/codemod@canary upgrade latest`, que reescribe la mayoría de los sitios de llamada sin intervención manual. El beneficio resultante está en el framework, ya que aplazar la resolución de valores vinculados a la solicitud permite a Next.js comenzar a renderizar las partes estáticas de una ruta antes de que los datos de la solicitud estén disponibles.

### Mejoras en la optimización de imágenes

Next.js 16 incluye un conjunto de cambios en `next/image` que refuerzan la configuración predeterminada y cierran algunas brechas de seguridad. El `minimumCacheTTL` predeterminado para imágenes optimizadas aumenta de 60 segundos a 4 horas, lo que reduce la frecuencia con la que las imágenes de origen sin encabezados `Cache-Control` explícitos se vuelven a solicitar y reoptimizar. El arreglo `imageSizes` predeterminado ya no incluye `16`, un ancho que rara vez se usa en la práctica porque las pantallas retina (`devicePixelRatio: 2`) ya solicitan una fuente de 32px para un espacio de 16px. Eliminarlo hace que el atributo `srcset` sea más corto y reduce los bytes enviados al navegador.

En el aspecto de la seguridad, Next.js 16 ahora bloquea la optimización de imágenes para URLs que resuelven direcciones IP locales de forma predeterminada. Esto cierra una clase de vulnerabilidades de tipo SSRF donde se podría usar una URL de imagen para sondear la red interna del host que ejecuta la aplicación. Si estás usando intencionalmente un proxy de imágenes desde una red privada confiable, puedes volver a habilitarlo con `images.dangerouslyAllowLocalIP`, pero la postura predeterminada es no permitirlo:

```ts
// next.config.ts
const nextConfig = {
  images: {
    // Only enable this for development or trusted private networks
    dangerouslyAllowLocalIP: true,
    // Updated default minimum cache time (4 hours)
    minimumCacheTTL: 14400,
    // Quality prop now coerced to closest value in this array
    qualities: [75],
    // Maximum redirects when following image URLs
    maximumRedirects: 3,
  },
};

export default nextConfig;
```

En conjunto, estos cambios cambian la postura predeterminada de `next/image` de permisiva a restrictiva. Vale la pena señalar algunos otros elementos: las fuentes de imágenes locales con cadenas de consulta (*query strings*) ahora requieren una entrada en `images.localPatterns`. `images.domains` está obsoleto en favor de `images.remotePatterns`, que restringe las fuentes por protocolo, nombre de host, puerto y nombre de ruta. `next/legacy/image` está obsoleto y debe migrarse a `next/image`. La propiedad `priority` se reemplaza por `preload`.

### Mirando hacia el futuro

Next.js 16 representa una etapa de consolidación. Las primitivas de solicitud asíncrona, el control explícito de la caché a través de `use cache` y Cache Components, y Turbopack como el empaquetador predeterminado ahora forman parte de la cadena de herramientas estable en lugar de estar ocultos tras configuraciones experimentales. El soporte para el compilador de React (**React Compiler**) también ha pasado de experimental a estable, y `middleware.ts` ha sido renombrado a `proxy.ts` para marcar el límite de red con mayor claridad. El énfasis está menos en introducir nuevas abstracciones y más en consolidar el modelo de ejecución, aclarar la semántica de invalidación y asegurar que el comportamiento de desarrollo y producción se alineen. Estas decisiones reflejan lecciones de la adopción a gran escala y priorizan la previsibilidad, la operatividad y la mantenibilidad a largo plazo.

Los cambios disruptivos requieren atención, en particular la transformación asíncrona para `params`, `searchParams`, `cookies()`, `headers()` y `draftMode()`, así como los valores predeterminados de `next/image`. El comando `npx @next/codemod@canary upgrade latest` cubre la mayor parte del trabajo mecánico, y los beneficios resultantes, como un mejor rendimiento, una semántica de caché más clara y una mayor seguridad en las imágenes, hacen que la actualización valga la pena para la mayoría de las aplicaciones. Vale la pena destacar: Next.js 16 requiere Node.js 20.9+ y TypeScript 5.1+, y Next.js 16.2 añade herramientas dirigidas a agentes de codificación con IA, incluido un archivo `AGENTS.md` generado por `create-next-app` y un servidor DevTools MCP para asistencia contextual en actualizaciones.

Lo que destaca de esta versión no es una única característica, sino la visión coherente que representa. Cada cambio, desde los ajustes predeterminados más pequeños hasta el cambio arquitectónico más grande, apunta en la misma dirección: hacia aplicaciones más rápidas, una ergonomía más clara para el desarrollador y patrones listos para producción que escalan desde prototipos hasta plataformas globales. A medida que Next.js continúa evolucionando, los cimientos establecidos aquí —es decir, el rendimiento mediante Turbopack, el control explícito de caché y las APIs asíncronas por defecto— son sobre los que se está construyendo la próxima ola de características del framework.

---

## Resumen

Este capítulo recorrió los cambios que Next.js 16 introduce sobre los patrones establecidos por las Actions de React 19, siguiendo el orden en que afectan a una aplicación típica, desde el empaquetador hasta la capa de enrutamiento.

Comenzamos con Turbopack, ahora el empaquetador predeterminado tanto para desarrollo como para producción. Las compilaciones se ejecutan de 2 a 5 veces más rápido que con webpack y Fast Refresh de 5 a 10 veces más rápido, y el almacenamiento en caché del sistema de archivos para desarrollo es estable y está activo de forma predeterminada a partir de la versión 16.1. Webpack permanece disponible como una opción para proyectos que no pueden migrar su configuración de inmediato.

De ahí pasamos al enrutamiento y la navegación. Los layouts compartidos se deduplican durante la precarga, y la precarga incremental solo descarga los segmentos que aún no están en la caché. Las precargas se cancelan cuando un enlace sale de la ventana gráfica y se reanudan cuando regresa, lo que cambia la distribución de las solicitudes sin aumentar el tamaño total transferido.

El capítulo luego cubrió el modelo de almacenamiento en caché revisado. El almacenamiento en caché ahora es explícito a través de Cache Components en lugar de implícito, y tres APIs de Server Actions cubren los casos comunes posteriores a una mutación: una actualiza las entradas de caché etiquetadas en segundo plano, otra las expira dentro de la misma solicitud para permitir lecturas inmediatas tras escrituras, y otra vuelve a renderizar la ruta actual cuando los datos subyacentes nunca se almacenaron en caché en primer lugar.

Luego analizamos la API de Adaptadores de Compilación, estabilizada en 16.2, que ofrece a las plataformas de despliegue una descripción tipada y versionada de la salida de compilación y una interfaz definida para conectarse al proceso. Esto reemplaza la situación anterior en la que las plataformas que no eran de Vercel tenían que depender de rutas de código internas no documentadas.

Posteriormente cubrimos los cambios disruptivos. Los parámetros de ruta y los metadatos de las solicitudes ahora son completamente asíncronos, habiéndose eliminado la compatibilidad síncrona. Los valores predeterminados de optimización de imágenes también se vuelven más estrictos, incluyendo una mayor duración de caché, una lista permitida de calidades más reducida, redirecciones limitadas, protección SSRF para direcciones IP locales y patrones requeridos para fuentes locales con cadenas de consulta. Un único codemod gestiona la mayor parte de la migración mecánica.

Cerramos con la dirección general del lanzamiento. El soporte de React Compiler es estable aunque opcional, `middleware.ts` ha sido renombrado a `proxy.ts`, la versión mínima de Node.js es ahora 20.9 y la versión 16.2 agrega herramientas para agentes de codificación de IA. Estas incorporaciones, combinadas con el modelo de solicitudes asíncrono y el almacenamiento en caché explícito, hacen que el comportamiento en tiempo de ejecución sea más predecible y la ruta de migración más clara que en versiones principales anteriores.

En el próximo capítulo, veremos el manejo avanzado de errores y la depuración en React 19.
