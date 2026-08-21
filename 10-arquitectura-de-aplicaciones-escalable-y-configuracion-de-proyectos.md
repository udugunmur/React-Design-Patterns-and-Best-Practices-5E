# Capítulo 10: Arquitectura de Aplicaciones Escalable y Configuración de Proyectos

Construir una aplicación React es fácil. ¿Construir una que escale con elegancia a medida que tu equipo y tu código crecen? Ese es un desafío completamente diferente. Las decisiones que tomas al inicio de un proyecto (cómo estructuras tus carpetas, gestionas las dependencias y aplicas la calidad del código) potenciarán a tu equipo o lo atormentarán durante los años venideros.

Este capítulo no trata de seguir reglas rígidas ni de copiar la estructura de carpetas de otra persona. Trata de comprender los principios detrás de una arquitectura escalable y tomar decisiones informadas que se adapten a las necesidades de tu proyecto. Exploraremos cómo los equipos exitosos organizan sus aplicaciones React, compararemos diferentes estrategias para gestionar bases de código grandes y configuraremos las herramientas que mantienen tu código consistente y mantenible.

En este capítulo, cubriremos los siguientes temas:

- Estructuración de proyectos React para escalabilidad y mantenibilidad
- Monorepo vs. multirepo: elección de la estrategia adecuada
- Mejores prácticas para organizar componentes, archivos y assets
- Aplicación de calidad de código: linting, formateo y pre-commit hooks
- Optimización de pipelines de compilación, despliegue y CI/CD

---

## Estructuración de proyectos React para escalabilidad y mantenibilidad

Cuando creas un nuevo proyecto de React con Next o Vite, obtienes una estructura mínima: unas pocas carpetas, algunos archivos de configuración y un solo componente. Esto funciona perfectamente para tutoriales y pequeños experimentos. Pero a medida que tu aplicación crece, esa estructura plana se convierte en una desventaja. Los archivos se acumulan, las dependencias se enredan y encontrar cualquier cosa toma más tiempo del debido.

El objetivo de una buena estructura de proyecto no es la complejidad, sino la claridad. Cuando un nuevo desarrollador se une a tu equipo, debería poder navegar por el código de forma intuitiva. Cuando necesites agregar una funcionalidad, deberías saber exactamente a dónde pertenece. Exploremos los patrones comunes que hacen esto posible.

### Estructuras de proyectos comunes y cuándo utilizarlas

No existe una única forma correcta de estructurar una aplicación React, pero han surgido ciertos patrones especialmente efectivos. La clave es hacer coincidir la estructura con la complejidad de tu proyecto y el flujo de trabajo de tu equipo. Veamos cada estructura:

**La estructura basada en funcionalidades (*Feature-Based Structure*)** agrupa todo lo relacionado con una funcionalidad específica en un solo lugar. En lugar de separar componentes, hooks y estilos en diferentes carpetas de nivel superior, los mantienes juntos. Este enfoque destaca en aplicaciones medianas o grandes donde las funcionalidades son relativamente independientes:

```
src/
│   ├── features/
│   ├── authentication/
│   │   ├── components/
│   │   │   ├── LoginForm.tsx
│   │   │   └── SignupForm.tsx
│   │   ├── hooks/
│   │   │   └── useAuth.ts
│   │   ├── api/
│   │   │   └── authApi.ts
│   │   └── types.ts
│   ├── dashboard/
│   │   ├── components/
│   │   ├── hooks/
│   │   └── utils/
│   └── profile/
├── shared/
│   ├── components/
│   ├── hooks/
│   └── utils/
└── App.tsx
```

Esta estructura facilita entender qué hace tu aplicación de un vistazo. Cada funcionalidad es autónoma, lo que reduce la carga cognitiva cuando estás trabajando en algo específico. También hace que la división de código (*code splitting*) sea más natural; puedes cargar perezosamente (*lazy-load*) funcionalidades completas con un esfuerzo mínimo.

**La estructura basada en capas (*Layer-Based Structure*)** organiza el código por responsabilidad técnica en lugar de por funcionalidad del negocio. Encontrarás esto en muchas bases de código establecidas, especialmente aquellas que comenzaron pequeñas y crecieron orgánicamente:

```
src/
├── components/
│   ├── auth/
│   ├── dashboard/
│   └── common/
├── hooks/
├── services/
├── store/
├── types/
├── utils/
└── App.tsx
```

Este enfoque funciona bien para aplicaciones más pequeñas donde la sobrecarga de carpetas de funcionalidades parece innecesaria. También resulta familiar: la mayoría de los desarrolladores han trabajado antes con una estructura como esta. Sin embargo, a medida que tu aplicación crece, notarás que el código relacionado queda disperso. Trabajar en la funcionalidad de autenticación significa tocar archivos en múltiples carpetas de nivel superior.

**El enfoque híbrido (*Hybrid Approach*)** combina lo mejor de ambos mundos. Organizas las funcionalidades principales por dominio, pero mantienes capas técnicas compartidas para el código verdaderamente reutilizable:

```
src/
├── features/
│   ├── authentication/
│   └── dashboard/
├── components/
│   ├── ui/          // Button, Input, Modal
│   └── layout/      // Header, Sidebar, Footer
├── hooks/
├── lib/             // Third-party integrations and utils
└── App.tsx
```

Esta estructura reconoce que cierto código es inherentemente compartido. Tu librería de componentes de interfaz de usuario, las funciones de utilidad y los hooks comunes no pertenecen a ninguna funcionalidad individual. Mantenerlos separados hace que sean más fáciles de mantener y versionar.

### Modularización y separación de código para una mejor mantenibilidad

Una buena estructura de carpetas es solo el comienzo. La verdadera escalabilidad proviene de cómo modularizas tu código: los límites que trazas entre las diferentes partes de tu aplicación y cómo se comunican.

Considera una aplicación típica de comercio electrónico. Puedes tener funcionalidades para navegación de productos, gestión de carritos, proceso de pago y cuentas de usuario. Cada funcionalidad debe ser relativamente independiente, con interfaces claras para la interacción. Cuando la funcionalidad del carrito necesita información sobre el usuario, no debe importar directamente desde la funcionalidad de autenticación. En su lugar, debe recibir esa información a través de props o un contexto bien definido.

Construyamos un ejemplo real. Así es como podrías estructurar una funcionalidad de productos con los límites adecuados:

```typescript
// features/products/types.ts
export interface Product {
  id: string;
  name: string;
  price: number;
  imageUrl: string;
  stock: number;
}

export interface ProductFilters {
  category?: string;
  minPrice?: number;
  maxPrice?: number;
  inStock?: boolean;
}
// features/products/api/productsApi.ts
import type { Product, ProductFilters } from '../types';

export const productsApi = {
  async getProducts(filters?: ProductFilters): Promise<Product[]> {
    const params = new URLSearchParams();
    if (filters?.category) params.append('category', filters.category);
    if (filters?.minPrice) params.append('minPrice', String(filters.minPrice));
    if (filters?.maxPrice) params.append('maxPrice', String(filters.maxPrice));
    if (filters?.inStock) params.append('inStock', 'true');

    const response = await fetch(`/api/products?${params}`);
    if (!response.ok) throw new Error('Failed to fetch products');
    return response.json();
  },

  async getProduct(id: string): Promise<Product> {
    const response = await fetch(`/api/products/${id}`);
    if (!response.ok) throw new Error('Failed to fetch product');
    return response.json();
  },

  async updateStock(id: string, quantity: number): Promise<void> {
    const response = await fetch(`/api/products/${id}/stock`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ quantity })
    });
    if (!response.ok) throw new Error('Failed to update stock');
  }
};
```

