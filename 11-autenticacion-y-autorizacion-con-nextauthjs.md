# Capítulo 11: Autenticación y Autorización con NextAuth.js

La autenticación se encuentra en el corazón de las aplicaciones web modernas, custodiando silenciosamente la puerta entre los espacios públicos y privados. Cada vez que un usuario inicia sesión, cada ruta protegida a la que accede, cada panel de control personalizado que visualiza, estas experiencias dependen de un sistema de autenticación robusto que funciona entre bastidores. En las aplicaciones de React y Next.js, este desafío se vuelve aún más matizado a medida que navegamos entre los límites del cliente y del servidor, equilibrando la seguridad con el rendimiento.

En este capítulo, cubriremos los siguientes temas:

- Introducción a la autenticación en React y Next.js
- Configuración de NextAuth.js en una aplicación Next.js 16
- Integración de proveedores de autenticación en NextAuth.js
- Manejo de autenticación en RSC
- Implementación de control de acceso basado en roles (RBAC) y permisos

---

## Introducción a la autenticación en React y Next.js

La autenticación responde a una pregunta fundamental: ¿quién eres? La autorización le sigue de cerca preguntando: ¿qué tienes permitido hacer? Estos dos conceptos forman la base de la seguridad de las aplicaciones; sin embargo, implementarlos correctamente requiere una cuidadosa consideración de la arquitectura, la experiencia del usuario y las mejores prácticas de seguridad.

### La importancia de la autenticación y la autorización

Considera una aplicación web típica, tal vez una herramienta de gestión de proyectos o una plataforma social. Sin autenticación, no existe el concepto de *tus* proyectos o *tu* feed. Sin autorización, cada usuario podría acceder a controles administrativos, modificar los datos de otros usuarios o realizar acciones destructivas. La autenticación establece la identidad, mientras que la autorización determina las capacidades en función de esa identidad.

Lo que está en juego es mucho. Un sistema de autenticación mal implementado expone los datos de los usuarios, abre las puertas a ataques de apropiación de cuentas (*account takeover*) y erosiona la confianza. Por el contrario, un sistema bien diseñado se vuelve invisible para los usuarios: fluido, seguro y confiable. Las aplicaciones modernas exigen este nivel de pulido, y Next.js, combinado con NextAuth.js, proporciona las herramientas para lograrlo.

### Autenticación del lado del cliente vs. del lado del servidor

Las aplicaciones de React tradicionalmente manejaban la autenticación en el lado del cliente, almacenando tokens en `localStorage`, administrando el estado de inicio de sesión con proveedores de contexto (*context providers*) y protegiendo rutas con renderizado condicional. Si bien este enfoque funciona, introduce varios desafíos. Los tokens del lado del cliente son vulnerables a ataques de Cross-Site Scripting (XSS), las cargas iniciales de página exponen contenido protegido antes de que se ejecute JavaScript, y la gestión de la renovación de tokens (*token refresh*) se vuelve compleja.

Next.js cambia este paradigma al introducir el renderizado del lado del servidor (*Server-Side Rendering*) y los Server Components. La autenticación ahora puede ocurrir en el servidor, antes de que el HTML llegue al navegador. Las sesiones se pueden validar en middleware, las rutas de API pueden autenticar solicitudes en el Edge, y las operaciones confidenciales nunca tocan el cliente. Este enfoque centrado en el servidor (*server-first*) mejora drásticamente la seguridad al tiempo que simplifica el modelo mental para los desarrolladores.

La naturaleza híbrida de Next.js significa que trabajamos con ambos paradigmas. Los Server Components obtienen datos autenticados, los Client Components manejan flujos de inicio de sesión interactivos, y el middleware protege las rutas incluso antes de que se rendericen. Comprender dónde ocurren las comprobaciones de autenticación y por qué se vuelve crucial para crear aplicaciones seguras.

### Por qué usar NextAuth.js para la autenticación

Construir la autenticación desde cero es tentador. ¿Qué tan difícil podría ser hacer hash de contraseñas y emitir tokens JWT?

La respuesta: sorprendentemente difícil. La gestión de sesiones, los flujos OAuth, la protección CSRF, la rotación de tokens, el manejo seguro de cookies... estas preocupaciones se multiplican rápidamente. Las vulnerabilidades de seguridad acechan en cada rincón, y el costo de equivocarse es severo.

NextAuth.js elimina esta carga al proporcionar una librería de autenticación probada en batalla y diseñada específicamente para Next.js. Maneja proveedores de OAuth de forma predeterminada, gestiona sesiones de forma segura, se integra a la perfección con RSC y sigue las mejores prácticas de seguridad por defecto. En lugar de reinventar la autenticación, podemos aprovechar una solución en la que confían miles de aplicaciones.

La filosofía de la librería se alinea perfectamente con Next.js: convención sobre configuración, pensamiento *server-first* y arquitectura lista para el Edge. Ya sea que necesites una autenticación simple por correo electrónico/contraseña o flujos complejos de OAuth con múltiples proveedores, NextAuth.js escala para satisfacer tus necesidades sin obligarte a convertirte en un experto en seguridad.

Ahora que entendemos por qué NextAuth.js es la opción correcta, la siguiente sección se centra en implementarlo en una aplicación Next.js 16, cubriendo la instalación, la configuración y los conceptos centrales que impulsan la autenticación.

---

## Configuración de NextAuth.js en una aplicación Next.js 16

Transformemos la teoría en práctica integrando NextAuth.js en una aplicación Next.js. El proceso de configuración establece las bases para todo lo que sigue: configuración de proveedores, gestión de sesiones y protección de rutas.

### Instalación y configuración de NextAuth.js

Primero, instalaremos las dependencias necesarias y crearemos la configuración principal de autenticación. NextAuth.js 5 (también conocido como Auth.js) introduce una nueva API que está más alineada con la arquitectura de Next.js 16:

```bash
npm install next-auth@beta
```

La configuración comienza creando un manejador de autenticación que define nuestros proveedores y la estrategia de sesión. Colocaremos esto en un archivo de configuración de autenticación dedicado que tanto nuestras rutas de API como nuestros Server Components puedan importar:

```typescript
// auth.ts
import NextAuth, { DefaultSession } from "next-auth";
import GitHub from "next-auth/providers/github";
import Google from "next-auth/providers/google";
import Credentials from "next-auth/providers/credentials";
import { JWT } from "next-auth/jwt";

// Extend the default Session type to include custom user properties.
// TypeScript's module augmentation lets us add fields without losing
// the built-in types that NextAuth provides.
declare module "next-auth" {
  interface Session {
    user: {
      id: string;
      role: string;
}&DefaultSession["user"];
  }
  interface User {
    role: string;
  }
}

// Extend the JWT type separately since tokens and sessions
// are handled by different parts of the authentication flow.
declare module "next-auth/jwt" {
  interface JWT {
    id: string;
    role: string;
  }
}

export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [
    // OAuth providers handle the entire authentication flow automatically.
    // Users are redirected to the provider, authenticate there, and return
    // with an access token that NextAuth exchanges for user information.
    GitHub({
      clientId: process.env.GITHUB_ID!,
      clientSecret: process.env.GITHUB_SECRET!,
    }),
    Google({
      clientId: process.env.GOOGLE_ID!,
      clientSecret: process.env.GOOGLE_SECRET!,
    }),
    // The Credentials provider enables traditional email/password authentication.
    // Unlike OAuth, you're responsible for validating credentials and managing
    // password security—NextAuth only handles session management.
    Credentials({
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" },
      },
      async authorize(credentials) {
        // This function receives the submitted credentials and must return
        // a user object if valid, or null/throw an error if invalid.
        const user = await validateUser(credentials.email, credentials.password);
       
        if (!user) {
          throw new Error("Invalid credentials");
        }

        // The returned object becomes the `user` parameter in the jwt callback.
        return {
          id: user.id,
          email: user.email,
          name: user.name,
          role: user.role || "user",
        };
      },
    }),
  ],
  callbacks: {
    // The jwt callback runs whenever a token is created or updated.
    // Use it to persist custom properties from the user object into the token.
    async jwt({ token, user }) {
      // The `user` object is only available on initial sign-in.
      // On subsequent requests, only the token is available.
      if (user) {
        token.id = user.id;
        token.role = user.role;
      }
      return token;
    },
    // The session callback controls what's exposed to the client.
    // Transfer only the properties you need from the token to the session.
    async session({ session, token }) {
      if (session.user) {
        session.user.id = token.id;
        session.user.role = token.role;
      }
      return session;
    },
  },
  // Custom pages override NextAuth's default UI with your own routes.
  pages: {
    signIn: "/auth/signin",
    error: "/auth/error",
  },
  session: {
    // JWT strategy stores session data in an encrypted cookie rather than
    // a database. This eliminates database lookups but means sessions
    // can't be invalidated server-side without additional infrastructure.
    strategy: "jwt",
    maxAge: 30 * 24 * 60 * 60, // 30 days in seconds
  },
});

// Mock user validation - replace with real database query.
// In production, hash the password with bcrypt and compare against
// the stored hash. Never store or compare plain-text passwords.
async function validateUser(email: string, password: string) {
  return { id: "1", email, name: "User", role: "user" };
}
```

Este archivo de configuración sirve como el núcleo de nuestro sistema de autenticación. Observa cómo hemos definido tres proveedores: GitHub y Google para OAuth, y un proveedor de Credentials para la autenticación tradicional por correo electrónico/contraseña. Los callbacks de JWT extienden la sesión con datos de usuario personalizados, en este caso, un ID de usuario y un rol que utilizaremos para la autorización más adelante.

Las declaraciones de tipos en la parte superior pueden parecer detalladas, pero son esenciales. Le indican a TypeScript exactamente qué datos residen en nuestras sesiones y tokens, evitando errores en tiempo de ejecución y habilitando el autocompletado en toda nuestra aplicación. Sin estas declaraciones, acceder a `session.user.role` generaría errores de tipo.

