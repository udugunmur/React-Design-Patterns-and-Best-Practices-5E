# Capítulo 9: Manejo Avanzado de Formularios en React

Los formularios son la puerta de entrada entre tus usuarios y los datos de tu aplicación. Están en todas partes, desde simples suscripciones a boletines hasta complejos asistentes paso a paso (*wizards*). Sin embargo, a pesar de su ubicuidad, los formularios siguen siendo uno de los aspectos más desafiantes del desarrollo en React. En este capítulo, exploraremos cómo las características más recientes de React 19, combinadas con librerías modernas como Zod, pueden transformar la forma en que construyes formularios.

En este capítulo, cubriremos los siguientes temas:

- Los desafíos del manejo de formularios en React
- Gestión del estado de formularios con `useActionState` y `useFormStatus`
- Estrategias de validación con Zod
- Aprovechamiento de `new FormData()` para el manejo de formularios
- Mejores prácticas para arquitecturas de formularios escalables

---

## Los desafíos del manejo de formularios en React

Todo desarrollador de React se ha enfrentado al momento de temor cuando se le encarga construir un formulario complejo. Los requisitos parecen bastante simples al principio: recopilar la entrada del usuario, validarla y enviarla a un servidor. Pero a medida que profundizas, la complejidad se revela como las capas de una cebolla.

Los formularios están en todas partes en el desarrollo web: autenticación, procesos de pago (*checkout*), gestión de contenidos, entrada de datos. Sin embargo, siguen siendo una de las partes más complejas de la creación de aplicaciones React. Cuando los formularios fallan, los usuarios lo perciben de inmediato. Una validación tosca, una retroalimentación lenta, errores confusos o datos de entrada perdidos alejan a las personas de tu aplicación. Del lado del desarrollador, los formularios mal diseñados se convierten en componentes desmesurados llenos de condicionales anidados y una gestión de estado enredada que nadie quiere mantener.

Este capítulo cambiará eso. A través de años de construcción de aplicaciones en producción, he aprendido que invertir tiempo en una arquitectura de formularios adecuada rinde grandes frutos. Cubriremos cómo eliminar el código repetitivo (*boilerplate*), crear formularios seguros en tipos con validación de esquemas y crear patrones reutilizables que escalen. Aprenderás a manejar escenarios complejos como formularios de varios pasos y matrices de campos dinámicos sin perder la cabeza. También nos sumergiremos en la optimización del rendimiento y las estrategias de prueba.

Más importante aún, nos centraremos en el *porqué* detrás de cada enfoque. Mi objetivo no es darte recetas para seguir a ciegas, sino ayudarte a comprender las ventajas y desventajas para que puedas tomar decisiones informadas para tus necesidades específicas.

Considera un formulario de registro de usuario típico. Necesitas rastrear el estado de los campos, validar formatos de correo electrónico, verificar la fortaleza de la contraseña, confirmar que las contraseñas coincidan, manejar estados de carga, mostrar errores con elegancia y tal vez verificar si un nombre de usuario ya está en uso, todo mientras mantienes la interfaz de usuario responsiva. El enfoque tradicional conduce a componentes inflados con lógica de validación dispersa y manejadores de eventos que se vuelven más difíciles de mantener con cada requisito. Así es como podría verse una implementación ingenua:

```tsx
// The problem: A form component that quickly becomes unwieldy
import React, { useState } from 'react';

interface FormData {
  email: string;
  password: string;
  confirmPassword: string;
}

export function TraditionalForm() {
  const [formData, setFormData] = useState<FormData>({
    email: '',
    password: '',
    confirmPassword: ''
  });
  const [errors, setErrors] = useState<Partial<FormData>>({});
  const [isSubmitting, setIsSubmitting] = useState(false);

  const validateEmail = (email: string) => {
    const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return regex.test(email);
  };

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const { name, value } = e.target;
    setFormData(prev => ({ ...prev, [name]: value }));
   
    // Inline validation logic that grows complex quickly
    if (name === 'email' && !validateEmail(value)) {
      setErrors(prev => ({ ...prev, email: 'Invalid email' }));
    } else if (name === 'password' && value.length < 8) {
      setErrors(prev => ({ ...prev, password: 'Too short' }));
    } else {
      setErrors(prev => ({ ...prev, [name]: undefined }));
    }
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setIsSubmitting(true);
    // Validation, API calls, error handling...
    // This function keeps growing!
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4 p-6">
      {/* Form fields with repetitive patterns */}
    </form>
  );
}
```

Este enfoque funciona para formularios simples, pero no escala. Cada nuevo campo requiere actualizar múltiples partes del componente, la lógica de validación se vuelve cada vez más compleja y las responsabilidades del componente se multiplican sin control.

### Problemas de rendimiento en formularios a gran escala

Los problemas de rendimiento en los formularios a menudo se infiltran gradualmente. Un formulario con cinco campos puede sentirse rápido, pero ¿qué sucede cuando tienes cincuenta? ¿O cuando cada campo requiere una validación compleja? ¿O cuando necesitas agregar y eliminar campos dinámicamente según las elecciones del usuario?

El principal culpable son los re-renderizados innecesarios que provocan retrasos al escribir (*typing lag*). En React, cada cambio de estado desencadena un re-renderizado del componente y de sus hijos. En el contexto de un formulario, esto significa que escribir en un campo hace que todo el formulario se vuelva a renderizar, incluidos los campos que no han cambiado. Para un formulario con muchos campos o una validación costosa, esto genera un retraso notable en la interfaz de usuario: los usuarios escriben, pero su entrada aparece demorada o entrecortada.

El problema se agrava a medida que crece tu formulario. Con 10 bloques de direcciones, una sola pulsación de tecla re-renderiza 10 componentes. Con 50, son 50 re-renderizados por pulsación de tecla. Si cada bloque de direcciones tiene lógica de validación, formateo o componentes secundarios complejos, el impacto en el rendimiento se multiplica.

Considera un formulario dinámico donde los usuarios pueden agregar múltiples direcciones:

```tsx
import React, { useState } from 'react';
interface Address {
  street: string;
  city: string;
  zipCode: string;
}
export function AddressForm() {
  const [addresses, setAddresses] = useState<Address[]>([
    { street: '', city: '', zipCode: '' }
  ]);
  const updateAddress = (index: number, field: keyof Address, value: string) => {
    setAddresses(prev => {
      const updated = [...prev];
      updated[index] = { ...updated[index], [field]: value };
      return updated; // This triggers re-render of ALL addresses
    });
  };
  const addAddress = () => {
    setAddresses(prev => [...prev, { street: '', city: '', zipCode: '' }]);
  };
  return (
    <div className="space-y-6">
      {addresses.map((address, index) => (
        <div key={index} className="border p-4 rounded-lg">
          <input
            value={address.street}
            onChange={(e) => updateAddress(index, 'street', e.target.value)}
            placeholder="Street"
            className="w-full p-2 border rounded mb-2"
          />
          <input
            value={address.city}
            onChange={(e) => updateAddress(index, 'city', e.target.value)}
            placeholder="City"
            className="w-full p-2 border rounded mb-2"
          />
          <input
            value={address.zipCode}
            onChange={(e) => updateAddress(index, 'zipCode', e.target.value)}
            placeholder="ZIP Code"
            className="w-full p-2 border rounded"
          />
        </div>
      ))}
      <button
        onClick={addAddress}
        className="px-4 py-2 bg-blue-500 text-white rounded"
      >
        Add Address
      </button>
    </div>
  );
}
```

Veamos qué sale mal aquí:

- **Re-renderizados en cascada**: Cuando escribes en el campo de calle de la dirección #3, React actualiza todo el arreglo `addresses`. Esto obliga a los 10 (o 50, o 100) bloques de direcciones a volver a renderizarse, aunque las direcciones #1, #2, #4-10 no hayan cambiado.
- **Sobrecarga de validación**: Si cada campo tiene lógica de validación (formato de correo electrónico, formato de código postal, etc.), esa validación se ejecuta en cada campo durante cada re-renderizado, no solo en el campo que cambió.
- **Costo de comparación del DOM Virtual (Diffing)**: React debe comparar todo el árbol del formulario en cada pulsación de tecla para determinar qué cambió realmente en el DOM. Con formularios grandes, este proceso de diferenciación en sí mismo se vuelve costoso.
- **Falta de aislamiento de componentes**: Debido a que todo el estado vive en el nivel superior, no hay forma de aislar los cambios a campos individuales o bloques de direcciones.

¿El resultado? Los usuarios experimentan un retraso en la escritura en el que los caracteres aparecen 100–200 ms después de presionar las teclas. En casos extremos con validación compleja o cientos de campos, la interfaz de usuario puede congelarse por completo durante un segundo o más por pulsación de tecla.

Esto no es una fuga de memoria, ya que el uso de memoria se mantiene relativamente constante. El problema es el desgaste de la CPU por re-renderizados excesivos, lo que hace que tu formulario se sienta lento y roto.

Estos problemas de rendimiento se derivan de un desafío fundamental: la gestión tradicional del estado de React no fue diseñada teniendo en cuenta formularios complejos. Cuando todo el estado de tu formulario reside en un solo componente, pierdes la capacidad de aislar los cambios y controlar los re-renderizados de manera efectiva. Si bien puedes mitigar estos problemas con `React.memo`, `useMemo` y una división cuidadosa del estado, esencialmente estás luchando contra los patrones naturales del framework. Afortunadamente, React 19 introduce un enfoque completamente diferente. En lugar de gestionar todo el estado del formulario en el cliente y luchar contra el rendimiento de los re-renderizados, los nuevos hooks `useActionState` y `useFormStatus` cambian el paradigma hacia un manejo de formularios impulsado por el servidor.

