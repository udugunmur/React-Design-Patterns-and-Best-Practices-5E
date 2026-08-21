# Capítulo 14: Pruebas Automatizadas para Aplicaciones React

Las pruebas son la base invisible que mantiene unidas a tus aplicaciones de React. Si bien los usuarios nunca ven tus pruebas, experimentan sus beneficios cada vez que una funcionalidad opera correctamente y cada vez que se detecta un error antes del despliegue.

Esta confiabilidad tiene un costo. Escribir pruebas requiere un tiempo que, de otro modo, podría dedicarse a crear funcionalidades. Las pruebas requieren mantenimiento a medida que tu aplicación evoluciona; cambia el comportamiento de un componente y también tendrás que actualizar sus pruebas. Una suite de pruebas integral puede ralentizar significativamente tu pipeline de CI: las pruebas unitarias se ejecutan en milisegundos, pero las pruebas end-to-end que inician navegadores e interactúan con servicios reales pueden tardar minutos. En una aplicación grande, una ejecución de prueba completa puede tardar de 20 a 30 minutos, bloqueando los despliegues y ralentizando los ciclos de retroalimentación. Las pruebas inestables (*flaky tests*), aquellas que pasan y fallan de forma intermitente, erosionan la confianza en tu suite de pruebas y hacen que los desarrolladores pierdan tiempo investigando fallos falsos.

El objetivo no es probarlo todo. Se trata de probar de forma estratégica, centrando el esfuerzo donde proporciona el mayor valor en relación con el costo. Un flujo de pago que procesa transacciones merece pruebas exhaustivas; una página estática de "Acerca de nosotros" probablemente no. La lógica de negocio crítica justifica pruebas unitarias con casos extremos cubiertos; los componentes de interfaz de usuario sencillos tal vez solo necesiten una prueba de humo (*smoke test*) que confirme que se renderizan.

El panorama de pruebas para React a menudo se describe en tres capas: unitarias, de integración y end-to-end. En la práctica, estos límites se difuminan. Cuando pruebas un componente que renderiza hijos y gestiona estado, ¿es una prueba unitaria o una prueba de integración? La distinción importa menos que comprender qué valida cada prueba. Las pruebas que examinan componentes de forma aislada con dependencias simuladas (*mocks*) detectan problemas diferentes a los de las pruebas que permiten a los componentes interactuar naturalmente. Las pruebas end-to-end validan toda tu infraestructura tecnológica (frontend, backend, base de datos), simulando el comportamiento real del usuario frente a una infraestructura real.

A lo largo de este capítulo, construiremos una comprensión práctica de las pruebas creando una aplicación de gestión de tareas. Aprenderás a escribir pruebas que sean fáciles de mantener, significativas y que te den confianza en tu código sin ralentizarte.

En este capítulo, cubriremos los siguientes temas:

- Escritura de pruebas unitarias con Jest y React Testing Library
- Pruebas de integración para componentes de React
- Configuración de Playwright para pruebas end-to-end
- Automatización de pruebas en pipelines de CI/CD con GitHub Actions

---

## Escritura de pruebas unitarias con Jest y React Testing Library

React Testing Library ha revolucionado la forma en que pensamos sobre las pruebas de componentes de React. En lugar de probar detalles de implementación, nos anima a probar los componentes de la forma en que los usuarios interactúan con ellos. Esta filosofía conduce a pruebas que son más resistentes a la refactorización y reflejan mejor el uso en el mundo real.

Comencemos con un componente simple pero realista: un elemento de tarea (*task item*) en nuestro administrador de tareas. Este componente muestra una tarea, permite a los usuarios alternar su estado de finalización y proporciona un botón para eliminar:

```tsx
// TaskItem.tsx
interface Task {
  id: string;
  title: string;
  completed: boolean;
}

interface TaskItemProps {
  task: Task;
  onToggle: (id: string) => void;
  onDelete: (id: string) => void;
}

export const TaskItem = ({ task, onToggle, onDelete }: TaskItemProps) => {
  return (
    <div className="flex items-center justify-between p-4 bg-white border border-gray-200 rounded-lg shadow-sm">
      <div className="flex items-center gap-3">
        <input
          type="checkbox"
          checked={task.completed}
          onChange={() => onToggle(task.id)}
          className="w-5 h-5 text-blue-600 rounded focus:ring-2 focus:ring-blue-500"
          aria-label={`Toggle ${task.title}`}
        />
        <span
          className={`text-lg ${
            task.completed ? 'line-through text-gray-400' : 'text-gray-800'
          }`}
        >
          {task.title}
        </span>
      </div>
      <button
        onClick={() => onDelete(task.id)}
        className="px-3 py-1 text-sm text-white bg-red-500 rounded hover:bg-red-600 focus:outline-none focus:ring-2 focus:ring-red-500"
        aria-label={`Delete ${task.title}`}
      >
        Delete
      </button>
    </div>
  );
};
```

Ahora escribamos pruebas que verifiquen que este componente se comporte correctamente. Probaremos la presentación visual, las interacciones del usuario y las características de accesibilidad:

