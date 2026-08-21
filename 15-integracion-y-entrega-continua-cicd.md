# Capítulo 15: Integración y Entrega Continua (CI/CD)

En el momento en que subes código a producción manualmente por primera vez, te das cuenta de lo frágil que puede ser el proceso. Un paso de compilación olvidado, una funcionalidad no probada o una variable de entorno mal configurada: cualquiera de estos factores puede hacer caer tu aplicación. Aquí es donde la Integración Continua y la Entrega Continua (CI/CD) transforman la forma en que distribuimos aplicaciones de React.

CI/CD no elimina el trabajo, cambia el tipo de trabajo que haces. En lugar de desplegar manualmente y esperar que nada se rompa, pasarás tiempo manteniendo pipelines, solucionando pruebas inestables (*flaky tests*) y actualizando configuraciones que silenciosamente se desincronizan. Esa es la compensación honesta. Lo que ganas es coherencia: las mismas comprobaciones se ejecutan en cada commit, ya sea un lunes por la mañana o un viernes a las 5 PM, ya sea que estés haciendo push tú o el miembro más nuevo del equipo. Seguirás encontrando problemas, pero detectarás más de ellos antes de que lleguen a producción, y depurar una compilación fallida es mucho más fácil que revertir un despliegue roto a las 2 AM.

Para las aplicaciones de React, CI/CD añade una capa de orquestación sobre tus herramientas de compilación existentes. Tu empaquetador (*bundler*, como Vite, webpack o Next.js) gestiona las variables de entorno, la división de código (*code splitting*) y la optimización de recursos; el pipeline simplemente ejecuta esas compilaciones en un entorno coherente y falla ruidosamente cuando algo sale mal. Esta distinción es importante al depurar: si tu paquete es demasiado grande, ese es un problema de configuración de compilación; si las compilaciones pasan localmente pero fallan en CI, generalmente es un problema de entorno o de dependencias. Un buen pipeline no reemplaza tu configuración de compilación, sino que saca a la superficie los problemas antes de que lleguen a producción.

GitHub Actions, GitLab CI y CircleCI son plataformas de CI/CD; ejecutan tus pruebas, compilaciones y scripts personalizados en respuesta a eventos del repositorio. Vercel y Netlify son plataformas de despliegue que incluyen algunas funciones de CI/CD, pero su trabajo principal es alojar y servir tu aplicación. A menudo usarás ambas juntas: una plataforma de CI/CD para ejecutar tu suite completa de pruebas y comprobaciones de calidad, y luego una plataforma de despliegue para gestionar el alojamiento real. A lo largo de este capítulo, nos centraremos en GitHub Actions. Ten en cuenta que, si bien los conceptos (disparadores, trabajos, almacenamiento en caché) se transfieren entre plataformas, la sintaxis y las capacidades no; la palabra clave `needs` de GitLab CI funciona de manera diferente a las dependencias de trabajos de GitHub Actions, y cada plataforma tiene sus propios límites y peculiaridades que deberás aprender.

En este capítulo, cubriremos los siguientes temas:

- Configuración de un pipeline de CI/CD con GitHub Actions
- Automatización de compilaciones y pruebas en pipelines de despliegue
- Gestión de configuraciones específicas de entorno en CI/CD

---

## Configuración de un pipeline de CI/CD con GitHub Actions

GitHub Actions es la plataforma de automatización integrada de GitHub. Te permite definir flujos de trabajo (*workflows*), que son secuencias de trabajos (*jobs*) que se ejecutan en respuesta a eventos del repositorio como pushes, pull requests o momentos programados. Cada workflow reside en un archivo YAML dentro del directorio `.github/workflows` de tu repositorio, lo que significa que tu configuración de CI/CD está versionada junto con tu código. Los trabajos se ejecutan en máquinas virtuales alojadas en GitHub (o en tus propios ejecutores autoalojados), y puedes componerlos utilizando bloques reutilizables llamados acciones (*actions*) del marketplace de GitHub o escribir las tuyas propias.