---

## Gestión del estado de formularios con useActionState y useFormStatus

React 19 introduce primitivas potentes que cambian fundamentalmente la forma en que pensamos sobre la gestión del estado de los formularios. Los hooks `useActionState` y `useFormStatus` proporcionan un enfoque centrado en el servidor (*server-first*) para formularios que se alinea perfectamente con las aplicaciones modernas full-stack de React.

### Comprensión de useActionState para transiciones de estado

El hook `useActionState` proporciona una forma estructurada de gestionar el estado producido por una acción, incluidas funciones asíncronas del lado del cliente y Server Actions. En lugar de coordinar manualmente el estado de envío, los indicadores de pendiente, los errores y las actualizaciones de resultados, el hook invoca una acción y utiliza su valor devuelto como el siguiente estado. En los flujos de trabajo de formularios, se puede conectar directamente a una acción de formulario, pero no se limita a la ejecución del lado del servidor. Esto lo hace útil para modelar transiciones de estado de manera consistente tanto en interacciones del cliente como del servidor.

Piensa en `useActionState` como una máquina de estados para tus formularios. Realiza un seguimiento no solo del estado actual, sino también de la transición entre estados durante el envío del formulario. Esto es particularmente potente cuando se combina con RSC y Server Actions.

Aquí hay un formulario de perfil que demuestra este patrón. Observa cómo la validación y la persistencia de datos ocurren completamente en el servidor, mientras que el cliente recibe mensajes de éxito o errores de validación:

```tsx
// Modern approach with useActionState
import { useActionState } from 'react';
interface FormState {
  message?: string;
  errors?: Record<string, string>;
}
async function updateProfile(
  prevState: FormState,
  formData: FormData
): Promise<FormState> {
  const name = formData.get('name') as string;
  const bio = formData.get('bio') as string;
  // Validation happens on the server
  if (!name || name.length < 2) {
    return {
      errors: { name: 'Name must be at least 2 characters' }
    };
  }
  try {
    // Server-side database update
    await updateUserProfile({ name, bio });
    return { message: 'Profile updated successfully!' };
  } catch (error) {
    return {
      errors: { general: 'Failed to update profile' }
    };
  }
}
export function ProfileForm() {
  const [state, formAction] = useActionState(
    updateProfile,
    { message: undefined, errors: undefined }
  );
  return (
    <form action={formAction} className="max-w-md mx-auto p-6">
      <div className="mb-4">
        <input
          type="text"
          name="name"
          placeholder="Your name"
          className="w-full p-3 border rounded-lg focus:ring-2 focus:ring-blue-500"
        />
        {state.errors?.name && (
          <p className="text-red-500 text-sm mt-1">
            {state.errors.name}
          </p>
        )}
      </div>
      <div className="mb-4">
        <textarea
          name="bio"
          placeholder="Tell us about yourself"
          className="w-full p-3 border rounded-lg h-32 focus:ring-2 focus:ring-blue-500"
        />
      </div>
      {state.message && (
        <div className="mb-4 p-3 bg-green-100 text-green-700 rounded-lg">
          {state.message}
        </div>
      )}
      <button
        type="submit"
        className="w-full py-3 bg-blue-500 text-white rounded-lg hover:bg-blue-600 transition">
        Update Profile
      </button>
    </form>
  );
}
```

La belleza de este enfoque radica en su simplicidad. El componente del formulario no necesita gestionar el estado, manejar la validación ni lidiar con llamadas a la API. Simplemente renderiza el estado actual y deja que la Server Action maneje el resto. Este patrón escala maravillosamente: agregar nuevos campos no requiere lógica adicional de gestión de estado.

Lo que hace que esto sea simple es lo siguiente:

- **Sin estado del lado del cliente**: `FormData` captura los valores de entrada automáticamente. Sin `useState` para cada campo, sin manejadores `onChange` en todas partes.
- **Única fuente de validación**: Todas las reglas residen en la Server Action. Agrega una regla de validación una vez, no dos.
- **Estados de carga/error integrados**: El objeto de estado maneja los mensajes de éxito y error. Sin llamadas `useState` separadas para `isLoading`, `error` o `success`.
- **Cero código repetitivo de API**: Sin `fetch`, sin `try/catch` en el componente, sin analizar respuestas. Solo pasa una función a `useActionState`.
- **Escalado trivial**: ¿Quieres agregar un nuevo campo? Agrega el input y actualiza la Server Action. El mismo patrón para 3 campos o 30.

### Uso de useFormStatus para mejorar la retroalimentación del usuario

Mientras que `useActionState` gestiona el estado general del formulario, `useFormStatus` proporciona un control granular sobre el proceso de envío. Debe usarse dentro de un componente secundario del formulario y te brinda acceso al estado pendiente (*pending*), lo que te permite crear elementos de interfaz de usuario responsivos que reaccionan al envío del formulario.

El hook devuelve un objeto con el estado pendiente y los datos que se están enviando, lo que te permite deshabilitar botones, mostrar indicadores de carga o mostrar actualizaciones de UI optimistas. Así es como transforma la experiencia del usuario:

```tsx
// SubmitButton component using useFormStatus
import { useFormStatus } from 'react-dom';
function SubmitButton({ children }: { children: React.ReactNode }) {
  const { pending } = useFormStatus();
  return (
    <button
      type="submit"
      disabled={pending}
      className={`w-full py-3 rounded-lg font-medium transition ${pending ? 'bg-gray-400 cursor-not-allowed' : 'bg-blue-500 hover:bg-blue-600 text-white'} `}>
      {pending ? (
        <span className="flex items-center justify-center">
          <svg className="animate-spin h-5 w-5 mr-2" viewBox="0 0 24 24">
            <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4" fill="none" />
            <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8v8H4z" />
          </svg>
          Processing...
        </span>
      ) : (
        children
      )}
    </button>
  );
}
// Using the SubmitButton in a form
export function ContactForm() {
  async function sendMessage(formData: FormData) {
    'use server';
    // Simulate API delay
    await new Promise(resolve => setTimeout(resolve, 2000));
    const email = formData.get('email');
    const message = formData.get('message');
    // Process the message
    console.log('Sending message from:', email);
  }
  return (
    <form action={sendMessage} className="space-y-4 max-w-lg mx-auto">
      <input
        type="email"
        name="email"
        required
        placeholder="Your email"
        className="w-full p-3 border rounded-lg"
      />
      <textarea
        name="message"
        required
        placeholder="Your message"
        className="w-full p-3 border rounded-lg h-32"
      />
      <SubmitButton>Send Message</SubmitButton>
    </form>
  );
}
```

La clave es que `useFormStatus` debe llamarse desde un componente secundario del formulario, no desde el componente del formulario en sí. Por eso `SubmitButton` está separado: `useFormStatus` lee el estado pendiente del `<form>` padre más cercano.

Cuando el usuario hace clic en *Send Message*, React establece inmediatamente `pending` en `true` antes de que se ejecute la Server Action. El botón se deshabilita y muestra un spinner al instante, evitando envíos duplicados. Sin manejadores `onClick`, sin `useState` para la carga, sin deshabilitación manual de botones; el estado pendiente fluye automáticamente desde el ciclo de vida de envío del formulario.

Este patrón destaca cuando múltiples componentes necesitan reaccionar al envío. Podrías tener un botón que se deshabilita, un indicador de progreso que aparece y campos que se vuelven de solo lectura, todos usando `useFormStatus` de forma independiente, respondiendo al mismo envío sin *prop drilling* ni contexto.

### Implementación de UI optimista en formularios

La UI optimista (*Optimistic UI*) es un patrón donde actualizamos inmediatamente la interfaz para reflejar el resultado esperado de una acción, y luego reconciliamos con el resultado real cuando este llega. Esto crea una sensación más ágil y responsiva que a los usuarios les encanta. Los hooks de formulario de React 19 hacen que la implementación de actualizaciones optimistas sea notablemente sencilla.

Considera una lista de tareas (*to-do list*) donde los usuarios pueden agregar elementos. En lugar de esperar a que el servidor confirme la adición, podemos mostrar inmediatamente el nuevo elemento mientras la solicitud se procesa en segundo plano:

```tsx
// Optimistic updates with useOptimistic
import { useOptimistic, useActionState } from 'react';
interface Todo {
  id: string;
  text: string;
  completed: boolean;
  pending?: boolean;
}
async function addTodo(
  todos: Todo[],
  formData: FormData
): Promise<Todo[]> {
  const text = formData.get('todo') as string;
  // Simulate server delay
  await new Promise(resolve => setTimeout(resolve, 1000));
  const newTodo: Todo = {
    id: Date.now().toString(),
    text,
    completed: false
  };
  return [...todos, newTodo];
}
export function TodoList({ initialTodos }: { initialTodos: Todo[] }) {
  const [todos, formAction] = useActionState(addTodo, initialTodos);
  const [optimisticTodos, addOptimisticTodo] = useOptimistic(
    todos,
    (state: Todo[], newTodo: string) => [
      ...state,
      {
        id: 'temp-' + Date.now(),
        text: newTodo,
        completed: false,
        pending: true
      }
    ]
  );
  return (
    <div className="max-w-md mx-auto p-6">
      <form
        action={async (formData) => {
          const todo = formData.get('todo') as string;
          addOptimisticTodo(todo);
          await formAction(formData);
        }}
        className="flex gap-2 mb-4"
      >
        <input
          type="text"
          name="todo"
          required
          placeholder="Add a todo..."
          className="flex-1 p-2 border rounded-lg"
        />
        <button
          type="submit"
          className="px-4 py-2 bg-blue-500 text-white rounded-lg"
        >
          Add
        </button>
      </form>
      <ul className="space-y-2">
        {optimisticTodos.map(todo => (
          <li
            key={todo.id}
            className={`p-3 rounded-lg border ${todo.pending ? 'opacity-50 bg-gray-50' : 'bg-white'} `}>
            {todo.text}
            {todo.pending && (
              <span className="ml-2 text-sm text-gray-500">
                (adding...)
              </span>
            )}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

La magia ocurre en el manejador del formulario. Cuando el usuario hace clic en *Add*, llamamos a `addOptimisticTodo(todo)` antes de llamar a `formAction(formData)`. Esto agrega inmediatamente la tarea a `optimisticTodos` con un indicador `pending: true`, mostrándola en la interfaz de usuario al instante. Luego, `formAction` se ejecuta de forma asíncrona; mientras se procesa en el servidor, el usuario ya ve su nueva tarea con un indicador *(adding...)*.

Cuando el servidor responde, `useOptimistic` reconcilia el estado optimista temporal con el resultado real del servidor. Si tiene éxito, la tarea temporal se reemplaza por la real (completa con un ID generado adecuadamente por el servidor). Si la solicitud falla, React revierte automáticamente al estado anterior, eliminando la tarea optimista. Este patrón crea una experiencia fluida que se siente instantánea mientras mantiene la integridad de los datos.

---

## Estrategias de validación con Zod

La validación de formularios es donde la teoría se pone en práctica. No basta con recopilar datos; debemos asegurarnos de que sean correctos, completos y seguros. Zod aporta una validación componible y segura en tipos (*type-safe*) a los formularios de React, convirtiendo la validación en tiempo de ejecución en un proceso declarativo y mantenible.

### Introducción a la validación basada en esquemas con Zod

Zod es una librería de validación de esquemas orientada a TypeScript que te permite definir la forma y las restricciones de tus datos en un solo lugar. A diferencia de los enfoques de validación tradicionales que dispersan las reglas por todos tus componentes, Zod centraliza la lógica de validación en esquemas reutilizables.

El poder de Zod radica en su capacidad para inferir tipos de TypeScript a partir de tus esquemas. Define tu validación una vez y obtendrás tanto validación en tiempo de ejecución como seguridad de tipos en tiempo de compilación. Esto elimina la desconexión entre tus tipos de TypeScript y las reglas de validación que a menudo conduce a errores:

```typescript
import { z } from "zod";

const UserSchema = z.object({
  email: z.email({
    error: issue =>
      issue.input === "" || issue.input === undefined
        ? "Email is required"
        : "Please enter a valid email address",
  }),

  age: z
    .number()
    .min(18, "You must be at least 18 years old")
    .max(120, "Please enter a valid age"),

  username: z
    .string()
    .min(3, "Username must be at least 3 characters")
    .max(20, "Username must be at most 20 characters")
    .regex(
      /^[a-zA-Z0-9_]+$/,
      "Username can only contain letters, numbers, and underscores",
    ),

  website: z
    .union([
      z.url("Please enter a valid URL"),
      z.literal(""),
    ])
    .optional(),

  bio: z
    .string()
    .max(500, "Bio must be 500 characters or less")
    .optional(),
});

type User = z.infer<typeof UserSchema>;

export function validateUser(data: unknown): User | null {
  const result = UserSchema.safeParse(data);

  if (!result.success) {
    console.error("Validation errors:", result.error.issues);
    return null;
  }

  return result.data;
}
```

El esquema encadena métodos de validación: `z.string().email()` garantiza que el campo sea tanto una cadena como un formato de correo electrónico válido. Los mensajes de error personalizados reemplazan los fallos genéricos con comentarios fáciles de usar.

El método `.optional().or(z.literal(''))` en `website` maneja un escenario común: campos opcionales que pueden ser cadenas vacías en lugar de `undefined`.

La línea crucial es `type User = z.infer<typeof UserSchema>`. Esto genera tipos de TypeScript directamente a partir del esquema. Si cambias el esquema, el tipo se actualiza automáticamente. No hay desfase entre la validación en tiempo de ejecución y los tipos en tiempo de compilación.

Cuando se ejecuta `UserSchema.parse(data)`, devuelve datos validados tipados como `User` o lanza un `ZodError` con información estructurada sobre qué campos fallaron y por qué. Esto hace que mostrar errores específicos de cada campo sea sencillo.

### Validación segura en tipos: creación y aplicación de esquemas de Zod

La verdadera magia ocurre cuando integramos Zod con nuestros formularios de React. Al combinar la validación de Zod con el manejo de formularios de React, creamos un sistema robusto que detecta errores a tiempo y proporciona comentarios útiles a los usuarios.

Construyamos un formulario de registro que demuestre una validación integral con Zod. Comenzaremos definiendo nuestro esquema de validación:

```typescript
import { z } from 'zod';
const RegistrationSchema = z.object({
  firstName: z.string()
    .min(2, 'First name must be at least 2 characters')
    .regex(/^[a-zA-Z\s]+$/, 'Only letters allowed'),
  lastName: z.string()
    .min(2, 'Last name must be at least 2 characters')
    .regex(/^[a-zA-Z\s]+$/, 'Only letters allowed'),
  email: z.string()
    .email('Invalid email format'),
  password: z.string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/[A-Z]/, 'Must contain uppercase letter')
    .regex(/[a-z]/, 'Must contain lowercase letter')
    .regex(/[0-9]/, 'Must contain a number')
    .regex(/[^A-Za-z0-9]/, 'Must contain special character'),
  confirmPassword: z.string(),
  terms: z.boolean()
    .refine(val => val === true, 'You must accept the terms')
}).refine(data => data.password === data.confirmPassword, {
  message: "Passwords don't match",
  path: ['confirmPassword']
});
type RegistrationData = z.infer<typeof RegistrationSchema>;
type FieldErrors = Partial<Record<keyof RegistrationData, string>>;
```

Este esquema demuestra varios patrones de validación. Los campos individuales utilizan métodos encadenados para la validación básica, pero observa el `.refine()` a nivel de esquema; esto valida a través de múltiples campos, asegurando que `password` y `confirmPassword` coincidan. El error se adjunta al campo `confirmPassword` mediante la opción `path`, para que los usuarios vean el error donde tiene sentido.

Ahora construyamos el componente de formulario con manejo de errores:

```tsx
import { useState } from 'react';

export function RegistrationForm() {
  const [errors, setErrors] = useState<FieldErrors>({});

  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
   
    const data = {
      firstName: formData.get('firstName'),
      lastName: formData.get('lastName'),
      email: formData.get('email'),
      password: formData.get('password'),
      confirmPassword: formData.get('confirmPassword'),
      terms: formData.get('terms') === 'on'
    };

    try {
      const validated = RegistrationSchema.parse(data);
      console.log('Valid data:', validated);
      // Submit to server
      setErrors({});
    } catch (error) {
      if (error instanceof z.ZodError) {
        const fieldErrors: FieldErrors = {};
        error.errors.forEach(err => {
          if (err.path[0]) {
            fieldErrors[err.path[0] as keyof RegistrationData] = err.message;
          }
        });
        setErrors(fieldErrors);
      }
    }
  };
```

El manejador de envío extrae los datos del formulario e intenta la validación. Si `RegistrationSchema.parse()` tiene éxito, obtenemos datos validados y seguros en tipos. Si falla, transformamos el arreglo de errores de Zod en un objeto con claves por campo para una fácil visualización de errores. La ruta (*path*) de cada error nos indica qué campo falló.

Finalmente, rendericemos el formulario con retroalimentación de errores:

```tsx
  return (
    <form onSubmit={handleSubmit} className="max-w-md mx-auto space-y-4">
      <h2 className="text-2xl font-bold mb-6">Create Account</h2>

      <div>
        <label htmlFor="firstName" className="block mb-2 font-medium">
          First Name
        </label>
        <input
          type="text"
          id="firstName"
          name="firstName"
          className="w-full p-2 border rounded-lg"
        />
        {errors.firstName && (
          <p className="mt-1 text-sm text-red-600">{errors.firstName}</p>
        )}
      </div>

      <div>
        <label htmlFor="lastName" className="block mb-2 font-medium">
          Last Name
        </label>
        <input
          type="text"
          id="lastName"
          name="lastName"
          className="w-full p-2 border rounded-lg"
        />
        {errors.lastName && (
          <p className="mt-1 text-sm text-red-600">{errors.lastName}</p>
        )}
      </div>

      <div>
        <label htmlFor="email" className="block mb-2 font-medium">
          Email
        </label>
        <input
          type="email"
          id="email"
          name="email"
          className="w-full p-2 border rounded-lg"
        />
        {errors.email && (
          <p className="mt-1 text-sm text-red-600">{errors.email}</p>
        )}
      </div>

      <div>
        <label htmlFor="password" className="block mb-2 font-medium">
          Password
        </label>
        <input
          type="password"
          id="password"
          name="password"
          className="w-full p-2 border rounded-lg"
        />
        {errors.password && (
          <p className="mt-1 text-sm text-red-600">{errors.password}</p>
        )}
      </div>

      <div>
        <label htmlFor="confirmPassword" className="block mb-2 font-medium">
          Confirm Password
        </label>
        <input
          type="password"
          id="confirmPassword"
          name="confirmPassword"
          className="w-full p-2 border rounded-lg"
        />
        {errors.confirmPassword && (
          <p className="mt-1 text-sm text-red-600">{errors.confirmPassword}</p>
        )}
      </div>

      <div className="flex items-center gap-2">
        <input
          type="checkbox"
          id="terms"
          name="terms"
          className="w-4 h-4"
        />
        <label htmlFor="terms" className="text-sm">
          I accept the terms and conditions
        </label>
      </div>
      {errors.terms && (
        <p className="text-sm text-red-600">{errors.terms}</p>
      )}

      <button
        type="submit"
        className="w-full py-3 bg-blue-500 text-white rounded-lg hover:bg-blue-600 transition"
      >
        Register
      </button>
    </form>
  );
}
```

Cada campo sigue el mismo patrón: renderizar la entrada y luego mostrar condicionalmente su mensaje de error. Los mensajes de error provienen de nuestras definiciones de esquema, lo que los hace consistentes y fáciles de mantener. Si cambias un mensaje de validación en el esquema, se actualiza en todos los lugares donde se use ese campo.

### Manejo de validación asíncrona y mensajes de error

La validación en el mundo real a menudo requiere consultar fuentes externas. ¿Este nombre de usuario ya está en uso? ¿Es válido este código de cupón? Estas comprobaciones requieren una validación asíncrona, que Zod maneja con elegancia mediante refinamientos.

La validación asíncrona introduce complejidad: necesitamos gestionar estados de carga, manejar errores de red y proporcionar retroalimentación sin bloquear la experiencia del usuario. Aquí hay un patrón que aborda estos desafíos:

```tsx
// Async validation with Zod
import { useState, useCallback } from 'react';
import { z } from 'zod';
import { debounce } from 'lodash';
// Simulate API call to check username availability
async function checkUsernameAvailable(username: string): Promise<boolean> {
  await new Promise(resolve => setTimeout(resolve, 500));
  const taken = ['admin', 'root', 'user123'];
  return !taken.includes(username.toLowerCase());
}
const UsernameSchema = z.string()
  .min(3, 'Username must be at least 3 characters')
  .max(20, 'Username must be at most 20 characters')
  .regex(/^[a-zA-Z0-9_]+$/, 'Invalid characters');