```tsx
// TaskItem.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { TaskItem } from './TaskItem';

describe('TaskItem', () => {
  const mockTask = {
    id: '1',
    title: 'Write unit tests',
    completed: false,
  };
  const mockOnToggle = jest.fn();
  const mockOnDelete = jest.fn();

  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('renders the task title correctly', () => {
    render(
      <TaskItem
        task={mockTask}
        onToggle={mockOnToggle}
        onDelete={mockOnDelete}
      />
    );
    expect(screen.getByText('Write unit tests')).toBeInTheDocument();
  });

  it('displays unchecked checkbox for incomplete tasks', () => {
    render(
      <TaskItem
        task={mockTask}
        onToggle={mockOnToggle}
        onDelete={mockOnDelete}
      />
    );
    const checkbox = screen.getByRole('checkbox', {
      name: /toggle write unit tests/i,
    });
    expect(checkbox).not.toBeChecked();
  });

  it('displays checked checkbox for completed tasks', () => {
    const completedTask = { ...mockTask, completed: true };
    render(
      <TaskItem
        task={completedTask}
        onToggle={mockOnToggle}
        onDelete={mockOnDelete}
      />
    );
    const checkbox = screen.getByRole('checkbox', {
      name: /toggle write unit tests/i,
    });
    expect(checkbox).toBeChecked();
  });

  it('applies strike-through styling to completed tasks', () => {
    const completedTask = { ...mockTask, completed: true };
    render(
      <TaskItem
        task={completedTask}
        onToggle={mockOnToggle}
        onDelete={mockOnDelete}
      />
    );
    const title = screen.getByText('Write unit tests');
    expect(title).toHaveClass('line-through');
  });

  it('calls onToggle when checkbox is clicked', async () => {
    const user = userEvent.setup();
    render(
      <TaskItem
        task={mockTask}
        onToggle={mockOnToggle}
        onDelete={mockOnDelete}
      />
    );
    const checkbox = screen.getByRole('checkbox', {
      name: /toggle write unit tests/i,
    });
    await user.click(checkbox);
    expect(mockOnToggle).toHaveBeenCalledTimes(1);
    expect(mockOnToggle).toHaveBeenCalledWith('1');
  });

  it('calls onDelete when delete button is clicked', async () => {
    const user = userEvent.setup();
    render(
      <TaskItem
        task={mockTask}
        onToggle={mockOnToggle}
        onDelete={mockOnDelete}
      />
    );
    const deleteButton = screen.getByRole('button', {
      name: /delete write unit tests/i,
    });
    await user.click(deleteButton);
    expect(mockOnDelete).toHaveBeenCalledTimes(1);
    expect(mockOnDelete).toHaveBeenCalledWith('1');
  });
});
```

Usamos `screen.getByRole` para buscar elementos por su rol de accesibilidad, lo que nos anima a pensar en cómo las tecnologías de asistencia interpretarán nuestros componentes y hace que las pruebas sean más resistentes a los cambios de marcado. Sin embargo, pasar estas consultas no garantiza la accesibilidad; solo confirma que los roles ARIA están presentes. La verdadera accesibilidad requiere consideraciones adicionales como navegación por teclado, gestión del foco, contraste de color y un etiquetado adecuado que estas pruebas no cubren.

El hook `beforeEach` restablece el estado de los mocks entre pruebas, asegurando que cada prueba se ejecute de forma independiente. Ten en cuenta que `clearAllMocks()` solo restablece el historial de llamadas (`.calls`, `.results`), por lo que si una prueba anterior cambió el valor de retorno de un mock, ese cambio persiste. Usa `resetAllMocks()` para restablecer también las implementaciones de los mocks, o `restoreAllMocks()` cuando trabajes con espías (*spies*) que necesiten restaurar sus funciones originales.

Exploremos un escenario más complejo con un componente que gestiona el estado internamente. Nuestro componente `TaskForm` debe manejar la entrada del usuario, la validación y el envío del formulario:

```tsx
// TaskForm.tsx
import { FormEvent, useState } from 'react';

interface TaskFormProps {
  onSubmit: (title: string) => void;
}

export const TaskForm = ({ onSubmit }: TaskFormProps) => {
  const [title, setTitle] = useState('');
  const [error, setError] = useState('');

  const handleSubmit = (e: FormEvent) => {
    e.preventDefault();
    const trimmedTitle = title.trim();

    if (!trimmedTitle) {
      setError('Task title cannot be empty');
      return;
    }

    if (trimmedTitle.length < 3) {
      setError('Task title must be at least 3 characters');
      return;
    }

    onSubmit(trimmedTitle);
    setTitle('');
    setError('');
  };

  return (
    <form onSubmit={handleSubmit} className="mb-6">
      <div className="flex gap-2">
        <div className="flex-1">
          <input
            type="text"
            value={title}
            onChange={(e) => {
              setTitle(e.target.value);
              setError('');
            }}
            placeholder="Enter a new task..."
            className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
            aria-label="Task title"
            aria-invalid={!!error}
            aria-describedby={error ? 'task-error' : undefined}
          />
          {error && (
            <p id="task-error" className="mt-1 text-sm text-red-600" role="alert">
              {error}
            </p>
          )}
        </div>
        <button
          type="submit"
          className="px-6 py-2 text-white bg-blue-600 rounded-lg hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500"
        >
          Add Task
        </button>
      </div>
    </form>
  );
};
```

Probar este formulario requiere verificar tanto los caminos felices (*happy paths*) como los casos de error. Necesitamos asegurarnos de que la validación funcione correctamente y de que el formulario se limpie después de un envío exitoso:

```tsx
// TaskForm.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { TaskForm } from './TaskForm';

describe('TaskForm', () => {
  const mockOnSubmit = jest.fn();

  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('allows users to type in the input field', async () => {
    const user = userEvent.setup();
    render(<TaskForm onSubmit={mockOnSubmit} />);
    const input = screen.getByLabelText(/task title/i);
    await user.type(input, 'New task');
    expect(input).toHaveValue('New task');
  });

  it('submits the form with valid input', async () => {
    const user = userEvent.setup();
    render(<TaskForm onSubmit={mockOnSubmit} />);
    const input = screen.getByLabelText(/task title/i);
    const submitButton = screen.getByRole('button', { name: /add task/i });
    await user.type(input, 'Valid task');
    await user.click(submitButton);
    expect(mockOnSubmit).toHaveBeenCalledWith('Valid task');
    expect(input).toHaveValue('');
  });

  it('shows error for empty input', async () => {
    const user = userEvent.setup();
    render(<TaskForm onSubmit={mockOnSubmit} />);
    const submitButton = screen.getByRole('button', { name: /add task/i });
    await user.click(submitButton);
    expect(screen.getByRole('alert')).toHaveTextContent(
      'Task title cannot be empty'
    );
    expect(mockOnSubmit).not.toHaveBeenCalled();
  });

  it('shows error for input that is too short', async () => {
    const user = userEvent.setup();
    render(<TaskForm onSubmit={mockOnSubmit} />);
    const input = screen.getByLabelText(/task title/i);
    const submitButton = screen.getByRole('button', { name: /add task/i });
    await user.type(input, 'ab');
    await user.click(submitButton);
    expect(screen.getByRole('alert')).toHaveTextContent(
      'Task title must be at least 3 characters'
    );
    expect(mockOnSubmit).not.toHaveBeenCalled();
  });

  it('clears error when user starts typing again', async () => {
    const user = userEvent.setup();
    render(<TaskForm onSubmit={mockOnSubmit} />);
    const input = screen.getByLabelText(/task title/i);
    const submitButton = screen.getByRole('button', { name: /add task/i });
    await user.click(submitButton);
    expect(screen.getByRole('alert')).toBeInTheDocument();
    await user.type(input, 'a');
    expect(screen.queryByRole('alert')).not.toBeInTheDocument();
  });
});
```

