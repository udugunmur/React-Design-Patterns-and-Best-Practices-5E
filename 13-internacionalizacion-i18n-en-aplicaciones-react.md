# Capítulo 13: Internacionalización (i18n) en Aplicaciones React

En el momento en que tu aplicación gana tracción más allá de tu mercado local, te enfrentas a un desafío práctico de producto: ¿cómo hacer que el software se sienta nativo para personas que hablan diferentes idiomas, usan diferentes formatos de fecha y leen en diferentes direcciones? Este desafío, la internacionalización, comúnmente abreviada como i18n, es más que un simple reemplazo de texto. Requiere una estrategia clara sobre cómo una aplicación almacena el contenido traducido, detecta el idioma preferido del usuario, renderiza el texto localizado y permite a los usuarios cambiar de idioma sin fricciones.

En este capítulo, construiremos un sistema completo de internacionalización del lado del cliente para una aplicación de React utilizando Next.js App Router. En lugar de depender de una pesada librería de terceros, construiremos un pequeño entorno de ejecución (*runtime*) de i18n desde los primeros principios. Este enfoque te ayuda a comprender exactamente cómo funciona la detección de locales, cómo se importan los diccionarios de traducción, cómo se aplican las traducciones de respaldo (*fallback translations*) y cómo React Context puede exponer una función de traducción síncrona al resto de la aplicación.

El sistema que estamos construyendo utiliza diccionarios JSON almacenados dentro del árbol de fuentes del proyecto. Estos archivos no se colocan en el directorio `public` y no se cargan con `fetch`. En su lugar, se importan directamente en el bundle de la aplicación. Esto mantiene la búsqueda de traducción de forma síncrona y predecible: los componentes pueden llamar a `t('hero.title')` y recibir una cadena de texto inmediatamente. La función de traducción no devuelve una promesa, y los componentes nunca necesitan escribir `await t(...)`.

Esta arquitectura del lado del cliente es particularmente útil para aplicaciones interactivas de React, paneles de control (*dashboards*), aplicaciones autenticadas y productos donde el idioma puede cambiar después de que la aplicación ya se haya cargado. También mantiene la capa de i18n independiente de las APIs de solicitudes del servidor. La desventaja es que el HTML inicial puede usar un idioma predeterminado hasta que la aplicación se hidrate y resuelva el locale almacenado o preferido por el navegador del usuario. Para aplicaciones donde el HTML inicial completamente localizado es obligatorio para SEO, una estrategia de i18n basada en rutas o a nivel de framework puede ser más apropiada. Sin embargo, para muchas interfaces de aplicaciones, un proveedor del lado del cliente ofrece una base simple y fácil de mantener.

En este capítulo se cubrirán los siguientes temas:

- Comprensión de la arquitectura i18n del lado del cliente
- Comprensión de los fundamentos de tipos
- Exploración de la normalización de locales
- Resolución del locale del navegador
- Comprensión de la estructura de archivos de traducción
- Importación directa de diccionarios
- Creación de una función de traducción síncrona
- Construcción de un `I18nProvider`
- Exposición de traducciones a través de `useI18n`
- Construcción de un selector de idioma
- Integración de traducciones en páginas y componentes
- Soporte para idiomas de derecha a izquierda (RTL)
- Pruebas de componentes que utilizan `useI18n`
- Errores comunes a evitar

---

## Comprensión de la arquitectura i18n del lado del cliente

Antes de sumergirnos en el código, establezcamos un modelo mental de cómo funciona la internacionalización del lado del cliente en una aplicación moderna de React. La arquitectura consta de cuatro capas: configuración, detección, almacenamiento y renderizado. Veámoslas en detalle:

1. **La capa de configuración** define qué locales admite la aplicación. Incluye el locale predeterminado, la lista de locales admitidos, la clave de almacenamiento utilizada para la persistencia y la lista de locales de derecha a izquierda (*RTL*). Mantener esta información centralizada evita que el comportamiento específico de cada locale se disperse entre los componentes.
2. **La capa de detección** determina qué idioma debe ver el usuario. En un sistema del lado del cliente, esto ocurre en el navegador. La aplicación comprueba primero una preferencia explícita del usuario, generalmente de `localStorage` o de una cookie escrita por el cliente. Si no existe ninguna preferencia guardada, comprueba `navigator.languages` y `navigator.language`. Si ninguna preferencia del navegador coincide con la lista de locales admitidos, la aplicación recurre a su locale predeterminado.
3. **La capa de almacenamiento** gestiona los diccionarios de traducción. Cada locale admitido tiene un archivo JSON correspondiente que contiene pares clave-valor. Las claves siguen una convención de notación de puntos como `hero.title`, `nav.home` y `language.switch`. Los archivos viven dentro del árbol de código fuente, por ejemplo en `src/i18n/dictionaries`. Se importan directamente a módulos de TypeScript en lugar de recuperarse a través de HTTP.
4. **La capa de renderizado** expone una función de traducción síncrona. Los componentes de React llaman a `t(key, vars?)` durante el renderizado y reciben una cadena de texto inmediatamente. La función de traducción busca la clave en el diccionario del locale activo, recurre al diccionario del locale predeterminado si es necesario, interpola variables y devuelve la cadena final.

Esta arquitectura simplifica el uso de traducciones a nivel de componente:

```tsx
  const { t } = useI18n()

  return <h1>{t('hero.title')}</h1>
```

No hay estado de carga para la función de traducción ni búsqueda asíncrona. El diccionario ya está disponible porque se importó en el proyecto. Esta es la restricción de diseño central del capítulo: los diccionarios son archivos fuente locales y la búsqueda de traducción es síncrona.

---

## Comprensión de los fundamentos de tipos

TypeScript proporciona la primera línea de defensa contra los errores de i18n. Un sistema de traducción generalmente comienza con algunos conceptos básicos: un código de locale, un diccionario, un mapa de variables para interpolación y el valor de contexto expuesto a los componentes de React:

```typescript
// src/i18n/types.ts
export type LocaleDictionary = Record<string, string>

export type TranslationVariables = Record<
  string,
  string | number | boolean | null | undefined
>

export type TranslationFunction = (
  key: string,
  vars?: TranslationVariables
) => string

export type LocaleDirection = 'ltr' | 'rtl'

export type I18nContextValue<Locale extends string = string> = {
  locale: Locale
  setLocale: (locale: Locale) => void
  t: TranslationFunction
  dir: LocaleDirection
}
```