Ahora necesitamos conectar esta configuración con el sistema de enrutamiento de Next.js. Next.js 16 utiliza el App Router, por lo que crearemos una ruta de API que maneje todas las solicitudes de autenticación: inicio de sesión, cierre de sesión y callbacks de OAuth:

```typescript
// app/api/auth/[...nextauth]/route.ts
import { handlers } from "@/auth";

export const { GET, POST } = handlers;
```

Esta ruta elegantemente simple aprovecha el enrutamiento catch-all de Next.js (`[...nextauth]`) para manejar cada endpoint de autenticación. Los manejadores exportados desde nuestra configuración de autenticación ya saben cómo procesar callbacks de OAuth, emitir sesiones y manejar solicitudes de cierre de sesión. Simplemente los exponemos como manejadores `GET` y `POST`.

### Comprensión de la autenticación basada en sesiones vs. JWT

La elección entre la autenticación basada en sesiones y la autenticación basada en JSON Web Token (JWT) define cómo tu aplicación almacena y valida la identidad del usuario. La autenticación tradicional basada en sesiones almacena los datos de la sesión en una base de datos o en una caché de Redis, guardando solo un ID de sesión en la cookie del usuario. Cada solicitud requiere una búsqueda en la base de datos para validar la sesión y recuperar los datos del usuario.

La autenticación JWT, que hemos configurado en nuestra instalación, incrusta los datos del usuario directamente en un token cifrado. La cookie contiene la sesión completa: ID de usuario, rol y cualquier otra notificación (*claim*) que hayamos agregado. Este enfoque elimina las consultas a la base de datos para la validación de sesiones, lo que lo hace ideal para despliegues *serverless* y en el Edge donde las conexiones a la base de datos son costosas.

La compensación es la flexibilidad. Con la autenticación basada en sesiones, invalidar la sesión de un usuario es inmediato: basta con eliminar el registro de la base de datos. Con los JWT, los tokens siguen siendo válidos hasta su vencimiento, incluso si deseas revocar el acceso. Para la mayoría de las aplicaciones, los beneficios de rendimiento de los JWT superan esta limitación, especialmente cuando se combinan con tiempos de expiración razonables y estrategias de tokens de actualización (*refresh tokens*).

### Protección de rutas con middleware de autenticación

El middleware de autenticación actúa como un guardián, verificando las credenciales antes de que las solicitudes lleguen a la lógica de tu aplicación. En Next.js, el middleware se ejecuta en el Edge, antes de que se rendericen los Server Components y antes de que se ejecuten las rutas de API. Esta posición lo convierte en el lugar perfecto para las comprobaciones de autenticación:

```typescript
// middleware.ts
import { auth } from "@/auth";
import { NextResponse } from "next/server";

// NextAuth's auth function wraps the middleware, providing access to the
// current session via req.auth. This runs on the Edge Runtime, executing
// before every matched request reaches your application.
export default auth((req) => {
  const { pathname } = req.nextUrl;
  const isAuthenticated = !!req.auth;
 
  // Define route groups for different access levels. Using arrays allows
  // easy maintenance—add or remove routes without changing the logic.
  const protectedRoutes = ["/dashboard", "/profile", "/settings"];
  const adminRoutes = ["/admin"];
 
  // Check if the current path falls within protected route groups.
  // startsWith enables matching nested routes like /dashboard/analytics.
  const isProtectedRoute = protectedRoutes.some((route) =>
    pathname.startsWith(route)
  );
  const isAdminRoute = adminRoutes.some((route) =>
    pathname.startsWith(route)
  );

  // Unauthenticated users attempting to access protected routes get
  // redirected to sign-in. The callbackUrl parameter ensures they
  // return to their intended destination after authentication.
  if (isProtectedRoute && !isAuthenticated) {
    const signInUrl = new URL("/auth/signin", req.url);
    signInUrl.searchParams.set("callbackUrl", pathname);
    return NextResponse.redirect(signInUrl);
  }

  // Admin routes require both authentication and the admin role.
  // This two-layer check prevents unauthorized access even if
  // someone guesses the URL structure.
  if (isAdminRoute && (!isAuthenticated || req.auth?.user?.role !== "admin")) {
    return NextResponse.redirect(new URL("/unauthorized", req.url));
  }

  // Prevent authenticated users from accessing sign-in pages.
  // This improves UX by redirecting them to a useful page instead
  // of showing a login form they don't need.
  if (pathname.startsWith("/auth/signin") && isAuthenticated) {
    return NextResponse.redirect(new URL("/dashboard", req.url));
  }

  // No conditions matched—allow the request to proceed normally.
  return NextResponse.next();
});

// The matcher config determines which routes invoke the middleware.
// This regex excludes API routes, static files, and images from
// authentication checks, improving performance for public assets.
export const config = {
  matcher: ["/((?!api|_next/static|_next/image|favicon.ico).*)"],
};
```

Este middleware se ejecuta en cada solicitud que coincida con nuestra configuración, verificando el estado de autenticación antes de que continúe la solicitud. Observa el patrón: definimos arreglos de rutas protegidas, verificamos si la ruta actual coincide y luego redirigimos si es necesario. El parámetro `callbackUrl` asegura que los usuarios regresen a su destino previsto después de iniciar sesión, un pequeño detalle que mejora significativamente la experiencia del usuario.

La comprobación de la ruta de administración introduce una protección basada en roles. No solo nos preguntamos "¿estás autenticado?", sino también "¿tienes los permisos adecuados?". Este patrón se extiende de forma natural a esquemas de autorización más complejos, que exploraremos más adelante en el capítulo.

Con la protección de rutas y la autorización en su lugar, el siguiente paso es definir cómo se autentican los usuarios, lo que nos lleva a configurar los proveedores de autenticación en NextAuth.js.

---

## Integración de proveedores de autenticación en NextAuth.js

Los proveedores de autenticación se dividen en dos categorías: OAuth (inicios de sesión sociales) y basados en credenciales (correo electrónico/contraseña). Cada uno atiende a diferentes casos de uso y presenta desafíos de implementación únicos. OAuth destaca por reducir la fricción; los usuarios aprovechan cuentas existentes sin crear nuevas contraseñas. La autenticación basada en credenciales proporciona control, permitiendo una lógica de validación personalizada y políticas de contraseñas a medida.

### Configuración de autenticación OAuth con Google y GitHub

Los proveedores de OAuth siguen una secuencia bien establecida: redirigir al proveedor, el usuario se autentica, el proveedor redirige de regreso con un código, intercambiar el código por tokens y crear la sesión. NextAuth.js orquesta todo este flujo, pero primero debemos registrar nuestra aplicación en cada proveedor.

Para GitHub, ve a **Settings | Developer settings | OAuth Apps | New OAuth App**. Establece la URL de callback de autorización en `http://localhost:3000/api/auth/callback/github` durante el desarrollo. GitHub proporciona un Client ID y un Client Secret; guárdalos en tus variables de entorno:

```env
// .env.local
GITHUB_ID=your_github_client_id
GITHUB_SECRET=your_github_client_secret
GOOGLE_ID=your_google_client_id
GOOGLE_SECRET=your_google_client_secret
NEXTAUTH_SECRET=generate_a_random_secret
NEXTAUTH_URL=http://localhost:3000
```

Configura el inicio de sesión de Google mediante Google Identity Services (OAuth 2.0) en Google Cloud Console: crea un proyecto, configura la pantalla de consentimiento de OAuth (agrega detalles de la aplicación y usuarios de prueba si es necesario) y crea credenciales de OAuth 2.0 para una aplicación web con el URI de redirección autorizado `http://localhost:3000/api/auth/callback/google` (más tu URI de producción). Utiliza los alcances (*scopes*) predeterminados de OpenID (`openid`, `email`, `profile`), y solo habilita la People API si requieres datos adicionales de perfil o contactos. En tu aplicación NextAuth/Auth.js, proporciona `GOOGLE_ID`, `GOOGLE_SECRET` y `NEXTAUTH_URL`.

El `NEXTAUTH_SECRET` es crucial, ya que cifra los tokens de sesión y firma las cookies. Genera una cadena aleatoria fuerte (de al menos 32 caracteres) y nunca la confirmes en el control de versiones. En producción, utiliza el sistema de variables de entorno de tu proveedor de hosting para inyectar secretos de forma segura.

### Implementación de autenticación basada en credenciales

Mientras que OAuth maneja proveedores de identidad externos, la autenticación basada en credenciales requiere una lógica de validación personalizada. Aquí es donde nos conectamos a nuestra base de datos, verificamos contraseñas y determinamos los roles de los usuarios. La implementación reside en la configuración del proveedor de Credentials que vimos anteriormente, pero examinemos una versión más realista con un hash de contraseña adecuado y consultas a la base de datos:

```typescript
// lib/auth-service.ts
import bcrypt from "bcryptjs";
import { eq } from "drizzle-orm";
import { db } from "@/lib/db";
import { users } from "@/db/schema";

// Define a safe user type that excludes sensitive fields like passwordHash.
// This interface represents what can be safely passed to NextAuth and
// exposed in sessions without leaking internal database details.
export interface AuthUser {
  id: string;
  email: string;
  name: string;
  role: string;
}

export async function validateCredentials(
  email: string,
  password: string
): Promise<AuthUser | null> {
  // Normalize email to lowercase to ensure consistent lookups.
  // This prevents duplicate accounts from case variations like
  // "User@Email.com" and "user@email.com".
  const normalizedEmail = email.toLowerCase();

  // Fetch only the fields needed for authentication. Selecting specific
  // columns rather than the entire row minimizes data transfer and
  // avoids accidentally exposing sensitive fields.
  const rows = await db
    .select({
      id: users.id,
      email: users.email,
      name: users.name,
      role: users.role,
      isActive: users.isActive,
      passwordHash: users.passwordHash,
    })
    .from(users)
    .where(eq(users.email, normalizedEmail))
    .limit(1);

  const user = rows[0];
 
  // Check both existence and active status. Deactivated accounts should
  // fail authentication silently—don't reveal whether the account exists.
  if (!user || !user.isActive) return null;

  // bcrypt.compare handles the salt extraction internally. The hash
  // contains the salt prefix, so we don't need to store it separately.
  // This comparison runs in constant time to prevent timing attacks.
  const isValidPassword = await bcrypt.compare(password, user.passwordHash);
  if (!isValidPassword) return null;

  // Return only safe, non-sensitive user data. The passwordHash is
  // intentionally excluded to prevent accidental exposure in logs,
  // error messages, or API responses.
  return {
    id: user.id,
    email: user.email,
    name: user.name,
    role: user.role,
  };
}

export async function createUser(
  email: string,
  password: string,
  name: string
): Promise<AuthUser> {
  const normalizedEmail = email.toLowerCase();
 
  // Cost factor of 12 provides strong security while keeping hash time
  // reasonable (~250ms). Higher values increase computational cost for
  // both legitimate hashing and brute-force attacks.
  const passwordHash = await bcrypt.hash(password, 12);

  // Drizzle's returning clause retrieves the inserted row in a single
  // query, avoiding a separate SELECT. Only request the fields needed
  // for the AuthUser response.
  const inserted = await db
    .insert(users)
    .values({
      email: normalizedEmail,
      name,
      passwordHash,
      role: "user",
      isActive: true,
    })
    .returning({
      id: users.id,
      email: users.email,
      name: users.name,
      role: users.role,
    });

  // If the email already exists, Drizzle throws a unique constraint error.
  // The calling code should catch this and return a user-friendly message
  // without revealing whether the email is registered (to prevent enumeration).
  const user = inserted[0]!;
  return user;
}

// Track login activity for security auditing and analytics. This enables
// detecting suspicious patterns like logins from new locations or
// identifying inactive accounts for cleanup.
export async function updateUserLastLogin(userId: string): Promise<void> {
  await db
    .update(users)
    .set({ lastLoginAt: new Date() })
    .where(eq(users.id, userId));
}
```

Este módulo de servicio encapsula todas las operaciones de base de datos relacionadas con la autenticación. La comparación de contraseñas se realiza a través de `bcrypt`, que utiliza un algoritmo de hash computacionalmente costoso para evitar ataques de fuerza bruta. El parámetro de rondas de sal (*salt rounds*, 12) equilibra la seguridad con el rendimiento; valores más altos aumentan la seguridad pero ralentizan la autenticación.

Observa cómo nunca devolvemos el hash de la contraseña a la aplicación. Los servicios de autenticación deben tratar las contraseñas como de solo escritura: entran, pero nunca salen. Incluso en los mensajes de error, mantenemos la ambigüedad: "Invalid credentials" en lugar de "Wrong password" evita que los atacantes enumeren direcciones de correo electrónico válidas.

### Manejo de inicios de sesión sociales vs. tradicionales

Los usuarios que se autentican mediante OAuth no tienen contraseñas en nuestro sistema. Esto crea un desafío interesante: ¿qué sucede si un usuario se registra con GitHub y luego intenta iniciar sesión con el mismo correo electrónico a través de credenciales? ¿O viceversa? Las estrategias de vinculación de cuentas (*account linking*) manejan estos escenarios, pero requieren un diseño cuidadoso para evitar vulnerabilidades de seguridad.

Auth.js no proporciona un callback `link` para la vinculación de cuentas. De forma predeterminada, cuando un usuario no autenticado inicia sesión a través de un proveedor de OAuth y ya existe otra cuenta con la misma dirección de correo electrónico, Auth.js no vincula automáticamente las cuentas. Esta restricción evita la apropiación de cuentas porque los proveedores de OAuth no garantizan universalmente que las direcciones de correo electrónico devueltas hayan sido verificadas de forma segura. La vinculación automática por correo electrónico idéntico se puede habilitar para un proveedor específico con `allowDangerousEmailAccountLinking: true`, pero solo cuando la aplicación confía explícitamente en el proceso de verificación de correo electrónico de ese proveedor. Para una mayor seguridad, las aplicaciones deben exigir a los usuarios que se autentiquen con su cuenta existente antes de conectar un proveedor adicional:

```typescript
// lib/account-linking.ts
import { and, eq } from "drizzle-orm";
import { db } from "@/lib/db";
import { accounts, users } from "@/db/schema";

type LinkedAccount = { provider: string; providerAccountId: string };

type UserWithAccounts = {
  id: string;
  email: string;
  name: string;
  role: string;
  isActive: boolean;
  emailVerified: Date | null;
  accounts: LinkedAccount[];
};

// Links an OAuth provider to an existing user. Call this when a signed-in
// user connects an additional provider from their account settings.
export async function linkOAuthAccount(
  userId: string,
  provider: string,
  providerAccountId: string
): Promise<LinkedAccount> {
  const [inserted] = await db
    .insert(accounts)
    .values({ userId, provider, providerAccountId, type: "oauth" })
    .returning({ provider: accounts.provider, providerAccountId: accounts.providerAccountId });

  return inserted!;
}

// Returns all OAuth provider names linked to a user (e.g., ["github", "google"]).
export async function getUserProviders(userId: string): Promise<string[]> {
  const rows = await db
    .select({ provider: accounts.provider })
    .from(accounts)
    .where(eq(accounts.userId, userId));

  return rows.map((r) => r.provider);
}

// Core OAuth sign-in handler. Implements automatic account linking:
// - Returns existing user if provider is already linked
// - Links provider to existing account with matching email
// - Creates new user if no account exists
// Transaction prevents race conditions on simultaneous first-time sign-ins.
export async function findOrCreateOAuthUser(
  email: string,
  name: string,
  provider: string,
  providerAccountId: string
): Promise<UserWithAccounts> {
  const normalizedEmail = email.toLowerCase();

  return await db.transaction(async (tx) => {
    // Check for existing user with this email
    const [existingUser] = await tx
      .select()
      .from(users)
      .where(eq(users.email, normalizedEmail))
      .limit(1);

    // Use existing user ID or create new user and get their ID
    const userId = existingUser?.id ?? (await createUser(tx, normalizedEmail, name));

    if (existingUser) {
      // Check if this specific provider/account combo is already linked
      const [linked] = await tx
        .select({ id: accounts.id })
        .from(accounts)
        .where(
          and(
            eq(accounts.userId, userId),
            eq(accounts.provider, provider),
            eq(accounts.providerAccountId, providerAccountId)
          )
        )
        .limit(1);

      // Link provider only if not already connected
      if (!linked) {
        await tx.insert(accounts).values({ userId, provider, providerAccountId, type: "oauth" });
      }
    } else {
      // New user—link their first OAuth provider
      await tx.insert(accounts).values({ userId, provider, providerAccountId, type: "oauth" });
    }

    // Return complete user object with all linked accounts
    const [user] = await tx.select().from(users).where(eq(users.id, userId));
    const linkedAccounts = await tx
      .select({ provider: accounts.provider, providerAccountId: accounts.providerAccountId })
      .from(accounts)
      .where(eq(accounts.userId, userId));

    return { ...user!, accounts: linkedAccounts };
  });
}

// Creates a new OAuth user within a transaction. Email is marked verified
// since OAuth providers have already confirmed ownership.
async function createUser(tx: any, email: string, name: string): Promise<string> {
  const [created] = await tx
    .insert(users)
    .values({ email, name, role: "user", isActive: true, emailVerified: new Date() })
    .returning({ id: users.id });

  return created!.id;
}

// Unlinks an OAuth provider from a user account. Includes safety check
// to prevent users from removing their only authentication method.
export async function unlinkOAuthAccount(
  userId: string,
  provider: string
): Promise<{ success: boolean; error?: string }> {
  // Fetch account count and password status in parallel for efficiency
  const [accountCount, userData] = await Promise.all([
    db.select({ id: accounts.id }).from(accounts).where(eq(accounts.userId, userId)),
    db.select({ passwordHash: users.passwordHash }).from(users).where(eq(users.id, userId)).limit(1),
  ]);

  // Block removal if this is the only auth method (no password + single OAuth)
  if (accountCount.length <= 1 && !userData[0]?.passwordHash) {
    return { success: false, error: "Cannot remove your only sign-in method" };
  }

  await db.delete(accounts).where(and(eq(accounts.userId, userId), eq(accounts.provider, provider)));

  return { success: true };
}
```

Este módulo demuestra una estrategia segura de vinculación de cuentas. Cuando un usuario inicia sesión con OAuth, verificamos si ya existe una cuenta con ese correo electrónico. Si es así, vinculamos el proveedor de OAuth a la cuenta existente. Si no, creamos un nuevo usuario. La medida de seguridad clave: solo vinculamos automáticamente cuentas de OAuth y marcamos los correos verificados por OAuth como verificados, ya que el proveedor de OAuth ya ha confirmado la propiedad del correo electrónico.

### Aseguramiento de flujos OAuth y gestión de perfiles de usuario

Los flujos de OAuth introducen varias consideraciones de seguridad más allá de la autenticación básica. Los parámetros `state` previenen ataques CSRF, las extensiones PKCE protegen contra la interceptación de códigos y la validación adecuada del URI de redirección previene vulnerabilidades de redirección abierta. NextAuth.js maneja la mayoría de estos aspectos automáticamente, pero comprender los mecanismos ayuda al depurar problemas o implementar proveedores personalizados.

La gestión de perfiles de usuario se vuelve más compleja con múltiples proveedores. Cada proveedor de OAuth devuelve datos de usuario diferentes: GitHub proporciona un nombre de usuario y una URL de avatar, mientras que Google proporciona un correo electrónico verificado y una foto de perfil. Necesitamos normalizar estos datos y manejar las actualizaciones cuando los usuarios cambian sus perfiles en el nivel del proveedor:

```typescript
// lib/profile-sync.ts
import { eq } from "drizzle-orm";
import { db } from "@/lib/db";
import { users } from "@/db/schema";

// Standardized profile structure across all OAuth providers.
// Each provider returns data in different formats—this interface
// normalizes them for consistent handling.
interface ProviderProfile {
  email: string;
  name?: string;
  image?: string;
  emailVerified?: boolean;
}

// Updates local user record with fresh data from an OAuth provider.
// Called during sign-in to keep profile info current (e.g., if user
// changes their GitHub avatar, it reflects here on next login).
export async function syncProviderProfile(
  userId: string,
  provider: string,
  profile: ProviderProfile
): Promise<void> {
  const updateData: Partial<{
    name: string;
    image: string;
    emailVerified: Date | null;
  }> = {};

  // Only include fields that have values to avoid overwriting
  // existing data with null/undefined
  if (profile.name) updateData.name = profile.name;
  if (profile.image) updateData.image = profile.image;
  if (profile.emailVerified) updateData.emailVerified = new Date();

  // Skip database call if there's nothing to update
  if (Object.keys(updateData).length > 0) {
    await db.update(users).set(updateData).where(eq(users.id, userId));
  }
}

// Transforms provider-specific profile formats into our standard structure.
// Each OAuth provider returns different field names (e.g., GitHub uses
// "avatar_url" while Google uses "picture" for the profile image).
export async function normalizeOAuthProfile(
  provider: string,
  rawProfile: any
): Promise<ProviderProfile> {
  switch (provider) {
    case "github":
      return {
        email: rawProfile.email,
        name: rawProfile.name || rawProfile.login, // Fallback to username if name is empty
        image: rawProfile.avatar_url,
        emailVerified: true, // GitHub verifies emails before allowing OAuth
      };

    case "google":
      return {
        email: rawProfile.email,
        name: rawProfile.name,
        image: rawProfile.picture,
        emailVerified: rawProfile.email_verified, // Google provides explicit verification status
      };

    default:
      // Conservative defaults for unknown providers—don't assume email is verified
      return {
        email: rawProfile.email,
        name: rawProfile.name,
        image: rawProfile.image,
        emailVerified: false,
      };
  }
}

// Merges new OAuth profile data with existing user data, preserving
// existing values when new ones aren't available. Prevents OAuth
// sign-ins from accidentally clearing previously set profile fields.
export async function mergeProfileData(
  existingUser: {
    name?: string | null;
    image?: string | null;
    emailVerified?: Date | null;
  },
  newProfile: ProviderProfile
): Promise<{
  name: string | null;
  image: string | null;
  emailVerified: Date | null;
}> {
  return {
    // Prefer new data, fall back to existing, then null
    name: newProfile.name ?? existingUser.name ?? null,
    image: newProfile.image ?? existingUser.image ?? null,
    // Convert boolean to Date if newly verified, otherwise keep existing
    emailVerified: newProfile.emailVerified
      ? new Date()
      : existingUser.emailVerified ?? null,
  };
}
```

La sincronización de perfiles garantiza que los datos de los usuarios se mantengan actualizados en todos los inicios de sesión. Cada proveedor estructura los datos de perfil de manera diferente, por lo que los normalizamos en un formato coherente. La estrategia de combinación prioriza los datos más nuevos y conserva la información existente cuando no hay nuevos datos disponibles; los usuarios no deberían perder su foto de perfil solo porque GitHub no proporciona una al iniciar sesión.

Después de garantizar que los perfiles de usuario sigan siendo coherentes en todos los inicios de sesión, el siguiente paso es aprovechar esos datos durante el renderizado, comenzando con la autenticación en RSC.

---

## Manejo de autenticación en RSC

Los React Server Components (RSC) revolucionan la forma en que pensamos sobre la autenticación en las aplicaciones Next.js. Las aplicaciones tradicionales de React verificaban la autenticación en el cliente, creando un parpadeo de contenido no autenticado antes de que JavaScript determinara el estado del usuario. Los Server Components se autentican antes de renderizar, entregando un HTML completamente personalizado en la primera carga.

### Obtención de sesiones de usuario en componentes de servidor

Los Server Components acceden al estado de autenticación a través de la función `auth` que exportamos desde nuestra configuración. Esta función lee la sesión de las cookies de la solicitud, valida el JWT y devuelve los datos del usuario. Debido a que es asíncrona, usamos `await` dentro de nuestros Server Components.

Primero creemos nuestra API en `app/api/dashboard/route.ts`:

```typescript
// app/api/dashboard/route.ts
import { NextResponse } from "next/server";
import { desc, eq } from "drizzle-orm";
import { auth } from "@/auth";
import { db } from "@/lib/db";
import { users, projects, activities } from "@/db/schema";

// Use Node.js runtime for full Drizzle ORM support. Switch to "edge"
// if your database driver supports it for faster cold starts.
export const runtime = "nodejs";

// Returns dashboard data: user profile, recent projects, and activity feed.
// Single endpoint reduces client-side request waterfall on page load.
export async function GET() {
  // Authenticate request using NextAuth's server-side session
  const session = await auth();
  if (!session?.user?.id) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const userId = session.user.id;

  // Verify user exists in database. Session could reference a deleted user
  // if account was removed but session token hasn't expired yet.
  const [user] = await db
    .select({
      id: users.id,
      name: users.name,
      email: users.email,
    })
    .from(users)
    .where(eq(users.id, userId))
    .limit(1);

  if (!user) {
    return NextResponse.json({ error: "Not found" }, { status: 404 });
  }

  // Fetch projects and activities concurrently. Promise.all cuts response
  // time roughly in half compared to sequential queries.
  const [proj, acts] = await Promise.all([
    db
      .select({
        id: projects.id,
        name: projects.name,
        updatedAt: projects.updatedAt,
      })
      .from(projects)
      .where(eq(projects.userId, userId))
      .orderBy(desc(projects.updatedAt))
      .limit(5), // Most recent projects for quick access

    db
      .select({
        id: activities.id,
        type: activities.type,
        createdAt: activities.createdAt,
        message: activities.message,
      })
      .from(activities)
      .where(eq(activities.userId, userId))
      .orderBy(desc(activities.createdAt))
      .limit(10), // Activity feed for dashboard timeline
  ]);

  return NextResponse.json({
    user,
    projects: proj,
    activities: acts,
  });
}
```

Luego este es nuestro RSC:

```tsx
// app/dashboard/page.tsx
import { auth } from "@/auth";
import { ActivityFeed } from "@/components/activity-feed";
import { UserProfile } from "@/components/user-profile";
import { db } from "@/lib/db";
import { redirect } from "next/navigation";

type DashboardData = {
  user: {
    id: string;
    name: string | null;
    email: string | null;
  };
  projects: Array<{
    id: string;
    name: string;
    updatedAt: string;
  }>;
  activities: Array<{
    id: string;
    type: string;
    createdAt: string;
    message: string | null;
  }>;
};

export const dynamic = "force-dynamic";

async function getDashboardData(
  userId: string,
): Promise<DashboardData | null> {
  const [user, projects, activities] = await db.$transaction([
    db.user.findUnique({
      where: {
        id: userId,
      },
      select: {
        id: true,
        name: true,
        email: true,
      },
    }),

    db.project.findMany({
      where: {
        userId,
      },
      orderBy: {
        updatedAt: "desc",
      },
      select: {
        id: true,
        name: true,
        updatedAt: true,
      },
    }),

    db.activity.findMany({
      where: {
        userId,
      },
      orderBy: {
        createdAt: "desc",
      },
      take: 20,
      select: {
        id: true,
        type: true,
        createdAt: true,
        message: true,
      },
    }),
  ]);

  if (!user) {
    return null;
  }

  return {
    user,
    projects: projects.map((project) => ({
      ...project,
      updatedAt: project.updatedAt.toISOString(),
    })),
    activities: activities.map((activity) => ({
      ...activity,
      createdAt: activity.createdAt.toISOString(),
    })),
  };
}

export default async function DashboardPage() {
  const session = await auth();

  if (!session?.user?.id) {
    redirect("/auth/signin");
  }

  const data = await getDashboardData(session.user.id);

  // Handles a stale session whose user was removed from the database.
  if (!data) {
    redirect("/auth/signin");
  }

  return (
    <div className="min-h-screen bg-gray-50">
      <header className="border-b border-gray-200 bg-white">
        <div className="mx-auto max-w-7xl px-4 py-4 sm:px-6 lg:px-8">
          <h1 className="text-2xl font-bold text-gray-900">
            Welcome back, {data.user.name ?? "friend"}
          </h1>
        </div>
      </header>

      <main className="mx-auto max-w-7xl px-4 py-8 sm:px-6 lg:px-8">
        <div className="grid grid-cols-1 gap-8 lg:grid-cols-3">
          <div className="lg:col-span-2">
            <ActivityFeed activities={data.activities} />
          </div>

          <div>
            <UserProfile
              user={data.user}
              projects={data.projects}
            />
          </div>
        </div>
      </main>
    </div>
  );
}
```

Este Server Component demuestra la obtención de datos con autenticación previa (*authentication-first data fetching*). Verificamos la sesión antes de consultar la base de datos, asegurándonos de nunca obtener datos para usuarios no autenticados. La redirección ocurre en el servidor, por lo que el navegador nunca ve contenido protegido.

El patrón es elegante: autenticar, obtener datos para ese usuario específico y renderizar. Sin estados de carga para la autenticación, sin parpadeos de contenido erróneo, sin redirecciones en el cliente. El usuario recibe una página completamente renderizada y personalizada en la primera carga.

### Autenticación de solicitudes a la API con Server Actions

Las Server Actions introducen un nuevo paradigma para manejar mutaciones en Next.js. Estas funciones se ejecutan en el servidor pero se llaman directamente desde Client Components, eliminando la necesidad de rutas de API explícitas para muchas operaciones. La autenticación en Server Actions sigue el mismo patrón que en los Server Components: leer la sesión, validar permisos y ejecutar la acción:

```typescript
// app/actions/project-actions.ts
"use server";

import { and, eq } from "drizzle-orm";
import { revalidatePath } from "next/cache";
import { z } from "zod";
import { auth } from "@/auth";
import { db } from "@/lib/db";
import { projects } from "@/db/schema";

// Zod schema validates input before database operations. Server Actions
// receive untrusted input—always validate even if client-side checks exist.
const projectSchema = z.object({
  name: z.string().min(3).max(100),
  description: z.string().max(500).optional(),
});

// Extracts string value from FormData, handling null and File edge cases.
// FormData.get() returns FormDataEntryValue (string | File) or null.
const fdString = (v: FormDataEntryValue | null) =>
  v === null ? undefined : typeof v === "string" ? v : (v as File).name;

// Shared auth check for all actions. Throws early to prevent any
// database operations from running for unauthenticated requests.
async function requireAuth(): Promise<string> {
  const session = await auth();
  if (!session?.user?.id) throw new Error("Unauthorized");
  return session.user.id;
}

// Shared form parsing logic. Extracts and validates project fields,
// throwing a ZodError if validation fails.
function parseProjectForm(formData: FormData) {
  return projectSchema.parse({
    name: fdString(formData.get("name")),
    description: fdString(formData.get("description")),
  });
}

// Creates a new project owned by the authenticated user.
// Returns the new project ID for client-side navigation.
export async function createProject(formData: FormData) {
  const userId = await requireAuth();
  const data = parseProjectForm(formData);

  const [project] = await db
    .insert(projects)
    .values({ ...data, userId })
    .returning({ id: projects.id });

  if (!project) throw new Error("Failed to create project");

  revalidatePath("/dashboard"); // Refresh dashboard to show new project
  return { success: true, projectId: project.id };
}

// Updates project only if owned by authenticated user. Combining ownership
// check with update in single query prevents TOCTOU race conditions.
export async function updateProject(projectId: string, formData: FormData) {
  const userId = await requireAuth();
  const data = parseProjectForm(formData);

  const [updated] = await db
    .update(projects)
    .set(data)
    .where(and(eq(projects.id, projectId), eq(projects.userId, userId)))
    .returning({ id: projects.id });

  if (!updated) throw new Error("Forbidden"); // No match = wrong owner or missing project

  revalidatePath("/dashboard");
  revalidatePath(`/projects/${projectId}`); // Refresh project detail page too
  return { success: true };
}

// Deletes project only if owned by authenticated user. Empty result
// means either project doesn't exist or belongs to another user.
export async function deleteProject(projectId: string) {
  const userId = await requireAuth();

  const [deleted] = await db
    .delete(projects)
    .where(and(eq(projects.id, projectId), eq(projects.userId, userId)))
    .returning({ id: projects.id });

  if (!deleted) throw new Error("Forbidden");

  revalidatePath("/dashboard");
  return { success: true };
}
```

Las Server Actions difuminan la línea entre el código del cliente y del servidor. Estas funciones están marcadas con `'use server'` y se pueden importar directamente en Client Components, pero se ejecutan en el servidor con acceso completo a la base de datos. Las comprobaciones de autenticación ocurren en cada invocación; aunque el código del cliente llame a estas funciones, se ejecutan en el servidor donde la validación de la sesión es segura.

La verificación de propiedad demuestra un patrón de seguridad crucial: autenticar al usuario no es suficiente. Debemos verificar que tenga permiso para modificar el recurso específico. Cualquiera podría llamar a `deleteProject` con cualquier ID de proyecto; la verificación de propiedad garantiza que solo puedan eliminar sus propios proyectos.

### Aseguramiento de rutas de API con middleware de NextAuth.js

Si bien las Server Actions manejan muchas mutaciones, las rutas de API tradicionales todavía tienen propósitos importantes: webhooks, integraciones de terceros y operaciones que necesitan semántica HTTP explícita. Estas rutas también requieren autenticación, y el patrón refleja el de los Server Components: leer la sesión, validar y responder:

```typescript
// app/api/projects/[id]/route.ts
import { eq } from "drizzle-orm";
import { NextRequest, NextResponse } from "next/server";

import { auth } from "@/auth";
import { projects, users } from "@/db/schema";
import { db } from "@/lib/db";

type RouteContext = {
  params: Promise<{ id: string }>;
};

const unauthorized = () =>
  NextResponse.json({ error: "Unauthorized" }, { status: 401 });

const notFound = () =>
  NextResponse.json({ error: "Not found" }, { status: 404 });

const forbidden = () =>
  NextResponse.json({ error: "Forbidden" }, { status: 403 });

async function getAuthUser() {
  const session = await auth();
  return session?.user ?? null;
}

function canAccess(
  projectUserId: string,
  user: { id: string; role?: string },
) {
  return projectUserId === user.id || user.role === "admin";
}

export async function GET(
  _request: NextRequest,
  { params }: RouteContext,
) {
  const { id } = await params;

  const user = await getAuthUser();

  if (!user) {
    return unauthorized();
  }

  const [row] = await db
    .select({
      project: {
        id: projects.id,
        userId: projects.userId,
        name: projects.name,
        description: projects.description,
        createdAt: projects.createdAt,
        updatedAt: projects.updatedAt,
      },
      user: {
        id: users.id,
        name: users.name,
        email: users.email,
      },
    })
    .from(projects)
    .leftJoin(users, eq(users.id, projects.userId))
    .where(eq(projects.id, id))
    .limit(1);

  if (!row) {
    return notFound();
  }

  if (!canAccess(row.project.userId, user)) {
    return forbidden();
  }

  return NextResponse.json({
    ...row.project,
    user: row.user,
  });
}

export async function PATCH(
  request: NextRequest,
  { params }: RouteContext,
) {
  const { id } = await params;

  const user = await getAuthUser();

  if (!user) {
    return unauthorized();
  }

  const [existing] = await db
    .select({ userId: projects.userId })
    .from(projects)
    .where(eq(projects.id, id))
    .limit(1);

  if (!existing) {
    return notFound();
  }

  if (!canAccess(existing.userId, user)) {
    return forbidden();
  }

  const body = await request.json();

  const [updated] = await db
    .update(projects)
    .set({
      name: body.name,
      description: body.description,
      updatedAt: new Date(),
    })
    .where(eq(projects.id, id))
    .returning({
      id: projects.id,
      userId: projects.userId,
      name: projects.name,
      description: projects.description,
      createdAt: projects.createdAt,
      updatedAt: projects.updatedAt,
    });

  return NextResponse.json(updated);
}

export async function DELETE(
  _request: NextRequest,
  { params }: RouteContext,
) {
  const { id } = await params;

  const user = await getAuthUser();

  if (!user) {
    return unauthorized();
  }

  const [existing] = await db
    .select({ userId: projects.userId })
    .from(projects)
    .where(eq(projects.id, id))
    .limit(1);

  if (!existing) {
    return notFound();
  }

  if (!canAccess(existing.userId, user)) {
    return forbidden();
  }

  await db.delete(projects).where(eq(projects.id, id));

  return NextResponse.json({ success: true });
}
```

Las rutas de API siguen un patrón de autenticación coherente en todos los métodos HTTP. Cada manejador comienza validando la sesión y luego verifica los permisos a nivel de recurso antes de continuar. Observa la distinción entre autenticación (401 Unauthorized) y autorización (403 Forbidden), sutil pero importante para la semántica de la API.

La comprobación de roles introduce nuestro primer vistazo al control de acceso basado en roles. Los administradores pueden modificar cualquier proyecto, mientras que los usuarios normales solo pueden modificar los suyos. Este patrón se escala a sistemas de permisos complejos, que exploraremos en detalle en breve.

### Almacenamiento y acceso a datos de autenticación de usuario

Los datos de la sesión fluyen a través de tu aplicación como un río: disponibles donde los necesites, consistentes en todas las solicitudes y seguros por defecto. Comprender dónde residen los datos de la sesión y cómo acceder a ellos evita errores comunes y permite patrones potentes.

En los Server Components y las Server Actions, hemos visto la función `auth()`. Esta lee la cookie de sesión cifrada, valida la firma JWT y devuelve el objeto de sesión. Para los Client Components, el enfoque difiere ligeramente; necesitamos acceder a los datos de la sesión sin ejecución en el servidor:

```tsx
// app/providers/session-provider.tsx
"use client";

import { SessionProvider as NextAuthSessionProvider } from "next-auth/react";
import { Session } from "next-auth";

interface Props {
  children: React.ReactNode;
  session: Session | null;
}

// Client-side wrapper for NextAuth's SessionProvider. This component must
// be marked "use client" since SessionProvider uses React Context internally.
//
// Accepts an optional session prop to hydrate the client with server-fetched
// session data, avoiding an extra round-trip on initial page load. When session
// is passed from a Server Component, the client renders immediately with
// authentication state instead of showing a loading state.
export function SessionProvider({ children, session }: Props) {
  return (
    <NextAuthSessionProvider session={session}>
      {children}
    </NextAuthSessionProvider>
  );
}
```

Este proveedor envuelve nuestra aplicación, haciendo que los datos de la sesión estén disponibles para todos los Client Components a través del hook `useSession`. La prop `session` se pasa desde un Server Component, lo que garantiza que tengamos datos de sesión en el render inicial sin necesidad de obtenerlos en el cliente:

```tsx
// app/layout.tsx
import { auth } from "@/auth";
import { SessionProvider } from "@/app/providers/session-provider";
import { Navigation } from "@/components/navigation";
import "./globals.css";

export default async function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const session = await auth();

  return (
    <html lang="en">
      <body>
        <SessionProvider session={session}>
          <Navigation />
          {children}
        </SessionProvider>
      </body>
    </html>
  );
}
```