export function UsernameField() {
  const [username, setUsername] = useState('');
  const [checking, setChecking] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [available, setAvailable] = useState<boolean | null>(null);
  const checkUsername = useCallback(
    debounce(async (value: string) => {
      setChecking(true);
      setError(null);
      try {
        // First, validate format
        UsernameSchema.parse(value);
        // Then check availability
        const isAvailable = await checkUsernameAvailable(value);
        setAvailable(isAvailable);
        if (!isAvailable) {
          setError('Username is already taken');
        }
      } catch (err) {
        if (err instanceof z.ZodError) {
          setError(err.errors[0].message);
        }
        setAvailable(null);
      } finally {
        setChecking(false);
      }
    }, 500),
    []
  );
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    setUsername(value);
    if (value.length >= 3) {
      checkUsername(value);
    } else {
      setError(null);
      setAvailable(null);
    }
  };
  return (
    <div className="relative">
      <input
        type="text"
        value={username}
        onChange={handleChange}
        placeholder="Choose a username"
        className={` w-full p-3 pr-10 border rounded-lg ${error ? 'border-red-500' : available ? 'border-green-500' : 'border-gray-300'} `}
      />
      {/* Status indicator */}
      <div className="absolute right-3 top-3.5">
        {checking && (…)}
        {!checking && available && (…)}
        {!checking && error && (…)}
      </div>
      {/* Error or success message */}
      {error && (
        <p className="text-red-500 text-sm mt-1">{error}</p>
      )}
      {available && !checking && (
        <p className="text-green-500 text-sm mt-1">
          Username is available!
        </p>
      )}
    </div>
  );
}
```

El componente utiliza una validación de dos fases para evitar llamadas innecesarias a la API. Primero, `UsernameSchema.parse()` valida el formato; solo si eso tiene éxito, realiza la solicitud al servidor para verificar la disponibilidad. La función envoltorio `debounce(async (...), 500)` espera 500 ms después de que el usuario deja de escribir antes de verificar, por lo que escribir *john* realiza una llamada a la API en lugar de cuatro. La función `handleChange` agrega otra protección, activando la validación solo cuando el nombre de usuario alcanza un mínimo de 3 caracteres.

Tres variables de estado: `checking`, `error` y `available`, proporcionan comentarios en tiempo real. Mientras se realiza la comprobación, los usuarios ven un spinner. Si tiene éxito, aparecen una marca de verificación y el mensaje *"Username is available!"*. Si falla, se muestra una X y el error específico (problema de formato o ya en uso). Los usuarios siempre saben qué está sucediendo sin que se bloquee su escritura. Este patrón se adapta a cualquier validación asíncrona: verificación de correo electrónico, códigos de descuento, enlaces de invitación o cualquier verificación que requiera un viaje de ida y vuelta al servidor.

---

## Aprovechamiento de new FormData() para el manejo de formularios

La API `FormData` ha formado parte de la plataforma web durante años, pero los desarrolladores de React a menudo la pasaban por alto en favor de componentes controlados. El énfasis de React 19 en patrones orientados al servidor vuelve a poner a `FormData` en el centro de atención, y por una buena razón: es potente, eficiente y funciona a la perfección con los patrones de formularios modernos.

### Introducción a la API FormData en React

`FormData` representa los datos del formulario como pares clave-valor, exactamente como se enviarían en una solicitud HTTP. Maneja todos los tipos de entrada de forma natural, desde campos de texto hasta cargas de archivos, sin requerir lógica de serialización personalizada. En React 19, `FormData` se convierte en el puente entre tus componentes y las Server Actions.

La belleza de `FormData` es su simplicidad. En lugar de mantener el estado de cada campo y construir manualmente objetos de carga útil (*payloads*), dejas que el navegador haga lo que mejor sabe hacer: recopilar datos de formularios. Este enfoque reduce el código repetitivo y elimina errores comunes relacionados con la sincronización del estado.

Aquí hay un formulario de suscripción a un boletín que demuestra la versatilidad de `FormData` con diferentes tipos de entrada:

```tsx
// Working with FormData in React
export function NewsletterSignup() {
  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    // FormData provides multiple ways to access data
    const email = formData.get('email') as string;
    const frequency = formData.get('frequency') as string;
    const topics = formData.getAll('topics') as string[];
    console.log('Subscription details:', {
      email,
      frequency,
      topics
    });
    // Convert to plain object if needed
    const data = Object.fromEntries(formData);
    // Or send directly to an API
    const response = await fetch('/api/subscribe', {
      method: 'POST',
      body: formData // FormData can be sent directly!
    });
  };
  return (
    <form onSubmit={handleSubmit} className="space-y-4 p-6">
      <div>
        <label className="block text-sm font-medium mb-2">
          Email Address
        </label>
        <input
          type="email"
          name="email"
          required
          className="w-full p-2 border rounded-lg"
        />
      </div>
      <div>
        <label className="block text-sm font-medium mb-2">
          Frequency
        </label>
        <select name="frequency" className="w-full p-2 border rounded-lg">
          <option value="daily">Daily</option>
          <option value="weekly">Weekly</option>
          <option value="monthly">Monthly</option>
        </select>
      </div>
      <div>
        <label className="block text-sm font-medium mb-2">
          Topics of Interest
        </label>
        <div className="space-y-2">
          <label className="flex items-center">
            <input type="checkbox" name="topics" value="tech" className="mr-2" />
            Technology
          </label>
          <label className="flex items-center">
            <input type="checkbox" name="topics" value="business" className="mr-2" />
            Business
          </label>
          <label className="flex items-center">
            <input type="checkbox" name="topics" value="design" className="mr-2" />
            Design
          </label>
        </div>
      </div>
      <button
        type="submit"
        className="w-full py-2 bg-blue-500 text-white rounded-lg"
      >
        Subscribe
      </button>
    </form>
  );
}
```

La llamada `new FormData(e.currentTarget)` captura todos los valores del formulario en una sola línea basándose en los atributos `name` de los inputs. Observa que las entradas son no controladas (*uncontrolled*), sin props `value` ni `onChange`. El navegador gestiona su estado de forma nativa. `FormData` proporciona `get()` para valores individuales y `getAll()` para múltiples valores con el mismo nombre. Las casillas de verificación de temas comparten `name="topics"`, por lo que `getAll('topics')` devuelve un arreglo como `['technology', 'design']` de los valores marcados. Puedes convertir `FormData` a un objeto plano con `Object.fromEntries(formData)` para APIs JSON, o enviarlo directamente a `fetch`, el cual maneja `FormData` de forma nativa, incluso con subidas de archivos.

El verdadero poder es la eliminación de código repetitivo. Compara esto con componentes controlados donde necesitarías `const [email, setEmail] = useState('')`, `const [frequency, setFrequency] = useState('daily')`, `const [topics, setTopics] = useState<string[]>([])`, más manejadores `onChange` actualizando cada variable de estado. `FormData` maneja toda esa complejidad automáticamente mientras preserva la validación HTML nativa y las funciones del navegador. El formulario simplemente funciona sin tener que gestionar el estado de los campos individuales.

### Serialización y envío de formularios usando new FormData()

`FormData` destaca cuando se trabaja con estructuras de datos complejas. Maneja objetos anidados, arreglos y tipos de datos mixtos con facilidad. La clave es comprender cómo estructurar los campos de tu formulario para producir la forma de datos deseada.

Las aplicaciones modernas a menudo necesitan enviar estructuras de datos complejas y anidadas. `FormData` puede manejar estos escenarios de manera elegante con convenciones de nomenclatura de campos adecuadas:

```tsx
// Complex form with nested data
interface Product {
  name: string;
  price: number;
  categories: string[];
  specifications: {
    weight: number;
    dimensions: {
      length: number;
      width: number;
      height: number;
    };
  };
}
export function ProductEditor() {
  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    // Build nested object from flat FormData
    const product: Product = {
      name: formData.get('name') as string,
      price: Number(formData.get('price')),
      categories: formData.getAll('categories') as string[],
      specifications: {
        weight: Number(formData.get('specs.weight')),
        dimensions: {
          length: Number(formData.get('specs.length')),
          width: Number(formData.get('specs.width')),
          height: Number(formData.get('specs.height'))
        }
      }
    };
    console.log('Structured product data:', product);
    // Send to server
    await fetch('/api/products', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(product)
    });
  };
  return (
    <form onSubmit={handleSubmit} className="space-y-6 max-w-2xl">
      <div className="grid grid-cols-2 gap-4">
        <input name="name" placeholder="Product Name" required className="p-2 border rounded" />
        <input name="price" type="number" step="0.01" placeholder="Price" required className="p-2 border rounded" />
      </div>
      <fieldset className="border p-4 rounded">
        <legend className="font-semibold">Categories</legend>
        <label className="block"><input type="checkbox" name="categories" value="electronics" /> Electronics</label>
        <label className="block"><input type="checkbox" name="categories" value="computers" /> Computers</label>
        <label className="block"><input type="checkbox" name="categories" value="accessories" /> Accessories</label>
      </fieldset>
      <fieldset className="border p-4 rounded">
        <legend className="font-semibold">Specifications</legend>
        <div className="grid grid-cols-2 gap-4">
          <input name="specs.weight" type="number" step="0.01" placeholder="Weight (kg)" className="p-2 border rounded" />
          <div className="grid grid-cols-3 gap-2">
            <input name="specs.length" type="number" placeholder="L" className="p-2 border rounded" />
            <input name="specs.width" type="number" placeholder="W" className="p-2 border rounded" />
            <input name="specs.height" type="number" placeholder="H" className="p-2 border rounded" />
          </div>
        </div>
      </fieldset>
      <button type="submit" className="px-4 py-2 bg-blue-500 text-white rounded">
        Save Product
      </button>
    </form>
  );
}
```

La técnica clave es utilizar la notación de puntos en los nombres de los campos para indicar el anidamiento: `name="specs.weight"`, `name="specs.length"`, etc. `FormData` los almacena como pares clave-valor planos, pero la convención de nomenclatura facilita la reconstrucción. Cuando llamas a `formData.get('specs.weight')`, obtienes el valor y luego construyes manualmente la estructura anidada. Observa `Number(formData.get('price'))` para la conversión de tipos: `FormData` devuelve todo como cadenas, por lo que debes convertir explícitamente los valores numéricos. El arreglo `categories` usa `getAll('categories')` para recopilar todas las casillas de verificación marcadas con el mismo nombre.

Este enfoque te brinda un control total sobre la estructura de datos final. Mapeas explícitamente las claves planas de `FormData` a las propiedades del objeto anidado, manejando la conversión de tipos y la transformación de datos según sea necesario. Si bien requiere más trabajo manual que las librerías de serialización automatizada, obtienes seguridad de tipos a través de la interfaz `Product` y claridad sobre cómo los datos del formulario se convierten exactamente en los datos de tu aplicación. Para formularios complejos con estructuras profundamente anidadas, librerías de validación como Zod pueden analizar y validar `FormData` directamente, eliminando el paso de construcción manual.

### Manejo de subidas de archivos y estructuras de datos complejas

Las subidas de archivos representan uno de los mayores puntos fuertes de `FormData`. A diferencia de JSON, `FormData` puede manejar datos binarios de forma nativa, lo que lo convierte en la opción ideal para formularios que incluyen cargas de archivos. El navegador maneja toda la complejidad de codificar y transmitir archivos, mientras tú te concentras en la experiencia del usuario.

Al trabajar con cargas de archivos, considera la experiencia del usuario de manera integral. Los usuarios necesitan comentarios sobre la selección de archivos, el progreso de la carga y la validación. Aquí hay un componente de carga de archivos completo que maneja múltiples archivos con vista previa y validación:

```tsx
// Advanced file upload with preview and validation
import { useState, useRef } from 'react';
interface FileWithPreview {
  file: File;
  preview: string;
  id: string;
}
const MAX_FILE_SIZE = 5 * 1024 * 1024; // 5MB
const ACCEPTED_TYPES = ['image/jpeg', 'image/png', 'image/webp'];
export function ImageUploader() {
  const [files, setFiles] = useState<FileWithPreview[]>([]);
  const [dragActive, setDragActive] = useState(false);
  const inputRef = useRef<HTMLInputElement>(null);
  const validateFile = (file: File): string | null => {
    if (!ACCEPTED_TYPES.includes(file.type)) {
      return `Invalid file type: ${file.type}`;
    }
    if (file.size > MAX_FILE_SIZE) {
      return `File too large: ${(file.size / 1024 / 1024).toFixed(2)}MB`;
    }
    return null;
  };
  const handleFiles = (fileList: FileList) => {
    const newFiles: FileWithPreview[] = [];
    Array.from(fileList).forEach(file => {
      const error = validateFile(file);
      if (!error) {
        newFiles.push({
          file,
          preview: URL.createObjectURL(file),
          id: `${Date.now()}-${Math.random()}`
        });
      } else {
        console.error(error);
      }
    });
    setFiles(prev => [...prev, ...newFiles]);
  };
  const handleDrop = (e: React.DragEvent) => {
    e.preventDefault();
    setDragActive(false);
    if (e.dataTransfer.files) {
      handleFiles(e.dataTransfer.files);
    }
  };
  const removeFile = (id: string) => {
    setFiles(prev => {
      const file = prev.find(f => f.id === id);
      if (file) {
        URL.revokeObjectURL(file.preview);
      }
      return prev.filter(f => f.id !== id);
    });
  };
  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData();
    files.forEach((fileWrapper, index) => {
      formData.append('images', fileWrapper.file);
    });
    // Add other form fields
    formData.append('title', 'Product Gallery');
    formData.append('timestamp', Date.now().toString());
    // Upload with progress tracking
    const xhr = new XMLHttpRequest();
    xhr.upload.addEventListener('progress', (e) => {
      if (e.lengthComputable) {
        const percentComplete = (e.loaded / e.total) * 100;
        console.log(`Upload progress: ${percentComplete.toFixed(2)}%`);
      }
    });
    xhr.onload = () => {
      if (xhr.status === 200) {
        console.log('Upload successful!');
        setFiles([]);
      }
    };
    xhr.open('POST', '/api/upload');
    xhr.send(formData);
  };
  return (
    <form onSubmit={handleSubmit} className="max-w-xl mx-auto p-6">
      {files.length > 0 && (
        <div className="grid grid-cols-3 gap-4 mt-6">
          {files.map(({ id, preview, file }) => (
            <div key={id} className="relative group">
              <img src={preview} alt="" className="w-full h-32 object-cover rounded" />
              <button type="button" onClick={() => removeFile(id)}> X
</button>
              <p className="text-xs text-gray-600 mt-1 truncate">{file.name}</p>
            </div>
          ))}
        </div>
      )}
      {files.length > 0 && (
        <button type="submit" className="w-full mt-6 py-2 bg-blue-500 text-white rounded hover:bg-blue-600">
          Upload {files.length} {files.length === 1 ? 'Image' : 'Images'}
        </button>
      )}
    </form>
  );
}
```

La función `validateFile` verifica el tipo y tamaño del archivo antes de aceptarlo, rechazando archivos no válidos en el lado del cliente. Cuando los archivos pasan la validación, `URL.createObjectURL(file)` genera una URL temporal del navegador para una vista previa instantánea, sin necesidad de subir nada al servidor todavía. De manera crítica, `URL.revokeObjectURL(file.preview)` limpia estas URLs cuando se eliminan los archivos, evitando fugas de memoria. El arrastrar y soltar (*drag-and-drop*) utiliza eventos HTML5 estándar (`onDragEnter`, `onDrop`) con `e.preventDefault()` para evitar que el navegador abra los archivos arrastrados.

Al enviar, `formData.append('images', fileWrapper.file)` agrega cada archivo con la misma clave, y el servidor los recibe como un arreglo. `FormData` mezcla archivos y campos de texto a la perfección: `formData.append('title', 'Product Gallery')`. El componente utiliza `XMLHttpRequest` en lugar de `fetch` porque solo `XMLHttpRequest` admite el seguimiento del progreso de carga mediante `xhr.upload.addEventListener('progress')`. Esto calcula el porcentaje de finalización con `(e.loaded / e.total) * 100`. El patrón maneja múltiples archivos, vistas previas instantáneas, validación del lado del cliente, seguimiento del progreso y una limpieza adecuada.

### Envío de FormData a Server Actions y APIs

Las Server Actions de React 19 proporcionan una forma fluida de manejar `FormData` en el servidor. La integración es tan suave que los formularios pueden funcionar sin JavaScript, proporcionando una mejora progresiva (*progressive enhancement*) perfecta. Las Server Actions reciben `FormData` directamente, eliminando la necesidad de serialización en el lado del cliente.

Así es como fluye `FormData` del cliente al servidor en una aplicación React moderna:

```tsx
// Server Action handling FormData
async function createPost(formData: FormData) {
  'use server';
  // Extract and validate data
  const title = formData.get('title') as string;
  const content = formData.get('content') as string;
  const tags = formData.getAll('tags') as string[];
  const image = formData.get('image') as File;
  // Handle file upload
  let imageUrl = null;
  if(image&&image.size> 0) {
    const bytes = await image.arrayBuffer();
    const buffer = Buffer.from(bytes);
    // Save to storage service
    imageUrl = await uploadToStorage(buffer, image.name);
  }
  // Save to database
  const post = await db.post.create({
    data: {
      title,
      content,
      tags,
      imageUrl,
      publishedAt: new Date()
    }
  });
  // Return result or redirect
  return { success: true, postId: post.id };
}
// Client component using the Server Action
export function BlogPostEditor() {
  const [state, formAction] = useActionState(
    createPost,
    { success: false, postId: null }
  );
  return (
    <form action={formAction} className="space-y-6 max-w-2xl mx-auto">
      <div>
        <label htmlFor="title" className="block text-sm font-medium mb-2">Post Title</label>
        <input id="title" name="title" required className="w-full p-3 border rounded-lg focus:ring-2 focus:ring-blue-500" placeholder="Enter your post title" />
      </div>
      <div>
        <label htmlFor="content" className="block text-sm font-medium mb-2">Content</label>
        <textarea id="content" name="content" required rows={10} className="w-full p-3 border rounded-lg focus:ring-2 focus:ring-blue-500" placeholder="Write your post content..." />
      </div>
      <div>
        <label htmlFor="image" className="block text-sm font-medium mb-2">Featured Image</label>
        <input id="image" name="image" type="file" accept="image/*" className="w-full p-2 border rounded-lg" />
      </div>
      <div>
        <label className="block text-sm font-medium mb-2">Tags</label>
        <div className="flex flex-wrap gap-2">
          {['React', 'TypeScript', 'Web Dev', 'Tutorial'].map(tag => (
            <label key={tag} className="flex items-center">
              <input type="checkbox" name="tags" value={tag} className="mr-1" />
              <span className="px-3 py-1 bg-gray-100 rounded-full text-sm">{tag}</span>
            </label>
          ))}
        </div>
      </div>
      <button type="submit" className="w-full py-3 bg-blue-500 text-white rounded-lg hover:bg-blue-600 transition">Publish Post</button>
      {state.success && (
        <div className="p-4 bg-green-100 text-green-700 rounded-lg">
          Post created successfully! ID: {state.postId}
        </div>
      )}
    </form>
  );
}
```

La Server Action recibe `FormData` directamente, sin necesidad de serialización JSON. La directiva `'use server'` marca esto como código exclusivo del servidor que nunca se envía al cliente. Extrae valores con `get()` y `getAll()`, convierte archivos a búferes con `await image.arrayBuffer()`, los sube al almacenamiento, los guarda en la base de datos y devuelve un objeto de resultado. Sin endpoints de API, sin manejadores de rutas, sin llamadas a `fetch`: React maneja toda la comunicación cliente-servidor.

El cliente pasa `formAction` al atributo `action` del formulario, lo que permite la mejora progresiva: el formulario funciona sin JavaScript mediante el envío estándar de HTML. Con JS habilitado, `useActionState` intercepta el envío, llama a la Server Action y actualiza el estado con los resultados. El formulario es no controlado (sin `onChange` ni `useState`), pero proporciona retroalimentación instantánea a través de `state.success`. Los archivos, textos y casillas de verificación fluyen sin problemas hacia el servidor, donde tiene acceso directo a la base de datos y al almacenamiento.

---

## Mejores prácticas para arquitecturas de formularios escalables

Crear formularios que escalen requiere más que solo manejar el estado y la validación. Exige una arquitectura reflexiva, patrones consistentes y atención a los detalles que impactan tanto en la experiencia del desarrollador como en la satisfacción del usuario. Exploremos las prácticas que separan los sistemas de formularios mantenibles de los desastres enredados.

### Estructuración de formularios grandes y dinámicos

Los formularios grandes son como ciudades: sin una planificación adecuada, se vuelven caóticos e inmantenibles. La clave para gestionar la complejidad es dividir los formularios en componentes lógicos y reutilizables que se comuniquen a través de interfaces bien definidas.

Considera un formulario de solicitud de empleo de varios pasos. En lugar de un componente monolítico, creamos un sistema de piezas interconectadas que pueden evolucionar de forma independiente:

```tsx
import { useState } from 'react';
// Define the shape of our application data
interface ApplicationData {
  personal: { firstName: string; lastName: string; email: string; };
  experience: { years: number; role: string; skills: string[]; };
  preferences: { location: string; salary: string; startDate: string; };
}
export default function ApplicationForm() {
  // Track which step (0, 1, or 2) the user is on
  const [step, setStep] = useState(0);

  // Store all form data in a single state object organized by section
  const [data, setData] = useState<ApplicationData>({
    personal: { firstName: '', lastName: '', email: '' },
    experience: { years: 0, role: '', skills: [] },
    preferences: { location: '', salary: '', startDate: '' }
  });
  
  // Helper to update nested state without losing other data
  const updateData = (section: keyof ApplicationData, key: string, val: any) => {
    setData(prevData => ({
      ...prevData,
      [section]: { ...prevData[section], [key]: val }
    }));
  };
  // Navigation functions for moving between steps 
  const next = () => setStep(s => Math.min(s + 1, 2));
  const prev = () => setStep(s => Math.max(s - 1, 0));

  // Final submission handler
  const submit = async () => {
    console.log('Submitting', data);
    alert('Application submitted successfully!');
  };
  // Progress indicator component showing current step  
  const Progress = () => (
    <div className="flex mb-6">
      {[0, 1, 2].map(i => (
        <div key={i} className={`flex-1 h-2 mx-1 rounded-full ${i <= step ? 'bg-blue-500' : 'bg-gray-200'}`} />
      ))}
    </div>
  );
  // Step 1: Personal information inputs
  const StepPersonal = () => (
    <div className="space-y-4">
      <h2 className="text-xl font-semibold mb-4">Personal Information</h2>
      <div className="grid grid-cols-2 gap-3">
        <input placeholder="First Name" value={data.personal.firstName} onChange={e => updateData('personal','firstName', e.target.value)} />
        <input placeholder="Last Name" value={data.personal.lastName} onChange={e => updateData('personal','lastName', e.target.value)} />
      </div>
      <input placeholder="Email" type="email" value={data.personal.email} onChange={e => updateData('personal','email', e.target.value)} />
    </div>
  );
  // Step 2: Professional experience inputs
  const StepExperience = () => (
    <div className="space-y-4">
      <h2 className="text-xl font-semibold mb-4">Experience</h2>
      <input placeholder="Years of Experience" type="number" value={data.experience.years || ''} onChange={e => updateData('experience','years', Number(e.target.value) || 0)} />
      <input placeholder="Current/Most Recent Role" value={data.experience.role} onChange={e => updateData('experience','role', e.target.value)} />
      <input placeholder="Skills (comma separated)" value={data.experience.skills.join(', ')} onChange={e => updateData('experience','skills', e.target.value.split(',').map(s => s.trim()).filter(Boolean))} />
    </div>
  );
  // Step 3: Job preferences inputs
  const StepPreferences = () => (
    <div className="space-y-4">
      <h2 className="text-xl font-semibold mb-4">Preferences</h2>
      <input placeholder="Preferred Location" value={data.preferences.location} onChange={e => updateData('preferences','location', e.target.value)} />
      <input placeholder="Expected Salary" value={data.preferences.salary} onChange={e => updateData('preferences','salary', e.target.value)} />
      <input placeholder="Preferred Start Date" type="date" value={data.preferences.startDate} onChange={e => updateData('preferences','startDate', e.target.value)} />
    </div>
  );
  // Render the appropriate step component based on current step
  const renderCurrentStep = () => {
    switch (step) {
      case 0: return <StepPersonal />;
      case 1: return <StepExperience />;
      case 2: return <StepPreferences />;
      default: return <StepPersonal />;
    }
  };
  return (
    <div className="max-w-2xl mx-auto p-6 space-y-6 bg-white rounded-lg shadow-lg">
      <div className="text-center">
        <h1 className="text-2xl font-bold">Job Application</h1>
        <p>Step {step + 1} of 3</p>
      </div>
      <Progress />
      {renderCurrentStep()}
      <div className="flex justify-between pt-4">
        <button onClick={prev} disabled={step === 0}>Previous</button>
        {step < 2 ? (
          <button onClick={next}>Next</button>
        ) : (
          <button onClick={submit}>Submit Application</button>
        )}
      </div>
    </div>
  );
}
```

Esta arquitectura proporciona varios beneficios. Cada paso está aislado y es testeable. El estado del formulario está centralizado pero se accede a través de una interfaz limpia. Agregar nuevos pasos o modificar los existentes no requiere tocar otras partes del sistema.

### Manejo adecuado del envío de formularios y efectos secundarios

El envío de formularios es donde muchas aplicaciones fallan. No se trata solo de enviar datos a un servidor; se trata de gestionar todo el ciclo de vida del envío, manejar los errores con elegancia, evitar envíos duplicados y proporcionar retroalimentación significativa.

Un manejador de envíos robusto considera todos los estados posibles y casos extremos:

```tsx
import { useState, useRef } from 'react';
// Define all possible submission states as a discriminated union
// This ensures type-safe state handling throughout the form lifecycle
type SubmissionState =
  | { status: 'idle' }
  | { status: 'submitting' }
  | { status: 'success'; data: any }
  | { status: 'error'; error: string };