GitHub Actions es una opción popular para proyectos de React porque vive junto a tu código y utiliza una sintaxis YAML familiar. GitLab CI, CircleCI y Jenkins siguen siendo ampliamente utilizados; tu elección a menudo depende de dónde reside tu repositorio y de lo que tu equipo ya conoce. GitHub Actions es gratuito para repositorios públicos, pero los repositorios privados obtienen 2,000 minutos por mes en el nivel gratuito, lo que suena como mucho hasta que comienzas a usar estrategias de matriz para probar en múltiples versiones de Node. Un workflow de 10 minutos que se ejecuta en dos versiones de Node consume 20 minutos por ejecución, por lo que un equipo ocupado puede agotar ese límite en un par de semanas. Mantén un ojo en tu uso en la configuración del repositorio y considera si cada push necesita la matriz completa o solo los pull requests. Con eso en mente, creemos un pipeline listo para producción y veamos cómo encaja cada pieza.

Nuestro primer workflow se centrará en la integración continua, asegurando que cada pull request cumpla con nuestros estándares de calidad antes de que pueda fusionarse:

```yaml
name: CI Pipeline
on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main]
jobs:
  quality-checks:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [22.x]
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      - name: Install dependencies
        run: npm ci
      - name: Run linter
        run: npm run lint
      - name: Type check
        run: npm run type-check
      - name: Run tests
        run: npm test -- --coverage --watchAll=false
      - name: Build application
        run: npm run build
        env:
          CI: true
```

Este workflow se ejecuta en cada pull request y push a `main`. El comando `npm ci` instala las dependencias de forma reproducible, utilizando versiones exactas de `package-lock.json`. Ten en cuenta que probar en múltiples versiones de Node.js tiene sentido para paquetes de npm, pero para una aplicación de Next.js, tus usuarios ejecutan el código en navegadores, no en Node.js. La versión de Node.js solo importa durante el tiempo de compilación y para los Server Components, así que prueba contra tu versión de Node en producción en lugar de una matriz. Si estás desplegando en Vercel con Node 20, probar también contra Node 16 y 18 consume minutos de CI sin una cobertura significativa. Cada paso se basa en el anterior, y si algún paso falla, el workflow se detiene; tu pull request no mostrará la marca de verificación verde hasta que todo pase.

Pero una aplicación de React robusta necesita más que solo pruebas aprobadas. También debemos rastrear el tamaño de nuestro bundle para evitar enviar inadvertidamente JavaScript inflado a nuestros usuarios. Creemos una acción complementaria que analice nuestra compilación:

```yaml
name: Analyze Next.js Bundle
on:
  pull_request:
    branches:
      - main
  workflow_dispatch:
jobs:
  analyze:
    name: Build and analyze bundle
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - name: Install dependencies
        run: npm ci
      - name: Build with bundle analyzer
        run: ANALYZE=true npm run build
      - name: Upload bundle analyzer reports
        uses: actions/upload-artifact@v4
        with:
          name: nextjs-bundle-analyzer
          path: |
            .next/analyze/*.html
          if-no-files-found: warn
```

Este workflow genera un informe visual de la composición de tu paquete y comenta en el pull request con información del tamaño. Es increíblemente valioso para detectar aumentos inesperados en el tamaño del bundle antes de que lleguen a producción.

Asegúrate de que `next.config.mjs` tenga habilitado el analizador:

```javascript
import bundleAnalyzer from '@next/bundle-analyzer'

const withBundleAnalyzer = bundleAnalyzer({
  enabled: process.env.ANALYZE === 'true'
})

/** @type {import('next').NextConfig} */
const nextConfig = {}

export default withBundleAnalyzer(nextConfig)
```

Y en `package.json`:

```json
{
  "scripts": {
    "build": "next build"
  },
  "devDependencies": {
    "@next/bundle-analyzer": "latest"
  }
}
```

Ahora hablemos del almacenamiento en caché. Hay dos tipos de almacenamiento en caché en un pipeline de CI típico de Next.js, y sirven para propósitos diferentes:
- **El almacenamiento en caché de dependencias** acelera `npm ci` almacenando en caché los paquetes descargados. La acción `actions/setup-node` maneja esto automáticamente con `cache: 'npm'`, que almacena en caché la caché de descargas de npm (no `node_modules`), de modo que npm no vuelve a descargar paquetes que ya tiene:

```yaml
- name: Setup Node.js
  uses: actions/setup-node@v4
  with:
    node-version: '22.x'
    cache: 'npm'
```