El layout raíz obtiene la sesión en el servidor y la pasa al proveedor. Esto garantiza que cada página tenga acceso a los datos de la sesión sin solicitudes de red adicionales. Los Client Components pueden luego usar el hook `useSession` para acceder a estos datos de manera reactiva:

```tsx
// components/user-menu.tsx
"use client";

import { useSession, signOut } from "next-auth/react";
import { useState } from "react";
import Link from "next/link";

// Dropdown menu displaying user info and navigation links. Handles three states:
// loading (skeleton), unauthenticated (sign-in button), authenticated (full menu).
export function UserMenu() {
  const { data: session, status } = useSession();
  const [isOpen, setIsOpen] = useState(false);

  // Show skeleton placeholder while session loads to prevent layout shift
  if (status === "loading") {
    return <div className="w-8 h-8 bg-gray-200 rounded-full animate-pulse" />;
  }

  // Unauthenticated users see a sign-in CTA instead of the menu
  if (!session?.user) {
    return (
      <Link
        href="/auth/signin"
        className="px-4 py-2 text-sm font-medium text-white bg-blue-600 rounded-lg hover:bg-blue-700 transition-colors"
      >
        Sign In
      </Link>
    );
  }

  const { user } = session;

  return (
    <div className="relative">
      {/* Clickable avatar and name toggles dropdown visibility */}
      <button
        onClick={() => setIsOpen(!isOpen)}
        className="flex items-center gap-2 p-2 rounded-lg hover:bg-gray-100 transition-colors"
      >
        <img
          src={user.image || "/default-avatar.png"}
          alt={user.name || "User"}
          className="w-8 h-8 rounded-full"
        />
        <span className="text-sm font-medium">{user.name}</span>
      </button>

     {/* Dropdown renders only when open. Positioned absolute to overlay content below. */}
      {isOpen && (
        <div className="absolute right-0 mt-2 w-48 bg-white rounded-lg shadow-lg border border-gray-200 py-1">
          {/* User info header showing email and role */}
          <div className="px-4 py-2 border-b border-gray-100">
            <p className="text-sm text-gray-500">{user.email}</p>
            <p className="text-xs text-gray-400 mt-1">Role: {user.role}</p>
          </div>

          {/* Navigation links available to all authenticated users */}
          <Link href="/profile" className="block px-4 py-2 text-sm text-gray-700 hover:bg-gray-50">
            Profile
          </Link>
          <Link href="/settings" className="block px-4 py-2 text-sm text-gray-700 hover:bg-gray-50">
            Settings
          </Link>

          {/* Conditionally render admin link based on user role */}
          {user.role === "admin" && (
            <Link href="/admin" className="block px-4 py-2 text-sm text-blue-600 hover:bg-blue-50">
              Admin Panel
            </Link>
          )}

          {/* Sign out redirects to home page after session is cleared */}
          <button
            onClick={() => signOut({ callbackUrl: "/" })}
            className="block w-full text-left px-4 py-2 text-sm text-red-600 hover:bg-red-50 border-t border-gray-100 mt-1"
          >
            Sign Out
          </button>
        </div>
      )}
    </div>
  );
}
```

Este Client Component demuestra el acceso reactivo a la sesión. El hook `useSession` proporciona la sesión actual y el estado de carga, lo que permite transiciones de interfaz suaves. Observa cómo renderizamos condicionalmente el enlace de administrador según el rol del usuario (autorización del lado del cliente para la visibilidad de la interfaz de usuario), aunque seguiremos aplicando los permisos en el servidor.

El patrón de comprobar `status === "loading"` evita el cambio de diseño (*layout shift*) durante la resolución del estado de autenticación. Los usuarios ven un esqueleto de carga (*skeleton*) en lugar de que el contenido parpadee entre estados autenticados y no autenticados.

Si bien los Client Components pueden renderizarse condicionalmente según los datos de la sesión, la verdadera autorización requiere un modelo de permisos centralizado, lo que nos lleva a implementar RBAC.

---

## Implementación de control de acceso basado en roles (RBAC) y permisos

El control de acceso basado en roles (RBAC) transforma la autenticación binaria (sesión iniciada o no) en un espectro de permisos. Los usuarios tienen roles (administrador, editor, espectador) y cada rol otorga capacidades específicas. RBAC separa quién puede acceder a qué, lo que permite una seguridad detallada sin una gestión compleja de permisos por usuario.

### Definición de roles de usuario y niveles de acceso

Un sistema de roles bien diseñado equilibra la granularidad con la simplicidad. Con muy pocos roles, necesitarás excepciones constantemente. Con demasiados, la gestión se vuelve abrumadora. La mayoría de las aplicaciones prosperan con tres a cinco roles principales, cada uno de los cuales representa un conjunto distinto de permisos:

```typescript
// lib/rbac/roles.ts
// Application roles ordered by privilege level (highest to lowest).
// Used as the primary identifier for user access levels.
export enum Role {
  ADMIN = "admin",
  EDITOR = "editor",
  USER = "user",
  GUEST = "guest",
}

// Granular permissions following "resource:action" naming convention.
// This pattern makes it easy to group and identify related permissions.
export enum Permission {
  // Project permissions
  PROJECT_CREATE = "project:create",
  PROJECT_READ = "project:read",
  PROJECT_UPDATE = "project:update",
  PROJECT_DELETE = "project:delete",

  // User management permissions
  USER_READ = "user:read",
  USER_UPDATE = "user:update",
  USER_DELETE = "user:delete",
  USER_MANAGE_ROLES = "user:manage_roles",

  // Admin-only permissions
  ADMIN_ACCESS = "admin:access",
  ADMIN_SETTINGS = "admin:settings",
}

// Maps each role to its allowed permissions. Higher roles typically
// include all permissions of lower roles plus additional privileges.
// Modify this matrix to adjust access control across the application.
const rolePermissions: Record<Role, Permission[]> = {
  [Role.ADMIN]: [
    Permission.PROJECT_CREATE,
    Permission.PROJECT_READ,
    Permission.PROJECT_UPDATE,
    Permission.PROJECT_DELETE,
    Permission.USER_READ,
    Permission.USER_UPDATE,
    Permission.USER_DELETE,
    Permission.USER_MANAGE_ROLES,
    Permission.ADMIN_ACCESS,
    Permission.ADMIN_SETTINGS,
  ],
  [Role.EDITOR]: [
    Permission.PROJECT_CREATE,
    Permission.PROJECT_READ,
    Permission.PROJECT_UPDATE,
    Permission.USER_READ,
  ],
  [Role.USER]: [
    Permission.PROJECT_CREATE,
    Permission.PROJECT_READ,
    Permission.PROJECT_UPDATE,
  ],
  [Role.GUEST]: [
    Permission.PROJECT_READ,
  ],
};

// Checks if a role has a specific permission. Returns false for
// unknown roles, providing safe default behavior.
export function hasPermission(role: Role, permission: Permission): boolean {
  return rolePermissions[role]?.includes(permission) ?? false;
}


// Returns true if role has at least one of the specified permissions.
// Useful for UI elements that should show if user can do any related action.
export function hasAnyPermission(role: Role, permissions: Permission[]): boolean {
  return permissions.some((permission) => hasPermission(role, permission));
}

// Returns true only if role has all specified permissions.
// Use for actions requiring multiple permissions simultaneously.
export function hasAllPermissions(role: Role, permissions: Permission[]): boolean {
  return permissions.every((permission) => hasPermission(role, permission));
}

// Returns the complete permission list for a role. Useful for
// debugging or displaying user capabilities in admin interfaces.
export function getRolePermissions(role: Role): Permission[] {
  return rolePermissions[role] || [];
}
```

Este sistema de permisos asigna roles a capacidades a través de una matriz explícita. En lugar de verificar `if (role === "admin")` en todo tu código, verificas `if (hasPermission(role, Permission.PROJECT_DELETE))`. Esta abstracción permite cambiar los permisos de los roles sin tocar la lógica de negocio.

El enfoque basado en enums proporciona seguridad de tipos y autocompletado. TypeScript garantiza que no puedas comprobar un permiso que no existe, y las herramientas de refactorización pueden rastrear el uso de permisos en toda tu base de código. Las funciones auxiliares `hasAnyPermission` y `hasAllPermissions` manejan escenarios comunes sin duplicar lógica.

### Restricción de componentes de UI basados en roles

La autorización de UI evita que los usuarios vean controles que no pueden usar. Un editor no debería ver el botón "Delete All Users" porque carece de ese permiso. Esto no es seguridad en sí (los usuarios pueden inspeccionar el DOM y ver elementos ocultos), pero es una buena experiencia de usuario (UX). La verdadera seguridad ocurre en el servidor; la autorización de UI guía a los usuarios hacia acciones válidas:

```tsx
// components/rbac/can.tsx
"use client";

import { useSession } from "next-auth/react";
import { hasPermission, hasAnyPermission, Permission, Role } from "@/lib/rbac/roles";

interface CanProps {
  permission?: Permission;          // Single permission to check
  permissions?: Permission[];       // Multiple permissions to check
  requireAll?: boolean;    // If true, all permissions required; if false, any one suffices
  fallback?: React.ReactNode;       // Content to show when access denied
  children: React.ReactNode;
}

// Declarative permission gate that conditionally renders children based on
// user's role permissions. Simplifies authorization checks in JSX without
// cluttering components with permission logic.
//
// Usage examples:
//   <Can permission={Permission.PROJECT_DELETE}><DeleteButton /></Can>
//   <Can permissions={[Permission.USER_READ, Permission.USER_UPDATE]} requireAll>...</Can>
export function Can({
  permission,
  permissions,
  requireAll = false,
  fallback = null,
  children,
}: CanProps) {
  const { data: session } = useSession();

  // No session or role means no permissions—show fallback
  if (!session?.user?.role) {
    return <>{fallback}</>;
  }

  const userRole = session.user.role as Role;
  let hasAccess = false;

  if (permission) {
    // Single permission check
    hasAccess = hasPermission(userRole, permission);
  } else if (permissions) {
    // Multiple permissions: requireAll determines AND vs OR logic
    hasAccess = requireAll
      ? permissions.every((p) => hasPermission(userRole, p))
      : hasAnyPermission(userRole, permissions);
  }

  return hasAccess ? <>{children}</> : <>{fallback}</>;
}
```