El tipo `LocaleDictionary` representa un diccionario plano de claves de traducción y cadenas traducidas. El tipo `TranslationVariables` admite valores primitivos comunes que se pueden insertar en los mensajes traducidos. El tipo `TranslationFunction` es intencionalmente síncrono. Devuelve `string`, no `Promise<string>`.

El tipo `I18nContextValue` describe lo que reciben los componentes del proveedor de i18n: el locale actual, un setter para cambiar el locale, la función de traducción y la dirección del texto.

### Configuración

El archivo de configuración es la ubicación canónica para los locales admitidos y las constantes relacionadas. Mantener esto centralizado hace que sea más fácil agregar nuevos idiomas más adelante:

```typescript
// src/i18n/config.ts
export const DEFAULT_LOCALE = 'en-us' as const

export const SUPPORTED_LOCALES = ['en-us', 'es-mx', 'ar-sa'] as const

export type LocaleCode = (typeof SUPPORTED_LOCALES)[number]

export const LOCALE_STORAGE_KEY = 'app.locale'
export const LOCALE_COOKIE_NAME = 'locale'

export const RTL_LOCALES = ['ar-sa', 'ar', 'he', 'fa', 'ur'] as const
```

El arreglo `SUPPORTED_LOCALES` está marcado con `as const`, lo que permite a TypeScript inferir un tipo de unión literal en lugar de un `string[]` genérico. El tipo `LocaleCode` resultante se convierte en:

```typescript
'en-us' | 'es-mx' | 'ar-sa'
```

Esto proporciona una mayor seguridad de tipos al llamar a `setLocale`, definir opciones de locale y construir mapas de diccionarios.

---

## Exploración de la normalización de locales

Los códigos de locale llegan en diferentes formatos: `en`, `en-US`, `en_us`, `ES-MX`, y así sucesivamente. Antes de comparar estos valores con los locales admitidos, los normalizamos a un formato coherente. Este capítulo utiliza códigos en minúsculas al estilo BCP 47 con guiones, como `en-us` y `es-mx`:

```typescript
// src/i18n/locale.ts
import {
  DEFAULT_LOCALE,
  LOCALE_COOKIE_NAME,
  LOCALE_STORAGE_KEY,
  RTL_LOCALES,
  SUPPORTED_LOCALES,
  type LocaleCode,
} from './config'
import type { LocaleDirection } from './types'

export const normalizeLocaleCode = (value: string): string => {
  const normalized = value.trim().toLowerCase().replace(/_/g, '-')

  if (normalized === 'en') {
    return 'en-us'
  }

  if (normalized === 'es') {
    return 'es-mx'
  }

  if (normalized === 'ar') {
    return 'ar-sa'
  }

  return normalized
}

export const isSupportedLocale = (value: string): value is LocaleCode => {
  return SUPPORTED_LOCALES.includes(value as LocaleCode)
}
```

El mapeo de códigos cortos es importante porque los navegadores a menudo reportan un idioma base como `en` o `es`, mientras que las aplicaciones comúnmente almacenan códigos de locale completos como `en-us` y `es-mx`. Normalizar los códigos cortos de forma temprana mantiene el resto del sistema más simple.

### Coincidencia con locales compatibles

No todos los locales del navegador serán compatibles con tu aplicación. Un usuario puede preferir `en-gb`, mientras que tu aplicación solo incluye `en-us`. En ese caso, mostrar inglés americano suele ser mejor que recurrir a un idioma completamente no relacionado:

```typescript
// src/i18n/locale.ts
export const matchLocale = (candidate: string | null | undefined): LocaleCode | null => {
  if (!candidate) {
    return null
  }

  const normalized = normalizeLocaleCode(candidate)

  if (isSupportedLocale(normalized)) {
    return normalized
  }

  const candidateBase = normalized.split('-')[0]

  return (
    SUPPORTED_LOCALES.find((supportedLocale) => {
      return supportedLocale.split('-')[0] === candidateBase
    }) ?? null
  )
}
```

Esta función intenta primero una coincidencia exacta. Si no existe una coincidencia exacta, compara el idioma base. Esto admite casos como `en-gb` coincidiendo con `en-us`, o `es-es` coincidiendo con `es-mx`.

### Lectura de preferencias del usuario en el cliente

Cuando los usuarios seleccionan explícitamente un idioma, la aplicación debe recordar su elección. En un sistema i18n del lado del cliente, `localStorage` es un mecanismo de persistencia simple. Una cookie escrita por el cliente también puede ser útil si otro código del navegador necesita acceso a la misma preferencia:

```typescript
// src/i18n/locale.ts
const readCookie = (name: string): string | null => {
  if (typeof document === 'undefined') {
    return null
  }

  const cookies = document.cookie.split(';').map((entry) => entry.trim())
  const cookie = cookies.find((entry) => entry.startsWith(`${name}=`))

  if (!cookie) {
    return null
  }

  return decodeURIComponent(cookie.split('=')[1] ?? '')
}

export const readStoredLocale = (): LocaleCode | null => {
  if (typeof window === 'undefined') {
    return null
  }

  const localStorageLocale = window.localStorage.getItem(LOCALE_STORAGE_KEY)
  const matchedLocalStorageLocale = matchLocale(localStorageLocale)

  if (matchedLocalStorageLocale) {
    return matchedLocalStorageLocale
  }

  const cookieLocale = readCookie(LOCALE_COOKIE_NAME)
  return matchLocale(cookieLocale)
}

export const writeStoredLocale = (locale: LocaleCode): void => {
  if (typeof window === 'undefined') {
    return
  }

  window.localStorage.setItem(LOCALE_STORAGE_KEY, locale)
  document.cookie = `${LOCALE_COOKIE_NAME}=${encodeURIComponent(
    locale
  )};path=/;samesite=lax;max-age=31536000`
}
```

La función de lectura comprueba primero `localStorage` y luego la cookie. La función de escritura almacena el locale seleccionado en ambos lugares. Esto mantiene la preferencia de idioma explícita del usuario estable a través de las recargas.

---

## Resolución del locale del navegador

Cuando no hay ninguna preferencia almacenada, la lista de idiomas del navegador se convierte en la siguiente señal. La mayoría de los navegadores exponen un arreglo ordenado a través de `navigator.languages`, con el idioma más preferido primero.