Observa cómo la capa de API está completamente separada de la UI. Expone una interfaz limpia que otras partes de la aplicación pueden usar sin saber nada sobre `fetch` o los detalles de manejo de errores. Esta separación facilita las pruebas y te permite cambiar implementaciones sin tocar los componentes.

Ahora veamos cómo un hook puede encapsular la lógica para trabajar con productos:

```typescript
// features/products/hooks/useProducts.ts
import { useState, useEffect } from 'react';
import { productsApi } from '../api/productsApi';
import type { Product, ProductFilters } from '../types';

interface UseProductsResult {
  products: Product[];
  loading: boolean;
  error: string | null;
  refetch: () => void;
}

export function useProducts(filters?: ProductFilters): UseProductsResult {
  const [products, setProducts] = useState<Product[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const fetchProducts = async () => {
    setLoading(true);
    setError(null);
    try {
      const data = await productsApi.getProducts(filters);
      setProducts(data);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'An error occurred');
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchProducts();
  }, [JSON.stringify(filters)]);

  return { products, loading, error, refetch: fetchProducts };
}
```

Este hook proporciona una interfaz limpia para los componentes. No necesitan saber sobre llamadas a la API, estados de carga o manejo de errores; todo eso está encapsulado. El componente simplemente llama a `useProducts()` y trabaja con los datos devueltos.

Aquí hay un componente que usa este hook:

```tsx
// features/products/components/ProductList.tsx
import { useProducts } from '../hooks/useProducts';
import type { ProductFilters } from '../types';

interface ProductListProps {
  filters?: ProductFilters;
  onSelectProduct: (id: string) => void;
}

export function ProductList({ filters, onSelectProduct }: ProductListProps) {
  const { products, loading, error, refetch } = useProducts(filters);

  if (loading) {
    return (
      <div className="flex justify-center items-center min-h-[400px]">
        <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-blue-600" />
      </div>
    );
  }

  if (error) {
    return (
      <div className="bg-red-50 border border-red-200 rounded-lg p-4">
        <p className="text-red-800 mb-2">{error}</p>
        <button
          onClick={refetch}
          className="text-sm text-red-600 hover:text-red-800 underline"
        >
          Try again
        </button>
      </div>
    );
  }

  return (
    <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
      {products.map((product) => (
        <button
          key={product.id}
          onClick={() => onSelectProduct(product.id)}
  className="bg-white rounded-lg shadow-sm hover:shadow-md transition-shadow p-4 text-left"
        >
          <img
            src={product.imageUrl}
            alt={product.name}
            className="w-full h-48 object-cover rounded mb-4"
          />
          <h3 className="font-semibold text-lg mb-2">{product.name}</h3>
          <div className="flex justify-between items-center">
            <span className="text-xl font-bold text-blue-600">
              ${product.price.toFixed(2)}
            </span>
            <span className={`text-sm ${product.stock > 0 ? 'text-green-600' : 'text-red-600'}`}>
              {product.stock > 0 ? `${product.stock} in stock` : 'Out of stock'}
            </span>
          </div>
        </button>
      ))}
    </div>
  );
}
```

Este componente demuestra la arquitectura basada en funcionalidades al residir en `features/products/components` con su propio hook personalizado (`useProducts`) y tipos, manteniendo toda la lógica relacionada con productos coubicada (*co-located*). Acepta filtros opcionales y un callback `onSelectProduct`, utilizando el hook para obtener productos y gestionar los estados de carga, error y éxito. El componente maneja tres estados: la carga muestra un mensaje centrado, el error muestra el texto del error con un botón de reintento que llama a `refetch`, y el éxito itera sobre los productos para renderizar tarjetas clickeables que muestran imagen, nombre, precio y estado de stock con estilos condicionales (verde para disponibilidad, rojo para agotado). Esta encapsulación hace que el componente sea autónomo y reutilizable: puedes colocarlo en cualquier lugar de tu aplicación, pasar filtros y un manejador de selección, y gestionará su propia obtención de datos y estados de UI sin acoplarse a los componentes padres.

### Aplicación de una estructura coherente de carpetas y archivos

La coherencia importa más que la estructura específica que elijas. Cuando todos los desarrolladores de tu equipo siguen los mismos patrones, la base de código se vuelve predecible. Sabes dónde encontrar las cosas, dónde agregar nuevo código y cómo nombrar los archivos.

Establece convenciones temprano y documéntalas. Algunos equipos prefieren `PascalCase` para los archivos de componentes y `camelCase` para las utilidades. Otros usan archivos index para crear importaciones limpias. Elijas lo que elijas, hazlo explícito. Aquí tienes un ejemplo de convenciones de nomenclatura claras:

```typescript
// Convention: Components are PascalCase, one component per file
ProductCard.tsx
ProductList.tsx
ProductFilters.tsx

// Convention: Hooks start with 'use' and are camelCase
useProducts.ts
useProductFilters.ts
useCart.ts

// Convention: API modules are camelCase and end with 'Api'
productsApi.ts
cartApi.ts
authApi.ts

// Convention: Types are defined in types.ts files
types.ts

// Convention: Utility functions are camelCase
formatPrice.ts
validateEmail.ts
debounce.ts
```

Los archivos index pueden limpiar significativamente las importaciones. En lugar de importar desde rutas profundamente anidadas, expones una API pública para cada funcionalidad:

```typescript
// features/products/index.ts
export { ProductList } from './components/ProductList';
export { ProductCard } from './components/ProductCard';
export { useProducts } from './hooks/useProducts';
export type { Product, ProductFilters } from './types';

// Now other parts of the app can import cleanly:
import { ProductList, useProducts } from '@/features/products';
```

Esto también te da control sobre lo que es público. Si no exportas algo desde el index, se considera interno de la funcionalidad: un detalle de implementación privado del cual otras funcionalidades no deberían depender.

---

## Monorepo vs. multirepo: elección de la estrategia adecuada

A medida que tu proyecto crece, es posible que te encuentres administrando múltiples aplicaciones relacionadas: tal vez una aplicación web orientada al cliente, un panel de administración y una aplicación móvil, todas compartiendo lógica de negocio común y componentes de interfaz de usuario. La forma en que organices estas bases de código afectará significativamente la velocidad de tu equipo y su capacidad para mantener la coherencia.

### ¿Qué es un monorepo? Beneficios y desafíos

Un monorepo es un repositorio único que contiene múltiples proyectos o paquetes. En lugar de tener repositorios separados para tu aplicación web, tu aplicación móvil y tu librería compartida de componentes, todo vive junto en un solo lugar.

La estructura podría verse así:

```
my-project/
├── packages/
│   ├── frontend/
│   │   ├── src/
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── admin-dashboard/
│   │   ├── src/
│   │   └── package.json
│   ├── ui/
│   │   ├── src/
│   │   └── package.json
│   └── shared-utils/
│       ├── src/
│       └── package.json
├── package.json
└── tsconfig.json
```

El principal beneficio es el **código compartido**. Cuando corriges un error en tu librería de componentes compartidos o agregas una nueva función de utilidad, todas las aplicaciones que dependen de ella pueden usar la versión actualizada de inmediato. No es necesario publicar paquetes, esperar pipelines de CI/CD ni actualizar manualmente las dependencias en varios repositorios.

Los monorepos también imponen la **consistencia**. Cuando todos tus proyectos están en un solo lugar, es más fácil hacer cumplir reglas de linting compartidas, configuraciones de TypeScript y estándares de código. Puedes refactorizar a través de los límites de los proyectos con confianza, sabiendo que tu IDE y el verificador de tipos detectarán cualquier problema.