- **El almacenamiento en caché de compilación** es independiente; acelera la compilación de Next.js reutilizando archivos compilados previamente. Next.js almacena datos de compilación incremental en `.next/cache`, y restaurar esto entre ejecuciones significa que los archivos sin cambios no necesitan recompilación:

```yaml
- name: Cache Next.js build
  uses: actions/cache@v4
  with:
    path: .next/cache
    key: ${{ runner.os }}-nextjs-${{ hashFiles('**/package-lock.json') }}-${{ hashFiles('**/*.ts', '**/*.tsx') }}
```

La clave de caché de compilación incluye tanto los hashes de dependencias como los de los archivos fuente; si alguno cambia, la caché se invalida. Esto es más agresivo que el almacenamiento en caché de dependencias porque la salida de la compilación depende de ambos. Pero el almacenamiento en caché no es gratuito. Las cachés obsoletas causan errores sutiles: la CI pasa con artefactos en caché mientras que una compilación limpia fallaría, o una caché desactualizada enmascara un cambio importante. Cuando algo funciona localmente pero falla en CI (o viceversa), la caché suele ser la culpable; tu primer instinto debe ser limpiarla y ejecutar desde cero. Utiliza el almacenamiento en caché donde el beneficio sea claro, pero no almacenes todo en caché solo porque puedes.

Hasta ahora nos hemos centrado en el lado de la CI, ejecutando comprobaciones en cada push y pull request. Ahora veamos qué sucede después de que se aprueban esas comprobaciones.

---

## Automatización de compilaciones y pruebas en pipelines de despliegue

Con la integración continua establecida, podemos centrarnos en la entrega continua: desplegar automáticamente tu aplicación cuando el código se fusiona con `main`. Esto significa que tu aplicación se despliega sin intervención manual cada vez que fusionas una pull request. Eso suena conveniente, y lo es, pero también significa que cada brecha en tu cobertura de pruebas es un incidente potencial de producción. No hay un punto de control humano para detectar lo que tus pruebas pasaron por alto. El despliegue automático recompensa a los equipos que invierten en comprobaciones exhaustivas de CI y castiga a los que no lo hacen. Antes de habilitarlo, asegúrate de confiar en tu pipeline lo suficiente como para dejar que entregue código en tu nombre.

Creemos un workflow de despliegue que compile nuestra aplicación de React y la despliegue en diferentes entornos según la rama. Este workflow introduce el concepto de despliegues específicos de entorno:

```yaml
name: Deploy
on:
  push:
    branches:
      - main
      - staging
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: ${{ github.ref == 'refs/heads/main' && 'production' || 'staging' }}
      url: ${{ steps.deploy.outputs.url }}
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22.x'
          cache: 'npm'
      - name: Install dependencies
        run: npm ci
      - name: Build application
        run: npm run build
        env:
          NEXT_PUBLIC_API_URL: ${{ secrets.API_URL }}
          NEXT_PUBLIC_ENVIRONMENT: ${{ github.ref == 'refs/heads/main' && 'production' || 'staging' }}
      - name: Deploy to environment
        id: deploy
        run: |
          if [ "${{ github.ref }}" == "refs/heads/main" ]; then
            npm run deploy:production
          else
            npm run deploy:staging
          fi
```

Los scripts `deploy:production` y `deploy:staging` se definen en tu `package.json` y dependen de tu plataforma de alojamiento. Aquí tienes ejemplos comunes:

```json
{
  "scripts": {
    "deploy:staging": "vercel --prod --scope=my-team --token=$DEPLOY_TOKEN",
    "deploy:production": "vercel --prod --scope=my-team --token=$DEPLOY_TOKEN"
  }
}
```

```json
{
  "scripts": {
    "deploy:staging": "netlify deploy --prod --site=$STAGING_SITE_ID --auth=$DEPLOY_TOKEN",
    "deploy:production": "netlify deploy --prod --site=$PRODUCTION_SITE_ID --auth=$DEPLOY_TOKEN"
  }
}
```

```json
{
  "scripts": {
    "deploy:staging": "aws s3 sync .next/static s3://staging-bucket && aws cloudfront create-invalidation --distribution-id $STAGING_CF_ID --paths '/*'",
    "deploy:production": "aws s3 sync .next/static s3://production-bucket && aws cloudfront create-invalidation --distribution-id $PROD_CF_ID --paths '/*'"
  }
}
```