Agrega la resolución del locale del navegador a `src/i18n/locale.ts`:

```typescript
// src/i18n/locale.ts
export const resolveBrowserLocale = (): LocaleCode => {
  const storedLocale = readStoredLocale()

  if (storedLocale) {
    return storedLocale
  }

  if (typeof navigator !== 'undefined') {
    const candidates = navigator.languages?.length
      ? navigator.languages
      : [navigator.language]

    for (const candidate of candidates) {
      const matchedLocale = matchLocale(candidate)

      if (matchedLocale) {
        return matchedLocale
      }
    }
  }

  return DEFAULT_LOCALE
}
```

La prioridad ahora está clara:

1. Preferencia explícita del usuario desde el almacenamiento del cliente
2. Preferencia de idioma del navegador
3. Locale predeterminado de la aplicación

Esta lógica se ejecuta en el navegador. No utiliza encabezados de solicitud ni APIs del servidor.

### Detección de la dirección del texto

Los idiomas de derecha a izquierda como el árabe, el hebreo, el persa y el urdu requieren que la dirección de la página cambie de izquierda a derecha (`ltr`) a derecha a izquierda (`rtl`). Esta dirección se puede derivar del locale actual:

```typescript
// src/i18n/locale.ts
export const getLocaleDirection = (locale: string): LocaleDirection => {
  const normalized = normalizeLocaleCode(locale)
  const language = normalized.split('-')[0]

  const isRtl = RTL_LOCALES.some((rtlLocale) => {
    return normalized === rtlLocale || language === rtlLocale
  })

  return isRtl ? 'rtl' : 'ltr'
}

export const getLocaleLanguage = (locale: string): string => {
  return normalizeLocaleCode(locale).split('-')[0]
}
```

El valor `dir` se puede aplicar más adelante a un elemento envoltorio (*wrapper*) y a `document.documentElement` después de la hidratación.

---

## Comprensión de la estructura de archivos de traducción

Los archivos de traducción utilizan una estructura JSON plana con claves en notación de puntos. Aunque el JSON anidado puede parecer más organizado al principio, los diccionarios planos suelen ser más fáciles de buscar, comparar, validar y usar en el código.

Coloca los diccionarios dentro del árbol de código fuente:

```
src/
  i18n/
    dictionaries/
      en-us.json
      es-mx.json
      ar-sa.json
      index.ts
```

Aquí está el diccionario en inglés:

```json
// src/i18n/dictionaries/en-us.json
{
  "site.title": "i18n Demo",
  "site.description": "A demonstration of client-side internationalization",
  "nav.home": "Home",
  "nav.about": "About",
  "nav.features": "Features",
  "hero.title": "Build Global Applications",
  "hero.subtitle": "Create multilingual experiences with synchronous translations",
  "hero.cta.primary": "Get Started",
  "hero.cta.secondary": "Learn More",
  "features.title": "Internationalization Features",
  "features.detection.title": "Browser Locale Detection",
  "features.detection.description": "Detect the user's preferred language from local storage or browser settings.",
  "features.sync.title": "Synchronous Translation Lookup",
  "features.sync.description": "Import dictionaries directly so components can call t() without promises.",
  "features.fallback.title": "Fallback Translations",
  "features.fallback.description": "Use the default dictionary whenever a translated key is missing.",
  "features.rtl.title": "RTL Support",
  "features.rtl.description": "Switch document direction for languages such as Arabic and Hebrew.",
  "language.switch": "Switch Language",
  "language.current": "Current: {language}",
  "language.english": "English",
  "language.spanish": "Spanish",
  "language.arabic": "Arabic"
}
```

El diccionario en español refleja las mismas claves:

```json
// src/i18n/dictionaries/es-mx.json
{
  "site.title": "Demo de i18n",
  "site.description": "Una demostración de internacionalización del lado del cliente",
  "nav.home": "Inicio",
  "nav.about": "Acerca de",
  "nav.features": "Características",
  "hero.title": "Construye aplicaciones globales",
  "hero.subtitle": "Crea experiencias multilingües con traducciones síncronas",
  "hero.cta.primary": "Comenzar",
  "hero.cta.secondary": "Más información",
  "features.title": "Características de internacionalización",
  "features.detection.title": "Detección de idioma del navegador",
  "features.detection.description": "Detecta el idioma preferido del usuario desde el almacenamiento local o la configuración del navegador.",
  "features.sync.title": "Búsqueda síncrona de traducciones",
  "features.sync.description": "Importa diccionarios directamente para que los componentes puedan llamar t() sin promesas.",
  "features.fallback.title": "Traducciones de respaldo",
  "features.fallback.description": "Usa el diccionario predeterminado cuando falta una clave traducida.",
  "features.rtl.title": "Soporte RTL",
  "features.rtl.description": "Cambia la dirección del documento para idiomas como árabe y hebreo.",
  "language.switch": "Cambiar idioma",
  "language.current": "Actual: {language}",
  "language.english": "Inglés",
  "language.spanish": "Español",
  "language.arabic": "Árabe"
}
```

El diccionario en árabe utiliza las mismas claves en inglés mientras proporciona valores en árabe:

```json
// src/i18n/dictionaries/ar-sa.json
{
  "site.title": "عرض i18n",
  "site.description": "عرض توضيحي للتدويل من جهة العميل",
  "nav.home": "الرئيسية",
  "nav.about": "حول",
  "nav.features": "الميزات",
  "hero.title": "أنشئ تطبيقات عالمية",
  "hero.subtitle": "أنشئ تجارب متعددة اللغات بترجمات متزامنة",
  "hero.cta.primary": "ابدأ الآن",
  "hero.cta.secondary": "اعرف المزيد",
  "features.title": "ميزات التدويل",
  "features.detection.title": "اكتشاف لغة المتصفح",
  "features.detection.description": "اكتشف اللغة المفضلة للمستخدم من التخزين المحلي أو إعدادات المتصفح.",
  "features.sync.title": "بحث ترجمة متزامن",
  "features.sync.description": "استورد القواميس مباشرة حتى تمكن المكونات من استدعاء t() دون وعود.",
  "features.fallback.title": "ترجمات احتياطية",
  "features.fallback.description": "استخدم القاموس الافتراضي عندما يكون مفتاح الترجمة مفقودًا.",
  "features.rtl.title": "دعم الاتجاه من اليمين إلى اليسار",
  "features.rtl.description": "غيّر اتجاه المستند للغات مثل العربية والعبرية.",
  "language.switch": "تغيير اللغة",
  "language.current": "الحالي: {language}",
  "language.english": "الإنجليزية",
  "language.spanish": "الإسبانية",
  "language.arabic": "العربية"
}
```