Observa cómo verificamos lo que los usuarios pueden observar: mensajes de error que aparecen y desaparecen adecuadamente. Testing Library proporciona tres variantes de consulta, cada una para diferentes escenarios:
- `getByRole` lanza un error inmediatamente si no se encuentra el elemento, lo que lo hace ideal para elementos que deben estar presentes.
- `queryByRole` devuelve `null` en lugar de lanzar un error, perfecto para afirmar que algo no debería existir sincrónicamente.
- `findByRole` espera de forma asíncrona y reintenta hasta que el elemento aparece o expira el tiempo de espera, esencial para elementos que aparecen después de actualizaciones de estado o llamadas a la API.

Para los elementos que desaparecen de forma asíncrona, como un indicador de carga (*spinner*) después de completarse una API, usa `waitForElementToBeRemoved` o envuelve tu consulta en `waitFor`. Una verificación simple con `queryByRole` pasaría inmediatamente antes de que el elemento tenga tiempo de desaparecer.

Dicho esto, probar si se llamó a `mockOnSubmit` es un compromiso pragmático más que una verdadera prueba desde la perspectiva del usuario; los usuarios experimentan mensajes de éxito, redirecciones o estados de carga, no invocaciones de callbacks. Estas aserciones prueban el contrato del componente con su padre, no el viaje completo del usuario.

---

## Pruebas de integración para componentes de React

Mientras que las pruebas unitarias verifican componentes individuales de forma aislada, las pruebas de integración aseguran que múltiples componentes funcionen juntos correctamente. Este nivel de pruebas detecta problemas que surgen de las interacciones entre componentes, el flujo de datos y la gestión del estado compartido.

Construyamos una lista de tareas completa que combine nuestros componentes anteriores con la gestión de estado. Esto representa una porción realista de funcionalidad con la que los usuarios interactúan como una característica cohesiva:

```tsx
// TaskList.tsx
import { useState } from 'react';
import { TaskItem } from './TaskItem';
import { TaskForm } from './TaskForm';

interface Task {
  id: string;
  title: string;
  completed: boolean;
}

export const TaskList = () => {
  const [tasks, setTasks] = useState<Task[]>([]);

  const addTask = (title: string) => {
    const newTask: Task = {
      id: crypto.randomUUID(),
      title,
      completed: false,
    };
    setTasks((prev) => [...prev, newTask]);
  };

  const toggleTask = (id: string) => {
    setTasks((prev) =>
      prev.map((task) =>
        task.id === id ? { ...task, completed: !task.completed } : task
      )
    );
  };

  const deleteTask = (id: string) => {
    setTasks((prev) => prev.filter((task) => task.id !== id));
  };

  const completedCount = tasks.filter((task) => task.completed).length;
  const totalCount = tasks.length;

  return (
    <div className="max-w-2xl mx-auto p-6">
      <h1 className="text-3xl font-bold text-gray-800 mb-2">Task Manager</h1>
      <p className="text-gray-600 mb-6">
        {completedCount} of {totalCount} tasks completed
      </p>
      <TaskForm onSubmit={addTask} />
      {tasks.length === 0 ? (
        <p className="text-center text-gray-500 py-8">
          No tasks yet. Add one above to get started!
        </p>
      ) : (
        <div className="space-y-2">
          {tasks.map((task) => (
            <TaskItem
              key={task.id}
              task={task}
              onToggle={toggleTask}
              onDelete={deleteTask}
            />
          ))}
        </div>
      )}
    </div>
  );
};
```

Nuestras pruebas de integración verificarán todo el flujo de trabajo desde la perspectiva del usuario: agregar tareas a través del formulario, verlas aparecer en la lista, alternar su estado de finalización, eliminarlas y asegurarse de que el conteo del resumen se actualice correctamente a lo largo de estas interacciones. A diferencia de nuestras pruebas unitarias, no simularemos `TaskForm` ni `TaskItem`; queremos verificar que estos componentes funcionen juntos tal como los experimentarán los usuarios en la realidad:

```tsx
// TaskList.test.tsx
import { render, screen, within } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { TaskList } from './TaskList';

describe('TaskList Integration', () => {
  it('displays empty state when no tasks exist', () => {
    render(<TaskList />);
    expect(screen.getByText(/no tasks yet/i)).toBeInTheDocument();
    expect(screen.getByText('0 of 0 tasks completed')).toBeInTheDocument();
  });

  it('adds a new task and displays it', async () => {
    const user = userEvent.setup();
    render(<TaskList />);
    const input = screen.getByLabelText(/task title/i);
    const addButton = screen.getByRole('button', { name: /add task/i });
    await user.type(input, 'Buy groceries');
    await user.click(addButton);
    expect(screen.getByText('Buy groceries')).toBeInTheDocument();
    expect(screen.queryByText(/no tasks yet/i)).not.toBeInTheDocument();
  });

  it('adds multiple tasks in sequence', async () => {
    const user = userEvent.setup();
    render(<TaskList />);
    const input = screen.getByLabelText(/task title/i);
    const addButton = screen.getByRole('button', { name: /add task/i });
    await user.type(input, 'First task');
    await user.click(addButton);
    await user.type(input, 'Second task');
    await user.click(addButton);
    await user.type(input, 'Third task');
    await user.click(addButton);
    expect(screen.getByText('First task')).toBeInTheDocument();
    expect(screen.getByText('Second task')).toBeInTheDocument();
    expect(screen.getByText('Third task')).toBeInTheDocument();
    expect(screen.getByText('0 of 3 tasks completed')).toBeInTheDocument();
  });

  it('toggles task completion and updates counter', async () => {
    const user = userEvent.setup();
    render(<TaskList />);
    const input = screen.getByLabelText(/task title/i);
    const addButton = screen.getByRole('button', { name: /add task/i });
    await user.type(input, 'Complete me');
    await user.click(addButton);
    const checkbox = screen.getByRole('checkbox', {
      name: /toggle complete me/i,
    });
    await user.click(checkbox);
    expect(checkbox).toBeChecked();
    expect(screen.getByText('1 of 1 tasks completed')).toBeInTheDocument();
    await user.click(checkbox);
    expect(checkbox).not.toBeChecked();
    expect(screen.getByText('0 of 1 tasks completed')).toBeInTheDocument();
  });
});
```