Pero los monorepos no están exentos de desafíos. El repositorio puede volverse muy grande, lo que hace que los clones iniciales sean lentos. Los tiempos de compilación pueden aumentar si no tienes cuidado con qué paquetes necesitan reconstruirse. Y necesitas herramientas para gestionar las dependencias entre paquetes; no puedes simplemente ejecutar `npm install` para todo y esperar que funcione sin problemas.

### ¿Qué es un multirepo? Beneficios y desafíos

Una estrategia multirepo utiliza repositorios separados para cada proyecto. Tu aplicación web, aplicación móvil y librería de componentes vivirían cada una en su propio repositorio con control de versiones, pipelines de CI/CD y ciclos de lanzamiento independientes:

```
frontend/             (separate repo)
admin-dashboard/      (separate repo)
ui/                   (separate repo, published to npm)
shared-utils/         (separate repo, published to npm)
```

Este enfoque ofrece **límites claros**. Cada equipo puede ser propietario absoluto de su repositorio, estableciendo sus propios procesos y calendarios de lanzamiento. Los repositorios se mantienen más pequeños y enfocados. La publicación de paquetes en npm o en un registro privado crea un versionado explícito, por lo que las aplicaciones pueden elegir cuándo actualizar las dependencias.

La desventaja es la **sobrecarga de coordinación**. Cuando necesitas realizar cambios en múltiples proyectos (actualizar una interfaz de API, por ejemplo), tienes que actualizar múltiples repositorios, a menudo en un orden específico. Podrías publicar una nueva versión de tus utilidades compartidas, luego actualizar la aplicación web para usarla y luego la aplicación móvil. Probar cambios entre proyectos requiere clonar múltiples repositorios y mantenerlos sincronizados.

### Elección del enfoque adecuado para tu equipo y proyecto

No existe una respuesta universal. La elección correcta depende del tamaño de tu equipo, la complejidad del proyecto y las preferencias de flujo de trabajo:

**Elige un monorepo cuando:**
- Tienes múltiples proyectos que comparten una cantidad significativa de código.
- Tu equipo es pequeño o mediano y trabaja en varios proyectos.
- Deseas commits atómicos que abarquen múltiples paquetes.
- Los lanzamientos coordinados entre proyectos son comunes.
- Valoras la coherencia y deseas imponer herramientas compartidas.

**Elige un multirepo cuando:**
- Los proyectos son verdaderamente independientes con un código compartido mínimo.
- Diferentes equipos poseen diferentes proyectos con distintos ciclos de lanzamiento.
- Necesitas límites estrictos y una propiedad clara.
- Tu organización ya cuenta con procesos sólidos para administrar paquetes npm.
- El tamaño del repositorio y el tiempo de clonación son motivo de preocupación.

Muchas empresas exitosas utilizan monorepos. Google guarda la mayor parte de su código en un solo repositorio. Meta utiliza un monorepo para sus productos principales. Pero muchas empresas exitosas también usan multirepos: Netflix, Amazon y muchas startups trabajan con repositorios separados. El patrón importa menos que qué tan bien lo ejecutes. Un monorepo bien administrado con buenas herramientas supera a un multirepo desordenado. Un multirepo limpio con versiones claras supera a un monorepo enredado. Los equipos que tienen más dificultades siguen cambiando de enfoque, esperando que el siguiente solucione sus problemas. El verdadero problema suele ser la disciplina, no la elección en sí. Elige un enfoque, mantenlo, configura las herramientas adecuadas y sigue mejorando. El éxito proviene de la ejecución, no de elegir el patrón perfecto.

### Herramientas para gestionar monorepos: NPM Workspaces vs Turborepo

Si decides optar por un monorepo, necesitarás herramientas para gestionarlo eficazmente. Dos opciones populares son **NPM Workspaces** y **Turborepo**.

**NPM Workspaces** está integrado en npm (yarn y pnpm tienen características similares). Es la forma más sencilla de comenzar con un monorepo porque requiere una configuración mínima.

Aquí tienes una configuración básica:

```json
// Root package.json
{
  "name": "myapp",
  "private": true,
  "workspaces": [
    "packages/*"
  ],
  "scripts": {
    "build": "npm run build --workspaces",
    "test": "npm run test --workspaces",
    "lint": "npm run lint --workspaces"
  },
  "devDependencies": {
    "typescript": "^5.9.3",
    "eslint": "^9.37.0"
  }
}


// packages/ui/package.json
{
  "name": "@myapp/ui",
  "version": "1.0.0",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "test": "jest"
  },
  "devDependencies": {
    "@types/react": "^19.0.0",
    "react": "^19.0.0"
  }
}

// packages/frontend/package.json
{
  "name": "@myapp/frontend",
  "version": "1.0.0",
  "dependencies": {
    "@myapp/ui": "1.0.0",
    "react": "^19.2.7",
    "react-dom": "^19.2.7"
  }
}
```

Cuando ejecutas `npm install` en la raíz, npm crea enlaces simbólicos (*symlinks*) entre tus paquetes. La aplicación web puede importar desde `@myapp/ui` como si estuviera instalado desde npm, pero en realidad apunta al paquete local. Los cambios en la librería de componentes están disponibles de inmediato para la aplicación web.

Esto funciona bien para monorepos sencillos, pero tiene limitaciones. La ejecución de scripts en todos los paquetes se realiza de forma secuencial, lo que puede ser lento. No hay almacenamiento en caché integrado, por lo que es posible que reconstruyas paquetes innecesariamente. La gestión de dependencias puede complicarse cuando los paquetes tienen dependencias superpuestas con diferentes versiones.

**Turborepo** se basa en los conceptos de workspaces con potentes funciones de optimización. Agrega almacenamiento en caché inteligente, ejecución paralela y programación de tareas basada en dependencias.

La configuración de Turborepo requiere solo dos archivos de configuración:

```json
// Root package.json
{
  "name": "myapp",
  "private": true,
  "workspaces": ["packages/*"],
  "scripts": {
    "build": "turbo run build",
    "test": "turbo run test",
    "dev": "turbo run dev --parallel"
  },
  "devDependencies": {
    "turbo": "^1.11.0"
  }
}

// turbo.json
{
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": []
    },
    "lint": {
      "outputs": []
    },
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

La configuración `turbo.json` le indica a Turborepo cómo se relacionan las tareas. La dependencia `^build` significa que antes de compilar este paquete, debe compilar todos los paquetes de los que depende. El campo `outputs` especifica qué archivos produce la tarea, lo que permite a Turborepo almacenar en caché los resultados.

Cuando ejecutas `turbo run build`, Turborepo analiza las dependencias de tus paquetes y las compila en el orden óptimo, ejecutando paquetes independientes en paralelo. Si vuelves a compilar sin cambiar nada, Turborepo utiliza los resultados en caché y finaliza al instante.

Esto se vuelve muy potente a gran escala. Imagina que tienes diez paquetes y cambias código en uno solo. Turborepo reconstruye únicamente ese paquete y los paquetes que dependen de él, dejando todo lo demás en caché. En un servidor de CI, Turborepo puede compartir el caché entre compilaciones, lo que agiliza enormemente las comprobaciones de solicitudes de extracción (*pull requests*).

Turborepo también maneja el **caché remoto**. Todo tu equipo puede compartir artefactos de compilación, de modo que cuando alguien descarga tu rama, no recompila los paquetes que ya compilaste; en su lugar, descarga los resultados en caché.

---

## Mejores prácticas para organizar componentes, archivos y assets

Una arquitectura escalable no se trata solo de la estructura de carpetas y las herramientas de compilación. También se trata de cómo organizas los componentes básicos de tu aplicación: componentes, estilos, gestión de estado y recursos estáticos (*static assets*).

### Estructuración de componentes para reutilización y escalabilidad

La forma en que estructuras y compones los componentes tiene un impacto masivo en la mantenibilidad. Los componentes mal diseñados se vuelven rígidos y difíciles de reutilizar. Los componentes bien diseñados se adaptan a diferentes contextos sin esfuerzo.

Un patrón potente es el **patrón de componentes compuestos (*compound component pattern*)**, donde componentes relacionados trabajan juntos para crear interfaces flexibles y componibles. Considera un componente `Dropdown`:

```tsx
// components/ui/Dropdown/Dropdown.tsx
import { createContext, useContext, useState, ReactNode } from 'react';