Las claves se mantienen estables en todos los idiomas. Solo cambian los valores. Esto facilita la comparación de diccionarios y la detección de traducciones faltantes.

---

## Importación directa de diccionarios

El archivo index de diccionarios importa cada archivo JSON directamente y expone un mapa tipado. No hay `fetch`, ni URL pública, ni paso de carga asíncrona:

```typescript
// src/i18n/dictionaries/index.ts
import type { LocaleCode } from '../config'
import type { LocaleDictionary } from '../types'
import arSa from './ar-sa.json'
import enUs from './en-us.json'
import esMx from './es-mx.json'

export const dictionaries: Record<LocaleCode, LocaleDictionary> = {
  'en-us': enUs,
  'es-mx': esMx,
  'ar-sa': arSa,
}

export const getDictionary = (locale: LocaleCode): LocaleDictionary => {
  return dictionaries[locale]
}
```

Debido a que los diccionarios se importan estáticamente, están disponibles de inmediato para el entorno de ejecución de JavaScript. Esto es lo que permite que la función de traducción permanezca síncrona.

Si TypeScript aún no está configurado para importaciones JSON, asegúrate de que `resolveJsonModule` esté habilitado:

```json
// tsconfig.json
{
  "compilerOptions": {
    "resolveJsonModule": true
  }
}
```

Los proyectos de Next.js generalmente admiten importaciones JSON de fábrica, pero esta configuración hace que el comportamiento sea explícito.

---

## Creación de una función de traducción síncrona

La función de traducción es la API principal que utilizan los desarrolladores. Recibe una clave y variables opcionales, y luego devuelve una cadena de texto. No debe cargar archivos, realizar solicitudes de red ni devolver una promesa:

```typescript
// src/i18n/translate.ts
import type { LocaleDictionary, TranslationVariables } from './types'

type TranslateOptions = {
  dictionary: LocaleDictionary
  fallbackDictionary: LocaleDictionary
  key: string
  vars?: TranslationVariables
}

const interpolate = (value: string, vars?: TranslationVariables): string => {
  if (!vars) {
    return value
  }

  return value.replace(/\{(\w+)\}/g, (match, token) => {
    if (Object.prototype.hasOwnProperty.call(vars, token)) {
      const replacement = vars[token]
      return replacement == null ? '' : String(replacement)
    }

    return match
  })
}

export const translate = ({
  dictionary,
  fallbackDictionary,
  key,
  vars,
}: TranslateOptions): string => {
  const raw = dictionary[key] ?? fallbackDictionary[key] ?? key
  return interpolate(raw, vars)
}
```

El comportamiento de respaldo (*fallback*) es sencillo:

1. Probar con el diccionario del locale activo
2. Probar con el diccionario del locale predeterminado
3. Devolver la clave misma

Devolver la clave es útil durante el desarrollo porque las traducciones faltantes se vuelven visibles en la interfaz. En lugar de representar silenciosamente una cadena vacía, la interfaz de usuario muestra `hero.title` u otra clave faltante.

La interpolación de variables admite mensajes como:

```json
{
  "language.current": "Current: {language}"
}
```

Un componente puede renderizar este mensaje de forma síncrona:

```tsx
  const { t } = useI18n()

  return <p>{t('language.current', { language: t('language.english') })}</p>
```

---

## Construcción de un `I18nProvider`

El proveedor es el propietario del locale actual y expone la función de traducción a través de React Context. Dado que este es un sistema del lado del cliente, el proveedor debe ser un Client Component:

```tsx
// src/i18n/i18n-provider.tsx
'use client'

import {
  createContext,
  useCallback,
  useEffect,
  useMemo,
  useState,
  type ReactNode,
} from 'react'
import { DEFAULT_LOCALE, type LocaleCode } from './config'
import { dictionaries, getDictionary } from './dictionaries'
import {
  getLocaleDirection,
  getLocaleLanguage,
  resolveBrowserLocale,
  writeStoredLocale,
} from './locale'
import { translate } from './translate'
import type { I18nContextValue } from './types'

export const I18nContext = createContext<I18nContextValue<LocaleCode> | null>(
  null
)

type I18nProviderProps = {
  children: ReactNode
  initialLocale?: LocaleCode
}

export function I18nProvider({
  children,
  initialLocale = DEFAULT_LOCALE,
}: I18nProviderProps) {
  const [locale, setLocaleState] = useState<LocaleCode>(initialLocale)

  useEffect(() => {
    const resolvedLocale = resolveBrowserLocale()
    setLocaleState(resolvedLocale)
  }, [])

  const dir = useMemo(() => getLocaleDirection(locale), [locale])
  const lang = useMemo(() => getLocaleLanguage(locale), [locale])

  useEffect(() => {
    document.documentElement.lang = lang
    document.documentElement.dir = dir
  }, [dir, lang])

  const setLocale = useCallback((nextLocale: LocaleCode) => {
    setLocaleState(nextLocale)
    writeStoredLocale(nextLocale)
  }, [])

  const dictionary = dictionaries[locale] ?? getDictionary(DEFAULT_LOCALE)
  const fallbackDictionary = getDictionary(DEFAULT_LOCALE)

  const t = useCallback(
    (key: string, vars?: Parameters<typeof translate>[0]['vars']) => {
      return translate({
        dictionary,
        fallbackDictionary,
        key,
        vars,
      })
    },
    [dictionary, fallbackDictionary]
  )

  const value = useMemo<I18nContextValue<LocaleCode>>(
    () => ({
      locale,
      setLocale,
      t,
      dir,
    }),
    [dir, locale, setLocale, t]
  )

  return (
    <I18nContext.Provider value={value}>
      <div lang={lang} dir={dir}>
        {children}
      </div>
    </I18nContext.Provider>
  )
}
```

Varios detalles son importantes en esta implementación:

- El proveedor comienza con `DEFAULT_LOCALE`. Después de la hidratación, el efecto se ejecuta en el navegador y resuelve el locale almacenado o preferido por el navegador. Esto evita leer `window`, `document` o `navigator` durante el renderizado del servidor.
- `t` se crea con `useCallback`, pero la función en sí sigue siendo síncrona. Devuelve el resultado de `translate` directamente.
- El proveedor actualiza `document.documentElement.lang` y `document.documentElement.dir` después de la hidratación. También envuelve la aplicación en un `div` con `lang` y `dir`, de modo que los componentes reciban la dirección correcta como parte del árbol renderizado.
- Cambiar de locale no requiere una actualización de página. La actualización del estado de React hace que los componentes suscritos se vuelvan a renderizar con el nuevo diccionario.

---

## Exposición de traducciones a través de `useI18n`

Los componentes no deben acceder al contexto directamente. Un hook pequeño proporciona una API limpia y un error útil si falta el proveedor:

```typescript
// src/i18n/use-i18n.ts
'use client'

import { useContext } from 'react'
import { I18nContext } from './i18n-provider'

export const useI18n = () => {
  const context = useContext(I18nContext)

  if (!context) {
    throw new Error('useI18n must be used inside I18nProvider')
  }

  return context
}
```

El hook expone el valor de contexto completo:

```tsx
  const { locale, setLocale, t, dir } = useI18n()
```

El uso más común es la función de traducción:

```tsx
  <h1>{t('hero.title')}</h1>
```

Ningún componente necesita cargar diccionarios antes de llamar a `t`. Los diccionarios ya se han importado en el grafo de módulos del proveedor.

### Integración del proveedor con Next.js App Router

El layout raíz puede seguir siendo un Server Component porque no realiza tareas de i18n. Simplemente envuelve la aplicación con el proveedor del cliente:

```tsx
// src/app/layout.tsx
import type { Metadata } from 'next'
import { I18nProvider } from '@/src/i18n/i18n-provider'
import './globals.css'

export const metadata: Metadata = {
  title: 'i18n Demo',
  description: 'Client-side internationalization demo',
}

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode
}>) {
  return (
    <html lang="en" dir="ltr">
      <body className="min-h-screen bg-white antialiased">
        <I18nProvider>{children}</I18nProvider>
      </body>
    </html>
  )
}
```

El layout utiliza `lang` y `dir` predeterminados porque la preferencia real del usuario se resuelve en el cliente. Después de la hidratación, el proveedor actualiza los atributos del documento.

Este diseño evita por completo la resolución de locales en el servidor. El layout no lee encabezados de solicitud, cookies ni APIs del servidor. No carga diccionarios de traducción. Solo monta el proveedor.

---

## Construcción de un selector de idioma

Los usuarios necesitan una forma de anular la detección automática del navegador. El selector de idioma lee el locale actual del contexto y llama a `setLocale` cuando el usuario elige un nuevo idioma:

```tsx
// src/components/language-switcher.tsx
'use client'

import { Globe } from 'lucide-react'
import { SUPPORTED_LOCALES, type LocaleCode } from '@/src/i18n/config'
import { useI18n } from '@/src/i18n/use-i18n'

const localeLabels: Record<LocaleCode, string> = {
  'en-us': 'language.english',
  'es-mx': 'language.spanish',
  'ar-sa': 'language.arabic',
}

export function LanguageSwitcher() {
  const { locale, setLocale, t } = useI18n()

  return (
    <label className="inline-flex items-center gap-2 rounded-lg border border-gray-200 bg-white px-3 py-2 text-sm font-medium text-gray-700 shadow-sm">
      <Globe className="h-4 w-4" aria-hidden="true" />
      <span className="sr-only">{t('language.switch')}</span>
      <select
        value={locale}
        onChange={(event) => setLocale(event.target.value as LocaleCode)}
        className="bg-transparent outline-none"
        aria-label={t('language.switch')}
      >
        {SUPPORTED_LOCALES.map((option) => (
          <option key={option} value={option}>
            {t(localeLabels[option])}
          </option>
        ))}
      </select>
    </label>
  )
}
```

El selector de idioma es intencionalmente simple. No llama a `router.refresh()` porque las traducciones no se resuelven en el servidor. Actualizar el estado del contexto es suficiente para volver a renderizar la interfaz de usuario en el idioma seleccionado.

Puedes reemplazar el `select` nativo con un menú desplegable personalizado más adelante, pero el comportamiento debe seguir siendo el mismo: llamar a `setLocale(locale)` y dejar que React actualice la interfaz traducida.

---

## Integración de traducciones en páginas y componentes

Un componente que llama a `useI18n` debe ser un Client Component. En App Router, esto significa agregar `'use client'` en la parte superior del archivo o mover la interfaz traducida a un componente secundario del cliente.

Para una página de demostración pequeña, la página misma puede ser un Client Component:

```tsx
// src/app/page.tsx
'use client'

import { ArrowRight, Globe, Languages, ShieldCheck, Zap } from 'lucide-react'
import { LanguageSwitcher } from '@/src/components/language-switcher'
import { useI18n } from '@/src/i18n/use-i18n'

export default function HomePage() {
  const { t } = useI18n()

  const features = [
    {
      icon: Globe,
      title: t('features.detection.title'),
      description: t('features.detection.description'),
    },
    {
      icon: Zap,
      title: t('features.sync.title'),
      description: t('features.sync.description'),
    },
    {
      icon: ShieldCheck,
      title: t('features.fallback.title'),
      description: t('features.fallback.description'),
    },
    {
      icon: Languages,
      title: t('features.rtl.title'),
      description: t('features.rtl.description'),
    },
  ]

  return (
    <main className="min-h-screen bg-gradient-to-b from-gray-50 to-white">
      <header className="sticky top-0 z-50 border-b border-gray-200 bg-white/80 backdrop-blur-sm">
        <div className="mx-auto flex h-16 max-w-6xl items-center justify-between px-4">
          <div className="flex items-center gap-2">
            <Globe className="h-6 w-6 text-blue-600" />
            <span className="font-semibold text-gray-900">{t('site.title')}</span>
          </div>

          <nav className="hidden items-center gap-6 md:flex">
            <a href="#" className="text-sm text-gray-600 hover:text-gray-900">
              {t('nav.home')}
            </a>
            <a
              href="#features"
              className="text-sm text-gray-600 hover:text-gray-900"
            >
              {t('nav.features')}
            </a>
            <a href="#" className="text-sm text-gray-600 hover:text-gray-900">
              {t('nav.about')}
            </a>
          </nav>

          <LanguageSwitcher />
        </div>
      </header>

      <section className="py-24 md:py-32">
        <div className="mx-auto max-w-4xl px-4 text-center">
          <h1 className="text-4xl font-bold tracking-tight text-gray-900 md:text-6xl">
            {t('hero.title')}
          </h1>
          <p className="mt-6 text-lg text-gray-600 md:text-xl">
            {t('hero.subtitle')}
          </p>
          <div className="mt-10 flex flex-col items-center gap-4 sm:flex-row sm:justify-center">
            <button className="inline-flex items-center gap-2 rounded-lg bg-blue-600 px-6 py-3 font-medium text-white shadow-md hover:bg-blue-700">
              {t('hero.cta.primary')}
              <ArrowRight className="h-4 w-4 rtl:rotate-180" />
            </button>
            <button className="rounded-lg border border-gray-300 px-6 py-3 font-medium text-gray-700 hover:bg-gray-50">
              {t('hero.cta.secondary')}
            </button>
          </div>
        </div>
      </section>

      <section id="features" className="border-t border-gray-200 py-24">
        <div className="mx-auto max-w-6xl px-4">
          <h2 className="text-center text-3xl font-bold text-gray-900">
            {t('features.title')}
          </h2>
          <div className="mt-16 grid gap-8 md:grid-cols-2">
            {features.map((feature) => {
              const Icon = feature.icon

              return (
                <article
                  key={feature.title}
                  className="rounded-xl border border-gray-200 bg-white p-6 shadow-sm"
                >
                  <Icon className="h-10 w-10 text-blue-600" />
                  <h3 className="mt-4 text-lg font-semibold text-gray-900">
                    {feature.title}
                  </h3>
                  <p className="mt-2 text-gray-600">{feature.description}</p>
                </article>
              )
            })}
          </div>
        </div>
      </section>
    </main>
  )
}
```

Esta página demuestra el patrón de uso previsto. Las traducciones se leen sincrónicamente durante el renderizado. No hay `await`, no hay función de carga ni paso de propiedades de traducción de servidor a cliente.

Para aplicaciones más grandes, puedes mantener los archivos de ruta como Server Components y mover la interfaz traducida a componentes secundarios del cliente:

```tsx
// src/app/page.tsx
import { HomeContent } from './home-content'

export default function HomePage() {
  return <HomeContent />
}
```

```tsx
// src/app/home-content.tsx
'use client'

import { useI18n } from '@/src/i18n/use-i18n'

export function HomeContent() {
  const { t } = useI18n()

  return <h1>{t('hero.title')}</h1>
}
```

Esto mantiene el límite de la página simple mientras coloca la lógica de traducción interactiva donde pertenece: en un componente cliente.

---

## Soporte para idiomas de derecha a izquierda (RTL)

Como se mencionó anteriormente, algunos idiomas se leen de derecha a izquierda (RTL). Admitir estos idiomas requiere más que cadenas traducidas. La alineación del texto, la dirección del diseño y los iconos direccionales deben responder al locale seleccionado.

El proveedor ya calcula `dir` y lo aplica a un elemento envoltorio:

```tsx
<div lang={lang} dir={dir}>
  {children}
</div>
```

También actualiza el elemento del documento después de la hidratación:

```tsx
useEffect(() => {
  document.documentElement.lang = lang
  document.documentElement.dir = dir
}, [dir, lang])
```

Una vez que `dir="rtl"` está presente, los navegadores ajustan automáticamente muchos comportamientos de diseño. El texto fluye de derecha a izquierda, el diseño en línea cambia de dirección y las propiedades lógicas de CSS se adaptan.

Tailwind CSS también proporciona utilidades lógicas y variantes de RTL. Prefiere clases de espaciado lógico como `ms-*` y `me-*` cuando necesites un espaciado consciente de la dirección. Utiliza la variante `rtl:` para iconos que implican dirección:

```tsx
<button className="inline-flex items-center gap-2">
  {t('hero.cta.primary')}
  <ArrowRight className="h-4 w-4 rtl:rotate-180" />
</button>
```

La flecha apunta hacia la derecha en idiomas de izquierda a derecha y se invierte en idiomas de derecha a izquierda. Este pequeño detalle importa porque los iconos direccionales transmiten significado.

Los problemas comunes de RTL incluyen:

- Iconos que implican movimiento pero no se invierten
- Posicionamiento codificado de izquierda y derecha de forma fija
- Márgenes y rellenos que utilizan dirección física en lugar de dirección lógica
- Texto traducido que es más largo que el original y desborda los contenedores
- Contenido en varios idiomas donde los números o nombres de productos necesitan un espaciado cuidadoso

Probar el diseño RTL requiere ver la aplicación con un locale RTL real seleccionado. Establecer el árabe en el selector de idioma debería exponer de inmediato problemas de diseño que son fáciles de pasar por alto en inglés o español.

### Manejo de traducciones faltantes

Las traducciones faltantes son inevitables durante el desarrollo. El sistema debe fallar de manera visible y predecible. Devolver la clave misma es un valor predeterminado práctico:

```typescript
const raw = dictionary[key] ?? fallbackDictionary[key] ?? key
```

Este comportamiento significa que una clave faltante se renderiza como texto legible como `settings.profile.title`. Eso es mucho más fácil de depurar que una etiqueta vacía.

Para aplicaciones de producción, es posible que también desees registrar las claves faltantes en desarrollo:

```typescript
  if (process.env.NODE_ENV !== 'production' && raw === key) {
    console.warn(`Missing translation: ${key}`)
  }
```

Mantén esta advertencia fuera de la ruta crítica si se vuelve ruidosa. El comportamiento central debe permanecer estable: diccionario activo, diccionario de respaldo y luego la clave.

### Reglas de interpolación

La interpolación de variables permite que las cadenas traducidas incluyan valores dinámicos:

```json
{
  "language.current": "Current: {language}"
}
```

La lógica de interpolación reemplaza los tokens coincidentes del objeto `vars`:

```tsx
  <p>{t('language.current', { language: t('language.spanish') })}</p>
```

Si falta un token, el marcador de posición permanece sin cambios. Por ejemplo, si no se proporciona `{language}`, la salida sigue siendo `Current: {language}`. Esto hace que las variables faltantes sean visibles durante el desarrollo.