Ahora creemos el componente opuesto `Cannot`:

```tsx
interface CannotProps {
  permission?: Permission;
  permissions?: Permission[];
  children: React.ReactNode;
}

// Inverse of Can—renders children only when user lacks the specified permissions.
// Useful for showing upgrade prompts, restricted notices, or alternative UI
// for users without certain access levels.
//
// Usage: <Cannot permission={Permission.ADMIN_ACCESS}><UpgradePrompt /></Cannot>
export function Cannot({ permission, permissions, children }: CannotProps) {
  const { data: session } = useSession();

  // No session means no permissions—user "cannot" do anything, so show children
  if (!session?.user?.role) {
    return <>{children}</>;
  }

  const userRole = session.user.role as Role;
  let hasAccess = false;

  if (permission) {
    hasAccess = hasPermission(userRole, permission);
  } else if (permissions) {
    // Uses OR logic: if user has any permission, they "can" so hide children
    hasAccess = hasAnyPermission(userRole, permissions);
  }

  // Render children only when user lacks access
  return hasAccess ? null : <>{children}</>;
}
```

Estos componentes envuelven elementos de interfaz de usuario que requieren permisos específicos. El componente `Can` muestra los elementos secundarios solo cuando el usuario tiene el permiso requerido, mientras que `Cannot` muestra los elementos secundarios cuando el usuario carece de permiso. Este enfoque declarativo hace que las comprobaciones de permisos sean legibles y fáciles de mantener:

```tsx
// components/project-card.tsx
"use client";

import { useState } from "react";
import { Can, Cannot } from "@/components/rbac/can";
import { Permission } from "@/lib/rbac/roles";
import { deleteProject } from "@/app/actions/project-actions";

interface ProjectCardProps {
  id: string;
  name: string;
  description: string;
  userId: string;           // Project owner's ID
  currentUserId: string;    // Logged-in user's ID
}

// Displays a project with permission-based action buttons. Combines RBAC
// permissions with ownership checks—users need both the permission AND
// ownership to edit/delete (unless they're admins via RBAC).
export function ProjectCard({
  id,
  name,
  description,
  userId,
  currentUserId,
}: ProjectCardProps) {
  const [isDeleting, setIsDeleting] = useState(false);
 
  // Ownership check layered on top of RBAC permissions. A user might have
  // PROJECT_DELETE permission but should only delete their own projects.
  const isOwner = userId === currentUserId;

  const handleDelete = async () => {
    // Confirm before destructive action to prevent accidental deletions
    if (!confirm("Are you sure you want to delete this project?")) return;

    setIsDeleting(true);
    try {
      await deleteProject(id);
      // No need to reset isDeleting on success—component unmounts after deletion
    } catch (error) {
      alert("Failed to delete project");
      setIsDeleting(false);
    }
  };

  return (
    <div className="bg-white rounded-lg shadow-md p-6 border border-gray-200">
      <h3 className="text-lg font-semibold text-gray-900 mb-2">{name}</h3>
      <p className="text-gray-600 text-sm mb-4">{description}</p>

      <div className="flex gap-2">
        {/* Edit button: requires UPDATE permission AND ownership */}
        <Can permission={Permission.PROJECT_UPDATE}>
          {isOwner && (
            <button className="px-3 py-1 text-sm bg-blue-600 text-white rounded hover:bg-blue-700 transition-colors">
              Edit
            </button>
          )}
        </Can>

        {/* Delete button: requires DELETE permission AND ownership */}
        <Can permission={Permission.PROJECT_DELETE}>
          {isOwner && (
            <button
              onClick={handleDelete}
              disabled={isDeleting}
              className="px-3 py-1 text-sm bg-red-600 text-white rounded hover:bg-red-700 transition-colors disabled:opacity-50"
            >
              {isDeleting ? "Deleting..." : "Delete"}
            </button>
          )}
        </Can>

        {/* Show "View only" label for users without edit permissions */}
        <Cannot permission={Permission.PROJECT_UPDATE}>
          <span className="text-xs text-gray-400 self-center">View only</span>
        </Cannot>
      </div>
    </div>
  );
}
```

Este componente combina comprobaciones de permisos con validación de propiedad. Los usuarios necesitan el permiso `PROJECT_DELETE` y deben ser dueños del proyecto para ver el botón de eliminar. Los administradores con el permiso pueden eliminar cualquier proyecto, mientras que los usuarios normales solo pueden eliminar los suyos propios si tienen el permiso.

El componente `Cannot` proporciona comentarios contextuales para los usuarios sin permisos de edición, explicando por qué ciertas acciones no están disponibles. Esta transparencia mejora la experiencia de usuario al establecer expectativas claras en lugar de ocultar la funcionalidad silenciosamente.

### Protección de rutas de API basadas en permisos de usuario

La aplicación de permisos en el servidor es obligatoria. Las restricciones de la interfaz de usuario guían a los usuarios, pero las rutas de la API deben validar los permisos de forma independiente. Un atacante puede eludir las comprobaciones del lado del cliente llamando a las APIs directamente, por lo que cada endpoint debe autenticar y autorizar antes de ejecutar la lógica:

```typescript
// lib/rbac/api-guards.ts
import { NextResponse } from "next/server";
import { auth } from "@/auth";
import { hasPermission, Permission, Role } from "@/lib/rbac/roles";

// Custom error class for authorization failures. Distinguishes auth errors
// from other exceptions, enabling appropriate HTTP status codes in responses.
export class AuthorizationError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "AuthorizationError";
  }
}

// Base authentication guard. Throws if no valid session exists.
// All other guards build on this, ensuring authentication before authorization.
export async function requireAuth() {
  const session = await auth();

  if (!session?.user) {
    throw new AuthorizationError("Authentication required");
  }

  return session;
}

// Single permission guard. Use for endpoints requiring one specific permission.
// Example: requirePermission(Permission.PROJECT_DELETE)
export async function requirePermission(permission: Permission) {
  const session = await requireAuth();
  const userRole = session.user.role as Role;

  if (!hasPermission(userRole, permission)) {
    throw new AuthorizationError(`Missing required permission: ${permission}`);
  }

  return session;
}

// Multiple permission guard with AND logic. User must have ALL listed permissions.
// Returns session on success, throws with list of missing permissions on failure.
export async function requirePermissions(permissions: Permission[]) {
  const session = await requireAuth();
  const userRole = session.user.role as Role;

  // Collect all missing permissions for informative error messages
  const missingPermissions = permissions.filter(
    (permission) => !hasPermission(userRole, permission)
  );

  if (missingPermissions.length > 0) {
    throw new AuthorizationError(
      `Missing required permissions: ${missingPermissions.join(", ")}`
    );
  }

  return session;
}

// Exact role match guard. Use when specific role is required regardless of
// permission inheritance (e.g., admin-only operations).
export async function requireRole(role: Role) {
  const session = await requireAuth();

  if (session.user.role !== role) {
    throw new AuthorizationError(`Requires ${role} role`);
  }

  return session;
}

// Higher-order function wrapping API handlers with standardized error handling.
// Converts AuthorizationErrors to proper HTTP responses (401/403) and logs
// unexpected errors while returning generic 500 to clients.
//
// Usage: export const GET = withErrorHandling(async (req) => { ... })
export function withErrorHandling(handler: Function) {
  return async (...args: any[]) => {
    try {
      return await handler(...args);
    } catch (error) {
      if (error instanceof AuthorizationError) {
        // 401 for authentication failures, 403 for permission/role failures
        return NextResponse.json(
          { error: error.message },
          { status: error.message.includes("Authentication") ? 401 : 403 }
        );
      }

      // Log unexpected errors server-side but hide details from clients
      console.error("API Error:", error);
      return NextResponse.json(
        { error: "Internal server error" },
        { status: 500 }
      );
    }
  };
}
```

Estas funciones guardianas (*guard functions*) proporcionan comprobaciones de permisos reutilizables para rutas de API. En lugar de repetir la lógica de autenticación y autorización, llamamos a `requirePermission` al inicio de cada manejador. Si la comprobación falla, lanza una excepción que el manejador de errores convierte en una respuesta HTTP adecuada.

La distinción entre los códigos de estado 401 y 403 es importante para los consumidores de la API. 401 significa "necesitas autenticarte", lo que provoca una redirección al inicio de sesión. 403 significa "estás autenticado, pero no tienes permiso", lo que indica que el usuario necesita un rol diferente o no debería intentar la acción:

```typescript
// app/api/admin/users/route.ts
import { NextResponse } from "next/server";
import { desc } from "drizzle-orm";
import { requirePermission, withErrorHandling } from "@/lib/rbac/api-guards";
import { Permission } from "@/lib/rbac/roles";
import { db } from "@/lib/db";
import { users } from "@/db/schema";

// GET /api/admin/users - List all users for admin dashboard.
// Protected by USER_READ permission; withErrorHandling converts
// authorization failures to appropriate 401/403 responses.
export const GET = withErrorHandling(async () => {
  await requirePermission(Permission.USER_READ);

  // Select only fields needed for admin user list, excluding sensitive
  // data like passwordHash. Ordered by newest first for relevance.
  const rows = await db
    .select({
      id: users.id,
      email: users.email,
      name: users.name,
      role: users.role,
      isActive: users.isActive,
      createdAt: users.createdAt,
      lastLoginAt: users.lastLoginAt,
    })
    .from(users)
    .orderBy(desc(users.createdAt));

  return NextResponse.json(rows);
});

// POST /api/admin/users - Create new user from admin panel.
// Requires USER_UPDATE permission (create is a form of update).
// Note: This creates users without passwords—intended for OAuth-only
// or invitation flows where users set passwords separately.
export const POST = withErrorHandling(async (request: Request) => {
  await requirePermission(Permission.USER_UPDATE);

  const body = await request.json();
  const { email, name, role } = body as {
    email?: string;
    name?: string;
    role?: string;
  };

  // Basic validation—consider using Zod for more complex requirements
  if (!email || !name) {
    return NextResponse.json(
      { error: "Email and name are required" },
      { status: 400 }
    );
  }

  // Normalize email to lowercase for consistent lookups.
  // Default role to "user" if not specified by admin.
  const inserted = await db
    .insert(users)
    .values({
      email: email.toLowerCase(),
      name,
      role: role ?? "user",
      isActive: true,
    })
    .returning({
      id: users.id,
      email: users.email,
      name: users.name,
      role: users.role,
      createdAt: users.createdAt,
    });

  // Return 201 Created with the new user object
  return NextResponse.json(inserted[0], { status: 201 });
});
```