Continuemos con escenarios de integración más complejos que prueban el flujo de trabajo completo:

```tsx
// TaskList.test.tsx (continued)
describe('TaskList Integration - Complex Workflows', () => {
  it('deletes a task and updates the list', async () => {
    const user = userEvent.setup();
    render(<TaskList />);
    const input = screen.getByLabelText(/task title/i);
    const addButton = screen.getByRole('button', { name: /add task/i });
    await user.type(input, 'Task to delete');
    await user.click(addButton);
    const deleteButton = screen.getByRole('button', {
      name: /delete task to delete/i,
    });
    await user.click(deleteButton);
    expect(screen.queryByText('Task to delete')).not.toBeInTheDocument();
    expect(screen.getByText(/no tasks yet/i)).toBeInTheDocument();
  });

  it('manages a realistic task workflow', async () => {
    const user = userEvent.setup();
    render(<TaskList />);
    const input = screen.getByLabelText(/task title/i);
    const addButton = screen.getByRole('button', { name: /add task/i });
    await user.type(input, 'Morning exercise');
    await user.click(addButton);
    await user.type(input, 'Check emails');
    await user.click(addButton);
    await user.type(input, 'Team meeting');
    await user.click(addButton);
    const morningCheckbox = screen.getByRole('checkbox', {
      name: /toggle morning exercise/i,
    });
    await user.click(morningCheckbox);
    const emailCheckbox = screen.getByRole('checkbox', {
      name: /toggle check emails/i,
    });
    await user.click(emailCheckbox);
    expect(screen.getByText('2 of 3 tasks completed')).toBeInTheDocument();
    const deleteButton = screen.getByRole('button', {
      name: /delete check emails/i,
    });
    await user.click(deleteButton);
    expect(screen.getByText('1 of 2 tasks completed')).toBeInTheDocument();
    expect(screen.getByText('Morning exercise')).toBeInTheDocument();
    expect(screen.queryByText('Check emails')).not.toBeInTheDocument();
    expect(screen.getByText('Team meeting')).toBeInTheDocument();
  });
});
```

La prueba final demuestra el valor de las pruebas de integración. En una sola prueba, ejercitamos tres componentes (`TaskList`, `TaskForm` y múltiples instancias de `TaskItem`), realizamos seis interacciones de usuario (tres adiciones, dos alternancias, una eliminación) y verificamos que las actualizaciones de estado se propaguen correctamente a través de todo el árbol de componentes. Observa lo que no estamos probando: nunca verificamos variables de estado internas, no verificamos que se hayan llamado funciones específicas y no simulamos ningún componente secundario. En su lugar, verificamos resultados que un usuario reconocería: las tareas aparecen cuando se agregan, las casillas de verificación reflejan el estado de finalización, el contador se mantiene preciso a medida que cambian las tareas y los elementos eliminados desaparecen. Si más adelante refactorizas `TaskList` para usar `useReducer` en lugar de `useState`, o extraes la lógica de tareas en un hook personalizado, estas pruebas continúan pasando sin cambios porque prueban el comportamiento en lugar de la implementación. Esta resistencia a la refactorización es el sello distintivo de las pruebas de integración bien diseñadas.

Si bien las pruebas de integración nos dan confianza de que los componentes funcionan juntos, todavía se ejecutan en un entorno controlado, por lo que a continuación entraremos en condiciones del mundo real con pruebas end-to-end.

---

## Configuración de Playwright para pruebas end-to-end

Las pruebas end-to-end nos llevan aún más lejos, ejecutando tu aplicación en un navegador real y simulando interacciones genuinas del usuario. Playwright se destaca en este nivel de pruebas, ofreciendo potentes APIs para interactuar con aplicaciones web en múltiples navegadores.

Primero, necesitarás instalar Playwright e inicializarlo en tu proyecto. El proceso de configuración crea archivos de configuración e instala binarios de navegadores:

```bash
npm init playwright@latest
```