interface DropdownContextValue {
  isOpen: boolean;
  toggle: () => void;
  close: () => void;
}

const DropdownContext = createContext<DropdownContextValue | null>(null);

function useDropdownContext() {
  const context = useContext(DropdownContext);
  if (!context) throw new Error('Dropdown components must be used within Dropdown');
  return context;
}

interface DropdownProps {
  children: ReactNode;
}

export function Dropdown({ children }: DropdownProps) {
  const [isOpen, setIsOpen] = useState(false);
  const toggle = () => setIsOpen(prev => !prev);
  const close = () => setIsOpen(false);

  return (
    <DropdownContext.Provider value={{ isOpen, toggle, close }}>
      <div className="relative inline-block">{children}</div>
    </DropdownContext.Provider>
  );
}


interface TriggerProps {
  children: ReactNode;
}

function Trigger({ children }: TriggerProps) {
  const { toggle } = useDropdownContext();
 
  return (
    <button
      onClick={toggle}
      className="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700"
    >
      {children}
    </button>
  );
}

interface MenuProps {
  children: ReactNode;
}

function Menu({ children }: MenuProps) {
  const { isOpen, close } = useDropdownContext();
 
  if (!isOpen) return null;

  return (
    <>
      <div className="fixed inset-0" onClick={close} />
      <div className="absolute top-full mt-2 bg-white rounded-lg shadow-lg border py-1 min-w-[200px] z-10">
        {children}
      </div>
    </>
  );
}

interface ItemProps {
  onClick?: () => void;
  children: ReactNode;
}

function Item({ onClick, children }: ItemProps) {
  const { close } = useDropdownContext();
  const handleClick = () => {
    onClick?.();
    close();
  };
  return (
    <button
      onClick={handleClick}
      className="w-full px-4 py-2 text-left hover:bg-gray-100 transition-colors"
    >
      {children}
    </button>
  );
}

Dropdown.Trigger = Trigger;
Dropdown.Menu = Menu;
Dropdown.Item = Item;
```

Este patrón crea flexibilidad a través de la composición. En lugar de hacer *prop-drilling* de cada opción de configuración posible, les das a los desarrolladores bloques de construcción que funcionan juntos de forma natural:

```tsx
// Using the Dropdown component
function UserMenu() {
  return (
    <Dropdown>
      <Dropdown.Trigger>
        My Account
      </Dropdown.Trigger>
      <Dropdown.Menu>
        <Dropdown.Item onClick={() => navigate('/profile')}>
          Profile
        </Dropdown.Item>
        <Dropdown.Item onClick={() => navigate('/settings')}>
          Settings
        </Dropdown.Item>
        <Dropdown.Item onClick={handleLogout}>
          Logout
        </Dropdown.Item>
      </Dropdown.Menu>
    </Dropdown>
  );
}
```

El componente es simultáneamente potente y simple. Maneja el estado, el posicionamiento y la accesibilidad internamente mientras expone una API limpia y declarativa. Puedes personalizarlo ampliamente sin agregar props; simplemente envuelve los elementos en diferentes diseños o agrega íconos.

Otro patrón crucial es **separar la presentación de la lógica**. Los componentes contenedores (*containers*) manejan los datos y el estado, mientras que los componentes presentacionales se enfocan puramente en el renderizado. Esto es especialmente valioso en el desarrollo de funcionalidades:

```tsx
// features/dashboard/components/DashboardContainer.tsx
import { useState } from 'react';
import { useDashboardData } from '../hooks/useDashboardData';
import { DashboardView } from './DashboardView';

export function DashboardContainer() {
  const [dateRange, setDateRange] = useState('week');
  const { data, loading, error } = useDashboardData(dateRange);

  if (loading) return <LoadingSpinner />;
  if (error) return <ErrorMessage message={error} />;

  return (
    <DashboardView
      data={data}
      dateRange={dateRange}
      onDateRangeChange={setDateRange}
    />
  );
}

// features/dashboard/components/DashboardView.tsx
import type { DashboardData } from '../types';


interface DashboardViewProps {
  data: DashboardData;
  dateRange: string;
  onDateRangeChange: (range: string) => void;
}

export function DashboardView({ data, dateRange, onDateRangeChange }: DashboardViewProps) {
  return (
    <div className="p-6 space-y-6">
      <header className="flex justify-between items-center">
        <h1 className="text-3xl font-bold">Dashboard</h1>
        <select
          value={dateRange}
          onChange={(e) => onDateRangeChange(e.target.value)}
          className="px-4 py-2 border rounded-lg"
        >
          <option value="week">Last Week</option>
          <option value="month">Last Month</option>
          <option value="year">Last Year</option>
        </select>
      </header>

      <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
        <StatCard title="Total Users" value={data.totalUsers} change={data.userGrowth} />
        <StatCard title="Revenue" value={`$${data.revenue}`} change={data.revenueGrowth} />
        <StatCard title="Active" value={data.sessions} change={data.sessionGrowth} />
      </div>
    </div>
  );
}
```

`DashboardView` no tiene idea de cómo se obtienen o gestionan los datos. Recibe props y renderiza la interfaz. Esto hace que sea trivial probarlo, reutilizarlo en diferentes contextos o incluso previsualizarlo en Storybook sin un backend.

### Organización de la lógica de gestión del estado

La gestión del estado es a menudo donde las aplicaciones React se vuelven inmanejables. La lógica se extiende por los componentes, las actualizaciones ocurren de manera inconsistente y la depuración se convierte en arqueología. Una estrategia organizacional clara mantiene el estado predecible y mantenible.

Para el estado global, agrupa el estado y las acciones relacionadas. Ya sea que uses Redux, Zustand o la Context API, agrupa todo lo perteneciente a un dominio:

```typescript
// store/authStore.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface User {
  id: string;
  email: string;
  name: string;
}

interface AuthState {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  updateProfile: (updates: Partial<User>) => void;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set, get) => ({
      user: null,
      token: null,
      isAuthenticated: false,

      login: async (email: string, password: string) => {
        const response = await fetch('/api/auth/login', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ email, password })
        });

        if (!response.ok) throw new Error('Login failed');
       
        const { user, token } = await response.json();
        set({ user, token, isAuthenticated: true });
      },

      logout: () => {
        set({ user: null, token: null, isAuthenticated: false });
      },

      updateProfile: (updates: Partial<User>) => {
        const currentUser = get().user;
        if (!currentUser) return;
        set({ user: { ...currentUser, ...updates } });
      }
    }),
    { name: 'auth-storage' }
  )
);
```

Todo lo relacionado con la autenticación vive en un solo lugar. Los componentes que necesitan el estado de autenticación importan este hook y seleccionan solo lo que necesitan. El store maneja su propia persistencia utilizando el middleware de Zustand, manteniendo esa lógica fuera de los componentes.

Para el estado específico de una funcionalidad que no necesita ser global, mantenlo local a la funcionalidad:

```typescript
// features/cart/store/cartStore.ts
import { create } from 'zustand';