// Generic hook: <T> represents the shape of data being submitted
// This allows the hook to work with any form data type while maintaining type safety
// Example: useFormSubmission<{email: string, message: string}>
export function useFormSubmission<T>(
  // submitFn accepts data of type T (whatever type you specify when using the hook)
  // and returns a Promise with any type of result
  submitFn: (data: T) => Promise<any>
) {
  const [state, setState] = useState<SubmissionState>({ status: 'idle' });

  // Store abort controller to cancel in-flight requests if user resubmits
  const abortControllerRef = useRef<AbortController | null>(null);
  const submit = async (data: T) => {
    // Prevent duplicate submissions
    if (state.status === 'submitting') return;
    // Cancel any pending submission
    if (abortControllerRef.current) {
      abortControllerRef.current.abort();
    }
    // Create new abort controller
    abortControllerRef.current = new AbortController();
    setState({ status: 'submitting' });
    try {
      const result = await submitFn(data);
      setState({ status: 'success', data: result });
      return result;
    } catch (error) {
      if (error instanceof Error) {
        if (error.name !== 'AbortError') {
          setState({ status: 'error', error: error.message });
        }
      } else {
        setState({ status: 'error', error: 'An unknown error occurred' });
      }
      throw error;
    }
  };
  const reset = () => setState({ status: 'idle' });
  return { state, submit, reset };
}
// Using the submission hook
export function ContactForm() {
  const { state, submit, reset } = useFormSubmission(async (data) => {
    const response = await fetch('/api/contact', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    });
    if (!response.ok) {
      throw new Error('Failed to send message');
    }
    return response.json();
  });
  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    try {
      await submit({
        email: formData.get('email'),
        message: formData.get('message')
      });
      // Clear form on success
      e.currentTarget.reset();
      // Auto-reset after 5 seconds
      setTimeout(reset, 5000);
    } catch (error) {
      // Error is already handled by the hook
    }
  };
  return (
    <form onSubmit={handleSubmit} className="space-y-4 max-w-md mx-auto">
      <input name="email" type="email" required disabled={state.status === 'submitting'} placeholder="Your email" />
      <textarea name="message" required disabled={state.status === 'submitting'} placeholder="Your message" rows={5} />
      {state.status === 'error' && (
        <div className="p-3 bg-red-100 text-red-700 rounded-lg">{state.error}</div>
      )}
      {state.status === 'success' && (
        <div className="p-3 bg-green-100 text-green-700 rounded-lg">Message sent successfully!</div>
      )}
      <button type="submit" disabled={state.status === 'submitting'}>
        {state.status === 'submitting' ? 'Sending...' : 'Send Message'}
      </button>
    </form>
  );
}
```

El hook `useFormSubmission` gestiona el estado de envío del formulario a través de cuatro fases: `idle`, `submitting`, `success` y `error`. El genérico `<T>` acepta cualquier forma de datos para envíos con tipos seguros. Evita envíos duplicados con una protección de estado y utiliza `AbortController` para cancelar solicitudes en curso. El hook devuelve `state` para la retroalimentación de la interfaz de usuario, `submit` para activar el envío y `reset` para borrar el estado. `ContactForm` demuestra el uso: pasa una función `fetch`, llama a `submit({email, message})` al enviar el formulario, muestra los estados de carga/éxito/error automáticamente, limpia el formulario en caso de éxito y se reinicia automáticamente después de 5 segundos.

### Mejora de la accesibilidad en formularios

La accesibilidad no es una ocurrencia tardía; es un aspecto fundamental del diseño de formularios. Un formulario accesible es utilizable por todos, independientemente de sus capacidades o de la tecnología que utilicen para interactuar con tu aplicación.

La base de los formularios accesibles comienza con HTML semántico y un etiquetado adecuado. Cada control de formulario necesita una etiqueta, los mensajes de error deben asociarse con sus campos y el formulario debe ser navegable usando solo el teclado. Aquí hay un ejemplo completo que demuestra las mejores prácticas de accesibilidad:

```tsx
import { useState, useId } from 'react';