La configuración de Playwright te permite especificar contra qué navegadores realizar las pruebas, establecer URLs base y configurar la ejecución paralela de pruebas. Aquí hay una configuración práctica para una aplicación de React:

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
  ],
  webServer: {
    command: 'npm run dev', // Normally you should prefer to run this as prod (npm run build && npm run start)
    url: process.env.PLAYWRIGHT_BASE_URL || 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

Esta configuración inicia automáticamente tu servidor de desarrollo antes de ejecutar las pruebas y lo cierra después. En entornos de CI, ejecuta pruebas con reintentos para manejar pruebas inestables y captura trazas para depurar fallos.

Ahora escribamos pruebas end-to-end para nuestro administrador de tareas. Dado que nuestra implementación actual almacena el estado en la memoria con `useState`, los datos no persistirán a través de las recargas de página. Estas pruebas verifican la experiencia del usuario dentro de una sola sesión: crear tareas, marcarlas como completadas y eliminarlas. Si tu aplicación persiste datos en una API o base de datos, agregarías pruebas que recarguen la página y verifiquen que los datos sobrevivan:

```typescript
// e2e/task-manager.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Task Manager E2E', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/');
  });

  test('displays the application with correct title', async ({ page }) => {
    await expect(page.locator('h1')).toHaveText('Task Manager');
    await expect(page.getByText('0 of 0 tasks completed')).toBeVisible();
  });

  test('shows empty state message initially', async ({ page }) => {
    await expect(
      page.getByText('No tasks yet. Add one above to get started!')
    ).toBeVisible();
  });

  test('creates a new task through the form', async ({ page }) => {
    await page.getByLabel('Task title').fill('Write E2E tests');
    await page.getByRole('button', { name: 'Add Task' }).click();
    await expect(page.getByText('Write E2E tests')).toBeVisible();
    await expect(page.getByText('0 of 1 tasks completed')).toBeVisible();
    await expect(page.getByLabel('Task title')).toHaveValue('');
  });

  test('validates form input and shows errors', async ({ page }) => {
    await page.getByRole('button', { name: 'Add Task' }).click();
    await expect(page.getByRole('alert')).toHaveText(
      'Task title cannot be empty'
    );
    await page.getByLabel('Task title').fill('ab');
    await page.getByRole('button', { name: 'Add Task' }).click();
    await expect(page.getByRole('alert')).toHaveText(
      'Task title must be at least 3 characters'
    );
  });

  test('completes a full task lifecycle', async ({ page }) => {
    await page.getByLabel('Task title').fill('Buy groceries');
    await page.getByRole('button', { name: 'Add Task' }).click();
    await page.getByLabel('Task title').fill('Do laundry');
    await page.getByRole('button', { name: 'Add Task' }).click();
    await expect(page.getByText('0 of 2 tasks completed')).toBeVisible();

    const groceriesCheckbox = page.getByRole('checkbox', {
      name: /toggle buy groceries/i,
    });
    await groceriesCheckbox.check();
    await expect(groceriesCheckbox).toBeChecked();
    await expect(page.getByText('1 of 2 tasks completed')).toBeVisible();

    const laundryCheckbox = page.getByRole('checkbox', {
      name: /toggle do laundry/i,
    });
    await laundryCheckbox.check();
    await expect(page.getByText('2 of 2 tasks completed')).toBeVisible();

    const deleteButton = page.getByRole('button', {
      name: /delete buy groceries/i,
    });
    await deleteButton.click();
    await expect(page.getByText('Buy groceries')).not.toBeVisible();
    await expect(page.getByText('1 of 1 tasks completed')).toBeVisible();
  });
});
```

La API de Playwright refleja cómo los usuarios piensan al interactuar con las páginas web. Haces clic en botones, completas formularios y realizas afirmaciones sobre lo que está visible. El framework se encarga de esperar a que aparezcan los elementos, se completen las animaciones y finalicen las solicitudes de red. Sin embargo, los usuarios reales interactúan con las aplicaciones de formas que van más allá de hacer clic en botones en secuencia: utilizan atajos de teclado, actúan rápidamente sin esperar animaciones y esperan retroalimentación visual para confirmar sus acciones. Las siguientes pruebas cubren cuatro escenarios que reflejan esta realidad: navegación por teclado (verificando el envío con la tecla Enter), retroalimentación visual (confirmando el estilo tachado en las tareas completadas), creación rápida de tareas (asegurando que las entradas sucesivas rápidas no pierdan datos) y consistencia del estado durante interacciones rápidas (detectando condiciones de carrera donde la interfaz de usuario podría desincronizarse con el estado subyacente):

```typescript
// e2e/task-manager-advanced.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Task Manager - Advanced Interactions', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/')
  })

  test('submits form using Enter key', async ({ page }) => {
    await page.getByLabel('Task title').fill('Test keyboard submission')
    await page.getByLabel('Task title').press('Enter')
    await expect(page.getByText('Test keyboard submission')).toBeVisible()
  })

  test('visually distinguishes completed tasks', async ({ page }) => {
    await page.getByLabel('Task title').fill('Style test task')
    await page.getByRole('button', { name: 'Add Task' }).click()
    const checkbox = page.getByRole('checkbox', { name: /toggle style test task/i })
    await checkbox.check()

    // Test the user-visible result, not the implementation
    // Screenshot comparison catches any visual change (strikethrough, opacity, color)
    await expect(page.getByTestId('task-list')).toHaveScreenshot('completed-task.png')

    // Or verify accessibility: completed tasks should be marked
    await expect(checkbox).toBeChecked()
    await expect(page.getByText('1 of 1 tasks completed')).toBeVisible()
  })

  test('creates multiple tasks sequentially', async ({ page }) => {
    // Renamed: this is sequential, not "rapid"
    const taskTitles = [
      'First task',
      'Second task',
      'Third task',
    ]

    for (const title of taskTitles) {
      await page.getByLabel('Task title').fill(title)
      await page.getByRole('button', { name: 'Add Task' }).click()
    }

    for (const title of taskTitles) {
      await expect(page.getByText(title)).toBeVisible()
    }

    await expect(page.getByText('0 of 3 tasks completed')).toBeVisible()
  })

  test('handles rapid checkbox toggling', async ({ page }) => {
    await page.getByLabel('Task title').fill('Quick task')
    await page.getByRole('button', { name: 'Add Task' }).click()
    const checkbox = page.getByRole('checkbox', { name: /toggle quick task/i })

    // Rapid clicks without waiting for state to settle
    // Tests that the UI doesn't break under fast interaction
    await checkbox.click()
    await checkbox.click()
    await checkbox.click()

    // Final state should be checked (odd number of clicks)
    await expect(checkbox).toBeChecked()
    await expect(page.getByText('1 of 1 tasks completed')).toBeVisible()
  })
})

// For actual race condition testing with API calls,
// you'd test at the API level, not through the UI:
//
// test('handles concurrent API requests', async ({ request }) => {
//   const createTask = (title: string) =>
//     request.post('/api/tasks', { data: { title } })
//
//   // Fire requests in parallel
//   const responses = await Promise.all([
//     createTask('Task A'),
//     createTask('Task B'),
//     createTask('Task C'),
//   ])
//
//   // All should succeed without conflicts
//   responses.forEach(res => expect(res.ok()).toBe(true))
//
//   // Verify all were created
//   const list = await request.get('/api/tasks')
//   const tasks = await list.json()
//   expect(tasks).toHaveLength(3)
// })
```

Playwright se destaca en probar el comportamiento responsivo y la compatibilidad entre navegadores. Puedes simular diferentes tamaños de ventana gráfica (*viewport*) y probar en Chrome, Firefox y Safari simultáneamente. El framework captura capturas de pantalla y trazas en caso de fallo, lo que facilita el diagnóstico de problemas que solo ocurren en navegadores o tamaños de pantalla específicos.

La mayoría de las aplicaciones reales requieren autenticación, e iniciar sesión antes de cada prueba desperdicia tiempo y agrega inestabilidad (*flakiness*). Playwright resuelve esto con `storageState`: te autenticas una vez, guardas la sesión y la reutilizas en todas las pruebas:

```typescript
// e2e/auth.setup.ts
import { test as setup, expect } from '@playwright/test'

// This runs once before all tests
setup('authenticate', async ({ page }) => {
  await page.goto('/login')
  await page.getByLabel('Email').fill('test@example.com')
  await page.getByLabel('Password').fill('password123')
  await page.getByRole('button', { name: 'Sign in' }).click()

  // Wait for authentication to complete
  await expect(page.getByText('Welcome back')).toBeVisible()

  // Save signed-in state to a file
  await page.context().storageState({ path: './e2e/.auth/user.json' })
})
```

Necesitarás actualizar tu `playwright.config.ts`:

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
  projects: [
    // Setup runs first—authenticates and saves session
    { name: 'setup', testMatch: /.*\.setup\.ts/ },

    // Desktop browsers
    {
      name: 'chromium',
      use: {
        ...devices['Desktop Chrome'],
        storageState: './e2e/.auth/user.json',
      },
      dependencies: ['setup'],
    },
    {
      name: 'firefox',
      use: {
        ...devices['Desktop Firefox'],
        storageState: './e2e/.auth/user.json',
      },
      dependencies: ['setup'],
    },
    {
      name: 'webkit',
      use: {
        ...devices['Desktop Safari'],
        storageState: './e2e/.auth/user.json',
      },
      dependencies: ['setup'],
    },

    // Mobile devices
    {
      name: 'mobile-chrome',
      use: {
        ...devices['Pixel 5'],
        storageState: './e2e/.auth/user.json',
      },
      dependencies: ['setup'],
    },
    {
      name: 'mobile-safari',
      use: {
        ...devices['iPhone 13'],
        storageState: './e2e/.auth/user.json',
      },
      dependencies: ['setup'],
    },

    // Tablet
    {
      name: 'tablet',
      use: {
        ...devices['iPad Pro 11'],
        storageState: './e2e/.auth/user.json',
      },
      dependencies: ['setup'],
    },
  ],
  webServer: {
    command: 'npm run build && npm run start',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI
  }
})
```

Luego, así es como lo utilizas:

```typescript
// e2e/dashboardWithResponsiveDesign.spec.ts
import { test, expect } from '@playwright/test'