Para Vercel y Netlify, sus herramientas de CLI manejan la subida y la invalidación de CDN. Para AWS, normalmente estás sincronizando con S3 e invalidando CloudFront. El workflow anterior es independiente de la plataforma; sustituye los comandos de despliegue por la infraestructura que utilices.

### Configuración de pruebas para entornos de CI

Las pruebas en pipelines de CI/CD requieren algunos ajustes. Tus pruebas se ejecutan en un entorno headless sin un navegador visible, por lo que cualquier prueba basada en navegador necesita una configuración adecuada. La velocidad también importa: cada segundo dedicado a ejecutar pruebas es un segundo añadido a tu ciclo de retroalimentación (y a tu factura). Esta configuración utiliza `@swc/jest`, un compilador basado en Rust que es de 10 a 20 veces más rápido que `ts-jest`. Instálalo con `npm install -D @swc/jest @swc/core`:

```typescript
// jest.config.ts
import type { Config } from 'jest'
import nextJest from 'next/jest'

// next/jest handles Next.js-specific configuration automatically:
// - SWC transforms (faster than ts-jest)
// - CSS/image imports
// - Path aliases from tsconfig.json
// - Loading .env files
const createJestConfig = nextJest({
  dir: './', // Path to your Next.js app
})

const config: Config = {
  testEnvironment: 'jsdom',
  setupFilesAfterEnv: ['<rootDir>/src/setupTests.ts'],
  testMatch: [
    '**/__tests__/**/*.(test|spec).{ts,tsx}',
    '**/*.(test|spec).{ts,tsx}',
  ],
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/**/*.stories.tsx',
  ],
  coverageThreshold: {
    global: {
      branches: 70,
      functions: 70,
      lines: 70,
      statements: 70,
    },
  },
  // Module aliases are automatically loaded from tsconfig.json
  // No need to manually configure moduleNameMapper for @/ paths
}

// createJestConfig wraps your config with Next.js defaults
export default createJestConfig(config)
```

Esta configuración impone umbrales de cobertura, asegurando que tu equipo mantenga una alta cobertura de pruebas. Cuando la cobertura cae por debajo del 70%, el pipeline de CI falla, evitando la fusión de código insuficientemente probado. Es una función de control suave que mantiene la calidad del código a lo largo del tiempo.

### Pruebas end-to-end con Playwright

Las pruebas end-to-end (pruebas E2E) se sitúan en la cima de la pirámide de pruebas. Mientras que las pruebas unitarias verifican funciones y componentes individuales de forma aislada, y las pruebas de integración comprueban cómo funcionan juntas las piezas, las pruebas E2E verifican flujos de usuario completos a través de tu aplicación real (hacer clic en botones, rellenar formularios, navegar por páginas). Esto las convierte en la aproximación más cercana al comportamiento real del usuario, pero conlleva compensaciones: son lentas (segundos o minutos por prueba en lugar de milisegundos), propensas a la inestabilidad (problemas de red, problemas de sincronización, peculiaridades del navegador) y costosas de mantener (los cambios en la interfaz de usuario rompen las pruebas incluso cuando la funcionalidad está bien). Debido a esto, la mayoría de los equipos escriben menos pruebas E2E que pruebas unitarias y se centran en rutas críticas como la autenticación, el pago o los flujos de trabajo centrales en lugar de probar cada caso extremo.

Playwright y Cypress son las dos opciones principales. Playwright es agnóstico del framework; no sabe ni le importa si tu aplicación utiliza React, Vue o Angular. Interactúa con el DOM y la red, lo que significa que funciona con cualquier aplicación web. Ejecuta pruebas en motores de navegador reales (Chromium, Firefox, WebKit) y admite la ejecución en paralelo de forma nativa. Cypress tiene una experiencia de desarrollo más interactiva con su depurador de viaje en el tiempo y ofrece pruebas de componentes para React, lo que te permite probar componentes en un navegador real en lugar de jsdom, llenando el vacío entre las pruebas unitarias y las pruebas E2E completas. Si estás escribiendo principalmente pruebas E2E para CI, Playwright suele ser la mejor opción. Si tu equipo valora el ejecutor de pruebas interactivo durante el desarrollo o desea pruebas de componentes basadas en navegador, vale la pena considerar Cypress. Ambos se integran bien con GitHub Actions:

```yaml
name: E2E Tests
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
jobs:
  e2e-tests:
    timeout-minutes: 10
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22.x'
          cache: 'npm'
      - name: Install dependencies
        run: npm ci
      - name: Install Playwright
        run: npx playwright install --with-deps
      - name: Build application
        run: npm run build
      - name: Run E2E tests
        run: npx playwright test
        env:
          CI: true
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
```

Las siguientes pruebas se ejecutan contra tu aplicación en `localhost` dentro del runner de CI. Si bien la documentación de Playwright llama a esto pruebas E2E, se describe con mayor precisión como pruebas de integración basadas en navegador. Estás verificando que tus componentes, enrutamiento y lógica del lado del cliente funcionen juntos en un navegador real, pero no estás probando la pila de producción completa.

Las verdaderas pruebas end-to-end se ejecutan contra un entorno desplegado: una URL de vista previa de Vercel o Netlify, un servidor de staging o un entorno de prueba dedicado. Solo entonces detectas malas configuraciones de CORS, problemas de almacenamiento en caché de CDN, errores de certificados SSL y latencia de red real. Si tu pipeline se despliega en una URL de vista previa, apunta Playwright hacia ella:

```yaml
- name: Run E2E tests
  run: npx playwright test
  env:
    BASE_URL: ${{ steps.deploy-preview.outputs.url }}
```

Dicho esto, las pruebas basadas en localhost aún detectan la mayoría de las regresiones de interfaz de usuario y son más rápidas de ejecutar. Aquí hay un ejemplo usando Playwright:

```typescript
// e2e/dashboard.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Dashboard', () => {
  test('should display user dashboard with metrics', async ({ page }) => {
    await page.goto('/')

    // Use semantic locators (getByRole) instead of CSS selectors
    // They're more resilient to UI changes and reflect user interaction
    await expect(page.getByRole('heading', { name: 'Dashboard' }))
      .toBeVisible()

    // getByTestId is acceptable when no semantic role exists
    // Ensure your components include data-testid attributes
    const metricsCards = page.getByTestId('metric-card')
    await expect(metricsCards).toHaveCount(4)

    // Verify cards contain actual data, not just placeholder text
    await expect(metricsCards.first()).toContainText(/\d+/)
  })

  test('should navigate to settings page', async ({ page }) => {
    // Each test navigates fresh—no shared state between tests
    // This makes tests independent and easier to debug
    await page.goto('/')
    await page.getByRole('link', { name: 'Settings' }).click()

    // Prefer toHaveURL with regex over waitForURL
    // It auto-waits and provides better error messages
    await expect(page).toHaveURL(/\/settings$/)
    await expect(page.getByRole('heading', { name: 'Settings' }))
      .toBeVisible()
  })

  test('should filter metrics by date range', async ({ page }) => {
    await page.goto('/')

    // Test real user interactions, not implementation details
    await page.getByRole('button', { name: 'Filter' }).click()
    await page.getByRole('option', { name: 'Last 7 days' }).click()

    // Verify the UI reflects the user's selection
    await expect(page.getByTestId('date-range-label'))
      .toContainText('Last 7 days')
  })

  test('should display empty state when no data', async ({ page }) => {
    // Test edge cases with dedicated routes or test data
    // Don't mock API responses—that defeats the purpose of integration tests
    await page.goto('/dashboard/empty-project')
    await expect(page.getByRole('heading', { name: 'No data yet' }))
      .toBeVisible()

    // Verify the user has a path forward from empty states
    await expect(page.getByRole('link', { name: 'Import data' }))
      .toBeVisible()
  })
})
```

Estas pruebas utilizan localizadores semánticos (`getByRole`) como estrategia de selección principal, recurriendo a `data-testid` solo cuando no existe ningún rol semántico. Cada prueba navega de forma independiente en lugar de depender del estado compartido de `beforeEach`, lo que facilita el aislamiento de los fallos. En lugar de esperar estados de red arbitrarios o simular respuestas de API (ambos antipatrones en Playwright), las pruebas esperan a que elementos específicos sean visibles, lo cual es más confiable y refleja mejor la experiencia real del usuario.