interface FormData {
  name: string;
  email: string;
  phone: string;
  subscribe: boolean;
}

export default function AccessibleRegistrationForm() {
  const [formData, setFormData] = useState<FormData>({
    name: '', email: '', phone: '', subscribe: false
  });
  const [errors, setErrors] = useState<Partial<Record<keyof FormData, string>>>({});
  const [touched, setTouched] = useState<Record<keyof FormData, boolean>>({
    name: false, email: false, phone: false, subscribe: false
  });
 
  const baseId = useId();
  const fid = (k: keyof FormData) => `${baseId}-${k}`;
 
  const FIELDS = [
    { key: 'name' as keyof FormData, label: 'Full Name', type: 'text' },
    { key: 'email' as keyof FormData, label: 'Email', type: 'email' },
    { key: 'phone' as keyof FormData, label: 'Phone', type: 'tel' },
  ] as const;

  const validate = (k: keyof FormData, v: any) => {
    let err: string | undefined;
    if (k === 'name' && (!v || v.length < 2)) err = 'Name must be at least 2 characters';
    if (k === 'email' && (!v || !String(v).includes('@'))) err = 'Please enter a valid email';
    if (k === 'phone' && (!v || String(v).replace(/\D/g, '').length < 10)) err = 'Enter a 10-digit phone';
    setErrors(prev => ({ ...prev, [k]: err }));
    return !err;
  };

  const onChange = (k: keyof FormData, v: any) => {
    setFormData(prev => ({ ...prev, [k]: v }));
    if (touched[k]) validate(k, v);
  };

  const onBlur = (k: keyof FormData) => {
    setTouched(prev => ({ ...prev, [k]: true }));
    validate(k, formData[k]);
  };

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const isValid = FIELDS.map(f => validate(f.key, formData[f.key])).every(Boolean);
    if (isValid) {
      console.log('Form submitted:', formData);
      alert('Form submitted successfully!');
    } else {
      const firstErrorField = FIELDS.find(f => errors[f.key]);
      if (firstErrorField) document.getElementById(fid(firstErrorField.key))?.focus();
    }
  };

  const hasErrors = Object.values(errors).some(error => error !== undefined);

  return (
    <div className="min-h-screen bg-gray-50 py-12 px-4">
      <form onSubmit={handleSubmit} className="max-w-md mx-auto bg-white rounded-lg shadow-md p-8" noValidate>
        <div className="mb-6">
          <h1 className="text-2xl font-bold text-gray-900">Create Your Account</h1>
          <p className="mt-2 text-sm text-gray-600">Fields marked with * are required</p>
        </div>

        {hasErrors && (
         <div role="alert" className="mb-6 p-4 bg-red-50 border border-red-200 rounded-md">
            <p className="text-sm text-red-800">Please correct the errors below</p>
          </div>
        )}

        {FIELDS.map(field => (
          <div key={field.key} className="mb-4">
            <label htmlFor={fid(field.key)} className="block text-sm font-medium text-gray-700 mb-1">
              {field.label} *
            </label>
            <input
              type={field.type}
              id={fid(field.key)}
              name={field.key}
              value={formData[field.key]}
              onChange={e => onChange(field.key, e.target.value)}
              onBlur={() => onBlur(field.key)}
              aria-required="true"
              aria-invalid={!!errors[field.key]}
              aria-describedby={errors[field.key] ? `${fid(field.key)}-err` : undefined}
              className={`w-full p-3 border rounded-md focus:outline-none focus:ring-2 ${
                errors[field.key] ? 'border-red-500 focus:ring-red-500' : 'border-gray-300 focus:ring-blue-500'
              }`}
              placeholder={`Enter your ${field.label.toLowerCase()}`}
            />
            {errors[field.key] && (
              <p id={`${fid(field.key)}-err`} className="mt-1 text-sm text-red-600" role="alert">
                {errors[field.key]}
              </p>
            )}
          </div>
        ))}

        <div className="mb-6">
          <label className="flex items-start">
            <input
              type="checkbox"
              id={fid('subscribe')}
              name="subscribe"
              checked={formData.subscribe}
              onChange={e => onChange('subscribe', e.target.checked)}
              className="w-4 h-4 mt-1 text-blue-600 border-gray-300 rounded focus:ring-blue-500"
            />
            <span className="ml-2 text-sm text-gray-700">Subscribe to newsletter</span>
          </label>
          <p id={`${fid('subscribe')}-hint`} className="ml-6 mt-1 text-xs text-gray-500">
            Get updates about new features and promotions (optional)
          </p>
        </div>

        <button type="submit" className="w-full py-3 px-4 bg-blue-600 text-white font-medium rounded-md hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2 transition-colors">
          Create Account
        </button>
      </form>
    </div>
  );
}
```

El formulario utiliza `useId()` para generar IDs únicos, evitando colisiones cuando existen múltiples formularios. Cada entrada tiene atributos ARIA: `aria-required`, `aria-invalid` y `aria-describedby` que se vinculan a los mensajes de error, de modo que los lectores de pantalla anuncian el estado de validación y los errores. En caso de fallo en el envío, `document.getElementById(fid(firstErrorField.key))?.focus()` enfoca automáticamente el primer campo no válido. El estado `touched` evita que los errores se muestren al cargar la página; la validación solo se activa en `onBlur` o `onChange` después de que un campo se haya tocado una vez. Este patrón de validar al perder el foco y luego validar en vivo evita frustrar a los usuarios a mitad de la escritura. El estilo del campo cambia dinámicamente: rojo para errores, azul para válido. El arreglo `FIELDS` con `.map()` hace que agregar campos sea trivial.

### Navegación por teclado y gestión del foco

La navegación por teclado es igualmente crucial. Los usuarios deben poder navegar por el formulario usando la tecla Tab, enviar con Enter e interactuar con componentes personalizados usando las teclas de flecha. La gestión del foco se vuelve particularmente importante en formularios dinámicos donde los campos aparecen y desaparecen. Así es como se maneja el foco adecuadamente en escenarios dinámicos:

```tsx
import { useState, useRef, useEffect } from 'react';

