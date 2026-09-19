# Análisis del Proyecto: Sistema de Información Web (React + MVC + JWT)

¡Hola! Te ayudo a desglosar este proyecto paso a paso. Es un buen ejemplo para entender cómo se estructura una SPA (Single Page Application) con React usando un patrón tipo MVC en el frontend.

---

## 1) Patrón Arquitectónico

El proyecto **no es MVC puro** (que es más típico de backend), sino una **adaptación del MVC al frontend en React**. Se le suele llamar **"MVVM ligero"** o simplemente **"arquitectura por capas"**. Veamos las capas:
```text
┌─────────────────────────────────────────────┐
│  VIEWS (src/views/)                         │  ← Componentes React (UI)
│  LoginView, RegisterView, DashboardView...  │
└──────────────────┬──────────────────────────┘
                   │ usan
┌──────────────────▼──────────────────────────┐
│  HOOKS (src/hooks/)                         │  ← Estado + lógica reactiva
│  useAuth, useUsers, useDashboard            │
└──────────────────┬──────────────────────────┘
                   │ llaman a
┌──────────────────▼──────────────────────────┐
│  CONTROLLERS (src/controllers/)             │  ← Orquestan, validan
│  AuthController, UserController...          │
└──────────────────┬──────────────────────────┘
                   │ usan
┌──────────────────▼──────────────────────────┐
│  MODELS (src/models/)                       │  ← Datos + reglas de negocio
│  AuthModel, UserModel, DashboardModel       │
└──────────────────┬──────────────────────────┘
                   │ usan
┌──────────────────▼──────────────────────────┐
│  SERVICES (src/services/)                   │  ← Infraestructura
│  httpService, storageService, jwtService    │
└─────────────────────────────────────────────┘
```

**Rol de cada capa:**

| **CapaResponsabilidadEjemplo** |                                                      |                                |
| ------------------------------ | ---------------------------------------------------- | ------------------------------ |
| **Views**                      | Solo UI (JSX + CSS). No saben de fetch ni de tokens. | `LoginView.jsx`                |
| **Hooks**                      | Conectan React con la lógica (estado, efectos).      | `useAuth.js`                   |
| **Controllers**                | Validan datos, orquestan llamadas, manejan errores.  | `AuthController.handleLogin()` |
| **Models**                     | Reglas de negocio y acceso a datos.                  | `AuthModel.login()`            |
| **Services**                   | Utilidades transversales (HTTP, storage, JWT).       | `httpService.js`               |

**Ventajas de esta separación:**

- Si cambias de JSON Server a una API real, solo tocas `services/` y `config/`.
- Si cambias la UI (React → Vue), reutilizas controllers/models.
- Cada capa se puede testear por separado.

---

## 2) Inicio del Proyecto (carga de la página)

Rastreemos qué pasa desde que abres el navegador en `http://localhost:5173`:

### Paso 1 — `index.html`
```html
<div id="root"></div>
<script type="module" src="/src/main.jsx"></script>
```

Vite sirve este HTML. Solo hay un `<div id="root">` vacío y un script que carga React.

### Paso 2 — `src/main.jsx`
```jsx
ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </React.StrictMode>,
)
```

- Monta React en `#root`.
- Envuelve todo en `BrowserRouter` (para que funcionen las rutas de `react-router-dom`).
- Carga `App.jsx`.

### Paso 3 — `src/App.jsx`
```jsx
<Navbar />
<Routes>
  <Route path="/" element={<Navigate to="/login" />} />
  <Route path="/login" element={<LoginView />} />
  <Route path="/register" element={<RegisterView />} />
  <Route path="/dashboard/*" element={
    <ProtectedRoute><DashboardView /></ProtectedRoute>
  } />
</Routes>
```

- Muestra `<Navbar />` (que se oculta si estás en `/dashboard`).
- Define las rutas. La raíz `/` redirige a `/login`.
- `/dashboard` está envuelto en `<ProtectedRoute>`, que verifica autenticación.

### Paso 4 — `ProtectedRoute`
```jsx
const { isAuthenticated, loading } = useAuth()
if (loading) return <LoadingSpinner />
if (!isAuthenticated) return <Navigate to="/login" replace />
return children
```