interface CartItem {
  productId: string;
  quantity: number;
  price: number;
}

interface CartState {
  items: CartItem[];
  addItem: (productId: string, quantity: number, price: number) => void;
  removeItem: (productId: string) => void;
  updateQuantity: (productId: string, quantity: number) => void;
  clearCart: () => void;
  total: () => number;
}

export const useCartStore = create<CartState>((set, get) => ({
  items: [],
  addItem: (productId, quantity, price) => {
    set((state) => {
      const existing = state.items.find(item => item.productId === productId);
      if (existing) {
        return {
          items: state.items.map(item =>
            item.productId === productId
              ? { ...item, quantity: item.quantity + quantity }
              : item
          )
        };
      }
      return { items: [...state.items, { productId, quantity, price }] };
    });
  },
  removeItem: (productId) => {
    set((state) => ({
      items: state.items.filter(item => item.productId !== productId)
    }));
  },
  updateQuantity: (productId, quantity) => {
    if (quantity <= 0) {
      get().removeItem(productId);
      return;
    }
    set((state) => ({
      items: state.items.map(item =>
        item.productId === productId ? { ...item, quantity } : item
      )
    }));
  },
  clearCart: () => set({ items: [] }),
  total: () => {
    return get().items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }
}));
```

El store del carrito es autónomo. No depende del estado externo ni realiza llamadas a la API directamente. Los componentes lo usan como cualquier otro hook, y si alguna vez necesitas refactorizar cómo funciona el estado del carrito, solo tocas este archivo.

### Gestión de recursos estáticos y consideraciones de rendimiento

Los recursos estáticos (imágenes, fuentes, iconos) pueden afectar significativamente el rendimiento de tu aplicación si no se gestionan con cuidado. La organización y la optimización van de la mano.

Estructura tus assets por tipo y funcionalidad:

```
public/
├── fonts/
│   ├── inter-var.woff2
│   └── inter-var-latin.woff2
├── images/
│   ├── brand/
│   │   ├── logo.svg
│   │   └── logo-dark.svg
│   ├── illustrations/
│   │   └── empty-state.svg
│   └── placeholders/
│       └── avatar.png
└── icons/
    └── favicon.ico
src/
└── assets/
    ├── images/
    │   └── hero-background.jpg
    └── icons/
        └── index.ts  // Icon component exports
```

Los archivos en `public/` se sirven tal cual y se referencian mediante rutas absolutas. Los archivos en `src/assets/` son procesados por tu empaquetador (*bundler*), que puede optimizarlos y generar nombres de archivo únicos para invalidar la caché (*cache-busting*).

Para los iconos, crea un sistema de iconos centralizado en lugar de importar Gráficos Vectoriales Escalables (SVG) individualmente:

```tsx
// assets/icons/index.tsx
import type { SVGProps } from 'react';

type IconProps = SVGProps<SVGSVGElement> & {
  size?: number;
};

export function ChevronDownIcon({ size = 24, ...props }: IconProps) {
  return (
    <svg width={size} height={size} viewBox="0 0 24 24" fill="none" {...props}>
      <path
        d="M6 9l6 6 6-6"
        stroke="currentColor"
        strokeWidth={2}
        strokeLinecap="round"
        strokeLinejoin="round"
      />
    </svg>
  );
}

export function SearchIcon({ size = 24, ...props }: IconProps) {
  return (
    <svg width={size} height={size} viewBox="0 0 24 24" fill="none" {...props}>
      <circle cx={11} cy={11} r={8} stroke="currentColor" strokeWidth={2} />
      <path d="M21 21l-4.35-4.35" stroke="currentColor" strokeWidth={2} strokeLinecap="round" />
    </svg>
  );
}

export function UserIcon({ size = 24, ...props }: IconProps) {
  return (
    <svg width={size} height={size} viewBox="0 0 24 24" fill="none" {...props}>
      <circle cx={12} cy={8} r={4} stroke="currentColor" strokeWidth={2} />
      <path
        d="M6 21v-2a4 4 0 0 1 4-4h4a4 4 0 0 1 4 4v2"
        stroke="currentColor"
        strokeWidth={2}
        strokeLinecap="round"
      />
    </svg>
  );
}
```

Este patrón mantiene la coherencia de los iconos, facilita su diseño con las utilidades de color de texto de Tailwind (ya que usan `currentColor`) y proporciona una única fuente de verdad para toda la iconografía.

Para las imágenes, crea componentes auxiliares que apliquen las mejores prácticas:

```tsx
// components/ui/OptimizedImage.tsx
import { ImgHTMLAttributes, useState } from 'react';
import { cn } from '@/lib/styles/cn';

interface OptimizedImageProps extends ImgHTMLAttributes<HTMLImageElement> {
  src: string;
  alt: string;
  aspectRatio?: 'square' | 'video' | 'portrait';
  fallback?: string;
}

export function OptimizedImage({
  src,
  alt,
  aspectRatio,
  fallback = '/images/placeholders/image.png',
  className,
  ...props
}: OptimizedImageProps) {
  const [error, setError] = useState(false);
  const [loaded, setLoaded] = useState(false);

  return (
    <div className={cn('relative overflow-hidden bg-gray-100', {
      'aspect-square': aspectRatio === 'square',
      'aspect-video': aspectRatio === 'video',
      'aspect-[3/4]': aspectRatio === 'portrait',
    }, className)}>
      {!loaded && (
        <div className="absolute inset-0 animate-pulse bg-gray-200" />
      )}
      <img
        src={error ? fallback : src}
        alt={alt}
        onLoad={() => setLoaded(true)}
        onError={() => setError(true)}
        className={cn(
          'h-full w-full object-cover transition-opacity duration-300',
          loaded ? 'opacity-100' : 'opacity-0'
        )}
        {...props}
      />
    </div>
  );
}
```

Este componente maneja estados de carga, errores y aplica relaciones de aspecto adecuadas. Proporciona una mejor experiencia de usuario que las etiquetas `img` simples, manteniendo la implementación consistente en toda tu aplicación.

La gestión de assets se reduce en última instancia a tres principios: organizar por tipo y funcionalidad, centralizar patrones comunes en componentes reutilizables y dejar que tu empaquetador se encargue de la optimización. Esto mantiene tus recursos manejables a medida que tu proyecto crece, al tiempo que garantiza que los usuarios obtengan el mejor rendimiento posible.

Con la estructura de tu proyecto y la gestión de assets en su lugar, la siguiente pieza crítica es mantener la calidad del código en todo tu equipo. Veamos cómo las herramientas automatizadas pueden hacer cumplir los estándares sin depender de revisiones de código manuales.

---

## Aplicación de calidad de código: linting, formateo y pre-commit hooks

Una base de código escalable no es solo cuestión de arquitectura: es cuestión de coherencia. Cuando cada desarrollador formatea el código de manera diferente, usa patrones distintos o toma decisiones diferentes, el código se vuelve más difícil de leer y mantener. Las herramientas pueden hacer cumplir los estándares automáticamente, eliminando la carga cognitiva y evitando que se introduzcan inconsistencias.

### Configuración de ESLint y Prettier para la coherencia del código

ESLint detecta posibles errores y hace cumplir los estándares de codificación. Prettier formatea el código de forma coherente. Juntos, forman la base de las herramientas de calidad del código.

¿Cómo mejoran la calidad del código en la práctica? Sin estas herramientas, tu base de código desarrolla rápidamente inconsistencias: un desarrollador usa comillas simples, otro usa comillas dobles y, lo que es más crítico, ESLint detecta errores reales como variables no utilizadas que indican una refactorización incompleta, propiedades `key` faltantes que causan problemas de renderizado en React, hooks `useEffect` a los que les faltan dependencias que conducen a clausuras obsoletas (*stale closures*), y funciones asíncronas utilizadas incorrectamente en los manejadores de eventos. Considera un escenario real: un desarrollador escribe `onClick={async () => await deleteItem()}` sin darse cuenta de que el error no se maneja; la regla `@typescript-eslint/no-misused-promises` de ESLint marca esto de inmediato, evitando un error de producción donde los fallos de eliminación desaparecen silenciosamente. Al refactorizar un componente y eliminar una propiedad, ESLint detecta las actualizaciones de PropTypes olvidadas antes de que lleguen a la revisión del código. Prettier garantiza que cuando tres desarrolladores tocan el mismo archivo, las diferencias solo muestren cambios significativos, no ruido de formato, por lo que las revisiones de código se centran en la lógica en lugar de discutir sobre el espaciado y los estilos de comillas.

Comienza con una configuración integral de ESLint:

```javascript
// eslint.config.mjs
import { dirname } from "node:path";
import { fileURLToPath } from "node:url";