export default function DynamicSurveyForm() {
  const [showOptional, setShowOptional] = useState(false);
  const [rating, setRating] = useState(0);
  // Ref to focus the optional field when it appears
  const optionalFieldRef = useRef<HTMLTextAreaElement>(null);
  // Live region for screen reader announcements
  const announcementRef = useRef<HTMLDivElement>(null);
 
  // Announce dynamic changes to screen readers
  const announce = (message: string) => {
    if (announcementRef.current) {
      announcementRef.current.textContent = message;
    }
  };
 
  // Focus management: when optional field appears, focus it automatically
  useEffect(() => {
    if (showOptional && optionalFieldRef.current) {
      optionalFieldRef.current.focus();
      announce('Additional feedback field is now available');
    }
  }, [showOptional]);
 
  // Handle rating changes and show optional field for low ratings
  const handleRatingChange = (newRating: number) => {
    setRating(newRating);
    setShowOptional(newRating <= 3);
    announce(newRating <= 3
      ? `Rating ${newRating} selected. Please provide additional feedback.`
      : `Rating ${newRating} selected. Thank you!`
    );
  };
 
  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    alert('Feedback submitted!');
  };
 
  return (
    <div className="max-w-md mx-auto p-6 bg-white rounded-lg shadow-md">
      <h2 className="text-2xl font-bold mb-6">How was your experience?</h2>
     
      {/* Screen reader live region for announcements */}
      <div ref={announcementRef} role="status" aria-live="polite" aria-atomic="true" className="sr-only" />
     
      <form onSubmit={handleSubmit} className="space-y-6">
        {/* Rating buttons with keyboard navigation */}
        <div role="group" aria-labelledby="rating-label">
          <label id="rating-label" className="block text-sm font-medium mb-3">
            Rate your experience from 1 to 5
          </label>
          <div className="flex gap-2">
            {[1, 2, 3, 4, 5].map((value) => (
              <button
                key={value}
                type="button"
                onClick={() => handleRatingChange(value)}
                // Arrow key navigation for better UX
                onKeyDown={(e) => {
if(e.key==='ArrowRight'&&value< 5) handleRatingChange(value + 1);
if(e.key==='ArrowLeft'&&value> 1) handleRatingChange(value - 1);
                }}
                aria-pressed={rating === value}
                className={`w-12 h-12 rounded-full font-bold focus:outline-none focus:ring-2 focus:ring-blue-500 ${
              rating === value ? 'bg-blue-500 text-white' : 'bg-gray-200 hover:bg-gray-300'
                }`}
              >
                {value}
              </button>
            ))}
          </div>
        </div>
       
        {/* Conditionally rendered field with automatic focus */}
        {showOptional && (
          <div>
            <label htmlFor="feedback" className="block text-sm font-medium mb-2">
              What could we improve? <span className="text-gray-500">(Optional)</span>
            </label>
            <textarea ref={optionalFieldRef} id="feedback" name="feedback" rows={4} aria-describedby="feedback-help" className="w-full p-3 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500" />
            <p id="feedback-help" className="text-sm text-gray-600 mt-1">Your feedback will be reviewed by our team</p>
          </div>
        )}
       
        <button type="submit" className="w-full py-2 bg-blue-500 text-white rounded hover:bg-blue-600 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2">
          Submit Feedback
        </button>
      </form>
    </div>
  );
}
```

El `announcementRef` crea una región en vivo ARIA (`role="status"`, `aria-live="polite"`) que está visualmente oculta con `sr-only`. Cuando los usuarios cambian las calificaciones, `announce()` actualiza el texto de este elemento, lo que hace que los lectores de pantalla anuncien automáticamente los cambios. El `useEffect` observa `showOptional`: cuando una calificación baja (≤3) hace que aparezca el campo opcional, se enfoca automáticamente con `optionalFieldRef.current.focus()` y anuncia el cambio, evitando que los usuarios de teclado pierdan su posición. Los botones de calificación admiten la navegación con teclas de flecha a través de `onKeyDown` (`ArrowRight`/`ArrowLeft` para cambiar las calificaciones), y `aria-pressed` indica el estado seleccionado. Este patrón funciona para cualquier formulario dinámico donde los campos aparecen/desaparecen según la entrada.

### Mensajes de error accesibles y anuncios para lectores de pantalla

El manejo de errores en formularios accesibles requiere una atención especial. Los errores deben anunciarse de inmediato a los usuarios de lectores de pantalla, y los mensajes de error deben ser claros y procesables. Aquí hay un patrón para la validación inline accesible con mensajes de error útiles:

```tsx
import { useState, useId } from 'react';