Con las pruebas configuradas, hay otro desafío que a menudo causa errores sutiles en los pipelines de CI/CD: asegurarse de que tu aplicación se compile con la configuración correcta para cada entorno.

---

## Gestión de configuraciones específicas de entorno en CI/CD

Uno de los aspectos más complicados de CI/CD es gestionar la configuración en diferentes entornos. Tu aplicación de React necesita diferentes URLs de API, feature flags y claves de analítica para desarrollo, staging y producción. Gestionar esto incorrectamente puede provocar que aparezcan datos de staging en producción o, peor aún, que las credenciales de producción se filtren en compilaciones públicas.

La base de la gestión de entornos es separar la configuración del código. Nunca codifiques de forma fija (*hardcode*) valores específicos del entorno en tus componentes de React. En su lugar, utiliza variables de entorno que se inyectan en el momento de la compilación:

```typescript
// src/config/environment.ts
interface EnvironmentConfig {
  apiUrl: string;
  environment: 'development' | 'staging' | 'production';
  features: {
    enableAnalytics: boolean;
    enableBetaFeatures: boolean;
    debugMode: boolean;
  };
  analytics: {
    googleAnalyticsId?: string;
    sentryDsn?: string;
  };
}

function getEnvironmentConfig(): EnvironmentConfig {
  const env = process.env.REACT_APP_ENVIRONMENT || 'development';
  const baseConfig: EnvironmentConfig = {
    apiUrl: process.env.REACT_APP_API_URL || 'http://localhost:3001',
    environment: env as EnvironmentConfig['environment'],
    features: {
      enableAnalytics: env === 'production',
      enableBetaFeatures: env !== 'production',
      debugMode: env === 'development',
    },
    analytics: {},
  };

  if (env === 'production') {
    baseConfig.analytics.googleAnalyticsId = process.env.REACT_APP_GA_ID;
    baseConfig.analytics.sentryDsn = process.env.REACT_APP_SENTRY_DSN;
  }

  return baseConfig;
}

export const config = getEnvironmentConfig();
```

Este módulo de configuración proporciona acceso seguro mediante tipos a las variables de entorno en toda tu aplicación. También aplica lógica específica del entorno, como deshabilitar las analíticas en entornos que no sean de producción.

En GitHub Actions, gestionas estas variables de entorno mediante secretos y configuraciones específicas de entorno. Aquí te mostramos cómo configurar un workflow que maneje múltiples entornos correctamente:

```yaml
on:
  push:
    branches:
      - main
      - staging
      - develop

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ github.ref == 'refs/heads/main' && 'production' || github.ref == 'refs/heads/staging' && 'staging' || 'development' }}
    env:
      API_URL: ${{ github.ref == 'refs/heads/main' && 'https://api.example.com' || github.ref == 'refs/heads/staging' && 'https://staging-api.example.com' || 'https://dev-api.example.com' }}
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22.x'
          cache: 'npm'
      - name: Install dependencies
        run: npm ci
      - name: Build
        run: npm run build
        env:
          NEXT_PUBLIC_API_URL: ${{ env.API_URL }}
          NEXT_PUBLIC_ENVIRONMENT: ${{ github.ref == 'refs/heads/main' && 'production' || github.ref == 'refs/heads/staging' && 'staging' || 'development' }}
          NEXT_PUBLIC_SENTRY_DSN: ${{ secrets.SENTRY_DSN }}
```

Las expresiones condicionales manejan la selección del entorno en el momento de la evaluación: un trabajo, un runner por push. La cadena ternaria mapea cada rama a su entorno correspondiente y URL de API, manteniendo la configuración en un solo lugar en lugar de dispersa en archivos de workflow separados. Para los secretos, utiliza la función de secretos específicos del entorno de GitHub (configurada en los ajustes del repositorio bajo Environments) en lugar de codificar valores de forma fija o construir dinámicamente nombres de secretos. Es más seguro, más fácil de auditar y te brinda controles adicionales como revisores obligatorios para despliegues a producción.