La sintaxis de interpolación en este capítulo utiliza llaves simples, como `{name}`. Algunas aplicaciones prefieren llaves dobles, como `{{name}}`, especialmente al alinearse con sistemas de plantillas de correo electrónico o herramientas de gestión de traducción. Cualquiera de las sintaxis es aceptable siempre que sea coherente. Si eliges llaves dobles, actualiza la expresión regular de interpolación en consecuencia.

### Consideraciones de rendimiento

Las importaciones de diccionarios del lado del cliente proporcionan una búsqueda síncrona, pero también afectan el tamaño del paquete (*bundle size*). Dado que los diccionarios se importan en el grafo de JavaScript, los archivos de traducción grandes pueden aumentar la cantidad de JavaScript enviado al navegador.

Para aplicaciones pequeñas y medianas, importar todos los diccionarios en un mapa central a menudo es aceptable y simplifica enormemente el tiempo de ejecución. La función de traducción es rápida, predecible y no requiere cascadas de red (*network waterfalls*).

Para aplicaciones más grandes, considera dividir los diccionarios por funcionalidad mientras sigues utilizando importaciones directas. Por ejemplo:

```
src/
  i18n/
    dictionaries/
      en-us/
        common.json
        dashboard.json
      es-mx/
        common.json
        dashboard.json
```

Luego compone los diccionarios con importaciones estáticas:

```typescript
// src/i18n/dictionaries/index.ts
import commonEnUs from './en-us/common.json'
import dashboardEnUs from './en-us/dashboard.json'
import commonEsMx from './es-mx/common.json'
import dashboardEsMx from './es-mx/dashboard.json'
import type { LocaleCode } from '../config'
import type { LocaleDictionary } from '../types'

export const dictionaries: Record<LocaleCode, LocaleDictionary> = {
  'en-us': {
    ...commonEnUs,
    ...dashboardEnUs,
  },
  'es-mx': {
    ...commonEsMx,
    ...dashboardEsMx,
  },
  'ar-sa': {},
}
```

Esto mantiene la búsqueda síncrona al tiempo que permite que la base de código organice las traducciones por dominio. La regla importante sigue siendo la misma: los diccionarios utilizados por `t()` ya deben estar importados cuando ocurre el renderizado.

Un sistema de i18n del lado del cliente tiene diferentes compensaciones en comparación con un sistema de i18n renderizado en el servidor:

- Evita las APIs de solicitud del servidor y la carga de diccionarios en el servidor.
- Permite cambiar de idioma sin recargar la ruta.
- Mantiene `t()` de forma síncrona.
- Puede renderizar el locale predeterminado antes de que la hidratación resuelva la preferencia guardada.
- Puede aumentar el tamaño del bundle del cliente si los diccionarios son grandes.

Estas compensaciones son aceptables para muchos paneles de aplicaciones e interfaces de productos autenticados. Para páginas de marketing públicas donde se requiere HTML localizado antes de la hidratación, utiliza una estrategia de i18n basada en rutas en su lugar.

### Pruebas de la función de traducción

Dado que `translate` es una función síncrona pura, es fácil de probar. No hay simulación de llamadas de red (*mocking*), ni temporizadores falsos, ni resolución de promesas:

```typescript
// __tests__/translate.test.ts
import { translate } from '@/src/i18n/translate'

const fallbackDictionary = {
  'hero.title': 'Build Global Applications',
  'language.current': 'Current: {language}',
}

const spanishDictionary = {
  'hero.title': 'Construye aplicaciones globales',
}

describe('translate', () => {
  it('returns a translation from the active dictionary', () => {
    const result = translate({
      dictionary: spanishDictionary,
      fallbackDictionary,
      key: 'hero.title',
    })

    expect(result).toBe('Construye aplicaciones globales')
  })

  it('falls back to the default dictionary', () => {
    const result = translate({
      dictionary: spanishDictionary,
      fallbackDictionary,
      key: 'language.current',
      vars: { language: 'Español' },
    })

    expect(result).toBe('Current: Español')
  })

  it('returns the key when no translation exists', () => {
    const result = translate({
      dictionary: spanishDictionary,
      fallbackDictionary,
      key: 'missing.key',
    })

    expect(result).toBe('missing.key')
  })
})
```

Las pruebas confirman los tres comportamientos más importantes: búsqueda en el diccionario activo, búsqueda en el diccionario de respaldo y manejo de claves faltantes.

---

## Pruebas de componentes que utilizan `useI18n`

Los componentes que llaman a `useI18n` se pueden probar simulando el hook de forma síncrona. La función `t` simulada debe devolver una cadena directamente:

```tsx
// __tests__/home-content.test.tsx
import { render, screen } from '@testing-library/react'
import HomePage from '@/src/app/page'

const mockTranslations: Record<string, string> = {
  'site.title': 'i18n Demo',
  'hero.title': 'Build Global Applications',
  'hero.subtitle': 'Create multilingual experiences',
  'hero.cta.primary': 'Get Started',
  'hero.cta.secondary': 'Learn More',
  'features.title': 'Internationalization Features',
  'features.detection.title': 'Browser Locale Detection',
  'features.detection.description': 'Detect browser language.',
  'features.sync.title': 'Synchronous Translation Lookup',
  'features.sync.description': 'Use direct imports.',
  'features.fallback.title': 'Fallback Translations',
  'features.fallback.description': 'Use default translations.',
  'features.rtl.title': 'RTL Support',
  'features.rtl.description': 'Support Arabic and Hebrew.',
  'nav.home': 'Home',
  'nav.features': 'Features',
  'nav.about': 'About',
  'language.switch': 'Switch Language',
  'language.english': 'English',
  'language.spanish': 'Spanish',
  'language.arabic': 'Arabic',
}

jest.mock('@/src/i18n/use-i18n', () => ({
  useI18n: () => ({
    locale: 'en-us',
    dir: 'ltr',
    setLocale: jest.fn(),
    t: (key: string) => mockTranslations[key] ?? key,
  }),
}))

describe('HomePage', () => {
  it('renders translated content', () => {
    render(<HomePage />)

    expect(screen.getByText('Build Global Applications')).toBeInTheDocument()
    expect(screen.getByText('Synchronous Translation Lookup')).toBeInTheDocument()
  })
})
```