export function AccessiblePasswordField() {
  const [password, setPassword] = useState('');
  const [showPassword, setShowPassword] = useState(false);
  const [focused, setFocused] = useState(false);
  const fieldId = useId();
  const helpId = useId();
  const errorId = useId();
 
  // Define password requirements with validation functions
  const requirements = [
    { test: (p: string) => p.length >= 8, message: 'At least 8 characters' },
    { test: (p: string) => /[A-Z]/.test(p), message: 'One uppercase letter' },
    { test: (p: string) => /[a-z]/.test(p), message: 'One lowercase letter' },
    { test: (p: string) => /[0-9]/.test(p), message: 'One number' },
    { test: (p: string) => /[^A-Za-z0-9]/.test(p), message: 'One special character' }
  ];
 
  const failedRequirements = requirements.filter(req => !req.test(password));
  const isValid = password.length > 0 && failedRequirements.length === 0;
 
  return (
    <div className="space-y-2">
      <label htmlFor={fieldId} className="block text-sm font-medium">
        Create Password <span className="text-red-500">*</span>
      </label>
     
      {/* Password input with toggle visibility button */}
      <div className="relative">
        <input
          type={showPassword ? 'text' : 'password'}
          id={fieldId}
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          onFocus={() => setFocused(true)}
          onBlur={() => setFocused(false)}
          aria-required="true"
          aria-invalid={password.length > 0 && !isValid}
          aria-describedby={`${helpId} ${failedRequirements.length > 0 ? errorId : ''}`}
          className={`w-full pr-10 p-2 border rounded ${password.length > 0 ? isValid ? 'border-green-500' : 'border-red-500' : 'border-gray-300'} focus:outline-none focus:ring-2 focus:ring-blue-500`}
        />
        <button type="button" onClick={() => setShowPassword(!showPassword)} aria-label={showPassword ? 'Hide password' : 'Show password'} className="absolute right-2 top-2 p-1 text-gray-500 hover:text-gray-700">
{showPassword ? '👁️' : '👁️🗨️'}
        </button>
      </div>
     
      {/* Visual requirements checklist */}
      {(focused || failedRequirements.length > 0) && (
        <div id={helpId} className="text-sm">
          <p className="font-medium mb-1">Password must contain:</p>
          <ul className="space-y-1">
            {requirements.map((req, index) => {
              const met = req.test(password);
              return (
                <li key={index} className={met ? 'text-green-600' : 'text-gray-600'}>
                  <span className="font-bold mr-2">
  {met ? '✓' : '○'}
</span>                  {req.message}
                </li>
              );
            })}
          </ul>
        </div>
      )}
     
      {/* Screen reader error announcement */}
      {password.length >0&&failedRequirements.length> 0 && (
        <div id={errorId} role="alert" aria-live="polite" className="sr-only">
          Password does not meet {failedRequirements.length} requirement(s): {failedRequirements.map(req => req.message).join(', ')}
        </div>
      )}
    </div>
  );
}
```

El componente valida contraseñas en tiempo real frente a cinco requisitos (longitud, mayúscula, minúscula, número, carácter especial), filtrándolos en `failedRequirements` para determinar la validez. Utiliza `useId()` para generar IDs únicos para vincular la entrada con el texto de ayuda y los errores a través de `aria-describedby`, lo que indica a los lectores de pantalla dónde encontrar información adicional. La entrada tiene `aria-invalid={password.length > 0 && !isValid}` para anunciar el estado de validación y cambia dinámicamente los colores de los bordes (verde para válido, rojo para inválido). La lista de verificación de requisitos muestra marcas de verificación (✓) para los requisitos cumplidos y círculos (○) para los no cumplidos, apareciendo cuando está enfocado o cuando existen errores. El elemento oculto `<div role="alert" aria-live="polite">` crea una región en vivo que los lectores de pantalla anuncian automáticamente cuando los errores cambian, proporcionando el conteo y la lista de requisitos fallidos sin que los usuarios tengan que navegar hasta el mensaje de error. El botón de alternancia de visibilidad de contraseña utiliza `aria-label` para describir claramente su función a los lectores de pantalla.

---

## Resumen

En este capítulo, redefinimos cómo se construyen los formularios en React aprovechando las características orientadas al servidor de React 19, la validación basada en esquemas y la API nativa de `FormData`. En lugar de tratar los formularios como una red enredada de entradas controladas, aprendimos a verlos como máquinas de estados con transiciones claras, impulsadas por `useActionState` y `useFormStatus`. Este cambio no solo reduce el código repetitivo, sino que también permite actualizaciones optimistas e interacciones de usuario más fluidas.

La validación se volvió más simple y segura con Zod, brindándonos una única fuente de verdad tanto para el cliente como para el servidor. Combinado con TypeScript, esto garantiza la seguridad de tipos, la validación en tiempo de ejecución y un manejo de errores reutilizable en toda la aplicación. Mientras tanto, `FormData` se integra de forma natural con estos patrones, manejando texto, archivos y envíos sin necesidad de lógica de serialización personalizada.

Finalmente, enfatizamos que la accesibilidad y la experiencia del usuario no son negociables. Al centrarnos en la mejora progresiva, la retroalimentación clara y los componentes reutilizables, podemos crear formularios que escalen desde páginas de contacto simples hasta flujos de trabajo de varios pasos. Con las nuevas herramientas de React 19 y una arquitectura reflexiva, los formularios pasan de ser una de las partes más frustrantes del desarrollo a una de las más elegantes y agradables.

En el próximo capítulo, nos alejaremos de las funciones individuales para examinar cómo estructurar aplicaciones completas de React para el éxito a largo plazo, explorando patrones arquitectónicos, estrategias de organización de proyectos y las herramientas que mantienen los repositorios mantenibles a medida que crecen los equipos y los requisitos.