// No login needed—session is already authenticated
test('shows user dashboard', async ({ page }) => {
  await page.goto('/dashboard')
  await expect(page.getByRole('heading', { name: 'Dashboard' })).toBeVisible()
})

test.describe('Responsive Design', () => {
  test('shows mobile menu on small screens', async ({ page, isMobile }) => {
    await page.goto('/')

    if (isMobile) {
      // Mobile: hamburger menu should be visible
      await expect(page.getByRole('button', { name: 'Open menu' })).toBeVisible()
      await expect(page.getByRole('navigation')).not.toBeVisible()

      // Open menu
      await page.getByRole('button', { name: 'Open menu' }).click()
      await expect(page.getByRole('navigation')).toBeVisible()
    } else {
      // Desktop: navigation should be visible without menu button
      await expect(page.getByRole('navigation')).toBeVisible()
      await expect(page.getByRole('button', { name: 'Open menu' })).not.toBeVisible()
    }
  })

  test('adjusts grid layout for screen size', async ({ page, viewport }) => {
    await page.goto('/products')

    // Screenshot comparison catches layout differences across viewports
    // Each device project generates its own baseline screenshot
    await expect(page.getByTestId('product-grid')).toHaveScreenshot('product-grid.png')
  })

  test('touch interactions work on mobile', async ({ page, isMobile }) => {
    test.skip(!isMobile, 'Touch-specific test')
    await page.goto('/gallery')

    // Test swipe gesture
    const gallery = page.getByTestId('image-gallery')
    await gallery.dragTo(gallery, {
      sourcePosition: { x: 300, y: 200 },
      targetPosition: { x: 50, y: 200 },
    })

    await expect(page.getByText('Image 2 of 10')).toBeVisible()
  })
})
```

El archivo `storageState` contiene cookies y datos de `localStorage` de la sesión autenticada. Agrega `.auth/` a tu `.gitignore`; estos archivos contienen tokens de sesión. Para aplicaciones con múltiples roles de usuario (administrador, usuario regular), crea archivos de configuración y estados de almacenamiento separados para cada rol.

El fixture `isMobile` es automáticamente `true` para proyectos de dispositivos móviles, lo que te permite escribir aserciones condicionales. Los descriptores de dispositivos de Playwright incluyen tamaño de viewport, agente de usuario (*user agent*), soporte táctil y factor de escala del dispositivo, coincidiendo con el comportamiento del dispositivo real de manera más precisa que simplemente cambiar el ancho del viewport.

---

## Automatización de pruebas en pipelines de CI/CD con GitHub Actions

Escribir pruebas es solo la mitad de la batalla. Para beneficiarse verdaderamente de las pruebas automatizadas, tus pruebas deben ejecutarse automáticamente cada vez que cambia el código. GitHub Actions proporciona una potente plataforma para la integración continua, ejecutando toda tu suite de pruebas en cada pull request y push a las ramas principales.

Creemos un workflow integral de CI que ejecute pruebas unitarias, pruebas de integración y pruebas E2E en paralelo, maximizando la velocidad de retroalimentación. Una advertencia sobre las métricas de cobertura: una base de código con un 100% de cobertura de líneas aún puede tener errores. La cobertura mide qué líneas se ejecutaron durante las pruebas, no si esas pruebas hacen aserciones significativas o cubren todas las combinaciones de entrada posibles. Puedes lograr una cobertura completa con pruebas que nunca afirmen nada, o pasar por alto casos extremos críticos por completo. Trata la cobertura como una herramienta para encontrar código no probado, no como prueba de que el código probado funciona correctamente:

```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  unit-and-integration:
    name: Unit & Integration Tests
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run unit and integration tests
        run: npm test -- --coverage --watchAll=false

      - name: Upload coverage reports
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/coverage-final.json
          flags: unittests
          name: codecov-umbrella

  e2e-tests:
    name: E2E Tests - ${{ matrix.browser }}
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        browser: [chromium, firefox, webkit]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Cache Playwright browsers
        uses: actions/cache@v4
        with:
          path: ~/.cache/ms-playwright
          key: playwright-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}

      - name: Install Playwright browsers
        run: npx playwright install --with-deps ${{ matrix.browser }}

      - name: Build application
        run: npm run build

      - name: Run Playwright tests
        run: npx playwright test --project=${{ matrix.browser }}

      - name: Upload Playwright report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report-${{ matrix.browser }}
          path: playwright-report/
          retention-days: 7