import js from "@eslint/js";
import { defineConfig, globalIgnores } from "eslint/config";
import eslintConfigPrettier from "eslint-config-prettier/flat";
import importPlugin from "eslint-plugin-import";
import jsxA11y from "eslint-plugin-jsx-a11y";
import react from "eslint-plugin-react";
import reactHooks from "eslint-plugin-react-hooks";
import globals from "globals";
import tseslint from "typescript-eslint";

const rootDirectory = dirname(fileURLToPath(import.meta.url));

const sourceFiles = [
  "**/*.{js,mjs,cjs,jsx,ts,mts,cts,tsx}",
];

const typescriptFiles = [
  "**/*.{ts,mts,cts,tsx}",
];

export default defineConfig(
  globalIgnores([
    "dist/**",
    "build/**",
    "coverage/**",
  ]),

  {
    files: sourceFiles,

    extends: [
      js.configs.recommended,
      react.configs.flat.recommended,
      react.configs.flat["jsx-runtime"],
      reactHooks.configs.recommended,
      jsxA11y.flatConfigs.recommended,
      importPlugin.flatConfigs.recommended,
    ],

    languageOptions: {
      ecmaVersion: "latest",
      globals: {
        ...globals.browser,
        ...globals.node,
      },
    },

    settings: {
      react: {
        version: "detect",
      },

      "import/resolver": {
        typescript: true,
      },
    },

    rules: {
      "react/react-in-jsx-scope": "off",
      "react/prop-types": "off",

      "import/order": [
        "error",
        {
          groups: [
            "builtin",
            "external",
            "internal",
            "parent",
            "sibling",
            "index",
          ],
          "newlines-between": "always",
          alphabetize: {
            order: "asc",
            caseInsensitive: true,
          },
        },
      ],

      "no-console": [
        "warn",
        {
          allow: ["warn", "error"],
        },
      ],

      "prefer-const": "error",
      "no-var": "error",
    },
  },

  {
    files: typescriptFiles,

    extends: [
      tseslint.configs.recommended,
      tseslint.configs.recommendedTypeChecked,
      importPlugin.flatConfigs.typescript,
    ],

    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: rootDirectory,
      },
    },

    rules: {
      "@typescript-eslint/no-unused-vars": [
        "error",
        {
          argsIgnorePattern: "^_",
          varsIgnorePattern: "^_",
        },
      ],

      "@typescript-eslint/explicit-module-boundary-types": "off",
      "@typescript-eslint/no-explicit-any": "warn",
    },
  },

  // Keep this last so it disables formatting rules that conflict with Prettier.
  eslintConfigPrettier,
);
```

Esta configuración detecta errores comunes, hace cumplir estándares de accesibilidad con `jsx-a11y` y organiza las importaciones automáticamente. La regla `import/order` mantiene tus importaciones consistentes en todo el código, lo que facilita el escaneo visual de los archivos.

### Implementación de Prettier

Prettier maneja el formateo:

```yaml
// .prettierrc
arrowParens: always
bracketSpacing: true
jsxSingleQuote: false
printWidth: 80
quoteProps: as-needed
semi: false
singleQuote: true
tabWidth: 2
trailingComma: none
useTabs: false
plugins: ['@ianvs/prettier-plugin-sort-imports']
importOrder: ['^react$', '', '^[@/]', '', '^[~/]', '', '^[./]']