Aquí entra en juego `useAuth`, que consulta si hay token válido en localStorage.

### Flujo completo gráficamente:
```text
index.html → main.jsx → App.jsx → Routes
                                    ├─ /login     → LoginView
                                    ├─ /register  → RegisterView
                                    └─ /dashboard → ProtectedRoute → DashboardView
```

---

## 3) Login, Autenticación, Token y "Encriptación"

Este es el punto más importante. Vamos paso a paso cuando el usuario hace clic en **"Iniciar Sesión"**:

### 3.1 — `LoginView.jsx` captura los datos
```jsx
const handleSubmit = async (e) => {
  e.preventDefault()
  const result = await login(credentials)  // llama al hook
  navigate('/dashboard')
}
```

### 3.2 — `useAuth.js` (hook)
```jsx
const login = async (credentials) => {
  return new Promise((resolve, reject) => {
    AuthController.handleLogin(
      credentials,
      (user, authToken) => { /* éxito */ resolve(...) },
      (error) => { /* error */ reject(...) }
    )
  })
}
```

### 3.3 — `AuthController.handleLogin()`

Hace validaciones del lado del cliente:

- Campos no vacíos.
- Formato de email con regex `/^[^\s@]+@[^\s@]+\.[^\s@]+$/`.

Luego delega a `AuthModel.login()`.

### 3.4 — `AuthModel.login()` (la parte clave)
```js
static async login(credentials) {
  // 1. Trae TODOS los usuarios de JSON Server
  const users = await httpService.get(API_CONFIG.ENDPOINTS.USERS, false)

  // 2. Busca al usuario por email
  const user = users.find(u => u.email === credentials.email)

  if (!user) return { success: false, error: 'Usuario no encontrado' }

  // 3. ⚠️ AQUÍ ESTÁ EL PROBLEMA: acepta CUALQUIER contraseña no vacía
  if (!credentials.password || credentials.password.length < 1) {
    return { success: false, error: 'Contraseña incorrecta' }
  }

  // 4. Genera un token "JWT" falso
  const token = jwtService.generateToken({
    id: user.id,
    email: user.email,
    name: user.email.split('@')[0],
    exp: Date.now() + 3600000 // 1 hora
  })

  // 5. Guarda token y usuario en localStorage
  storageService.setToken(token)
  storageService.setUser({ id, email, name })

  return { success: true, token, user: {...} }
}
```

### 3.5 — El "JWT" que genera `jwtService.js`

⚠️ **Ojo:** este token **NO es un JWT real**. Es una simulación:
```js
generateToken(payload) {
  const header = { alg: 'HS256', typ: 'JWT' }
  const encodedHeader = btoa(JSON.stringify(header))       // Base64
  const encodedPayload = btoa(JSON.stringify(payload))     // Base64
  const signature = btoa('fake_signature_' + Date.now())   // ¡Falsa!
  return `${encodedHeader}.${encodedPayload}.${signature}`
}
```

**Diferencias con un JWT real:**

| **AspectoJWT realEste "JWT"** |                                     |                               |
| ----------------------------- | ----------------------------------- | ----------------------------- |
| Header/Payload                | Base64URL                           | Base64 (`btoa`)               |
| Firma                         | HMAC-SHA256 o RSA con clave secreta | Texto plano codificado        |
| Verificación                  | Recalcula la firma                  | Solo decodifica y mira `exp`  |
| Seguridad                     | El servidor valida                  | Cualquiera puede falsificarlo |

### 3.6 — Sobre la "encriptación"

En este proyecto **NO hay encriptación real**. Los conceptos que aparecen:

- **Base64 (****`btoa`****/****`atob`****)**: NO es encriptación, es codificación. Cualquiera puede decodificarla.
- **Hash de contraseñas (bcrypt)**: aparece comentado como nota ("en producción deberías hashear"), pero no se usa.
- **HTTPS**: no aplica en localhost.

La única "protección" es que el token se guarda en `localStorage` y se valida contra su fecha de expiración.

### 3.7 — Verificación de sesión (`isAuthenticated`)
```js
static isAuthenticated() {
  const token = storageService.getToken()
  if (!token) return false
  const isValid = jwtService.verifyToken(token)  // decodifica y mira exp
  if (!isValid) storageService.clearSession()
  return true
}
```

`verifyToken` hace:
```js
const parts = token.split('.')
if (parts.length !== 3) return false
const payload = JSON.parse(atob(parts[1]))
if (payload.exp && payload.exp < Date.now()) return false  // expirado
return true
```

### 3.8 — Logout
```js
static async logout() {
  storageService.clearSession()  // borra auth_token y user_data
  return { success: true }
}
```

### Resumen del flujo de autenticación:
```text
Usuario escribe email+pass
   ↓
LoginView → useAuth.login() → AuthController.handleLogin()
   ↓
AuthModel.login() → GET /users → busca por email
   ↓
Genera token falso → guarda en localStorage
   ↓
Navigate a /dashboard → ProtectedRoute verifica token
```

---

## 4) Conexión con JSON Server

### 4.1 — Configuración (`src/config/api.js`)
```js
const API_CONFIG = {
  BASE_URL: 'http://localhost:3000',  // JSON Server
  ENDPOINTS: {
    USERS: '/users',
    POSTS: '/posts',
    PRODUCTS: '/products',
    // ...
  }
}
```

### 4.2 — `httpService.js` (el wrapper de fetch)

Es una clase que envuelve `fetch` con métodos `get`, `post`, `put`, `patch`, `delete`. Todos:

1. Construyen la URL: `${BASE_URL}${endpoint}`.
2. Añaden headers (incluido `Authorization: Bearer <token>` si `includeAuth=true`).
3. Hacen `fetch`.
4. Llaman a `handleResponse()`.
```js
getHeaders(includeAuth = true) {
  const headers = { 'Content-Type': 'application/json' }
  if (includeAuth) {
    const token = this.getToken()
    if (token) headers['Authorization'] = `Bearer ${token}`
  }
  return headers
}

async get(endpoint, includeAuth = true) {
  const url = `${this.baseURL}${endpoint}`
  const response = await fetch(url, {
    method: 'GET',
    headers: this.getHeaders(includeAuth)
  })
  return await this.handleResponse(response)
}
```

### 4.3 — Operaciones CRUD en `UserModel.js`

| **MétodoVerbo HTTPEndpointUso** |        |              |                     |
| ------------------------------- | ------ | ------------ | ------------------- |
| `getAllUsers()`                 | GET    | `/users`     | Listar              |
| `getUserById(id)`               | GET    | `/users/:id` | Ver uno             |
| `createUser()`                  | POST   | `/users`     | Crear               |
| `updateUser(id)`                | PUT    | `/users/:id` | Reemplazar todo     |
| `patchUser(id)`                 | PATCH  | `/users/:id` | Actualizar un campo |
| `deleteUser(id)`                | DELETE | `/users/:id` | Eliminar            |

### 4.4 — Detalles importantes

- **`includeAuth=false`** **en login/registro**: porque aún no hay token.
```js
  httpService.get(API_CONFIG.ENDPOINTS.USERS, false)  // sin token
```
  - **Generación manual de IDs** (porque JSON Server no lo hace solo):
```js
  const lastId = users.length > 0 ? Math.max(...users.map(u => u.id)) : 0
  const newId = lastId + 1
```
  - **Manejo de errores**: `handleResponse` lanza `Error` con el mensaje del servidor.

### 4.5 — Cómo arrancar JSON Server

Necesitas un `db.json` con algo como:
```json
{
  "users": [
    { "id": 1, "email": "oliver@stone.com", "password": "123456" }
  ]
}
```

Y ejecutar:
```bash
npx json-server --watch db.json --port 3000
```

---

## 5) Otras Cosas Importantes

### 5.1 — Rutas (`src/config/routes.js`)

Centraliza las rutas para no escribirlas a mano:
```js
export const ROUTES = {
  LOGIN: '/login',
  DASHBOARD: '/dashboard',
  HOME: '/'
}
```

### 5.2 — Gestión de estado con hooks personalizados

- `useAuth()` → autenticación.
- `useUsers()` → CRUD de usuarios (con `loadUsers`, `createUser`, etc.).
- `useDashboard()` → estadísticas.

Cada uno expone `{ data, loading, error, acciones }`. Patrón muy común en React.

### 5.3 — DashboardView con pestañas

Usa un estado `activeTab` para alternar entre "Dashboard" y "Usuarios" sin recargar la página:
```jsx
const [activeTab, setActiveTab] = useState('dashboard')
// ...
{activeTab === 'dashboard' && <DashboardStats ... />}
{activeTab === 'users' && <UsersView />}
```

### 5.4 — El Navbar se oculta en el dashboard
```jsx
if (location.pathname.includes('/dashboard')) return null
```

Porque `DashboardHeader` ya tiene su propia barra.

### 5.5 — Problemas / cosas mejorables (¡para tu análisis crítico!)

1. **Seguridad nula en contraseñas**: cualquier contraseña no vacía sirve. En producción usar **bcrypt** en backend.
2. **Token JWT falso**: la firma es `btoa('fake_signature_...')`, no se verifica con clave secreta.
3. **`btoa`** **con caracteres no-ASCII**: falla con tildes/ñ. Un JWT real usa Base64URL.
4. **El token se guarda en** **`localStorage`**: vulnerable a XSS. Mejor `httpOnly cookies`.
5. **CORS/Authorization**: JSON Server no valida el `Bearer token`, así que el header es decorativo.
6. **`DashboardModel`** **con datos aleatorios**: `Math.random()` para "estadísticas" — es solo demo.
7. **`process.env = {}`** **en vite.config**: parche raro; normalmente se usa `import.meta.env`.
8. **Archivos vacíos**: `constants.js`, `helpers.js`, `validators.js`, `MainLayout.jsx` — pendientes.
9. **`useUsers`** **en cada componente**: crea una instancia nueva, no comparte estado. Podría usar Context.
10. **Contraseñas en texto plano en** **`db.json`**: visible para cualquiera que abra el archivo.

### 5.6 — Flujo completo de un CRUD de usuario (ejemplo)
```text
UsersView (click "Nuevo Usuario")
   ↓
UserForm se abre
   ↓
Usuario llena email+password → submit
   ↓
useUsers.createUser(data)
   ↓
UserController.createUser(data)  → valida email
   ↓
UserModel.createUser(data)       → verifica duplicado, calcula nuevo ID
   ↓
httpService.post('/users', newUser, true)
   ↓
JSON Server guarda en db.json
   ↓
loadUsers() recarga la lista
   ↓
UsersView re-renderiza con el nuevo usuario
```

### 5.7 — Mapa mental de archivos clave
```text
src/
├── config/          → Constantes (URLs, rutas)
├── controllers/     → Orquestación + validaciones
├── models/          → Reglas de negocio + acceso a datos
├── services/        → HTTP, storage, JWT
├── hooks/           → Puente React ↔ Controllers
├── views/
│   ├── auth/        → Login, Register
│   ├── common/      → Navbar, Spinner, Alert
│   └── dashboard/   → Dashboard, Users, Forms
├── styles/          → CSS por componente
├── App.jsx          → Rutas
└── main.jsx         → Punto de entrada
```

---

## Conclusión para tu análisis

Este proyecto es un **excelente caso de estudio** porque:

✅ **Bien**: separación en capas, hooks personalizados, rutas protegidas, CRUD completo con PUT/PATCH/DELETE, manejo de estados de carga/error.

⚠️ **Mal (intencionalmente didáctico)**: la autenticación es simulada, el "JWT" es falso, no hay hashing de contraseñas, no hay validación del token en el servidor.

Para tu presentación al profesor, podrías enfatizar:

1. La **arquitectura por capas** y cómo se comunican.
2. El **flujo de autenticación** paso a paso.
3. La **diferencia entre el JWT real y el simulado** (¡esto demuestra comprensión profunda!).
4. Las **mejoras** que harías en producción (bcrypt, JWT firmado con `jsonwebtoken`, cookies httpOnly, validación server-side).