```

Esta configuración de workflow ejecuta pruebas unitarias y pruebas E2E en paralelo, reduciendo la duración total del pipeline. Sin embargo, la ejecución en paralelo no siempre es óptima para la velocidad de retroalimentación. Si las pruebas unitarias fallan en dos minutos, ya has identificado el problema; no hay necesidad de esperar a que termine una suite E2E de diez minutos. Una estrategia secuencial (unitaria → integración → E2E) que falle rápido (*fail-fast*) puede mostrar los problemas antes y conservar los recursos de CI. La elección depende de tus prioridades: la ejecución paralela minimiza el tiempo total cuando las pruebas pasan, mientras que la ejecución secuencial minimiza el tiempo hasta el fallo cuando no pasan.

Ten en cuenta que `fail-fast: false` se establece en la configuración de la matriz, lo que significa que las tres pruebas de navegador se ejecutan hasta completarse incluso si una falla. Esto asegura que detectes problemas en todos los navegadores en una sola ejecución, pero también consume recursos en pruebas que pueden ser irrelevantes si solo estás intentando solucionar el primer fallo. Considera configurar `fail-fast: true` durante el desarrollo activo y deshabilitarlo solo para pruebas integrales previas al lanzamiento.

Cuando las pruebas fallan, Playwright genera informes HTML detallados y el paso `actions/upload-artifact` los sube a GitHub. El acceso a estos informes requiere descargar el artefacto, extraerlo y abrir el archivo HTML localmente, lo cual es funcional pero no precisamente ágil. Para los equipos que necesitan una depuración más rápida, considera la integración con el visor de trazas de Playwright o un servicio de informes de terceros que proporcione enlaces directos a los detalles del fallo.

Mejoremos nuestro workflow con comprobaciones de calidad adicionales y optimizaciones:

```yaml
# .github/workflows/quality.yml
name: Code Quality

# Trigger on pushes and pull requests to main branches
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  lint:
    name: Lint Code
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      # Cache is set to 'npm' which caches ~/.npm (the download cache)
      # This speeds up npm ci but still performs a clean install
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      # npm ci is preferred over npm install in CI environments
      # It's faster and ensures reproducible builds from package-lock.json
      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint

      # Runs tsc --noEmit to catch type errors without generating output files
      # Catches errors that ESLint might miss (type mismatches, missing imports)
      - name: Check TypeScript types
        run: npm run type-check

  # Runs in parallel with lint job since there's no dependency between them
  # Consider adding 'needs: lint' if you want sequential execution to fail faster
  test-coverage:
    name: Test Coverage Check
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      # --watchAll=false ensures tests run once and exit (required for CI)
      # Coverage output goes to /coverage directory by default
      - name: Run tests with coverage
        run: npm test -- --coverage --watchAll=false

      # Manual threshold check using shell commands
      # Alternative: configure coverageThreshold in vitest.config.ts or jest.config.js
      # to fail automatically when coverage drops below threshold
      - name: Check coverage thresholds
        run: |
          COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage is below 80%: $COVERAGE%"
            exit 1
          fi
          echo "Coverage is acceptable: $COVERAGE%"
```

### Comprobación de cobertura

El enfoque del script de shell que utiliza `jq`, `bc` y `cat` para analizar el JSON de cobertura es frágil. Puede romperse si el formato JSON cambia entre versiones, si `jq` no está instalado en el runner o si `bc` se comporta de manera diferente en distintos entornos. El umbral del 80% también está codificado de forma fija en el YAML, enterrado en la configuración de CI en lugar de residir con tu base de código. Un enfoque más robusto es configurar los umbrales de cobertura directamente en `vitest.config.ts` (o `jest.config.js`), donde `npm test -- --coverage` falla automáticamente cuando la cobertura cae por debajo de los límites definidos. Esto mantiene los umbrales bajo control de versiones junto con tu código y asegura un comportamiento coherente en el desarrollo local y en los entornos de CI.

Más allá de ejecutar pruebas, este workflow hace cumplir comprobaciones de calidad de código de referencia: linting para la coherencia de estilo, compilación de TypeScript para la seguridad de tipos y umbrales de cobertura para marcar el código no probado:

```typescript
// RECOMMENDED: Configure thresholds in vitest.config.ts instead:
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      thresholds: {
        lines: 80,
        branches: 80,
        functions: 80,
        statements: 80,
      }
    }
  }
})
```

Estas son prácticas fundamentales para cualquier proyecto moderno, no estándares excepcionales. Ten en cuenta que la retroalimentación de CI se mide en minutos, no en segundos; para cuando ves un fallo, es probable que ya hayas cambiado de contexto a otra cosa. Para una retroalimentación verdaderamente rápida, ejecuta `npm run lint` y `npm run type-check` localmente antes de hacer push, o configura tu editor para que muestre estos problemas a medida que escribes. El pipeline de CI sirve como una red de seguridad, no como un mecanismo de retroalimentación principal. También ten en cuenta que este workflow ejecuta trabajos en paralelo sin dependencias `needs`, lo que significa que un fallo de linting no detendrá la ejecución del trabajo de prueba. Si prefieres conservar los recursos de CI y fallar antes, agrega `needs: lint` al trabajo de prueba para que solo se ejecute después de que pase el linting.

Para equipos más grandes o proyectos donde los entornos de staging son importantes, los despliegues de vista previa (*deployment previews*) agregan otra capa de validación. La idea es sencilla: desplegar cada pull request en un entorno temporal, ejecutar tus pruebas E2E contra él y dar a los revisores una URL en vivo para verificar los cambios antes de fusionar. Esto detecta problemas específicos del entorno que no aparecen al probar localmente, como variables de entorno mal configuradas, comportamientos de CDN, tiempos de espera de funciones serverless o integraciones de terceros que se comportan de manera diferente en producción.

Este enfoque tiene ventajas y desventajas. Cada despliegue de vista previa consume minutos de compilación y recursos de hosting, y el workflow añade varios minutos a tu ciclo de retroalimentación de PR. Es más valioso cuando tu aplicación depende en gran medida de una infraestructura que es difícil de replicar localmente, o cuando las partes interesadas necesitan revisar los cambios en un entorno realista. Para equipos más pequeños o aplicaciones más simples, ejecutar pruebas E2E contra una compilación local durante la CI puede ser suficiente.

El siguiente workflow se despliega en Vercel, pero el patrón se aplica a cualquier servicio de hosting con vistas previas (Netlify, AWS Amplify, Cloudflare Pages). Requiere almacenar las credenciales de despliegue como secretos del repositorio:

```yaml
# .github/workflows/deploy-preview.yml
name: Deploy Preview