Para aplicaciones más complejas, podrías considerar la configuración en tiempo de ejecución (*runtime configuration*): feature flags obtenidos de una API que pueden cambiar sin recompilar. Pero esto agrega complejidad real: latencia de red al inicio, manejo de errores cuando el servicio no está disponible y comportamiento inconsistente cuando los flags se cargan a mitad de sesión. Si tu respaldo es usar valores predeterminados en tiempo de compilación, habrás agregado una dependencia que generalmente solo devuelve lo que ya tenías integrado.

Los flags en tiempo de ejecución tienen sentido cuando necesitas alternar funciones para usuarios específicos, realizar pruebas A/B o hacer despliegues graduales. Para diferencias simples basadas en el entorno, la configuración en tiempo de compilación es más confiable:

```typescript
// src/config/featureFlags.ts
interface FeatureFlags {
  newDashboard: boolean
  experimentalEditor: boolean
  advancedAnalytics: boolean
}

export const featureFlags: FeatureFlags = {
  newDashboard: process.env.NEXT_PUBLIC_ENABLE_NEW_DASHBOARD === 'true',
  experimentalEditor: process.env.NEXT_PUBLIC_ENABLE_EXPERIMENTAL_EDITOR === 'true',
  advancedAnalytics: process.env.NEXT_PUBLIC_ENABLE_ANALYTICS === 'true',
}
```

Luego, en tu componente de React, utilizas el feature flag:

```tsx
// src/components/Dashboard.tsx
import { featureFlags } from '@/config/featureFlags'

export function Dashboard() {
  return featureFlags.newDashboard ? <NewDashboard /> : <LegacyDashboard />
}
```

Sin solicitudes de red, sin estados de carga, sin manejo de errores: los flags se integran en el momento de la compilación y se eliminan del árbol (*tree-shaken*) en producción. Si más adelante necesitas flags específicos para usuarios o pruebas A/B, considera un servicio dedicado como LaunchDarkly o Statsig en lugar de construir el tuyo propio.

Unamos todo con un workflow de despliegue integral que incluya promoción de entornos, capacidades de reversión (*rollback*) y notificaciones de despliegue:

```yaml
name: Build and Release
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact-name: ${{ steps.upload.outputs.artifact-name }}
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22.x'
          cache: 'npm'
      - name: Install dependencies
        run: npm ci

      # Build once - this artifact will be used for both staging and production
      # Environment-specific config should be runtime, not build-time
      - name: Build
        run: npm run build

      # Store artifact for deployment and potential rollbacks
      - name: Upload release artifact
        id: upload
        uses: actions/upload-artifact@v4
        with:
          name: release-${{ github.sha }}
          path: .next/
          retention-days: 30

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.example.com
    steps:
      # Download the exact artifact we built - no rebuild
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: release-${{ github.sha }}
          path: .next/

      - name: Deploy to staging
        run: |
          # Your deploy command here (Vercel, Netlify, AWS, etc.)
          echo "Deploying to staging..."
        env:
          DEPLOY_TOKEN: ${{ secrets.STAGING_DEPLOY_TOKEN }}

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.example.com
    steps:
      # Same artifact that passed staging - true promotion
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: release-${{ github.sha }}
          path: .next/

      - name: Deploy to production
        run: |
          echo "Deploying to production..."
        env:
          DEPLOY_TOKEN: ${{ secrets.PRODUCTION_DEPLOY_TOKEN }}

      # Only notify on failure - avoid notification fatigue
      # Success is the default; failures need attention
      - name: Notify on failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": "Production deployment failed",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "Production deployment *failed*\nCommit: `${{ github.sha }}`\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View logs>"
                  }
                }
              ]
            }
```

Este workflow demuestra la promoción de artefactos: el trabajo de compilación crea un solo artefacto, staging lo despliega primero y solo después de que staging tiene éxito, producción despliega exactamente el mismo artefacto. La línea `needs: deploy-staging` impone este orden; producción no se ejecutará si staging falla. Las notificaciones solo se activan en caso de fallo para evitar el problema del canal de Slack silenciado, donde los desarrolladores ignoran el ruido del despliegue y pasan por alto los fallos reales. El artefacto se almacena durante 30 días con el SHA del commit en su nombre, lo que permite reversiones al descargar y volver a desplegar un artefacto anterior en lugar de recompilar.

Finalmente, creemos un workflow de reversión que pueda volver rápidamente a un despliegue anterior si algo sale mal:

```yaml
name: Rollback Production
on:
  workflow_dispatch:
    inputs:
      commit_sha:
        description: 'Commit SHA to rollback to'
        required: true
        type: string

jobs:
  rollback:
    runs-on: ubuntu-latest
    environment: production
    steps:
      # Download the exact artifact from the previous successful deployment
      # No rebuild - guaranteed identical to what was running before
      - name: Download release artifact
        uses: actions/download-artifact@v4
        with:
          name: release-${{ inputs.commit_sha }}
          path: .next/

      - name: Deploy rollback
        run: |
          echo "Rolling back to ${{ inputs.commit_sha }}..."
        env:
          DEPLOY_TOKEN: ${{ secrets.PRODUCTION_DEPLOY_TOKEN }}

      - name: Notify rollback
        uses: slackapi/slack-github-action@v1
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": " Production rolled back to ${{ inputs.commit_sha }}"
            }
```

Este workflow utiliza `workflow_dispatch`, lo que significa que lo activas manualmente desde la interfaz de usuario de GitHub Actions proporcionando el SHA del commit de un despliegue anterior. Debido a que descarga un artefacto almacenado en lugar de recompilar desde el código fuente, la reversión se completa en menos de un minuto: simplemente descargar y desplegar. Esa es la diferencia entre una verdadera reversión y una recompilación: estás restaurando el paquete exacto que se estaba ejecutando antes, no esperando que el mismo código fuente produzca la misma salida semanas después con versiones de dependencias potencialmente diferentes.

La limitación es la retención de artefactos. El workflow de compilación almacena artefactos durante 30 días, por lo que solo puedes revertir a despliegues dentro de esa ventana. Para aplicaciones críticas donde necesitas una capacidad de reversión más prolongada, sube los artefactos de lanzamiento a S3 o a un registro de contenedores en lugar de depender del almacenamiento de artefactos de GitHub. Alternativamente, las plataformas de despliegue como Vercel y Netlify mantienen los despliegues anteriores disponibles indefinidamente para una reversión instantánea a través de sus paneles.

La notificación de Slack se activa en cada reversión. A diferencia de las notificaciones de despliegue, donde solo alertamos en caso de fallo, las reversiones son eventos excepcionales que el equipo debe conocer independientemente del resultado. Si alguien está revirtiendo la producción, esa es información que vale la pena compartir.

---

## Resumen

Este capítulo cubrió los fundamentos de CI/CD con GitHub Actions: ejecutar pruebas en pull requests, compilar con configuraciones específicas de entorno y desplegar en diferentes entornos. Estos workflows proporcionan un punto de partida, pero tienen limitaciones.

Los pipelines dependen de servicios externos, registros de npm, la infraestructura de GitHub y acciones de terceros. Cuando esos servicios tienen problemas, tu pipeline se detiene. La verdadera confiabilidad requiere almacenar en caché las dependencias en tu propia infraestructura, fijar las versiones de las acciones a SHAs específicos y contar con planes de contingencia.

Tampoco logramos compilaciones verdaderamente reproducibles. Una compilación reproducible produce una salida binaria idéntica a partir de la misma fuente, donde los checksums coinciden independientemente de cuándo o dónde compiles. Nuestros workflows recompilan a partir del código fuente cada vez, lo que significa que la resolución de dependencias y las diferencias de entorno pueden producir salidas diferentes. Para una reproducibilidad genuina, necesitarías archivos lockfile verificados, imágenes base fijadas y almacenamiento de artefactos para que despliegues exactamente lo que probaste.

Varios patrones valen la pena explorar más allá de este capítulo: pruebas de regresión visual para detectar cambios involuntarios en la interfaz de usuario, despliegues blue-green para lanzamientos sin tiempo de inactividad (*zero-downtime*) y lanzamientos canary para enrutar gradualmente el tráfico a nuevas versiones. Estos agregan complejidad operativa y requieren soporte de infraestructura más allá de GitHub Actions por sí solo.

CI/CD es un trabajo continuo, no una configuración única. Lo que has construido es una base, útil, pero lejos de estar completa. Con los pipelines de despliegue en su lugar, el siguiente desafío es garantizar que lo que despliegas realmente tenga un buen rendimiento, lo que nos lleva al [Capítulo 16](https://subscription.packtpub.com/book/web-development/9781806108251/16).