La prueba no ejecuta `await HomePage()`, y no simula `t` como una promesa. La función de traducción simulada sigue el mismo contrato que la real: entra una clave, sale una cadena de texto.

### Pruebas del proveedor

También puedes probar el proveedor renderizando un pequeño componente consumidor dentro de `I18nProvider`:

```tsx
// __tests__/i18n-provider.test.tsx
import { render, screen } from '@testing-library/react'
import { I18nProvider } from '@/src/i18n/i18n-provider'
import { useI18n } from '@/src/i18n/use-i18n'

function TestConsumer() {
  const { t } = useI18n()
  return <p>{t('hero.title')}</p>
}

describe('I18nProvider', () => {
  it('provides synchronous translations', () => {
    render(
      <I18nProvider initialLocale="en-us">
        <TestConsumer />
      </I18nProvider>
    )

    expect(screen.getByText('Build Global Applications')).toBeInTheDocument()
  })
})
```

Esta prueba confirma que el proveedor y el hook funcionan juntos sin requerir una configuración asíncrona.

### Adición de un nuevo idioma

Agregar un nuevo idioma sigue una secuencia predecible:

1. Agregar el locale a `SUPPORTED_LOCALES`.
2. Agregar un diccionario JSON en `src/i18n/dictionaries`.
3. Importar el diccionario en `src/i18n/dictionaries/index.ts`.
4. Agregar la clave de etiqueta de idioma a cada diccionario.
5. Agregar la opción de idioma al selector de idioma si es necesario.
6. Agregar el locale a `RTL_LOCALES` si utiliza una escritura RTL.

Por ejemplo, para agregar francés:

```typescript
// src/i18n/config.ts
export const SUPPORTED_LOCALES = ['en-us', 'es-mx', 'ar-sa', 'fr-fr'] as const
```

```typescript
// src/i18n/dictionaries/index.ts
import frFr from './fr-fr.json'

export const dictionaries: Record<LocaleCode, LocaleDictionary> = {
  'en-us': enUs,
  'es-mx': esMx,
  'ar-sa': arSa,
  'fr-fr': frFr,
}
```

El resto del sistema continúa funcionando porque la coincidencia de locales, el manejo de respaldos y la búsqueda de traducciones están centralizados.

### Estructura de carpetas recomendada

Una implementación completa puede utilizar la siguiente estructura:

```
src/
  app/
    layout.tsx
    page.tsx
  components/
    language-switcher.tsx
  i18n/
    config.ts
    i18n-provider.tsx
    locale.ts
    translate.ts
    types.ts
    use-i18n.ts
    dictionaries/
      ar-sa.json
      en-us.json
      es-mx.json
      index.ts
```

Esta estructura mantiene la infraestructura de i18n en un solo lugar. Los componentes solo importan `useI18n`, mientras que el proveedor importa los diccionarios y las utilidades de locale.

---

## Errores comunes a evitar

- **Evita hacer que `t` sea asíncrono.** Una vez que `t` devuelve una promesa, cada componente que renderice texto traducido debe lidiar con el comportamiento asíncrono. Esa complejidad se extiende rápidamente por toda la base de código.
- **Evita colocar diccionarios en el directorio `public` para esta arquitectura.** Los archivos públicos son útiles cuando deseas obtener recursos por URL, pero este capítulo utiliza importaciones de fuentes específicamente para mantener la búsqueda de traducción de forma síncrona.
- **Evita llamar a APIs del servidor para la detección de locales.** Esta arquitectura utiliza el almacenamiento del navegador y las preferencias de idioma del navegador. No debe importar `next/headers`, leer cookies de solicitud ni depender del estado de solicitud de RSC.
- **Evita actualizar la ruta cuando el usuario cambie de idioma.** Dado que las traducciones son estado del lado del cliente, una actualización de página es innecesaria. Actualizar el estado del proveedor es suficiente.
- **Evita codificar etiquetas traducidas fijas dentro de los componentes.** Si aparece una cadena en la interfaz de usuario, generalmente debería residir en el diccionario. Esto mantiene la cobertura del idioma auditable.

---

## Resumen

La internacionalización transforma la forma en que las aplicaciones se comunican con los usuarios a través de diferentes idiomas y culturas. En este capítulo, construimos un sistema i18n completo del lado del cliente para aplicaciones React y Next.js utilizando importaciones directas de JSON, React Context y una función de traducción síncrona.

Comenzamos definiendo una arquitectura del lado del cliente con capas claras: configuración, detección, almacenamiento y renderizado. En lugar de cargar diccionarios desde URLs públicas, los almacenamos dentro del árbol de código fuente y los importamos directamente. Este diseño permitió que la búsqueda de traducciones fuera síncrona y simple.

Implementamos la normalización y coincidencia de locales para que los valores del navegador como `en`, `en-GB` o `es_ES` puedan resolverse a los locales admitidos por la aplicación. Persistimos las preferencias explícitas del usuario en el almacenamiento del navegador y usamos `navigator.languages` como señal de respaldo.

Construimos un ayudante `translate` puro que busca claves en el diccionario activo, recurre al diccionario predeterminado, interpola variables y devuelve la clave cuando no existe ninguna traducción. La función `t` expuesta por el proveedor sigue el mismo contrato: recibe una clave y devuelve una cadena de texto inmediatamente.

Creamos un `I18nProvider` que posee el estado del locale, expone `setLocale`, calcula la dirección del texto, actualiza los atributos del documento después de la hidratación y proporciona `t` a través de React Context. Luego construimos un selector de idioma que actualiza el locale sin recargar la ruta.

Finalmente, cubrimos el soporte para idiomas RTL, las compensaciones de rendimiento y las estrategias de prueba. Debido a que este sistema es síncrono, las pruebas pueden simular `useI18n` con una función simple que devuelve cadenas de texto. No es necesario resolver promesas de traducción ni esperar a que se carguen los diccionarios.

Este enfoque es una base sólida para aplicaciones interactivas que necesitan una interfaz de usuario multilingüe confiable sin la complejidad de i18n del lado del servidor. A partir de aquí, puedes ampliar el sistema con reglas de pluralización, formato de fechas y monedas, linting de traducciones o integración con un flujo de trabajo de gestión de traducciones.

En el próximo capítulo, desarrollaremos una comprensión práctica de las pruebas creando una aplicación de gestión de tareas.