Esta ruta de API de administración demuestra la protección basada en permisos en la práctica. El manejador `GET` requiere el permiso `USER_READ`, mientras que `POST` requiere `USER_UPDATE`. El contenedor `withErrorHandling` captura los errores de autorización y los convierte en respuestas apropiadas, eliminando bloques `try-catch` repetitivos.

Observa cómo separamos las responsabilidades: los guards manejan la autorización, la lógica de negocio maneja la validación y las operaciones de la base de datos, y los manejadores de errores administran el formato de la respuesta. Esta separación mantiene el código enfocado y comprobable.

### Gestión de autorización en middleware y rutas de API

La autorización en middleware se ejecuta antes de que se ejecuten las rutas, lo que proporciona una salida temprana para las solicitudes no autorizadas. Esto mejora el rendimiento (¿por qué consultar la base de datos si rechazaremos la solicitud de todos modos?) y simplifica la lógica de las rutas al garantizar un acceso autenticado:

```typescript
// middleware-advanced.ts
import { NextResponse } from "next/server";
import { auth } from "@/auth";
import { Role } from "@/lib/rbac/roles";

// Route-to-role mapping. Each path prefix maps to an array of roles
// that can access it. More restrictive routes list fewer roles.
const routePermissions: Record<string, Role[]> = {
  "/admin": [Role.ADMIN],
  "/admin/users": [Role.ADMIN],
  "/admin/settings": [Role.ADMIN],
  "/editor": [Role.ADMIN, Role.EDITOR],
  "/dashboard": [Role.ADMIN, Role.EDITOR, Role.USER],
  "/profile": [Role.ADMIN, Role.EDITOR, Role.USER],
};

// Finds the most specific route match for a given pathname. Sorts matches
// by length descending so "/admin/users" takes precedence over "/admin".
// Returns null for public routes not in the permissions map.
function getRequiredRoles(pathname: string): Role[] | null {
  const matchingRoutes = Object.keys(routePermissions)
    .filter((route) => pathname.startsWith(route))
    .sort((a, b) => b.length - a.length);

  return matchingRoutes.length > 0
    ? routePermissions[matchingRoutes[0]]
    : null;
}

// Main middleware using NextAuth's auth wrapper for session access.
// Runs on Edge Runtime before requests reach route handlers.
export default auth((req) => {
  const { pathname } = req.nextUrl;
  const isAuthenticated = !!req.auth;
  const userRole = req.auth?.user?.role as Role | undefined;

  // Skip auth checks for public routes: auth endpoints, static assets, home
  if (
    pathname.startsWith("/api/auth") ||
    pathname.startsWith("/_next") ||
    pathname === "/"
  ) {
    return NextResponse.next();
  }

  // Look up required roles for this path
  const requiredRoles = getRequiredRoles(pathname);

  // No role requirements means public route—allow access
  if (!requiredRoles) {
    return NextResponse.next();
  }

  // Protected route but user not authenticated—redirect to sign-in
  // with callback URL to return after authentication
  if (!isAuthenticated) {
    const signInUrl = new URL("/auth/signin", req.url);
    signInUrl.searchParams.set("callbackUrl", pathname);
    return NextResponse.redirect(signInUrl);
  }

  // User authenticated but lacks required role—show unauthorized page
  if (userRole && !requiredRoles.includes(userRole)) {
    return NextResponse.redirect(new URL("/unauthorized", req.url));
  }

  // Audit log for admin area access. In production, send to
  // logging service (e.g., Datadog, Sentry) instead of console.
  if (pathname.startsWith("/admin")) {
    console.log(`Admin access: ${userRole} - ${pathname} - ${req.auth?.user?.email}`);
  }

  return NextResponse.next();
});

// Matcher excludes paths that don't need auth checks: NextAuth API routes,
// static files, and images. This improves performance by skipping middleware
// for assets that never require authentication.
export const config = {
  matcher: ["/((?!api/auth|_next/static|_next/image|favicon.ico).*)"],
};
```

Este middleware avanzado asigna rutas a los roles requeridos, verificando la autorización antes de que se ejecute la lógica de cualquier página. La función `getRequiredRoles` encuentra la coincidencia de ruta más específica, lo que permite permisos jerárquicos: `/admin/settings` puede tener requisitos diferentes a `/admin`.

El registro de auditoría (*audit log*) para las rutas de administración demuestra cómo el middleware puede agregar preocupaciones transversales (*cross-cutting concerns*) sin tocar rutas individuales. Cada acceso a una página de administración se registra automáticamente, lo que proporciona monitoreo de seguridad y seguimiento del cumplimiento.

El sistema de permisos que hemos creado transforma la autenticación de una puerta binaria en un sofisticado mecanismo de control de acceso. Los usuarios tienen roles, los roles otorgan permisos y los permisos controlan acciones específicas. Esta arquitectura escala desde aplicaciones simples hasta sistemas empresariales complejos, todo mientras mantiene la claridad y la seguridad de tipos.

A medida que construyes aplicaciones autenticadas, recuerda que la seguridad ocurre en capas. El middleware protege las rutas, los guards de API validan los permisos, los componentes de UI guían a los usuarios y las consultas a la base de datos hacen cumplir la propiedad. Ninguna capa individual proporciona una seguridad completa, pero juntas crean una defensa en profundidad (*defense in depth*), el sello distintivo de los sistemas de autenticación robustos.

NextAuth.js maneja la complejidad de la gestión de sesiones, los flujos de OAuth y la seguridad de tokens, lo que te permite concentrarte en crear funciones en lugar de luchar contra la autenticación. Combinado con la arquitectura *server-first* de Next.js y el modelo de componentes de React, tienes todo lo necesario para crear experiencias autenticadas seguras, escalables y agradables. Los patrones de este capítulo forman la base para cualquier aplicación que requiera identidad de usuario y control de acceso; domínalos y construirás con confianza.

---

## Resumen

En este capítulo, aprendiste a diseñar e implementar una autenticación y autorización seguras y listas para producción en aplicaciones Next.js modernas utilizando NextAuth.js. Comenzamos aclarando la distinción crítica entre autenticación (establecer quién es un usuario) y autorización (determinar qué se le permite hacer a ese usuario). A partir de ahí, exploramos cómo la arquitectura *server-first* de Next.js cambia fundamentalmente la autenticación, trasladando la responsabilidad de las frágiles comprobaciones de tokens en el cliente a la validación de sesiones en el servidor antes de que se renderice el HTML, mejorando tanto la seguridad como la experiencia del usuario.

Implementaste NextAuth.js con múltiples proveedores de autenticación, incluidos OAuth (Google y GitHub) y autenticación basada en credenciales, y configuraste sesiones basadas en JWT optimizadas para entornos *serverless* y en el Edge. En el camino, aprendiste a proteger rutas mediante middleware, aplicar comprobaciones de autenticación y roles en el Edge, y manejar escenarios avanzados como la vinculación de cuentas, la sincronización de perfiles entre proveedores y la validación segura de contraseñas con `bcrypt`. Estos patrones garantizan una gestión de identidad coherente incluso cuando los usuarios inician sesión a través de diferentes proveedores a lo largo del tiempo.

Sobre la base de estos fundamentos de autenticación, el capítulo presentó un sistema RBAC integral que transforma las simples comprobaciones de "sesión iniciada o no" en un modelo de permisos escalable. Diseñaste jerarquías de roles utilizando enums de TypeScript para garantizar la seguridad de tipos, asignaste roles a capacidades y aplicaste la autorización en múltiples capas: middleware para protección de rutas, guards de API reutilizables como `requirePermission()` y `requireRole()`, Server Actions que validan tanto los permisos como la propiedad de los recursos, y componentes de UI declarativos que guían a los usuarios sin exponer funcionalidades confidenciales. Este enfoque en capas de defensa en profundidad garantiza que la seguridad no dependa de un único mecanismo.

La información de este capítulo es valiosa porque la autenticación y la autorización forman el límite de confianza de cada aplicación. Los errores en este punto conducen directamente a fugas de datos, escalamiento de privilegios y sistemas comprometidos. Al aprovechar NextAuth.js, RSC, Server Actions y middleware en el Edge en conjunto, ahora cuentas con un modelo mental robusto y un conjunto de herramientas prácticas para crear aplicaciones seguras que escalen, permanezcan mantenibles y brinden experiencias de usuario fluidas sin sacrificar la seguridad.

En el próximo capítulo, cambiarás el enfoque de quién puede acceder a tu sistema a cómo tu sistema expone los datos. Aprenderás a crear APIs dinámicas y automáticas utilizando Express, Node.js, PostgreSQL y Drizzle ORM: APIs que se generan a sí mismas directamente a partir del esquema de tu base de datos. Mientras que este capítulo aseguró la puerta principal, el siguiente capítulo muestra cómo diseñar un backend que evolucione sin esfuerzo a medida que crece tu modelo de datos, eliminando el código repetitivo de Crear, Leer, Actualizar y Eliminar (CRUD) mientras se mantiene la estructura, el rendimiento y el control.