// .prettierignore
dist
build
coverage
node_modules
.next
*.min.js
```

Los valores específicos importan menos que tenerlos definidos. Elige las reglas de formato que funcionen para tu equipo y deja que Prettier las aplique. Los desarrolladores nunca vuelven a discutir sobre el formato porque Prettier toma las decisiones.

Agrega scripts a tu `package.json` para un acceso fácil:

```json
{
  "scripts": {
    "lint": "eslint . --ext .ts,.tsx --max-warnings 0",
    "lint:fix": "eslint . --ext .ts,.tsx --fix",
    "format": "prettier --write \"src/**/*.{ts,tsx,css,md}\"",
    "format:check": "prettier --check \"src/**/*.{ts,tsx,css,md}\"",
    "type-check": "tsc --noEmit"
  }
}
```

Ahora los desarrolladores pueden ejecutar `npm run lint:fix` para corregir automáticamente la mayoría de los problemas, o `npm run format` para dar formato a todos los archivos. En CI, ejecutas `lint`, `format:check` y `type-check` para asegurarte de que todo pase antes de fusionar (*merge*).

Además, puedes crear un archivo de configuración de VSCode para habilitar el formateo al guardar:

```json
// .vscode/settings.json
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,
  "prettier.requireConfig": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  }
}
```

Esta configuración automatiza el formateo y el linting en VS Code. `editor.defaultFormatter` establece Prettier como el formateador predeterminado, mientras que `editor.formatOnSave: true` lo ejecuta automáticamente al guardar. La comprobación de seguridad `prettier.requireConfig: true` garantiza que Prettier solo se ejecute si existe un archivo `.prettierrc` en el proyecto. Finalmente, `source.fixAll.eslint: "explicit"` corrige automáticamente las infracciones de ESLint como puntos y comas faltantes, importaciones incorrectas o variables no utilizadas al guardar. Juntas, estas configuraciones imponen coherencia en todo el equipo sin esfuerzo manual; los desarrolladores nunca piensan en el formato o en correcciones simples de linting porque VS Code lo maneja silenciosamente en segundo plano.

### Automatización de correcciones de código con Husky

El linting manual es útil, pero es fácil de olvidar. Los pre-commit hooks ejecutan comprobaciones automáticamente antes de que se confirme el código, detectando problemas antes de que ingresen al control de versiones. Confiar en que los desarrolladores ejecuten manualmente `npm run lint` no funciona en la práctica: alguien tiene prisa por corregir un error y hace un commit directo, otro olvida que el comando existe, un tercero ve advertencias y hace el commit de todos modos pensando que lo arreglará más tarde. Sin hooks, un desarrollador confirma código con una importación no utilizada, lo envía a una rama, abre un PR, el CI falla 5 minutos después, lo corrige, lo vuelve a enviar y el ciclo se repite. Los pre-commit hooks hacen que las comprobaciones de calidad sean obligatorias: Git no te permitirá hacer commit si las comprobaciones fallan. Esa importación no utilizada se detecta localmente en segundos, se corrige automáticamente mediante ESLint y el commit tiene éxito sin esperar al CI, sin crear un commit de "fix lint" y sin hacer perder el tiempo al equipo. Husky simplifica los hooks de Git: en lugar de crear manualmente scripts en `.git/hooks`, configuras todo en tu `package.json` y Husky se encarga del resto:

```json
// package.json
{
  "scripts": {
    "prepare": "husky install"
  },
  "devDependencies": {
    "husky": "^9.1.7",
    "lint-staged": "^16.2.3"
  }
}
```

Después de instalar, inicializa Husky:

```bash
npx husky install
npx husky add .husky/pre-commit "npx lint-staged"
```

Esto crea un pre-commit hook que ejecuta `lint-staged`, el cual solo aplica linting a los archivos que has modificado:

```javascript
// .lintstagedrc.js
module.exports = {
  '*.{ts,tsx}': [
    'eslint --fix',
    'prettier --write'
  ],
  '*.{css,md,json}': [
    'prettier --write'
  ]
};
```

Ahora, cuando haces un commit, Husky aplica linting y formato automáticamente solo a los archivos modificados. Si hay errores que no se pueden corregir automáticamente, el commit falla, solicitándote que los corrijas primero.

### Escritura de Git hooks efectivos para la colaboración en equipo

Los pre-commit hooks son solo el comienzo. Puedes hacer cumplir otros estándares a través de hooks de Git, haciendo que la colaboración sea más fluida y evitando errores comunes.

Un hook de mensaje de commit asegura mensajes de commit significativos:

```bash
# .husky/commit-msg
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npx --no -- commitlint --edit $1
```

```javascript
// commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      [
        'feat',
        'fix',
        'docs',
        'style',
        'refactor',
        'perf',
        'test',
        'chore',
        'ci'
      ]
    ],
    'subject-case': [2, 'never', ['start-case', 'pascal-case', 'upper-case']],
    'subject-max-length': [2, 'always', 100]
  }
};
```

Esto impone el formato de conventional commits, lo que facilita la generación de changelogs y la comprensión del historial del proyecto. Un commit como `feat: add user profile page` indica claramente qué cambió y por qué.

Un pre-push hook puede ejecutar pruebas antes de subir cambios:

```bash
# .husky/pre-push
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npm run type-check
npm run test -- --run
```

Esto evita que llegue código roto al repositorio, detectando problemas localmente antes de que rompan el CI o el trabajo de otros desarrolladores.

### Integración de TypeScript para una mejor mantenibilidad

TypeScript no se trata solo de detectar errores. Es documentación que nunca queda desactualizada. El código con tipos adecuados se autodocumenta y facilita la refactorización. El compilador se convierte en tu aliado, detectando errores durante el desarrollo en lugar de en producción.

Una configuración estricta de TypeScript detecta más problemas:

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "esnext",
    "useDefineForClassFields": true,
    "lib": ["esnext", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,

    /* Bundler mode */
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",

    /* Linting */
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "forceConsistentCasingInFileNames": true,

    /* Path mapping */
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@/components/*": ["./src/components/*"],
      "@/features/*": ["./src/features/*"],
      "@/lib/*": ["./src/lib/*"],
      "@/hooks/*": ["./src/hooks/*"]
    }
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

La opción `strict` habilita todas las comprobaciones estrictas de tipos, `noUncheckedIndexedAccess` hace que el acceso a arreglos sea más seguro al tratar todo acceso a elementos como potencialmente `undefined`, y `noImplicitReturns` garantiza que las funciones devuelvan valores de manera consistente o se marquen como `void`.

El mapeo de rutas (`@/*`) crea importaciones limpias y facilita la refactorización:

```typescript
// Instead of:
import { Button } from '../../../components/ui/Button';

// You write:
import { Button } from '@/components/ui/Button';
```

Cuando mueves archivos, estas importaciones no se rompen. Tu IDE puede refactorizar en toda la base de código con total confianza.

---

## Optimización de pipelines de compilación, despliegue y CI/CD

Una arquitectura escalable se extiende más allá de tu código fuente hasta cómo lo compilas y despliegas. Las compilaciones rápidas, los despliegues automatizados y los pipelines de CI/CD confiables te permiten publicar código con frecuencia y seguridad. La diferencia entre una compilación de 2 minutos y una de 10 minutos se acumula rápidamente: en un equipo de cinco desarrolladores que realizan 20 commits al día, esa diferencia de 8 minutos cuesta casi 14 horas de tiempo de CI diarias, tiempo empleado esperando retroalimentación o cambiando de contexto en lugar de crear nuevas funcionalidades. Las compilaciones lentas desalientan a los desarrolladores de ejecutarlas localmente, lo que genera más fallos en CI, mientras que los pipelines de despliegue deficientes crean pasos manuales que se olvidan, entornos inconsistentes y reversiones (*rollbacks*) que tardan horas en lugar de segundos. En esta sección, compararemos las capacidades de optimización de Webpack con la velocidad de Vite, exploraremos técnicas de optimización de bundles como code splitting y lazy loading que pueden reducir el tamaño del bundle en un 50% o más, configuraremos pipelines de CI/CD con GitHub Actions que ejecuten pruebas en paralelo y almacenen dependencias en caché de manera inteligente, e implementaremos estrategias de despliegue como previsualizaciones para cada PR y lanzamientos a producción sin tiempo de inactividad (*zero-downtime*).

### Uso de Webpack para Devtools

Si bien Vite se ha convertido en la opción predeterminada para nuevos proyectos React debido a su velocidad, comprender Webpack sigue siendo valioso. Muchas aplicaciones en producción todavía lo usan y ofrece potentes capacidades de optimización que pueden ser cruciales para requisitos de compilación complejos.

Una configuración moderna de Webpack para React con TypeScript podría verse así:

```typescript
// webpack.config.ts
import path from 'path';
import HtmlWebpackPlugin from 'html-webpack-plugin';
import MiniCssExtractPlugin from 'mini-css-extract-plugin';
import type { Configuration } from 'webpack';

const isDevelopment = process.env.NODE_ENV !== 'production';

const config: Configuration = {
  mode: isDevelopment ? 'development' : 'production',
  entry: './src/index.tsx',
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: isDevelopment ? '[name].js' : '[name].[contenthash].js',
    clean: true,
    publicPath: '/'
  },
  resolve: {
    extensions: ['.ts', '.tsx', '.js', '.jsx'],
    alias: {
      '@': path.resolve(__dirname, 'src'),
      '@/components': path.resolve(__dirname, 'src/components'),
      '@/features': path.resolve(__dirname, 'src/features'),
      '@/lib': path.resolve(__dirname, 'src/lib')
    }
  },
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: 'ts-loader',
        exclude: /node_modules/
      },
      {
        test: /\.css$/,
        use: [
          isDevelopment ? 'style-loader' : MiniCssExtractPlugin.loader,
          'css-loader',
          'postcss-loader'
        ]
      },
      {
        test: /\.(png|svg|jpg|jpeg|gif)$/i,
        type: 'asset/resource'
      }
    ]
  },
  plugins: [
    new HtmlWebpackPlugin({
      template: './index.html'
    }),
    ...(!isDevelopment ? [new MiniCssExtractPlugin({
      filename: '[name].[contenthash].css'
    })] : [])
  ],
  devServer: {
    historyApiFallback: true,
    hot: true,
    port: 3000
  },
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          priority: 10
        }
      }
    }
  }
};

export default config;
```

Esta configuración de Webpack se adapta según el entorno: el modo de desarrollo utiliza nombres de archivo simples y CSS en línea para reconstrucciones rápidas, mientras que la producción agrega hashes de contenido como `[contenthash]` a los nombres de archivo para invalidación de caché (los navegadores almacenan en caché los archivos indefinidamente hasta que cambia el hash). La sección `resolve.alias` habilita importaciones limpias como `import Button from '@/components/Button'` en lugar de rutas relativas como `../../../components/Button`. Las reglas del módulo definen cómo se procesan los diferentes tipos de archivos: `ts-loader` compila TypeScript, la cadena de CSS utiliza `style-loader` en desarrollo (inyecta CSS en la página para recarga en caliente) pero `MiniCssExtractPlugin` en producción (extrae CSS en archivos separados para un mejor almacenamiento en caché), y las imágenes usan el tipo `asset/resource` de Webpack 5 para copiarlas con nombres de archivo hasheados. `HtmlWebpackPlugin` genera `index.html` con las etiquetas de script correctas, mientras que `devServer` habilita el reemplazo de módulos en caliente (*HMR*) y maneja el enrutamiento del lado del cliente con `historyApiFallback`. Finalmente, `optimization.splitChunks` separa el código de terceros (`node_modules`) en un paquete separado que cambia con menos frecuencia que el código de tu aplicación, lo que mejora el almacenamiento en caché a largo plazo ya que los usuarios no vuelven a descargar React cada vez que lanzas una funcionalidad.

### Implementación de pipelines de CI/CD con GitHub Actions

CI/CD automatiza las pruebas, la compilación y el despliegue de tu aplicación. GitHub Actions proporciona una plataforma potente y flexible para los flujos de trabajo de CI/CD.

Aquí tienes un flujo de trabajo integral que prueba, compila y despliega:

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

# Trigger on pushes to main/develop and PRs targeting those branches
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

# Define environment variables used across all jobs
env:
  NODE_VERSION: "22"

jobs:
  # Job 1: Run quality checks
  lint-and-test:
    runs-on: ubuntu-latest

    steps:
      # Check out the repository code
      - name: Checkout repository
        uses: actions/checkout@v4

      # Setup Node.js and cache npm's package cache
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: npm

      # Install exact versions from package-lock.json
      - name: Install dependencies
        run: npm ci

      # Check TypeScript types without emitting files
      - name: Type check
        run: npm run type-check

      # Run ESLint to catch code quality issues
      - name: Lint
        run: npm run lint

      # Verify Prettier formatting
      - name: Format check
        run: npm run format:check

      # Run tests with coverage
      - name: Run tests
        run: npm test -- --run --coverage

      # Upload coverage to Codecov
      - name: Upload coverage
        uses: codecov/codecov-action@v5
        with:
          files: ./coverage/coverage-final.json
          fail_ci_if_error: true
          token: ${{ secrets.CODECOV_TOKEN }}

  # Job 2: Build the application after quality checks pass
  build:
    runs-on: ubuntu-latest
    needs: lint-and-test

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: npm

      - name: Install dependencies
        run: npm ci

      # Build with environment variables from GitHub secrets
      - name: Build
        run: npm run build
        env:
          VITE_API_URL: ${{ secrets.API_URL }}

      # Save build output for the deployment job
      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build
          path: dist/
          if-no-files-found: error

  # Job 3: Deploy only after a successful push to main
  deploy:
    runs-on: ubuntu-latest
    needs: build
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    steps:
      # Download the build artifacts from the build job
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: build
          path: dist/

      # Deploy to your hosting platform
      - name: Deploy to production
        run: |
          # Add your deployment command here:
          # vercel deploy, netlify deploy, aws s3 sync, etc.
          echo "Deploying to production..."
```

Este flujo de trabajo se ejecuta en cada push y pull request, detectando problemas antes de fusionar. El trabajo `lint-and-test` se ejecuta primero y falla rápidamente si el código no cumple con los estándares. Solo después de que se aprueban esas comprobaciones, compila la aplicación. Finalmente, si el envío es a la rama principal (`main`), se despliega automáticamente.

El flujo de trabajo utiliza almacenamiento en caché para acelerar las ejecuciones posteriores; `actions/setup-node@v4` con `cache: 'npm'` almacena en caché `node_modules`, lo que reduce drásticamente el tiempo de instalación.

### Gestión de variables de entorno y archivos de configuración

Los diferentes entornos (desarrollo, staging, producción) requieren configuraciones distintas. Las URLs de las APIs, los feature flags y las claves de terceros varían según el entorno. Gestionarlos de forma segura y cómoda es crucial.

Crea archivos específicos para cada entorno:

```env
// .env.development
VITE_API_URL=http://localhost:3001/api
VITE_APP_ENV=development
VITE_ENABLE_ANALYTICS=false

// .env.production
VITE_API_URL=https://api.production.com
VITE_APP_ENV=production
VITE_ENABLE_ANALYTICS=true

// .env.example (committed to repo)
VITE_API_URL=
VITE_APP_ENV=
VITE_ENABLE_ANALYTICS=
VITE_STRIPE_PUBLIC_KEY=
```

Vite carga automáticamente los archivos `.env` según el modo. Accede a las variables en tu código utilizando `import.meta.env`:

```typescript
// lib/config.ts
interface Config {
  apiUrl: string;
  environment: string;
  enableAnalytics: boolean;
}

function getConfig(): Config {
  const apiUrl = import.meta.env.VITE_API_URL;
  const environment = import.meta.env.VITE_APP_ENV;
  const enableAnalytics = import.meta.env.VITE_ENABLE_ANALYTICS === 'true';

  if (!apiUrl || !environment) {
    throw new Error('Missing required environment variables');
  }

  return { apiUrl, environment, enableAnalytics };
}

export const config = getConfig();


// Usage in components:
import { config } from '@/lib/config';


async function fetchData() {
  const response = await fetch(`${config.apiUrl}/users`);
  return response.json();
}
```

Esta configuración centralizada facilita la validación de las variables de entorno al iniciar y proporciona seguridad de tipos en toda la aplicación. Si falta una variable requerida, la aplicación falla de inmediato con un error claro en lugar de romperse misteriosamente más adelante.

Para secretos y valores confidenciales, utiliza la gestión de secretos de tu plataforma de CI/CD. Los secretos de GitHub Actions, por ejemplo, te permiten almacenar valores de forma segura e inyectarlos durante las compilaciones sin exponerlos en tu repositorio.

---

## Resumen

Las decisiones de arquitectura que tomas al inicio de un proyecto resuenan a lo largo de toda su vida útil. Una base de código bien estructurada con límites claros, herramientas coherentes y comprobaciones de calidad automatizadas crece con elegancia. Las nuevas funcionalidades encajan de forma natural en los patrones existentes. La refactorización se vuelve segura y sencilla. La incorporación de nuevos desarrolladores lleva horas en lugar de semanas.

Pero recuerda: no existe una arquitectura perfecta. Los patrones de este capítulo son puntos de partida, no dogmas. Las necesidades de tu equipo, el dominio de tu aplicación y las limitaciones de tu organización influyen en lo que funciona mejor. Comienza con patrones comprobados, mide lo que importa y ajusta a medida que aprendes.

El objetivo no es la perfección, es la **sostenibilidad**. Una arquitectura que le permita a tu equipo publicar funcionalidades con confianza hoy y seguir moviéndose rápidamente dentro de seis meses: esa es la marca de una aplicación escalable.

Con tu arquitectura y herramientas en su lugar, es hora de abordar una de las características más críticas en las aplicaciones modernas: la **autenticación**. En el próximo capítulo, exploraremos cómo implementar una autenticación segura y lista para producción utilizando NextAuth.js, manejando desde inicios de sesión sociales hasta control de acceso basado en roles mientras navegamos por las complejidades de la autenticación cliente-servidor en Next.js.