on:
  pull_request:
    branches: [main]

jobs:
  deploy-preview:
    name: Deploy Preview
    runs-on: ubuntu-latest

    # Only run on non-draft PRs to save resources during active development
    if: github.event.pull_request.draft == false

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build application
        run: npm run build

      - name: Deploy to Vercel
        id: deploy
        uses: amondnet/vercel-action@v20
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          scope: ${{ secrets.VERCEL_ORG_ID }}

      # Simple health check to verify deployment succeeded
      - name: Verify deployment
        run: curl --fail --retry 3 --retry-delay 5 ${{ steps.deploy.outputs.preview-url }}

      - name: Comment preview URL on PR
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `Preview deployed to: ${{ steps.deploy.outputs.preview-url }}`
            })
```

Este workflow crea un despliegue de vista previa para cada pull request, brindando a los revisores una URL en vivo para verificar los cambios en un entorno similar a producción.

Hemos omitido intencionalmente la ejecución de pruebas E2E contra la vista previa, ya que ya se ejecutan en el workflow de pruebas principal; duplicarlas solo aumentaría los costos de CI sin detectar errores adicionales.

La automatización de pruebas realmente transforma la forma en que los equipos entregan software, pero los beneficios dependen enteramente de cómo escribes tus pruebas. Los ejemplos de este capítulo utilizaron consultas como `getByText('Buy groceries')` y `getByRole('button', { name: 'Add Task' })` para mayor claridad, pero estos selectores son frágiles en la práctica. Cambia la etiqueta de un botón de "Add Task" a "Create Task" y cada prueba que haga referencia a ella se romperá. Multiplica esto en una base de código grande y la refactorización se convierte en una carga en lugar de un impulso de confianza; los desarrolladores evitan los cambios porque no quieren pasar medio día arreglando pruebas rotas.

Para escribir pruebas que realmente permitan una refactorización confiable, prefiere selectores que estén desacoplados del texto visible para el usuario: atributos `data-testid` para la selección estable de elementos, o consultas basadas en roles con comparadores flexibles. Estructura tus pruebas en torno a los flujos de trabajo del usuario en lugar de los detalles de implementación, y mantén la disposición de eliminar las pruebas cuyo mantenimiento cueste más que los errores que previenen. Una suite más pequeña de pruebas resistentes supera a una cobertura integral en la que nadie confía.

A medida que construyes tu infraestructura de pruebas, recuerda que las pruebas también son código. Necesitan mantenimiento, refactorización y un diseño cuidadoso. Escribe pruebas que documenten el comportamiento de tu aplicación, detecten errores reales y te den confianza para avanzar rápidamente. Cuando tu suite de pruebas se vuelva confiable y completa, te preguntarás cómo pudiste entregar código sin ella.

El viaje desde escribir tu primera prueba hasta construir una cultura de pruebas sólida lleva tiempo, pero cada prueba que escribes hace que tu aplicación sea más estable y tu proceso de desarrollo sea más agradable. Tu futuro yo, tus compañeros de equipo y tus usuarios te agradecerán por el tiempo invertido en las pruebas automatizadas.

---

## Resumen

En este capítulo, aprendiste a construir una estrategia de pruebas integral para aplicaciones de React mediante el desarrollo de un administrador de tareas completo con cobertura de pruebas total. Comenzando con los fundamentos de las pruebas unitarias utilizando Jest y React Testing Library, descubriste cómo escribir pruebas que se centran en el comportamiento del usuario en lugar de en los detalles de implementación, asegurando que tus componentes funcionen de la manera en que los usuarios reales interactúan con ellos. Luego, el capítulo avanzó hacia las pruebas de integración, donde verificaste que múltiples componentes funcionen juntos correctamente, gestionando las actualizaciones de estado y los flujos de trabajo de los usuarios a través de funcionalidades completas en lugar de piezas aisladas.

En este capítulo, también dominaste las pruebas end-to-end con Playwright, aprendiendo a simular interacciones reales de usuarios en los navegadores Chrome, Firefox y Safari. Finalmente, automatizaste toda tu suite de pruebas utilizando GitHub Actions, creando pipelines de CI/CD que ejecutan pruebas en cada cambio de código, hacen cumplir los estándares de calidad y despliegan entornos de vista previa para las pull requests. Al completar este capítulo, adquiriste las habilidades necesarias para construir una infraestructura de pruebas robusta que brinda confianza para refactorizar código, entregar funcionalidades más rápido y mantener aplicaciones de alta calidad a lo largo de su ciclo de vida.

En el próximo capítulo, ampliaremos las bases de CI/CD introducidas aquí. Mientras que este capítulo se centró en ejecutar pruebas automáticamente, el [Capítulo 15](https://subscription.packtpub.com/book/web-development/9781806108251/15) profundiza en el lado de la entrega: compilar aplicaciones para diferentes entornos, monitorizar tamaños de paquetes, gestionar secretos y variables de entorno, implementar estrategias de despliegue como lanzamientos blue-green y canary, y configurar mecanismos de reversión (*rollback*) cuando las cosas salen mal. Aprenderás a construir pipelines que lleven tu código desde el commit hasta producción con confianza.
