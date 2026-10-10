# Análisis rápido del cambio

Su frontend actual está diseñado para JSON Server, lo que implica:

| Aspecto | JSON Server (actual) | Node.js + Express (nuevo) |
| :--- | :--- | :--- |
| **URL base** | `http://localhost:3000` | `http://192.168.78.145:3000` |
| **Endpoint login** | `GET /users` + filtrar en cliente | `POST /api/users/login` |
| **Endpoint registro** | `POST /users` | `POST /api/users/create` |
| **Listar usuarios** | `GET /users` | `GET /api/users` (requiere token) |
| **Actualizar** | `PUT /users/:id` | `PUT /api/users/:id` |
| **Eliminar** | `DELETE /users/:id` | `DELETE /api/users/delete/:id` |
| **Contraseña** | Texto plano | bcrypt hasheado |
| **Token** | Simulado (btoa) | JWT real firmado |
| **Respuesta login** | Objeto directo | `{ success, message, data }` |

**Conclusión clave:** El backend real hace hashing de contraseñas con bcrypt, valida con JWT firmado y devuelve respuestas envueltas en `{success, message, data}`. Todo esto debe reflejarse en el frontend.

## 🎯 PASO 1: src/config/api.js

Este es el archivo más profundo que define cómo se comunican todas las capas superiores con el backend. Es la base de todo. Cambiamos:
- La BASE_URL para apuntar al backend Express (con /api como prefijo de las rutas)
- Los endpoints para que coincidan con las rutas reales de userRoutes.js

Archivo completo a reemplazar:
```javascript
// src/config/api.js
const API_CONFIG = {
  // Backend Node.js + Express (no JSON Server)
  BASE_URL: 'http://192.168.78.145:3000/api',
  TIMEOUT: 10000,
  ENDPOINTS: {
    // Autenticación
    LOGIN: '/users/login',      // POST - Rutas públicas
    REGISTER: '/users/create',  // POST - Rutas públicas
    
    // CRUD de usuarios (rutas protegidas con JWT)
    USERS: '/users',            // GET (listar), GET/:id, PUT/:id
    USER_DELETE: '/users/delete' // DELETE /users/delete/:id
  }
}

export default API_CONFIG
```

📌 Explicación

1. **BASE_URL:** apunta a `http://192.168.78.145:3000/api` porque en `index.js` del backend usted montó las rutas así:
   ```javascript
   app.use('/api/users', usersRoutes);
   ```
   Por eso todas las rutas llevan el prefijo `/api/users`.

2. **LOGIN y REGISTER:** ahora son POST a rutas específicas, ya no se consultan todos los usuarios para filtrar en el cliente. Esto es una mejora de seguridad importante.

3. **USER_DELETE:** se separa de USERS porque su backend usa `DELETE /api/users/delete/:id` (no `DELETE /api/users/:id`).


✅ Antes de continuar
Por favor:
1. Reemplace src/config/api.js con el contenido de arriba.
2. Levante el backend Express en 192.168.78.145:3000 (o ajuste la IP a la que use en su salón de clases).
3. Pruebe con la app corriendo — el login fallará (esperado, aún no modificamos AuthModel), pero en la consola del navegador debería ver que las peticiones ya van hacia /api/users/... y no hacia /users.

## 🎯 PASO 2: src/services/jwtService.js

## ¿Por qué este archivo es el siguiente?

Este servicio actualmente falsifica tokens JWT usando `btoa()` (codificación Base64 simple, sin firma criptográfica). Eso era necesario con JSON Server porque no había backend que emitiera tokens reales.

Ahora, con el backend Express:
- El backend es quien emite y firma el token con `jsonwebtoken` y `keys.secretOrKey`.
- El frontend NO debe firmar tokens (no tiene la clave secreta, y aunque la tuviera, sería una vulnerabilidad grave).
- El frontend solo debe decodificar el token para leer su payload (`id`, `email`, `role`, `exp`) y verificar la expiración localmente para saber cuándo cerrar sesión.

## Diferencias técnicas importantes

| Antes (JSON Server) | Ahora (Express) |
| :--- | :--- |
| `generateToken()` con `btoa` + firma fake | ❌ Se elimina — el backend lo genera |
| `verifyToken()` decodifica con `atob` | ✅ Se mantiene, pero solo verifica expiración |
| `decodeToken()` con `atob` | ✅ Se mejora para manejar Base64URL (estándar JWT) |
| `exp` en milisegundos | ⚠️ El backend usa segundos (`Math.floor(Date.now()/1000) + 3600`) |

---

## ⚠️ Punto crítico: Base64URL vs Base64

Los JWT estándar (RFC 7519) usan Base64URL, no Base64 normal. La diferencia:
- Base64 usa `+` y `/`
- Base64URL uses `-` y `_`

`atob()` del navegador falla o produce basura con Base64URL. Debemos convertir antes de decodificar. Este es un error muy común que sus estudiantes deben conocer.

---

## Archivo completo a reemplazar:

```javascript
// src/services/jwtService.js

// NOTA: Con el backend Express, el token es emitido y firmado por el servidor
// (ver backend/nodejs_express/controllers/userController.js -> login).
// El frontend SOLO decodifica y verifica la expiración, NUNCA firma tokens.

class JWTService {
  // Convierte Base64URL (estándar JWT) a Base64 normal
  base64UrlDecode(str) {
    // Reemplazar caracteres URL-safe
    let base64 = str.replace(/-/g, '+').replace(/_/g, '/')
    // Agregar padding si falta
    while (base64.length % 4) {
      base64 += '='
    }
    return base64
  }

  // Decodificar token (sin verificar firma - el backend ya lo hizo al emitirlo)
  decodeToken(token) {
    try {
      if (!token) return null
      const parts = token.split('.')
      if (parts.length !== 3) {
        console.warn('⚠️ Token mal formado (no tiene 3 partes)')
        return null
      }

      // Decodificar payload usando Base64URL
      const payloadBase64 = this.base64UrlDecode(parts[1])
      const payload = JSON.parse(atob(payloadBase64))
      return payload
    } catch (error) {
      console.error('❌ Error al decodificar token:', error)
      return null
    }
  }

  // Verificar token: solo comprobamos estructura y expiración
  // La firma la valida el backend en cada petición protegida
  verifyToken(token) {
    try {
      if (!token) return false
      const payload = this.decodeToken(token)
      if (!payload) return false

      // El backend expresa 'exp' en SEGUNDOS (estándar JWT)
      // Date.now() está en MILISEGUNDOS
      if (payload.exp) {
        const nowInSeconds = Math.floor(Date.now() / 1000)
        if (payload.exp < nowInSeconds) {
          console.warn('⏰ Token expirado')
          return false
        }
      }
      return true
    } catch (error) {
      console.error('Error al verificar token:', error)
      return false
    }
  }

  // Tiempo restante en milisegundos
  getTokenRemainingTime(token) {
    try {
      const payload = this.decodeToken(token)
      if (!payload || !payload.exp) return 0

      const nowInSeconds = Math.floor(Date.now() / 1000)
      const remainingSeconds = payload.exp - nowInSeconds
      return remainingSeconds > 0 ? remainingSeconds * 1000 : 0
    } catch (error) {
      return 0
    }
  }

  // Obtener información útil del usuario desde el token
  getUserFromToken(token) {
    const payload = this.decodeToken(token)
    if (!payload) return null
    return {
      id: payload.id,
      email: payload.email,
      role: payload.role
    }
  }
}

export default new JWTService()
```

---

## 📌 Explicación

1. **`generateToken()` desaparece:** este es un concepto fundamental de seguridad. Le enseña a sus estudiantes que el cliente nunca debe tener la clave secreta, y por eso nunca debe firmar tokens. Solo el servidor firma, el cliente solo lee.
2. **Base64URL:** aproveche para explicar el estándar JWT (RFC 7519). Los tres segmentos `header.payload.signature` separados por puntos, y por qué el payload NO es cifrado (cualquiera lo puede leer) — solo está firmado para garantizar integridad.
3. **`exp` en segundos vs milisegundos:** es el bug #1 en implementaciones JWT caseras. El backend usa `Math.floor(Date.now() / 1000) + 3600`, y el frontend antes comparaba contra milisegundos, lo cual nunca expiraba. Ahora está corregido.
4. **`verifyToken()` no verifica la firma:** aquí hay que ser honesto con los estudiantes — verificar la firma en el cliente requiere la clave secreta, y eso rompe el modelo de seguridad. Por eso la verificación real ocurre en cada petición protegida en el backend (`authMiddleware.js`). Lo que hacemos en el cliente es solo "verificar lo verificable localmente": estructura y expiración, para dar buena UX (cerrar sesión a tiempo).
5. **Nuevo método `getUserFromToken()`:** útil para las vistas. Lo usaremos más adelante por si necesita leer el role desde el token sin tener que confiar solo en `localStorage`.

✅ Antes de continuar
Por favor:
1. Reemplace src/services/jwtService.js con el contenido de arriba.
2. Importante: aún la app no funcionará porque AuthModel.js todavía llama a jwtService.generateToken(...), que ya no existe. No se preocupe, ese será el siguiente paso.
3. Puede probar en consola del navegador para verificar la nueva lógica sin afectar la app aún. Por ejemplo, después de que el login falle, abra la consola y ejecute manualmente para probar la decodificación (opcional).

# 🎯 PASO 3: `src/services/httpService.js`

## ¿Por qué este archivo ahora?
Este servicio es el mensajero entre el frontend y el backend. Todas las capas superiores (`Models` → `Controllers` → `Hooks` → `Views`) dependen de que este servicio les entregue los datos ya "limpios" y con el formato esperado.

---

## ¿Qué cambia con el backend Express?
Aquí hay un detalle arquitectónico muy importante que sus estudiantes deben entender:

| Aspecto | JSON Server | Express Backend |
| :--- | :--- | :--- |
| **Formato de éxito** | `{ id: 1, email: "..." }` o `[array]` | `{ success: true, message: "...", data: {...} }` |
| **Formato de error** | HTTP `4xx` con texto plano | HTTP `4xx` con `{ success: false, message: "..." }` |
| **Autenticación** | Ignorada (cualquier token sirve) | Validada con JWT en el backend |
| **Token en header** | `Bearer <token>` | `JWT <token>` ⚠️ ¡Ojo aquí! |

---

## 🔍 Punto crítico: el prefijo JWT vs Bearer
Mire con atención su backend:

* En `userController.js` → `login`:
```js
session_token: `JWT ${token}`
```

* En `authMiddleware.js` → `verifyToken`:
```js
const token = authHeader.split(" ")[1];
```

Su middleware **no** distingue entre `JWT` y `Bearer`. Simplemente hace `split(" ")[1]` y toma la segunda parte. Eso significa que ambos prefijos funcionan. Pero por coherencia didáctica y para que sus estudiantes no se confundan luego, lo mejor es usar `Bearer` en el frontend (estándar OAuth 2.0, el que verán en el mundo real). El backend lo aceptará sin problema.

Si en algún momento quiere hacerlo estricto, tendría que modificar el middleware para que solo acepte `Bearer`. Pero por ahora, seguir con `Bearer` en el frontend es lo más profesional.

---

## 🎯 Decisiones de diseño para este servicio
Tengo dos estrategias posibles:
* **Opción A:** Que `httpService` devuelva la respuesta completa `{success, message, data}` y que cada `Model` decida cómo usarla.
* **Opción B:** Que `httpService` desenvuelva `data` automáticamente y devuelva solo el contenido útil.

Para su clase, recomiendo **Opción A**, porque:
- Es más explícita y didáctica (el estudiante ve la estructura real de la respuesta del backend).
- Permite que los `Models` accedan a `message` si lo necesitan.
- Es menos "mágica" — no oculta lo que pasa en la red.

---

## Archivo completo a reemplazar:
```js
// src/services/httpService.js
import API_CONFIG from '../config/api'

class HttpService {
  constructor() {
    this.baseURL = API_CONFIG.BASE_URL
  }

  // Obtener token del localStorage
  getToken() {
    return localStorage.getItem('auth_token')
  }

  // Configurar headers
  getHeaders(includeAuth = true) {
    const headers = {
      'Content-Type': 'application/json'
    }

    if (includeAuth) {
      const token = this.getToken()
      if (token) {
        // El backend Express espera el formato "Bearer <token>"
        // (aunque su middleware también acepta "JWT <token>")
        headers['Authorization'] = `Bearer ${token}`
      }
    }

    return headers
  }

  // Manejar errores de respuesta
  // El backend Express responde con: { success, message, data } o { success, message, error }
  async handleResponse(response) {
    console.log('📡 Response status:', response.status)

    // Intentamos parsear SIEMPRE el body (éxito o error vienen en JSON)
    let body = null
    try {
      body = await response.json()
    } catch (e) {
      // Si no es JSON (ej. error 500 con HTML), body queda null
      body = null
    }

    if (!response.ok) {
      // El backend devuelve { success: false, message, error }
      const errorMessage =
        (body && (body.message || body.error)) ||
        `Error HTTP: ${response.status}`

      console.error('❌ Error del servidor:', errorMessage)
      throw new Error(errorMessage)
    }

    // Respuesta exitosa: devolvemos el body tal cual
    // (incluye { success, message, data })
    console.log('📡 Response data:', body)
    return body
  }

  // POST request
  async post(endpoint, data, includeAuth = true) {
    try {
      const url = `${this.baseURL}${endpoint}`
      console.log(`📡 POST a: ${url}`)
      console.log('📡 Datos enviados:', data)

      const response = await fetch(url, {
        method: 'POST',
        headers: this.getHeaders(includeAuth),
        body: JSON.stringify(data)
      })

      return await this.handleResponse(response)
    } catch (error) {
      console.error(`❌ POST ${endpoint} error:`, error)
      throw error
    }
  }

  // GET request
  async get(endpoint, includeAuth = true) {
    try {
      const url = `${this.baseURL}${endpoint}`
      console.log(`📡 GET: ${url}`)

      const response = await fetch(url, {
        method: 'GET',
        headers: this.getHeaders(includeAuth)
      })

      return await this.handleResponse(response)
    } catch (error) {
      console.error(`GET ${endpoint} error:`, error)
      throw error
    }
  }

  // PUT request
  async put(endpoint, data, includeAuth = true) {
    try {
      const url = `${this.baseURL}${endpoint}`
      console.log(`📡 PUT: ${url}`)

      const response = await fetch(url, {
        method: 'PUT',
        headers: this.getHeaders(includeAuth),
        body: JSON.stringify(data)
      })

      return await this.handleResponse(response)
    } catch (error) {
      console.error(`PUT ${endpoint} error:`, error)
      throw error
    }
  }

  // PATCH request (por si lo usa en el futuro)
  async patch(endpoint, data, includeAuth = true) {
    try {
      const url = `${this.baseURL}${endpoint}`
      console.log(`📡 PATCH: ${url}`)

      const response = await fetch(url, {
        method: 'PATCH',
        headers: this.getHeaders(includeAuth),
        body: JSON.stringify(data)
      })

      return await this.handleResponse(response)
    } catch (error) {
      console.error(`PATCH ${endpoint} error:`, error)
      throw error
    }
  }

  // DELETE request
  async delete(endpoint, includeAuth = true) {
    try {
      const url = `${this.baseURL}${endpoint}`
      console.log(`📡 DELETE: ${url}`)

      const response = await fetch(url, {
        method: 'DELETE',
        headers: this.getHeaders(includeAuth)
      })

      return await this.handleResponse(response)
    } catch (error) {
      console.error(`DELETE ${endpoint} error:`, error)
      throw error
    }
  }
}

export default new HttpService()
```

---

## 📌 Notas didácticas clave
1. **Antes el error se leía con `response.text()`** como fallback. Ahora siempre intentamos `response.json()` porque su backend Express responde consistentemente en JSON (mire `userController.js`: todos los `res.status(...).json({...})`).
2. **El body de error ahora es el `message`** que viene del backend. Esto es mucho mejor UX: el usuario verá mensajes como *"El email no existe en la base de datos"* o *"Contraseña o correo incorrecto"* — los mismos que usted escribió en el controlador.
3. **Nuevo método `patch()`:** su backend no tiene endpoint PATCH todavía, pero si más adelante lo agrega (por ejemplo, para actualizaciones parciales), el servicio ya lo soporta.
4. **Se mantiene `includeAuth` como parámetro:** útil porque login y register son rutas públicas (no requieren token), mientras que listar/actualizar/eliminar sí.
5. **Diferencia Bearer vs JWT:** aproveche para explicar el estándar `Authorization: Bearer <token>` de OAuth 2.0. Su backend es tolerante, pero en el mundo real muchos servidores son estrictos.
6. **`handleResponse` unificado:** antes había lógica condicional extra para el 204. Como su backend Express siempre responde con un body JSON (incluso en DELETE: `res.status(200).json({success: true, message: "Usuario eliminado"})`), ya no necesitamos ese caso especial.

---

## ✅ Antes de continuar
Por favor:
1. Reemplace `src/services/httpService.js` con el contenido de arriba.
2. Verifique que el backend esté corriendo en `192.168.78.145:3000` (o donde lo tenga).
3. Puede hacer una prueba aislada desde la consola del navegador para verificar que el servicio funciona correctamente:

```js
// En la consola del navegador (con la app corriendo en Vite):
// Importar el servicio no es directo desde consola por ser módulo ES,
// pero puede probar con fetch manual:
fetch('http://192.168.78.145:3000/api/users/login', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ email: 'oliver@stone.com', password: 'su_password' })
}).then(r => r.json()).then(console.log)
```

Debería ver en consola el objeto `{ success: true, message: "Usuario autenticado", data: {...} }` con el token incluido.

# 🎯 PASO 4: src/models/AuthModel.js

## ¿Por qué este archivo ahora?
Este Model es el corazón de la autenticación. Es donde su código actual "hace trampa" — y aquí es donde sus estudiantes verán el cambio más grande y didáctico.

---

## 🔴 Los 3 problemas graves del código actual

* **Problema 1: Login inseguro sin validación real de contraseña**
  ```javascript
  // Actual:
  const users = await httpService.get(API_CONFIG.ENDPOINTS.USERS, false)
  const user = users.find(u => u.email === credentials.email)
  // ...
  if (!credentials.password || credentials.password.length < 1) {  
    // ❌ Acepta cualquier contraseña no vacía
  }
  ```
* **Problema 2:** Trae **TODOS** los usuarios para encontrar uno → ineficiente y filtra datos sensibles.
* **Problema 3:** Genera el token en el cliente → imposible con bcrypt + JWT firmado del backend.

---

## 🟢 La nueva arquitectura

```text
ANTES:                                    AHORA:
LoginView                                 LoginView
  └─ useAuth                                └─ useAuth
       └─ AuthController                         └─ AuthController
            └─ AuthModel                              └─ AuthModel
                 ├─ GET /users (todos) ←───┐              ├─ POST /api/users/login  ←──┐
                 ├─ find en cliente         │              ├─ Backend valida bcrypt     │
                 ├─ ❌ no valida password   │              ├─ Backend firma JWT real    │
                 └─ btoa() genera token ────┘              └─ Guardar token recibido ───┘
```

---

## ⚠️ Ajuste importante: register en el backend

Mire su `userController.js` $\rightarrow$ `register`:
```javascript
register(req, res) {  
  const user = req.body;  
  if (!user.role) user.role = "user";  
  User.create(user, ...);
}
```

Y su `userModel.create` requiere `name`:
```javascript
const sql = `INSERT INTO users(name, lastname, email, password, phone, image, role, ...)`;
```

Por lo tanto, el frontend debe enviar al menos: `email`, `password`, y opcionalmente `name`/`lastname`. Su `RegisterView.jsx` ya pide `name` — perfecto.

---

## 🔑 El id de usuarios en el backend

En el backend, `id` viene de MySQL (auto-incremental). El frontend ya no debe calcular el id (como hacía antes con `Math.max(...users.map(u => u.id)) + 1`). El backend lo asigna.

---

## 📋 Estructura de la respuesta del login (backend)

```json
{
  "success": true,
  "message": "Usuario autenticado",
  "data": {
    "id": 1,
    "email": "oliver@stone.com",
    "name": "Oliver",
    "lastname": "Stone",
    "image": null,
    "phone": null,
    "role": "admin",
    "session_token": "JWT eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

> **Observación importante:** el token viene como `session_token` con prefijo `JWT`. Hay que extraer solo la parte del token (quitar el prefijo `JWT`).

---

## Archivo completo a reemplazar:

```javascript
// src/models/AuthModel.js
import httpService from '../services/httpService'
import storageService from '../services/storageService'
import jwtService from '../services/jwtService'
import API_CONFIG from '../config/api'

class AuthModel {
  // Iniciar sesión contra el backend Express
  static async login(credentials) {
    try {
      console.log('🔐 Intentando login con:', credentials.email)
      
      // POST /api/users/login  ->  { success, message, data }
      const response = await httpService.post(
        API_CONFIG.ENDPOINTS.LOGIN,
        {
          email: credentials.email,
          password: credentials.password
        },
        false // Ruta pública: no requiere token
      )

      // El backend responde { success: true, message: "...", data: {...} }
      if (!response || !response.success || !response.data) {
        return {
          success: false,
          error: response?.message || 'Credenciales incorrectas'
        }
      }

      const userData = response.data

      // El backend envía el token como "session_token": "JWT <token>"
      // Extraemos solo el JWT
      let token = userData.session_token || ''
      if (token.startsWith('JWT ')) {
        token = token.substring(4)
      }

      if (!token) {
        return {
          success: false,
          error: 'El servidor no devolvió un token válido'
        }
      }

      // Construir usuario público (sin password, sin session_token)
      const user = {
        id: userData.id,
        email: userData.email,
        name: userData.name,
        lastname: userData.lastname,
        role: userData.role,
        image: userData.image,
        phone: userData.phone
      }

      // Guardar token y usuario en localStorage
      storageService.setToken(token)
      storageService.setUser(user)

      console.log('✅ Login exitoso para:', user.email)
      return {
        success: true,
        token,
        user
      }
    } catch (error) {
      console.error('❌ Error en login:', error)
      return {
        success: false,
        error: error.message || 'Error de conexión con el servidor'
      }
    }
  }

  // Registrar nuevo usuario
  static async register(userData) {
    try {
      console.log('📝 Intentando registrar usuario:', userData.email)
      
      // POST /api/users/create  ->  { success, message, data }
      // El backend hashea la contraseña con bcrypt y asigna el ID
      const payload = {
        name: userData.name || '',
        lastname: userData.lastname || '',
        email: userData.email,
        password: userData.password,
        phone: userData.phone || null,
        image: userData.image || null
        // No enviamos 'role': el backend asigna "user" por defecto
      }

      const response = await httpService.post(
        API_CONFIG.ENDPOINTS.REGISTER,
        payload,
        false // Ruta pública
      )

      if (!response || !response.success) {
        return {
          success: false,
          error: response?.message || 'Error al registrar usuario'
        }
      }

      console.log('✅ Usuario registrado:', response.data)
      return {
        success: true,
        user: response.data,
        message: response.message || 'Usuario registrado exitosamente'
      }
    } catch (error) {
      console.error('❌ Error en registro:', error)
      return {
        success: false,
        error: error.message || 'Error al registrar usuario'
      }
    }
  }

  // Cerrar sesión
  static async logout() {
    try {
      storageService.clearSession()
      return { success: true }
    } catch (error) {
      console.error('Error en logout:', error)
      return { success: false, error: error.message }
    }
  }

  // Verificar si está autenticado
  static isAuthenticated() {
    const token = storageService.getToken()
    if (!token) return false

    const isValid = jwtService.verifyToken(token)
    if (!isValid) {
      storageService.clearSession()
      return false
    }

    return true
  }

  // Obtener usuario actual
  static getCurrentUser() {
    return storageService.getUser()
  }

  // Obtener token actual
  static getCurrentToken() {
    return storageService.getToken()
  }
}

export default AuthModel
```

---

## 📌 Notas didácticas clave para su clase

1. **El cliente ya NO compara contraseñas:** ese es el punto más importante. La contraseña viaja por HTTPS (o HTTP en desarrollo local), el backend la compara con `bcrypt.compare()`, y devuelve éxito o fracaso. El frontend nunca ve el hash.
2. **El cliente ya NO genera tokens:** el backend firma con `jsonwebtoken` usando `keys.secretOrKey`. El frontend solo recibe el token y lo guarda. Esta es la regla de oro: el cliente nunca debe conocer la clave secreta.
3. **Extracción del token:** el backend devuelve `session_token: "JWT eyJ..."` (con prefijo). Lo limpiamos a `eyJ...` puro. Luego el `httpService` lo enviará como `Authorization: Bearer eyJ....` Ambos prefijos son válidos porque su middleware usa `split(" ")[1]`.
4. **Registro con POST /api/users/create:** noten que el backend hashea antes de guardar (`User.create` $\rightarrow$ `await bcrypt.hash(user.password, 10)`). El frontend envía la contraseña en texto plano por la red (por eso HTTPS es obligatorio en producción), pero nunca la guarda ni la compara.
5. **role no se envía:** el backend lo asigna como `"user"` por defecto. Es una buena práctica de seguridad — el cliente no debería poder auto-asignarse `role: "admin"`.
6. **Ya no hay `Math.max(...users.map(u => u.id))`:** el ID lo genera MySQL autoincremental. Si sus estudiantes preguntan, es una lección sobre por qué la lógica de negocio crítica vive en el servidor.

---

## 🚨 Aviso: AuthController aún puede fallar

Su `AuthController.js` actual valida la longitud de la contraseña ($< 6$) antes de llamar al Model. Eso es correcto para `register`, pero para `login` es una validación superflua. Ese archivo lo ajustaremos en un paso posterior (PASO 5). Por ahora, el flujo debería funcionar aunque con esa validación redundante.

También su `LoginView.jsx` muestra un mensaje hardcodeado:
```text
Prueba con los usuarios de tu base de datos:
Email: oliver@stone.com
```
Eso lo ajustaremos en su paso correspondiente.

---

## ✅ Antes de continuar

Por favor:
1. Reemplace `src/models/AuthModel.js` con el contenido de arriba.
2. Verifique que el backend esté corriendo y que en la tabla `users` de MySQL tenga al menos un usuario con password hasheado con bcrypt (el que use para las pruebas).
3. Pruebe el login en el navegador:
   * Ingrese con un email y contraseña válidos.
   * Abra la consola del navegador para ver los logs (`📡 POST`, `📡 Response data`, `✅ Login exitoso`).
   * Si el login es exitoso, debería redirigir a `/dashboard`.
   * En localStorage (`DevTools` $\rightarrow$ `Application` $\rightarrow$ `Local Storage`) debe ver `auth_token` y `user_data`.

### Errores esperados (normales en este punto):
* Si el backend devuelve `401` con *"Contraseña o correo incorrecto"* $\rightarrow$ credenciales erróneas, flujo correcto.
* Si el backend devuelve `403` en `/dashboard` $\rightarrow$ aún no tocamos eso, es solo el `ProtectedRoute` con verificación local, debería funcionar.

# 🎯 PASO 5: src/controllers/AuthController.js

## ¿Por qué este archivo ahora?

Este controlador es el intermediario entre el Hook (`useAuth`) y el Model (`AuthModel`). Su trabajo es validar la entrada del usuario antes de enviarla al backend, y delegar el trabajo pesado al Model.

## 🔍 Análisis del código actual

```js
static async handleLogin(credentials, onSuccess, onError) {  
  // 1. Valida campos vacíos ✅ (útil)  
  // 2. Valida formato de email ✅ (útil)  
  // 3. Llama a AuthModel.login() ✅  
  // 4. Maneja resultado ✅
}
```

```js
static async handleRegister(userData, onSuccess, onError) {  
  // 1. Valida campos vacíos ✅ (útil)  
  // 2. Valida formato de email ✅ (útil)  
  // 3. Valida longitud de contraseña ✅ (útil)  
  // 4. Llama a AuthModel.register() ✅
}
```

Buena noticia: `AuthController` ya está bien estructurado. Su única labor era validar antes de llamar al Model, y lo hace correctamente.

## 🟡 ¿Por qué entonces hay que modificarlo?

Hay 3 ajustes pequeños pero importantes que surgen de los cambios anteriores:

### Ajuste 1: Coherencia del mensaje de error del login
El Model ahora devuelve mensajes del backend real (*"Contraseña o correo incorrecto"*, *"El email no existe en la base de datos"*), no mensajes genéricos. El Controller antes tenía un fallback:

```js
onError(result.error || 'Credenciales incorrectas')
```

Esto está bien, pero el orden ideal es priorizar el mensaje del backend. Actualmente lo hace. ✅ Se puede dejar igual.

### Ajuste 2: register requiere más campos
Su backend (`userModel.create`) inserta en la tabla:

```sql
INSERT INTO users(name, lastname, email, password, phone, image, role, ...)
```

Campos que no pueden ser `NULL` en MySQL (asumiendo un esquema típico): `name`, `email`, `password`. El Controller ya valida `name`, `email`, `password`. ✅ Está bien.

Pero ojo: en el `RegisterView.jsx` el usuario ingresa `name`, y en `AuthModel.register` lo enviamos como `name`. El backend lo usa. Todo consistente. ✅

### Ajuste 3 (el único real): Mensaje de error de conexión
Cuando `AuthModel.login` captura una excepción de red (backend apagado), devuelve `error.message` que puede ser algo como `"Failed to fetch"` — en inglés y poco amigable para el usuario final. Vale la pena mejorar este mensaje en el Controller:

```js
// Antes:
onError('Error al conectar con el servidor')

// Después:
onError(result.error || 'Error al conectar con el servidor')
```

Espera — ya lo hace así en el catch. El catch del Controller solo se dispara si el Model lanza una excepción, pero nuestro Model nunca lanza, siempre retorna `{success, error}`. Entonces el catch del Controller es código muerto. 🧟

---

## 🎯 Conclusión: este archivo casi no necesita cambios

Pero hay una mejora didáctica valiosa que sí vale la pena hacer: agregar logs más claros y alinear los mensajes con el backend. También voy a agregar un pequeño método utilitario para validar email (que puede reusarse).

### Archivo completo a reemplazar:

```js
// src/controllers/AuthController.js
import AuthModel from '../models/AuthModel'

class AuthController {  
  // Expresión regular para validar email (reutilizable)  
  static isValidEmail(email) {    
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/    
    return emailRegex.test(email)  }  

  // Manejar login  
  static async handleLogin(credentials, onSuccess, onError) {    
    try {      
      console.log('🔧 AuthController: Procesando login para:', credentials.email)      
      // Validar que los campos no estén vacíos      
      if (!credentials.email || !credentials.password) {        
        onError('Por favor complete todos los campos')        
        return      }      
      // Validar formato de email      
      if (!this.isValidEmail(credentials.email)) {        
        onError('Por favor ingrese un email válido')        
        return      }      
      // Delegar al Model (que hace POST /api/users/login)      
      const result = await AuthModel.login(credentials)      
      console.log('📊 Resultado del login:', result)      
      if (result.success) {        
        onSuccess(result.user, result.token)      } else {        
        // Priorizar el mensaje del backend (ej. "Contraseña o correo incorrecto")        
        onError(result.error || 'Credenciales incorrectas')      }    
    } catch (error) {      
      console.error('❌ Error inesperado en handleLogin:', error)      
      onError('Error inesperado al iniciar sesión')    }  }  

  // Manejar registro de usuario  
  static async handleRegister(userData, onSuccess, onError) {    
    try {      
      console.log('🔧 AuthController: Procesando registro para:', userData.email)      
      // Validaciones básicas      
      if (!userData.email || !userData.password || !userData.name) {        
        onError('Por favor complete todos los campos')        
        return      }      
      if (!this.isValidEmail(userData.email)) {        
        onError('Por favor ingrese un email válido')        
        return      }      
      // Validación de contraseña (aplica solo al registro, no al login)      
      if (userData.password.length < 6) {        
        onError('La contraseña debe tener al menos 6 caracteres')        
        return      }      
      // Delegar al Model (que hace POST /api/users/create)      
      const result = await AuthModel.register(userData)      
      console.log('📊 Resultado del registro:', result)      
      if (result.success) {        
        onSuccess(result.user)      } else {        
        // Priorizar el mensaje del backend (ej. "El email ya está registrado")        
        onError(result.error || 'Error al registrar usuario')      }    } 
    catch (error) {      
      console.error('❌ Error inesperado en handleRegister:', error)      
      onError('Error inesperado al registrar usuario')    }  }  

  // Manejar logout  
  static async handleLogout(onSuccess, onError) {    
    try {      
      const result = await AuthModel.logout()      
      if (result.success) {        
        onSuccess()      } else {        
        onError(result.error || 'Error al cerrar sesión')      }    } 
    catch (error) {      
      console.error('❌ Error en handleLogout:', error)      
      onError('Error al cerrar sesión')    }  }  

  // Obtener estado de autenticación  
  static getAuthState() {    
    return {      
      isAuthenticated: AuthModel.isAuthenticated(),      
      currentUser: AuthModel.getCurrentUser(),      
      currentToken: AuthModel.getCurrentToken()    }  }  

  // Verificar autenticación (útil al inicio de la app)  
  static async checkAuth() {    
    const isValid = AuthModel.isAuthenticated()    
    if (!isValid) {      
      await AuthModel.logout()    }    
    return isValid  }
}

export default AuthController
```

---

## 📌 Notas didácticas clave para su clase

1. **`isValidEmail` como método estático reutilizable:** antes la regex estaba duplicada en `handleLogin` y `handleRegister`. Ahora se define una vez. Principio DRY (*Don't Repeat Yourself*) — concepto fundamental de ingeniería de software.
2. **Priorizar el mensaje del backend:** `result.error || 'Credenciales incorrectas'`. Si el backend dice *"Contraseña o correo incorrecto"*, mostramos eso. El fallback solo se usa cuando el Model no da mensaje (ej. error de red, que ya viene con su propio mensaje de `error.message`).
3. **Validación de contraseña solo en registro:** noten que en `handleLogin` no validamos la longitud de la contraseña. ¿Por qué? Porque si el usuario tiene una contraseña heredada de 5 caracteres (o incluso menos), no queremos bloquearlo. La validación de longitud es una regla de negocio para nuevos registros, no para login. Este es un excelente punto para discutir con sus estudiantes: las validaciones deben tener un contexto y un propósito.
4. **Código muerto eliminado:** el catch de antes decía `'Error al conectar con el servidor'`, pero nuestro Model nunca lanza excepciones (siempre retorna `{success, error}`). Ahora el mensaje del catch es más honesto: `'Error inesperado al iniciar sesión'`, porque solo llegaría aquí por un bug real, no por red.
5. **`AuthModel.isAuthenticated()` verifica la expiración del JWT:** aquí está la conexión con el PASO 2. Cuando usted escribió `jwtService.verifyToken(token)`, ese método ahora interpreta `exp` en segundos (correcto para JWT estándar). Antes estaba mal y nunca expiraba. Sus estudiantes pueden probar esto manipulando `localStorage` para ver el comportamiento.

---

## ✅ Antes de continuar

Por favor:
1. Reemplace `src/controllers/AuthController.js` con el contenido de arriba.
2. Pruebe el login con credenciales válidas e inválidas:
   - **Caso 1:** email y password correctos → debe redirigir a `/dashboard`.
   - **Caso 2:** email correcto, password incorrecto → debe mostrar *"Contraseña o correo incorrecto"* (mensaje del backend).
   - **Caso 3:** email que no existe → debe mostrar *"El email no existe en la base de datos"* (mensaje del backend).
   - **Caso 4:** email inválido (`"abc"`) → debe mostrar *"Por favor ingrese un email válido"* (validación local, no llega al backend).

En la consola del navegador debería ver los mensajes:

```text
🔧 AuthController: Procesando login para: oliver@stone.com
🔐 Intentando login con: oliver@stone.com
📡 POST a: http://192.168.78.145:3000/api/users/login
📡 Response data: { success: true, message: "Usuario autenticado", data: {...} }
📊 Resultado del login: { success: true, user: {...}, token: "..." }
✅ Login exitoso para: oliver@stone.com
```

# 🎯 PASO 6: src/models/UserModel.js

## ¿Por qué este archivo ahora?
Ya tenemos el login funcionando (Autenticación). Ahora toca el CRUD de usuarios (Autorización). Este Model es el encargado de listar, crear, actualizar y eliminar usuarios desde el panel del dashboard.

---

## 🔴 Problemas del código actual

### Problema 1: Usa endpoints de JSON Server
```js
const users = await httpService.get(API_CONFIG.ENDPOINTS.USERS, true)
// Esperaba: [ {id, email, password}, ... ]  (array plano)
```

### Problema 2: La respuesta del backend Express viene envuelta
```js
// Backend devuelve:
{ success: true, message: "Lista de usuarios", data: [...] }
// Pero el código actual espera:
[ ... ] // array directo
```

### Problema 3: createUser calcula el ID en el cliente
```js
const lastId = Math.max(...users.data.map(u => u.id))
const newId = lastId + 1  // ❌ MySQL lo asigna
```

### Problema 4: El backend tiene rutas específicas para DELETE
```text
Backend: DELETE /api/users/delete/:id  (no DELETE /api/users/:id)
```

---

## ⚠️ Punto crítico: rol del usuario logueado vs autorización

Mire su `authMiddleware.js` en el backend:
```js
router.get('/', verifyToken, authorizeRoles(['admin', 'seller']), ...)
router.put('/:id', verifyToken, authorizeRoles(['admin', 'seller']), ...)
router.delete('/delete/:id', verifyToken, authorizeRoles(['admin']), ...)
```

* **Implicación didáctica:** el CRUD de usuarios solo funciona si el usuario logueado tiene `role: 'admin'` o `'seller'`. Si su usuario de prueba (`oliver@stone.com`) tiene `role: 'user'` en MySQL, verá errores `403 Acceso denegado`.

* **Acción para usted antes de las pruebas:** verifique en su base de datos MySQL:
```sql
SELECT id, email, role FROM users;
-- Asegurarse de que el usuario de prueba tenga role = 'admin' o 'seller'
```

Si no lo tiene, actualícelo:
```sql
UPDATE users SET role = 'admin' WHERE email = 'oliver@stone.com';
```

---

## 🔍 Diferencia en la respuesta de cada endpoint

Su backend responde siempre con `{ success, message, data }`:

| Endpoint | Respuesta |
| :--- | :--- |
| `GET /api/users` | `{ success: true, message: "Lista de usuarios", data: [...] }` |
| `GET /api/users/:id` | `{ success: true, message: "Usuario encontrado", data: {...} }` |
| `POST /api/users/create` | `{ success: true, message: "Usuario creado correctamente", data: {...} }` |
| `PUT /api/users/:id` | `{ success: true, message: "Usuario actualizado", data: {...} }` |
| `DELETE /api/users/delete/:id` | `{ success: true, message: "Usuario eliminado", data: {...} }` |

---

## 🟡 Sobre PATCH

Su backend no tiene endpoint PATCH. En el `UserForm.jsx` hay un toggle para modo PUT/PATCH. Aquí tengo dos opciones:

* **Opción A (recomendada para clase):** Hacer que `patchUser` use PUT internamente, enviando solo los campos que se quieren actualizar. Su `userModel.update` en el backend ya maneja campos opcionales:
```js
if (user.password) { fields.push("password = ?"); ... }
if (user.email) { fields.push("email = ?"); ... }
// etc.
```
Esto significa que PUT con un solo campo funciona — el backend solo actualiza lo que envía.

* **Opción B:** Eliminar el toggle de PATCH del `UserForm.jsx` (lo haremos en un paso posterior). Pero como ya está en la UI, mejor mantenerlo funcional.

> **Voy con Opción A.** Didácticamente es valiosa: enseña que PUT y PATCH en un backend mal diseñado pueden converger, y que PUT es idempotente (enviar el mismo body dos veces da el mismo resultado).

---

## 🎯 Decisión de diseño: ¿qué hace cada método?

Mantengo los 5 métodos (`getAllUsers`, `getUserById`, `createUser`, `updateUser`, `patchUser`, `deleteUser`) para no romper los hooks ni las vistas. Pero internamente:
* `createUser` → `POST /api/users/create`
* `updateUser` → `PUT /api/users/:id`
* `patchUser` → también `PUT /api/users/:id` pero con body parcial (documentado)
* `deleteUser` → `DELETE /api/users/delete/:id`
* `getAllUsers` → `GET /api/users`
* `getUserById` → `GET /api/users/:id`

---

## Archivo completo a reemplazar:

```js
// src/models/UserModel.js

import httpService from '../services/httpService'
import API_CONFIG from '../config/api'

class UserModel {
  // GET /api/users - Listar todos los usuarios (requiere role admin o seller)
  static async getAllUsers() {
    try {
      console.log('📋 Obteniendo todos los usuarios')
      const response = await httpService.get(API_CONFIG.ENDPOINTS.USERS, true)
      
      // Backend: { success, message, data: [...] }
      if (!response || !response.success) {
        return {
          success: false,
          error: response?.message || 'Error al obtener usuarios'
        }
      }
      return {
        success: true,
        data: response.data
      }
    } catch (error) {
      console.error('Error al obtener usuarios:', error)
      return {
        success: false,
        error: error.message
      }
    }
  }

  // GET /api/users/:id - Obtener un usuario por ID
  static async getUserById(id) {
    try {
      console.log(`🔍 Obteniendo usuario con ID: ${id}`)
      const response = await httpService.get(
        `${API_CONFIG.ENDPOINTS.USERS}/${id}`,
        true
      )
      if (!response || !response.success) {
        return {
          success: false,
          error: response?.message || 'Usuario no encontrado'
        }
      }
      return {
        success: true,
        data: response.data
      }
    } catch (error) {
      console.error('Error al obtener usuario:', error)
      return {
        success: false,
        error: error.message
      }
    }
  }

  // POST /api/users/create - Crear nuevo usuario
  // NOTA: El backend hashea la contraseña con bcrypt y asigna el ID
  static async createUser(userData) {
    try {
      console.log('➕ Creando nuevo usuario:', userData.email)
      
      // El backend valida y hashea; aquí solo estructuramos el payload
      const payload = {
        name: userData.name || '',
        lastname: userData.lastname || '',
        email: userData.email,
        password: userData.password,
        phone: userData.phone || null,
        image: userData.image || null
        // No enviamos 'role': el backend asigna "user" por defecto
      }

      const response = await httpService.post(
        API_CONFIG.ENDPOINTS.REGISTER,
        payload,
        true // Ruta protegida (admin o seller)
      )

      if (!response || !response.success) {
        return {
          success: false,
          error: response?.message || 'Error al crear usuario'
        }
      }

      return {
        success: true,
        data: response.data,
        message: response.message || 'Usuario creado exitosamente'
      }
    } catch (error) {
      console.error('Error al crear usuario:', error)
      return {
        success: false,
        error: error.message
      }
    }
  }

  // PUT /api/users/:id - Actualizar todos los campos enviados
  // El backend solo actualiza los campos presentes en el body
  static async updateUser(id, userData) {
    try {
      console.log(`✏️ Actualizando usuario ID: ${id}`, userData)
      
      if (!id) {
        return { success: false, error: 'ID de usuario no proporcionado' }
      }

      // Estructura de datos que acepta el backend (userModel.update)
      const payload = {}
      if (userData.name !== undefined) payload.name = userData.name
      if (userData.lastname !== undefined) payload.lastname = userData.lastname
      if (userData.email !== undefined) payload.email = userData.email
      if (userData.password) payload.password = userData.password // solo si viene con valor
      if (userData.phone !== undefined) payload.phone = userData.phone
      if (userData.image !== undefined) payload.image = userData.image
      if (userData.role !== undefined) payload.role = userData.role

      const response = await httpService.put(
        `${API_CONFIG.ENDPOINTS.USERS}/${id}`,
        payload,
        true
      )

      if (!response || !response.success) {
        return {
          success: false,
          error: response?.message || 'Error al actualizar usuario'
        }
      }

      return {
        success: true,
        data: response.data,
        message: response.message || 'Usuario actualizado'
      }
    } catch (error) {
      console.error('Error al actualizar usuario:', error)
      return {
        success: false,
        error: error.message
      }
    }
  }

  // "PATCH" - Actualizar un solo campo
  // El backend no tiene PATCH real; usamos PUT con body parcial
  // (su userModel.update solo toca los campos presentes)
  static async patchUser(id, partialData) {
    try {
      console.log(`📝 Actualizando campo(s) de usuario ID: ${id}`, partialData)
      
      if (!id) {
        return { success: false, error: 'ID de usuario no proporcionado' }
      }

      const response = await httpService.put(
        `${API_CONFIG.ENDPOINTS.USERS}/${id}`,
        partialData, // solo el campo o campos a actualizar
        true
      )

      if (!response || !response.success) {
        return {
          success: false,
          error: response?.message || 'Error al actualizar campo'
        }
      }

      return {
        success: true,
        data: response.data,
        message: response.message || 'Campo(s) actualizado(s) correctamente'
      }
    } catch (error) {
      console.error('Error al actualizar campo:', error)
      return {
        success: false,
        error: error.message
      }
    }
  }

  // DELETE /api/users/delete/:id - Eliminar usuario (solo admin)
  static async deleteUser(id) {
    try {
      console.log(`🗑️ Eliminando usuario ID: ${id}`)
      
      if (!id) {
        return { success: false, error: 'ID de usuario no proporcionado' }
      }

      const response = await httpService.delete(
        `${API_CONFIG.ENDPOINTS.USER_DELETE}/${id}`,
        true
      )

      if (!response || !response.success) {
        return {
          success: false,
          error: response?.message || 'Error al eliminar usuario'
        }
      }

      return {
        success: true,
        message: response.message || 'Usuario eliminado exitosamente'
      }
    } catch (error) {
      console.error('Error al eliminar usuario:', error)
      return {
        success: false,
        error: error.message
      }
    }
  }
}

export default UserModel
```

---

## 📌 Notas didácticas clave para su clase

1. **De JSON Server a Express: la respuesta envuelta:** antes el JSON Server devolvía arrays/objetos planos. Ahora todo viene envuelto en `{success, message, data}`. Esto es un patrón de diseño muy común en APIs REST. Enseñe a sus estudiantes que cada backend puede tener su propio contrato de respuesta — el Model es el lugar adecuado para adaptarlo.
2. **El ID lo asigna MySQL:** adiós al `Math.max(...users.map(u => u.id)) + 1`. Los IDs autoincrementales son responsabilidad de la base de datos. Es un principio importante: la fuente de verdad de los identificadores únicos es el motor de persistencia, no el cliente.
3. **`patchUser` como "PUT parcial":** aquí hay una lección profunda. En REST puro:
   * `PUT` = reemplaza el recurso completo (envía todos los campos)
   * `PATCH` = modifica parcialmente (envía solo lo que cambia)
4. Su backend implementa `User.update` de forma que solo actualiza los campos presentes. Eso lo hace semánticamente más cercano a PATCH que a PUT, aunque la ruta sea PUT. Es un excelente ejemplo de "REST pragmático" vs "REST puro". Vale la pena discutirlo.
5. **`role` NO se envía en `createUser`:** por seguridad, el cliente no auto-asigna roles. El backend asigna "user" por defecto. Pero en `updateUser` sí permitimos enviar `role` — porque un admin legítimamente puede cambiar el rol de otro usuario. Discuta con sus estudiantes por qué esta asimetría tiene sentido.
6. **Filtrado de `undefined`:** en `updateUser` usamos `if (userData.x !== undefined)` para no enviar campos nulos al backend. Esto evita sobrescribir campos existentes con `null`. Es una técnica defensiva clave cuando se usa PUT parcial.
7. **Rutas específicas para DELETE:** `/users/delete/:id` en lugar de `/users/:id`. Es una decisión de diseño del backend (a veces por conflictos con otras rutas). El Model lo abstrae: los componentes de UI no saben cómo está la ruta por debajo, solo llaman `deleteUser(id)`.

---

## 🚨 Recordatorio antes de probar

Verifique en MySQL:
```sql
SELECT id, email, role FROM users WHERE email = 'oliver@stone.com';
```

Si el role es `'user'`, actualice:
```sql
UPDATE users SET role = 'admin' WHERE email = 'oliver@stone.com';
```
*(Sin esto, los GET/PUT/DELETE del CRUD devolverán `403 Acceso denegado` — comportamiento correcto del backend, pero no podrá probar la funcionalidad).*

---

## ✅ Antes de continuar

Por favor:
1. Reemplace `src/models/UserModel.js` con el contenido de arriba.
2. Verifique el rol de su usuario de prueba en MySQL (debe ser `admin` o `seller`).
3. Haga login en la app con ese usuario.
4. Vaya al tab "Usuarios" del dashboard.
5. Pruebe el flujo completo:
   * **Listar:** debería ver la tabla con los usuarios de MySQL.
   * **Crear:** `+ Nuevo Usuario` → llene el formulario → debería crear en MySQL (verifique con `SELECT * FROM users`).
   * **Ver:** clic en 👁️ → debería mostrar los detalles.
   * **Editar (PUT):** clic en ✏️ → modifique email → guardar.
   * **Editar (PATCH):** en el modal, elija "Actualizar campo específico" → cambie solo un campo.
   * **Eliminar:** clic en 🗑️ → confirmar → debería desaparecer de MySQL.

### Errores esperados (y su significado):
* `403` en `GET /api/users` → el usuario no tiene rol `admin`/`seller`.
* `500` en `POST /api/users/create` → puede faltar algún campo `NOT NULL` en MySQL (`name`, `lastname`, etc.) o fallar el hash de bcrypt.
* `401` en cualquier petición → token expirado o inválido; cerrar sesión y volver a entrar.


# 🎯 PASO 7: src/views/dashboard/UserForm.jsx

## ¿Por qué este archivo ahora?
Este es el formulario de creación/edición de usuarios. Ya tenemos UserModel con los métodos correctos, pero UserForm todavía tiene lógica heredada de la era JSON Server que puede causar problemas al probar el CRUD:

## 🔴 Problemas del código actual

### Problema 1: En modo PATCH envía password en texto plano sin advertencia
En el modo PATCH, el usuario selecciona "email" o "password". Si elige password, se envía como `{ password: "nuevoValor" }` al backend. Eso está bien (el backend hashea), pero el `<input type="text">` para password no oculta el valor.

```jsx
<input  
  type={patchField === 'email' ? 'email' : 'text'}  // ❌ password en texto plano  
  value={patchValue}  
  ...
/>
```

### Problema 2: updateUser en modo PUT envía user.password como fallback
```jsx
const updateData = {  
  email: formData.email,  
  password: formData.password || user.password  // ❌ envía el hash como "password"
}
```
Aquí hay un bug lógico grave: `user.password` en el frontend nunca debería existir (el backend nunca lo devuelve en el GET). Pero el código lo usa como fallback. Si el backend por error devolviera el hash, se re-hashearía el hash → contraseña rota. Eliminar ese fallback.

### Problema 3: El UserForm no acepta los campos name, lastname, phone
Su backend requiere (o al menos recomienda) name en el registro. Pero UserForm solo maneja email y password. Cuando el usuario crea un nuevo usuario desde el dashboard, el name va vacío (`''`). Puede causar problemas si MySQL tiene NOT NULL en name.

*Decisión para la clase:* Agregar los campos name, lastname y phone al formulario. Es más completo y didáctico.

### Problema 4: No hay validación de formato de email ni longitud de contraseña
El backend lo valida con bcrypt y MySQL, pero es mejor UX validar antes de enviar.

### Problema 5: El modo PATCH usa type="text" para contraseña
Cuando el campo es password en modo PATCH, debe usar `type="password"` para ocultar visualmente el valor.

---

## 🎯 ¿Qué cambios hago?
Voy a:
1. Agregar campos `name`, `lastname`, `phone` al formulario (alineados con el backend).
2. Corregir el fallback `user.password` en modo PUT — simplemente no enviar password si está vacío.
3. Corregir el type del input en modo PATCH para que password use `type="password"`.
4. Validar email y longitud de contraseña en el cliente antes de enviar.
5. Mantener el flujo PUT/PATCH tal como está, pero con las correcciones anteriores.
6. Mejorar los mensajes de error: mostrar el mensaje del backend tal cual (ya lo hace `setError(err.error || ...)` pero hay que asegurar que se propague).

---

## ⚠️ Un detalle sobre PUT y campos obligatorios
Recuerde que en su backend, `User.update` construye el SQL dinámicamente:

```js
if (user.name) { fields.push("name = ?"); values.push(user.name); }
if (user.email) { fields.push("email = ?"); values.push(user.email); }
// etc.
```

Eso significa que un PUT con solo `{ email: "nuevo@email.com" }` es válido — solo actualiza email. Perfecto para nuestros propósitos.

Pero en modo PUT puro del formulario, enviamos todos los campos visibles. Aquí está el dilema didáctico:
- **PUT semánticamente correcto:** enviar TODOS los campos (nombre, email, teléfono, etc.).
- **PUT pragmático con backend flexible:** enviar solo los campos que el usuario tocó.

Voy con **PUT pragmático**: enviamos los campos presentes pero solo los que tengan valor (o explícitamente vacíos si el usuario los vació). El backend actualiza lo que recibe.


## Archivo completo a reemplazar:


```js
// src/views/dashboard/UserForm.jsx
import { useState, useEffect } from 'react'
import useUsers from '../../hooks/useUsers'

const UserForm = ({ user, isEdit, onSuccess, onClose }) => {
  const { createUser, updateUser, patchUser } = useUsers()

  const [formData, setFormData] = useState({
    name: '',
    lastname: '',
    email: '',
    phone: '',
    password: '',
    confirmPassword: ''
  })
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState('')

  // Modo PATCH
  const [patchMode, setPatchMode] = useState(false)
  const [patchField, setPatchField] = useState('')
  const [patchValue, setPatchValue] = useState('')

  // Cargar datos del usuario en modo edición
  useEffect(() => {
    if (user && isEdit) {
      setFormData({
        name: user.name || '',
        lastname: user.lastname || '',
        email: user.email || '',
        phone: user.phone || '',
        password: '',
        confirmPassword: ''
      })
    }
  }, [user, isEdit])

  const handleChange = (e) => {
    const { name, value } = e.target
    setFormData(prev => ({ ...prev, [name]: value }))
    if (error) setError('')
  }

  // Validación reutilizable
  const validateEmail = (email) => {
    const re = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
    return re.test(email)
  }

  // ============ MODO POST (crear) y PUT (actualizar completo) ============
  const handleSubmit = async (e) => {
    e.preventDefault()
    setError('')

    // Validaciones comunes
    if (!formData.email) {
      setError('El email es requerido')
      return
    }
    if (!validateEmail(formData.email)) {
      setError('Por favor ingrese un email válido')
      return
    }
    if (!formData.name || !formData.name.trim()) {
      setError('El nombre es requerido')
      return
    }

    // Validaciones específicas para creación
    if (!isEdit) {
      if (!formData.password) {
        setError('La contraseña es requerida')
        return
      }
      if (formData.password.length < 6) {
        setError('La contraseña debe tener al menos 6 caracteres')
        return
      }
      if (formData.password !== formData.confirmPassword) {
        setError('Las contraseñas no coinciden')
        return
      }
    }

    // Validación de contraseña en edición (solo si se ingresó algo)
    if (isEdit && formData.password) {
      if (formData.password.length < 6) {
        setError('La contraseña debe tener al menos 6 caracteres')
        return
      }
      if (formData.password !== formData.confirmPassword) {
        setError('Las contraseñas no coinciden')
        return
      }
    }

    setLoading(true)

    try {
      if (isEdit) {
        // PUT - Actualizar campos enviados
        const updateData = {
          name: formData.name,
          lastname: formData.lastname,
          email: formData.email,
          phone: formData.phone || null
        }
        // Solo enviamos password si el usuario escribió una nueva
        if (formData.password) {
          updateData.password = formData.password
        }

        await updateUser(user.id, updateData)
      } else {
        // POST - Crear nuevo usuario
        await createUser({
          name: formData.name,
          lastname: formData.lastname,
          email: formData.email,
          phone: formData.phone || null,
          password: formData.password
        })
      }
      onSuccess()
    } catch (err) {
      setError(err.error || 'Error al guardar usuario')
    } finally {
      setLoading(false)
    }
  }

  // ============ MODO PATCH (actualizar un solo campo) ============
  const handlePatchSubmit = async (e) => {
    e.preventDefault()
    setError('')

    if (!patchField) {
      setError('Seleccione un campo a actualizar')
      return
    }
    if (!patchValue) {
      setError('Ingrese el nuevo valor')
      return
    }

    // Validación específica por campo
    if (patchField === 'email' && !validateEmail(patchValue)) {
      setError('Por favor ingrese un email válido')
      return
    }
    if (patchField === 'password' && patchValue.length < 6) {
      setError('La contraseña debe tener al menos 6 caracteres')
      return
    }

    setLoading(true)
    try {
      const patchData = {}
      patchData[patchField] = patchValue
      await patchUser(user.id, patchData)
      onSuccess()
    } catch (err) {
      setError(err.error || 'Error al actualizar campo')
    } finally {
      setLoading(false)
    }
  }

  // Determinar el type del input en modo PATCH
  const getPatchInputType = () => {
    if (patchField === 'email') return 'email'
    if (patchField === 'password') return 'password'
    return 'text'
  }

  // Placeholder para modo PATCH según campo
  const getPatchPlaceholder = () => {
    const placeholders = {
      email: 'nuevo@email.com',
      password: 'Nueva contraseña (mín. 6 caracteres)',
      name: 'Nuevo nombre',
      lastname: 'Nuevo apellido',
      phone: 'Nuevo teléfono'
    }
    return placeholders[patchField] || 'Nuevo valor'
  }

  return (
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        <div className="modal-header">
          <h3>{isEdit ? 'Editar Usuario' : 'Nuevo Usuario'}</h3>
          <button className="modal-close" onClick={onClose}>×</button>
        </div>

        {error && <div className="form-error">{error}</div>}

        {isEdit && (
          <div className="patch-toggle">
            <label>
              <input
                type="radio"
                checked={!patchMode}
                onChange={() => setPatchMode(false)}
              />
              Actualizar todos los datos (PUT)
            </label>
            <label>
              <input
                type="radio"
                checked={patchMode}
                onChange={() => setPatchMode(true)}
              />
              Actualizar campo específico (PATCH)
            </label>
          </div>
        )}

        {patchMode && isEdit ? (
          // ============ FORM PATCH ============
          <form onSubmit={handlePatchSubmit} className="user-form">
            <div className="form-group">
              <label>Campo a actualizar</label>
              <select
                value={patchField}
                onChange={(e) => setPatchField(e.target.value)}
                className="form-control"
                required
              >
                <option value="">Seleccionar campo</option>
                <option value="name">Nombre</option>
                <option value="lastname">Apellido</option>
                <option value="email">Email</option>
                <option value="phone">Teléfono</option>
                <option value="password">Contraseña</option>
              </select>
            </div>

            <div className="form-group">
              <label>Nuevo valor</label>
              <input
                type={getPatchInputType()}
                value={patchValue}
                onChange={(e) => setPatchValue(e.target.value)}
                placeholder={getPatchPlaceholder()}
                className="form-control"
                required
              />
            </div>

            <div className="form-actions">
              <button type="submit" className="btn-primary" disabled={loading}>
                {loading ? 'Actualizando...' : 'Actualizar Campo'}
              </button>
              <button type="button" className="btn-secondary" onClick={onClose}>
                Cancelar
              </button>
            </div>
          </form>
        ) : (
          // ============ FORM POST / PUT ============
          <form onSubmit={handleSubmit} className="user-form">
            <div className="form-group">
              <label>Nombre *</label>
              <input
                type="text"
                name="name"
                value={formData.name}
                onChange={handleChange}
                placeholder="Nombre del usuario"
                className="form-control"
                disabled={loading}
                required
              />
            </div>

            <div className="form-group">
              <label>Apellido</label>
              <input
                type="text"
                name="lastname"
                value={formData.lastname}
                onChange={handleChange}
                placeholder="Apellido del usuario"
                className="form-control"
                disabled={loading}
              />
            </div>

            <div className="form-group">
              <label>Email *</label>
              <input
                type="email"
                name="email"
                value={formData.email}
                onChange={handleChange}
                placeholder="usuario@ejemplo.com"
                className="form-control"
                required
                disabled={loading}
              />
            </div>

            <div className="form-group">
              <label>Teléfono</label>
              <input
                type="text"
                name="phone"
                value={formData.phone}
                onChange={handleChange}
                placeholder="+57 300 000 0000"
                className="form-control"
                disabled={loading}
              />
            </div>

            {!isEdit && (
              <>
                <div className="form-group">
                  <label>Contraseña *</label>
                  <input
                    type="password"
                    name="password"
                    value={formData.password}
                    onChange={handleChange}
                    placeholder="Mínimo 6 caracteres"
                    className="form-control"
                    required
                    disabled={loading}
                  />
                </div>

                <div className="form-group">
                  <label>Confirmar Contraseña *</label>
                  <input
                    type="password"
                    name="confirmPassword"
                    value={formData.confirmPassword}
                    onChange={handleChange}
                    placeholder="Repita la contraseña"
                    className="form-control"
                    required
                    disabled={loading}
                  />
                </div>
              </>
            )}

            {isEdit && !patchMode && (
              <>
                <div className="form-group">
                  <label>Nueva Contraseña (opcional)</label>
                  <input
                    type="password"
                    name="password"
                    value={formData.password}
                    onChange={handleChange}
                    placeholder="Dejar en blanco para mantener la actual"
                    className="form-control"
                    disabled={loading}
                  />
                </div>
                {formData.password && (
                  <div className="form-group">
                    <label>Confirmar Nueva Contraseña</label>
                    <input
                      type="password"
                      name="confirmPassword"
                      value={formData.confirmPassword}
                      onChange={handleChange}
                      placeholder="Repita la nueva contraseña"
                      className="form-control"
                      disabled={loading}
                    />
                  </div>
                )}
              </>
            )}

            <div className="form-actions">
              <button type="submit" className="btn-primary" disabled={loading}>
                {loading ? 'Guardando...' : (isEdit ? 'Actualizar' : 'Crear')}
              </button>
              <button type="button" className="btn-secondary" onClick={onClose}>
                Cancelar
              </button>
            </div>
          </form>
        )}
      </div>
    </div>
  )
}

export default UserForm
```

# 📌 Notas didácticas clave para su clase

1. El bug del fallback `user.password`: en el código original había `password: formData.password || user.password`. Aquí hay una lección de seguridad importante. El frontend nunca debe tener ni manejar el hash de la contraseña. En el `getUserById` del backend, su `SELECT` no incluye password, y eso es correcto. Eliminamos el fallback y ahora solo enviamos password si el usuario escribe una nueva.
2. Separación de validaciones por contexto: mire las validaciones en `handleSubmit`. Hay tres bloques:
  - Comunes: email válido, nombre no vacío.
  - Solo creación (`!isEdit`): contraseña obligatoria + confirmación.
  - Solo edición con password nueva: si el usuario dejó el campo vacío, no se valida; si escribió algo, se valida.
3. Esto enseña que la misma operación (guardar) tiene reglas distintas según el contexto. Es una lección sobre UX y reglas de negocio.
4. Modo PATCH real y funcional: ahora el usuario puede elegir entre 5 campos (`name`, `lastname`, `email`, `phone`, `password`) y solo se envía ese campo al backend. Su `User.update` construye un `UPDATE` dinámico, así que funciona perfecto.
5. `type="password"` condicional: fíjese en `getPatchInputType()`. Antes el input para password era `type="text"` (¡visible!). Ahora es `type="password"`. Detalle pequeño, pero didáctico: los detalles de seguridad importan.
6. `formData.phone || null`: cuando el usuario deja el teléfono vacío, enviamos `null` en lugar de `""`. Esto es semánticamente más correcto para la base de datos (un teléfono vacío se representa mejor como `NULL` que como string vacío).
7. Confirmación de contraseña condicional en edición: en modo PUT, si el usuario no escribe nueva contraseña, no mostramos el campo de confirmación. Si la escribe, aparece. Es UX adaptativo.

## 🚨 Advertencia sobre `user.name` y `user.lastname`
En el `UserModel.getAllUsers` de su backend, el `SELECT` incluye:
```sql
SELECT id, email, name, lastname, phone, image, role, created_at, updated_at FROM users
```
Perfecto. Ahora `UserForm` al editar puede poblar `name`, `lastname`, `phone` desde los datos del usuario. ✅

## 🚨 Advertencia sobre `UsersView`
Su `UsersView.jsx` no muestra los campos `name`, `lastname` en la tabla, solo `id`, `email`, `createdAt`. Los usuarios creados desde el dashboard aparecerán sin nombre visible en la tabla. Eso lo mejoraremos en el PASO 8 (opcional) si el tiempo lo permite.

## ✅ Antes de continuar
Por favor:
1. Reemplace `src/views/dashboard/UserForm.jsx` con el contenido de arriba.
2. Pruebe el flujo completo del dashboard:
  - Crear usuario con nombre, apellido, email, teléfono y contraseña → verifique en MySQL que se guardó con bcrypt:
  ```sql
  SELECT id, name, lastname, email, role, LEFT(password, 20) FROM users ORDER BY id DESC LIMIT 1;
  ```
  - El campo password debe empezar con `$2a$` o `$2b$` (bcrypt).
  - Editar (PUT): cambie el nombre o el teléfono, deje la contraseña vacía → verifique que el resto de campos no cambió.
  - Editar (PUT con cambio de contraseña): escriba una nueva contraseña → verifique que ahora puede hacer login con ella.
  - Editar (PATCH): cambie solo el teléfono → verifique que en MySQL solo cambió el teléfono y `updated_at`.
  - Eliminar → verifique que desaparece de MySQL.

### Errores esperados y su significado:
- `"El email ya está registrado"` — no es un error de tu código, es correcto (tu backend no lo valida por defecto, pero si lo agregas, es buena práctica). Aquí en el frontend no lo validamos, así que el error vendría de MySQL como `ER_DUP_ENTRY`. Si quiere, en un paso extra lo agregamos al backend.
- `500 al crear` — probablemente falte algún campo `NOT NULL` en MySQL (`lastname` si está definido como `NOT NULL`). Verifique el esquema de la tabla.

# 🎯 PASO 8: src/views/dashboard/UsersView.jsx

## ¿Por qué este archivo ahora?
Este es el listado principal de usuarios en el dashboard. La tabla actual solo muestra ID, Email y Fecha Registro, pero su backend ahora devuelve muchos más campos (`name`, `lastname`, `phone`, `role`, `image`, etc.). Aprovechémoslos para hacer la tabla más informativa y didáctica.

## 🔴 Problemas del código actual

### Problema 1: La tabla solo muestra 4 columnas pobres
```jsx
<tr>  <th>ID</th>  <th>Email</th>  <th>Fecha Registro</th>  <th>Acciones</th></tr>
```

Con el backend, el `getAllUsers` devuelve:
```json
{  "id": 1,  "email": "oliver@stone.com",  "name": "Oliver",  "lastname": "Stone",  "phone": "3001234567",  "image": null,  "role": "admin",  "created_at": "2024-01-15T10:30:00.000Z",  "updated_at": "2024-01-15T10:30:00.000Z"}
```

Tenemos mucha información que no se muestra. Enseñar a sus estudiantes a aprovechar los datos del backend es una lección importante.

### Problema 2: La búsqueda solo filtra por email e ID
```jsx
user.email.toLowerCase().includes(searchTerm.toLowerCase()) ||user.id.toString().includes(searchTerm)
```

Debería poder buscar por nombre completo y por rol también. Es más útil.

### Problema 3: El campo `createdAt` no existe — es `created_at`
```jsx
<td>{user.createdAt ? new Date(user.createdAt).toLocaleDateString() : 'N/A'}</td>
```

Su backend MySQL devuelve `created_at` (snake_case). El código actual espera `createdAt` (camelCase). Nunca mostrará la fecha correctamente. Este es un bug silencioso heredado de JSON Server, donde los nombres pueden variar.

### Problema 4: No se muestra el `role` con estilo
El `role` es un dato clave (`admin`/`seller`/`customer`/`user`). Vale la pena mostrarlo con un badge de color para hacerlo visual.

### Problema 5: La tabla tiene `colSpan="4"` — debe actualizarse
Si agregamos columnas, ese `colSpan` debe reflejar el nuevo total.

---

## 🎯 ¿Qué cambios hago?
1. Agregar columnas: Nombre completo, Email, Teléfono, Rol (con badge), Fecha de registro.
2. Corregir `createdAt` → `created_at`.
3. Ampliar la búsqueda: nombre, apellido, email, rol.
4. Añadir badges de color por rol: `admin` (rojo), `seller` (naranja), `customer` (azul), `user` (gris).
5. Reducir el ID a un badge pequeño — menos prominente, ya que la info clave es el nombre y el email.
6. Mejorar el estado vacío con un mensaje contextual (dependiendo de si es por búsqueda o por falta de datos).

## ⚠️ Decisión sobre `image`
El campo `image` del backend puede ser una URL o `null`. Podría mostrar una imagen, pero eso complica el archivo por ahora. Lo dejamos para un paso opcional posterior — la clase avanza mejor si mantenemos el foco en lo esencial.

**Archivo completo a reemplazar:**

```jsx
// src/views/dashboard/UsersView.jsx
import { useState } from 'react'
import useUsers from '../../hooks/useUsers'
import AlertMessage from '../common/AlertMessage'
import UserForm from './UserForm'
import UserDetails from './UserDetails'
import '../../styles/Users.css'

const UsersView = () => {
  const { users, loading, error, deleteUser, loadUsers } = useUsers()
  const [showForm, setShowForm] = useState(false)
  const [showDetails, setShowDetails] = useState(false)
  const [selectedUser, setSelectedUser] = useState(null)
  const [editMode, setEditMode] = useState(false)
  const [message, setMessage] = useState(null)
  const [searchTerm, setSearchTerm] = useState('')

  // Filtrar usuarios: nombre, apellido, email, rol o ID
  const filteredUsers = users.filter(user => {
    const term = searchTerm.toLowerCase().trim()
    if (!term) return true

    const fullName = `${user.name || ''} ${user.lastname || ''}`.toLowerCase()
    return (
      fullName.includes(term) ||
      (user.email || '').toLowerCase().includes(term) ||
      (user.role || '').toLowerCase().includes(term) ||
      user.id.toString().includes(term)
    )
  })

  // Manejar eliminación
  const handleDelete = async (user) => {
    if (window.confirm(`¿Estás seguro de eliminar al usuario ${user.email}?`)) {
      try {
        await deleteUser(user.id)
        setMessage({ type: 'success', text: 'Usuario eliminado exitosamente' })
        setTimeout(() => setMessage(null), 3000)
      } catch (err) {
        setMessage({ type: 'error', text: err.error || 'Error al eliminar usuario' })
      }
    }
  }

  // Manejar edición
  const handleEdit = (user) => {
    setSelectedUser(user)
    setEditMode(true)
    setShowForm(true)
    setShowDetails(false)
  }

  // Manejar ver detalles
  const handleViewDetails = (user) => {
    setSelectedUser(user)
    setShowDetails(true)
    setShowForm(false)
    setEditMode(false)
  }

  // Manejar éxito del formulario
  const handleFormSuccess = () => {
    setShowForm(false)
    setEditMode(false)
    setSelectedUser(null)
    loadUsers()
    setMessage({ type: 'success', text: 'Operación completada exitosamente' })
    setTimeout(() => setMessage(null), 3000)
  }

  const handleCloseForm = () => {
    setShowForm(false)
    setEditMode(false)
    setSelectedUser(null)
  }

  const handleCloseDetails = () => {
    setShowDetails(false)
    setSelectedUser(null)
  }

  // Renderizar badge según rol
  const getRoleBadge = (role) => {
    const colors = {
      admin: { bg: '#fee2e2', color: '#dc2626', label: 'Admin' },
      seller: { bg: '#fed7aa', color: '#ea580c', label: 'Seller' },
      customer: { bg: '#dbeafe', color: '#2563eb', label: 'Customer' },
      user: { bg: '#e5e7eb', color: '#6b7280', label: 'User' }
    }
    const style = colors[role] || colors.user
    return (
      <span
        style={{
          background: style.bg,
          color: style.color,
          padding: '4px 10px',
          borderRadius: '12px',
          fontSize: '12px',
          fontWeight: 600,
          textTransform: 'capitalize'
        }}
      >
        {style.label}
      </span>
    )
  }

  // Formatear fecha (backend devuelve created_at en snake_case)
  const formatDate = (dateStr) => {
    if (!dateStr) return 'N/A'
    try {
      return new Date(dateStr).toLocaleDateString('es-CO', {
        year: 'numeric',
        month: 'short',
        day: 'numeric'
      })
    } catch {
      return 'N/A'
    }
  }

  if (loading && users.length === 0) {
    return <div className="loading-container">Cargando usuarios...</div>
  }

  return (
    <div className="users-container">
      <div className="users-header">
        <h2>Gestión de Usuarios</h2>
        <button
          className="btn-primary"
          onClick={() => {
            setSelectedUser(null)
            setEditMode(false)
            setShowForm(true)
            setShowDetails(false)
          }}
        >
          + Nuevo Usuario
        </button>
      </div>

      {message && (
        <AlertMessage
          type={message.type}
          message={message.text}
          onClose={() => setMessage(null)}
        />
      )}

      <div className="users-search">
        <input
          type="text"
          placeholder="Buscar por nombre, email, rol o ID..."
          value={searchTerm}
          onChange={(e) => setSearchTerm(e.target.value)}
          className="search-input"
        />
      </div>

      {error && (
        <div className="error-message">
          Error: {error}
          <button onClick={loadUsers}>Reintentar</button>
        </div>
      )}

      <div className="users-table-container">
        <table className="users-table">
          <thead>
            <tr>
              <th>ID</th>
              <th>Nombre</th>
              <th>Email</th>
              <th>Teléfono</th>
              <th>Rol</th>
              <th>Registro</th>
              <th>Acciones</th>
            </tr>
          </thead>
          <tbody>
            {filteredUsers.length === 0 ? (
              <tr>
                <td colSpan="7" className="no-data">
                  {searchTerm
                    ? `No se encontraron usuarios que coincidan con "${searchTerm}"`
                    : 'No hay usuarios registrados'}
                </td>
              </tr>
            ) : (
              filteredUsers.map((user) => (
                <tr key={user.id}>
                  <td>
                    <span
                      style={{
                        background: '#f3f4f6',
                        padding: '2px 8px',
                        borderRadius: '6px',
                        fontSize: '12px',
                        fontWeight: 600,
                        color: '#374151'
                      }}
                    >
                      #{user.id}
                    </span>
                  </td>
                  <td>
                    <div style={{ fontWeight: 500, color: '#333' }}>
                      {user.name || user.lastname
                        ? `${user.name || ''} ${user.lastname || ''}`.trim()
                        : '—'}
                    </div>
                  </td>
                  <td>{user.email}</td>
                  <td>{user.phone || '—'}</td>
                  <td>{getRoleBadge(user.role)}</td>
                  <td>{formatDate(user.created_at)}</td>
                  <td className="actions">
                    <button
                      className="btn-view"
                      onClick={() => handleViewDetails(user)}
                      title="Ver detalles"
                    >
                      👁️
                    </button>
                    <button
                      className="btn-edit"
                      onClick={() => handleEdit(user)}
                      title="Editar"
                    >
                      ✏️
                    </button>
                    <button
                      className="btn-delete"
                      onClick={() => handleDelete(user)}
                      title="Eliminar"
                    >
                      🗑️
                    </button>
                  </td>
                </tr>
              ))
            )}
          </tbody>
        </table>
      </div>

      {showForm && (
        <UserForm
          user={selectedUser}
          isEdit={editMode}
          onSuccess={handleFormSuccess}
          onClose={handleCloseForm}
        />
      )}

      {showDetails && selectedUser && (
        <UserDetails
          user={selectedUser}
          onClose={handleCloseDetails}
          onEdit={() => {
            handleCloseDetails()
            handleEdit(selectedUser)
          }}
        />
      )}
    </div>
  )
}

export default UsersView
```

# 📌 Notas didácticas clave para su clase

1. El bug createdAt vs created_at: este es un excelente caso para discutir. JSON Server (JavaScript puro) usa camelCase, MySQL (base de datos relacional) usa snake_case. El Model debería normalizar los nombres (convertir snake_case a camelCase). Aquí lo solucionamos directo en la Vista, pero la solución arquitectónicamente correcta sería hacerlo en el UserModel. Vale la pena discutirlo con los estudiantes.
2. Búsqueda ampliada: ahora el estudiante puede buscar por nombre completo. La línea const fullName = \user.name∣∣′′
3. user.name∣∣′′
4. {user.lastname || ''}`.toLowerCase()` une ambos campos y busca en el resultado. Es un patrón muy útil.
5. Badges de rol con colores semánticos: rojo para admin (poder), naranja para seller (comercio), azul para customer (cliente), gris para user (básico). No es solo estético — es semántica visual. Los colores comunican jerarquía.
6. colSpan="7": cada vez que agregan o quitan columnas a la tabla, deben actualizar el colSpan. Es un bug clásico que los estudiantes cometen: agregan una columna y la fila "no hay datos" queda desalineada.
7. formatDate con es-CO: usamos la localización colombiana para mostrar fechas (15 ene 2024). Es un detalle profesional de UX. Puede variarlo según el país del salón.
8. Estado vacío contextual: si el usuario escribió algo en el buscador y no hay resultados, mostramos un mensaje diferente al de "no hay usuarios registrados". Es un detalle de UX que mejora la experiencia.

## 🎨 Sugerencia opcional: estilos en Users.css
El código anterior usa estilos inline para los badges y las etiquetas. Podría moverlos a Users.css para mantener la separación de responsabilidades (CSS en CSS, JSX en JSX). Si quiere, en un paso extra lo hacemos. Por ahora, inline funciona y es más fácil de leer para los estudiantes en un solo archivo.

## ✅ Antes de continuar
Por favor:
1. Reemplace src/views/dashboard/UsersView.jsx con el contenido de arriba.
2. Recargue la app y vaya al dashboard → tab "Usuarios".
3. Verifique la nueva tabla:
  - Debe mostrar 7 columnas: ID | Nombre | Email | Teléfono | Rol | Registro | Acciones.
  - El ID debe aparecer como un badge gris pequeño (#1, #2).
  - El rol debe aparecer como badge de color según su tipo.
  - La fecha de registro ahora SÍ debe aparecer (antes salía "N/A" por el bug createdAt).
4. Pruebe el buscador:
  - Escriba parte del nombre → debe filtrar.
  - Escriba "admin" → debe mostrar solo admins.
  - Escriba un email parcial → debe filtrar.
5. Pruebe la interacción completa: ver detalles, editar, eliminar.

### Errores esperados (ninguno debería aparecer, pero por si acaso):
- Si la fecha sigue mostrando N/A → revise en la consola el objeto user para confirmar el nombre del campo (created_at vs createdAt).
- Si el rol aparece sin color → verifique que user.role no sea undefined. Si MySQL lo devuelve en otro nombre, ajústelo.

# 🎯 PASO 9: src/views/dashboard/UserDetails.jsx

## ¿Por qué este archivo ahora?
Este es el modal de detalles que se abre al hacer clic en 👁️. Es el hermano pequeño del UsersView: mientras la tabla muestra un resumen, este modal debe mostrar todos los datos del usuario seleccionado.

## 🔴 Problemas del código actual

### Problema 1: Mismo bug createdAt → created_at
```jsx
{user.createdAt ? new Date(user.createdAt).toLocaleString() : 'N/A'}
{user.updatedAt ? new Date(user.updatedAt).toLocaleString() : 'N/A'}
```
Su backend MySQL devuelve `created_at` y `updated_at` en `snake_case`. La vista muestra `N/A` en ambos campos siempre. Es el mismo bug que ya corregimos en UsersView.

### Problema 2: Solo muestra 4 campos, pero el backend devuelve 10
Su `getAllUsers` y `getUserById` devuelven:
```text
id, email, name, lastname, phone, image, role, created_at, updated_at
```
Pero el modal actual solo muestra: `id`, `email`, `createdAt`, `updatedAt`. Se pierde información valiosa: nombre, apellido, teléfono, rol.

### Problema 3: No hay distinción visual del rol
Aquí conviene reutilizar el mismo patrón de badge que usamos en UsersView. Consistencia visual = buena UX.

### Problema 4: Falta el botón "Eliminar" en el modal
En el flujo actual, el usuario tiene que cerrar el modal y buscar el 🗑️ en la tabla. Es más natural tener el botón de eliminar también en el modal, con la misma confirmación.

## ⚠️ Decisión sobre la imagen (image)
Su backend devuelve `image`, que puede ser:
- `null` (usuario sin imagen — caso más común en pruebas)
- Una URL a imagen (si se sube al backend)

Voy a incluirlo de forma simple: si existe `image`, mostrar `<img src={user.image} />`. Si no, mostrar un avatar generado con iniciales (bonito y didáctico). No complica demasiado y agrega valor visual a la clase.

## 🎯 ¿Qué cambios hago?
1. Corregir `createdAt`/`updatedAt` → `created_at`/`updated_at`.
2. Agregar todos los campos: `name`, `lastname`, `phone`, `role`.
3. Badge de rol con los mismos colores que UsersView (consistencia).
4. Avatar con iniciales si no hay imagen.
5. Botón "Eliminar" dentro del modal (además del que está en la tabla).
6. Formato de fecha con localización `es-CO` y hora completa (más informativo en un modal de detalles).

## 🚨 Sobre el botón "Eliminar" en el modal
Para que funcione, el componente necesita recibir un callback `onDelete`. Eso implica un cambio pequeño en UsersView (donde se instancia `<UserDetails />`). Lo incluyo aquí y le indico el cambio de una línea para UsersView.

## Archivo completo a reemplazar:

```jsx
// src/views/dashboard/UserDetails.jsx
const UserDetails = ({ user, onClose, onEdit, onDelete }) => {
  if (!user) return null

  // Generar iniciales para el avatar
  const getInitials = () => {
    const first = (user.name || '').charAt(0).toUpperCase()
    const second = (user.lastname || '').charAt(0).toUpperCase()
    if (first && second) return `${first}${second}`
    if (first) return first
    return (user.email || '?').charAt(0).toUpperCase()
  }

  // Colores por rol (mismos que UsersView para consistencia)
  const getRoleBadge = (role) => {
    const colors = {
      admin: { bg: '#fee2e2', color: '#dc2626', label: 'Administrador' },
      seller: { bg: '#fed7aa', color: '#ea580c', label: 'Vendedor' },
      customer: { bg: '#dbeafe', color: '#2563eb', label: 'Cliente' },
      user: { bg: '#e5e7eb', color: '#6b7280', label: 'Usuario' }
    }
    const style = colors[role] || colors.user
    return (
      <span
        style={{
          background: style.bg,
          color: style.color,
          padding: '4px 12px',
          borderRadius: '12px',
          fontSize: '12px',
          fontWeight: 600,
          display: 'inline-block'
        }}
      >
        {style.label}
      </span>
    )
  }

  // Formatear fecha completa con hora
  const formatDateTime = (dateStr) => {
    if (!dateStr) return 'N/A'
    try {
      return new Date(dateStr).toLocaleString('es-CO', {
        year: 'numeric',
        month: 'long',
        day: 'numeric',
        hour: '2-digit',
        minute: '2-digit'
      })
    } catch {
      return 'N/A'
    }
  }

  const handleDelete = () => {
    if (window.confirm(`¿Estás seguro de eliminar al usuario ${user.email}?`)) {
      if (onDelete) onDelete(user)
    }
  }

  return (
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        <div className="modal-header">
          <h3>Detalles del Usuario</h3>
          <button className="modal-close" onClick={onClose}>×</button>
        </div>

        {/* Cabecera con avatar */}
        <div
          style={{
            display: 'flex',
            alignItems: 'center',
            gap: '20px',
            padding: '20px',
            background: 'linear-gradient(135deg, #667eea15 0%, #764ba215 100%)',
            borderBottom: '1px solid #e1e5e9'
          }}
        >
          {user.image ? (
            <img
              src={user.image}
              alt={`${user.name} ${user.lastname}`}
              style={{
                width: '80px',
                height: '80px',
                borderRadius: '50%',
                objectFit: 'cover',
                border: '3px solid white',
                boxShadow: '0 4px 10px rgba(0,0,0,0.1)'
              }}
            />
          ) : (
            <div
              style={{
                width: '80px',
                height: '80px',
                borderRadius: '50%',
                background: 'linear-gradient(135deg, #667eea 0%, #764ba2 100%)',
                color: 'white',
                display: 'flex',
                alignItems: 'center',
                justifyContent: 'center',
                fontSize: '28px',
                fontWeight: 700,
                border: '3px solid white',
                boxShadow: '0 4px 10px rgba(0,0,0,0.1)',
                flexShrink: 0
              }}
            >
              {getInitials()}
            </div>
          )}
          <div style={{ flex: 1, minWidth: 0 }}>
            <div
              style={{
                fontSize: '20px',
                fontWeight: 600,
                color: '#333',
                marginBottom: '5px',
                overflow: 'hidden',
                textOverflow: 'ellipsis',
                whiteSpace: 'nowrap'
              }}
            >
              {user.name || user.lastname
                ? `${user.name || ''} ${user.lastname || ''}`.trim()
                : 'Sin nombre'}
            </div>
            <div style={{ color: '#666', fontSize: '14px', marginBottom: '8px' }}>
              {user.email}
            </div>
            {getRoleBadge(user.role)}
          </div>
        </div>

        {/* Cuerpo con todos los detalles */}
        <div className="user-details">
          <div className="detail-row">
            <span className="detail-label">ID:</span>
            <span className="detail-value">
              <span
                style={{
                  background: '#f3f4f6',
                  padding: '2px 8px',
                  borderRadius: '6px',
                  fontSize: '12px',
                  fontWeight: 600,
                  color: '#374151'
                }}
              >
                #{user.id}
              </span>
            </span>
          </div>

          <div className="detail-row">
            <span className="detail-label">Nombre:</span>
            <span className="detail-value">{user.name || '—'}</span>
          </div>

          <div className="detail-row">
            <span className="detail-label">Apellido:</span>
            <span className="detail-value">{user.lastname || '—'}</span>
          </div>

          <div className="detail-row">
            <span className="detail-label">Email:</span>
            <span className="detail-value">{user.email}</span>
          </div>

          <div className="detail-row">
            <span className="detail-label">Teléfono:</span>
            <span className="detail-value">{user.phone || '—'}</span>
          </div>

          <div className="detail-row">
            <span className="detail-label">Rol:</span>
            <span className="detail-value">{getRoleBadge(user.role)}</span>
          </div>

          <div className="detail-row">
            <span className="detail-label">Fecha Registro:</span>
            <span className="detail-value">{formatDateTime(user.created_at)}</span>
          </div>

          <div className="detail-row">
            <span className="detail-label">Última Actualización:</span>
            <span className="detail-value">{formatDateTime(user.updated_at)}</span>
          </div>
        </div>

        {/* Footer con acciones */}
        <div className="modal-footer">
          <button
            className="btn-delete"
            onClick={handleDelete}
            title="Eliminar usuario"
            style={{
              padding: '10px 16px',
              borderRadius: '8px',
              border: 'none',
              cursor: 'pointer',
              fontWeight: 500
            }}
          >
            🗑️ Eliminar
          </button>
          <div style={{ flex: 1 }} />
          <button className="btn-secondary" onClick={onClose}>
            Cerrar
          </button>
          <button className="btn-primary" onClick={onEdit}>
            Editar Usuario
          </button>
        </div>
      </div>
    </div>
  )
}

export default UserDetails
```

# 🔧 Cambio complementario en UsersView.jsx

Como agregamos el callback `onDelete` al `UserDetails`, hay que pasarlo desde `UsersView`. Solo modifique el bloque final donde se renderiza `<UserDetails />`:

**Antes:**
```jsx
{showDetails && selectedUser && (
  <UserDetails
    user={selectedUser}
    onClose={handleCloseDetails}
    onEdit={() => {
      handleCloseDetails()
      handleEdit(selectedUser)
    }}
  />
)}
```

**Después:**
```jsx
{showDetails && selectedUser && (
  <UserDetails
    user={selectedUser}
    onClose={handleCloseDetails}
    onEdit={() => {
      handleCloseDetails()
      handleEdit(selectedUser)
    }}
    onDelete={async (user) => {
      handleCloseDetails()
      await handleDelete(user)
    }}
  />
)}
```

Esto ya está contemplado en el `UsersView` del paso anterior: `handleDelete` existe, solo necesitamos invocarlo desde el modal y cerrar primero.

---

## 📌 Notas didácticas clave para su clase

1. **El bug createdAt vuelve a aparecer:** este es el tercer archivo con el mismo bug. Excelente oportunidad para hablar de código duplicado y deuda técnica. La solución correcta sería:
   - Crear una capa de normalización en `UserModel` que convierta `snake_case` $\rightarrow$ `camelCase` al recibir datos.
   - O crear un util compartido `formatDate()` en `src/utils/helpers.js` (que está vacío).
2. **Pregúntele a sus estudiantes:** ¿qué pasa si mañana el backend cambia `created_at` a `createdAt`? Hay que actualizar todos los archivos donde se usa. Con un util centralizado, solo cambia en un lugar. Esto es el principio DRY aplicado a la vida real.
3. **Avatar con iniciales:** cuando `user.image` es `null`, generamos iniciales. Es un patrón muy común (Slack, Gmail, GitHub). La lógica: primera letra del nombre + primera del apellido. Si falta alguna, usar la inicial del email. Es un buen ejercicio de lógica condicional.
4. **`flexShrink: 0` en el avatar:** sin esto, si el nombre es muy largo, el avatar se encoge. Es un detalle sutil de CSS que vale la pena comentar.
5. **Badge de rol duplicado:** entre `UsersView` y `UserDetails` hay código duplicado en `getRoleBadge()`. Es el momento perfecto para hablar de componentes reutilizables. En un siguiente paso (opcional) podríamos extraer `<RoleBadge role={...} />` como componente compartido.
6. **Confirmación de eliminación en dos lugares:** ahora el usuario puede eliminar desde la tabla (🗑️) y desde el modal. Ambos usan el mismo `handleDelete` — la lógica no se duplica, solo el disparador. Es un buen ejemplo de separación entre "evento" y "lógica".
7. **`formatDateTime` vs `toLocaleDateString`:** en el modal usamos formato con hora (`toLocaleString`) porque en una vista de detalles el usuario quiere ver la hora exacta. En la tabla usamos solo fecha (`toLocaleDateString`) porque el espacio es limitado. Contexto determina presentación — otra lección de UX.

---

## 🎯 Bonus didáctico: ¿por qué no normalizamos en el Model?

Sus estudiantes podrían preguntar: *"¿por qué no arreglamos esto de una vez en el UserModel?"*.

La respuesta correcta es: sí, es lo que debería hacerse. Aquí lo hicimos en las vistas para mantener el enfoque paso a paso. En un paso final (o como tarea para los estudiantes), podríamos agregar a `UserModel.getAllUsers`:

```js
const normalized = response.data.map(u => ({
  ...u,
  createdAt: u.created_at,
  updatedAt: u.updated_at
}))
```

Y luego las vistas usarían `user.createdAt` siempre. Esto es un ejercicio excelente para dejar como tarea: pedir a los estudiantes que hagan la normalización en el Model y refactoricen las vistas.

---

## ✅ Antes de continuar

Por favor:
1. Reemplace `src/views/dashboard/UserDetails.jsx` con el contenido de arriba.
2. Modifique `UsersView.jsx` para pasar `onDelete` al `<UserDetails />` (ver el bloque "Cambio complementario" arriba).
3. Recargue la app y vaya al dashboard $\rightarrow$ tab "Usuarios".
4. Haga clic en 👁️ de cualquier usuario:
   - Debe aparecer la cabecera con avatar (imagen o iniciales).
   - Debe mostrar todos los campos: ID, Nombre, Apellido, Email, Teléfono, Rol, Fecha Registro, Última Actualización.
   - Las fechas ahora SÍ deben aparecer (antes salía "N/A").
   - El rol debe aparecer como badge de color.
5. Pruebe el botón "🗑️ Eliminar" del modal:
   - Debe pedir confirmación.
   - Debe cerrar el modal.
   - Debe eliminar de la tabla.
   - Debe mostrar el mensaje de éxito.
6. Pruebe el botón "Editar Usuario": debe cerrar el modal de detalles y abrir el modal de edición.

### Errores esperados:
- Si el avatar no muestra iniciales correctas $\rightarrow$ revise que `user.name` y `user.lastname` no estén vacíos en MySQL.
- Si el rol aparece sin badge $\rightarrow$ revise que `user.role` esté definido y sea uno de los valores esperados.

🎯 PASO 10: src/views/dashboard/DashboardHeader.jsx

¿Por qué este archivo ahora?
Este es el encabezado del dashboard que muestra el nombre del usuario logueado y el botón de cerrar sesión. Ya casi todo está bien, pero hay un detalle silencioso que surgió con el backend: el name del usuario puede estar vacío o ausente.

🔴 Problema del código actual
```jsx
<div style={{ fontWeight: 600, color: '#333', fontSize: '14px' }}>  {user?.name || user?.email?.split('@')[0] || 'Usuario'}</div>
```
Este código asume que si user.name está vacío, se usa la parte del email antes del @. Está bien como fallback, pero hay tres casos reales con el backend que el código actual no maneja bien:

* **Caso 1: user.name es string vacío ""**
En MySQL, si el usuario fue creado sin nombre, name puede ser "" (string vacío). "" es falsy en JavaScript, así que la lógica || funciona. ✅

* **Caso 2: user.name es null**
Si el campo name es NULL en MySQL, el backend lo devuelve como null. También es falsy. ✅

* **Caso 3: user.name = " " (espacio en blanco)**
Si el usuario, por error, guardó un nombre con solo espacios, " " es truthy en JavaScript. La lógica actual lo aceptaría y mostraría espacios vacíos. ❌

Además, hay un problema visual: no muestra el rol del usuario (admin, seller, etc.), información que ahora sí está disponible.

🎯 ¿Qué cambios hago?
1. Limpieza de strings vacíos: usar .trim() para detectar nombres con solo espacios.
2. Mostrar el rol con badge pequeño: igual que en la tabla de usuarios, para consistencia visual.
3. Nombre completo: si tiene name y lastname, mostrarlos juntos ("Oliver Stone").
4. Mejor estructura interna con un helper para el nombre a mostrar.
5. Mantener los estilos inline (así está el archivo actual), pero limpiar un poco.

⚠️ Sobre useAuth y el objeto user
Mire su useAuth actual:
```js
const [currentUser, setCurrentUser] = useState(null)// ...const checkAuth = async () => {  const authState = AuthController.getAuthState()  setCurrentUser(authState.currentUser)  // viene de localStorage}
```
El currentUser viene del localStorage (donde guardamos el usuario en el login con storageService.setUser(user)). En el AuthModel.login guardamos:
```js
storageService.setUser({  id: userData.id,  email: userData.email,  name: userData.name,  lastname: userData.lastname,  role: userData.role,  image: userData.image,  phone: userData.phone})
```
Así que sí llegan name, lastname y role. El DashboardHeader solo necesita aprovecharlos mejor.

## Archivo completo a reemplazar:

```jsx
// src/views/dashboard/DashboardHeader.jsx
const DashboardHeader = ({ user, onLogout, activeTab, onTabChange }) => {
  // Construye el nombre a mostrar con fallbacks robustos
  const getDisplayName = () => {
    const name = (user?.name || '').trim()
    const lastname = (user?.lastname || '').trim()

    if (name && lastname) return `${name} ${lastname}`
    if (name) return name
    if (lastname) return lastname

    // Fallback: parte del email antes del @
    const email = user?.email || ''
    if (email.includes('@')) return email.split('@')[0]

    return 'Usuario'
  }

  // Badge de rol (mismo esquema que UsersView y UserDetails)
  const getRoleBadge = (role) => {
    const colors = {
      admin: { bg: '#fee2e2', color: '#dc2626', label: 'Admin' },
      seller: { bg: '#fed7aa', color: '#ea580c', label: 'Seller' },
      customer: { bg: '#dbeafe', color: '#2563eb', label: 'Customer' },
      user: { bg: '#e5e7eb', color: '#6b7280', label: 'User' }
    }
    const style = colors[role] || colors.user
    return (
      <span
        style={{
          background: style.bg,
          color: style.color,
          padding: '2px 8px',
          borderRadius: '10px',
          fontSize: '11px',
          fontWeight: 600,
          textTransform: 'capitalize',
          display: 'inline-block',
          marginTop: '2px'
        }}
      >
        {style.label}
      </span>
    )
  }

  // ============ Estilos ============
  const headerStyles = {
    background: 'white',
    boxShadow: '0 2px 10px rgba(0,0,0,0.1)',
    padding: '0 30px',
    display: 'flex',
    justifyContent: 'space-between',
    alignItems: 'center',
    position: 'sticky',
    top: 0,
    zIndex: 100,
    flexWrap: 'wrap',
    gap: '15px'
  }

  const headerLeftStyles = {
    display: 'flex',
    alignItems: 'center',
    gap: '15px',
    padding: '15px 0'
  }

  const headerRightStyles = {
    display: 'flex',
    alignItems: 'center',
    gap: '20px'
  }

  const badgeStyles = {
    background: 'linear-gradient(135deg, #667eea 0%, #764ba2 100%)',
    color: 'white',
    padding: '4px 12px',
    borderRadius: '20px',
    fontSize: '12px',
    fontWeight: 600
  }

  const tabStyles = {
    display: 'flex',
    gap: '5px',
    background: '#f0f0f0',
    padding: '4px',
    borderRadius: '12px'
  }

  const tabBtnStyles = (isActive) => ({
    padding: '8px 20px',
    border: 'none',
    background: isActive
      ? 'linear-gradient(135deg, #667eea 0%, #764ba2 100%)'
      : 'transparent',
    cursor: 'pointer',
    fontSize: '14px',
    fontWeight: 500,
    borderRadius: '8px',
    transition: 'all 0.3s ease',
    color: isActive ? 'white' : '#666'
  })

  const logoutBtnStyles = {
    background: '#ef4444',
    color: 'white',
    border: 'none',
    padding: '8px 16px',
    borderRadius: '8px',
    cursor: 'pointer',
    fontSize: '14px',
    fontWeight: 500
  }

  return (
    <header style={headerStyles}>
      <div style={headerLeftStyles}>
        <h1 style={{ fontSize: '20px', color: '#333', margin: 0 }}>
          Sistema de Información Web
        </h1>
        <span style={badgeStyles}>Admin Panel</span>
      </div>

      <div style={tabStyles}>
        <button
          style={tabBtnStyles(activeTab === 'dashboard')}
          onClick={() => onTabChange('dashboard')}
        >
          Dashboard
        </button>
        <button
          style={tabBtnStyles(activeTab === 'users')}
          onClick={() => onTabChange('users')}
        >
          Usuarios
        </button>
      </div>

      <div style={headerRightStyles}>
        <div style={{ textAlign: 'right' }}>
          <div
            style={{
              fontWeight: 600,
              color: '#333',
              fontSize: '14px',
              lineHeight: 1.2
            }}
          >
            {getDisplayName()}
          </div>
          <div
            style={{
              fontSize: '12px',
              color: '#666',
              marginBottom: '2px'
            }}
          >
            {user?.email}
          </div>
          {user?.role && getRoleBadge(user.role)}
        </div>
        <button onClick={onLogout} style={logoutBtnStyles}>
          Cerrar Sesión
        </button>
      </div>
    </header>
  )
}

export default DashboardHeader
```

# 📌 Notas didácticas clave para su clase

1. El bug del string vacío: `' '` (espacio) es `truthy` en JavaScript, mientras que `''` y `null` son `falsy`. Este es un error muy común entre estudiantes. El `.trim()` antes de la comparación lo resuelve. Enseñe a sus estudiantes a **SIEMPRE** hacer `.trim()` cuando comparan strings que vienen del usuario o de una API.
2. Construcción del nombre completo: en lugar de solo `user.name`, construimos "Nombre Apellido" cuando ambos existen. Es un pequeño detalle de UX que mejora mucho la experiencia. La función `getDisplayName()` encapsula toda esa lógica — una sola responsabilidad.
3. Badge de rol en el header: por tercera vez usamos `getRoleBadge()`. Ahora sí es el momento de hablar de refactorización:
4. "Hemos escrito esta función 3 veces. ¿Cómo podríamos evitar esa duplicación?"
5. Respuesta: crear un componente `src/views/common/RoleBadge.jsx` y reutilizarlo. Lo dejamos como tarea para los estudiantes — es un ejercicio perfecto de refactorización DRY.
6. `marginTop: '2px'` en el badge: pequeña decisión de diseño. Cuando un badge va debajo de un nombre, necesita un poco de espacio para respirar. Pequeños detalles así hacen la diferencia entre "se ve apretado" y "se ve profesional".
7. `lineHeight: 1.2` en el nombre: cuando el nombre tiene acentos (ej. "José Rodríguez") y se muestra con `fontWeight: 600`, la línea puede "respirar" de más. `lineHeight: 1.2` lo compacta. Detalle didáctico de CSS.
8. Fallback chain robusto:
9. `text`
10. `name + lastname` → `name` → `lastname` → parte del email → `'Usuario'`
11. Es una cadena de fallbacks ordenada de lo más específico a lo más genérico. Es un patrón que sus estudiantes deben dominar.

## 🎨 Sugerencia didáctica para profundizar
Aproveche este paso para introducir el concepto de null safety con optional chaining (`user?.name`). Su código ya lo usa, pero vale la pena explicar:
- `user?.name` → devuelve `undefined` si `user` es `null`/`undefined` en lugar de lanzar error.
- Encadenarlo con `||` da un fallback elegante.
- Sin `?.`, si `user` fuera null, toda la app explotaría. Optional chaining = defensive programming.

## 🚨 ¿Y el DashboardView?
Su `DashboardView.jsx` también muestra el nombre del usuario en el mensaje de bienvenida:
```jsx
<h2>Bienvenido, {currentUser?.name || currentUser?.email?.split('@')[0] || 'Usuario'}!</h2>
```
Este código tiene el mismo problema del `.trim()` (aunque en menor grado). Si quiere, puede hacer un ajuste similar. Pero como ya tiene el `DashboardHeader` funcionando bien, tal vez no sea crítico. Si le sobra tiempo, lo revisamos como paso opcional.

## ✅ Antes de continuar
Por favor:
1. Reemplace `src/views/dashboard/DashboardHeader.jsx` con el contenido de arriba.
2. Recargue la app con una sesión activa (haga login).
3. Verifique el header del dashboard:
  - Debe mostrar el nombre completo del usuario ("Oliver Stone").
  - Debe mostrar el email debajo del nombre.
  - Debe mostrar el badge de rol (Admin, Seller, Customer o User).
  - El botón "Cerrar Sesión" debe seguir funcionando.
4. Pruebe el caso borde (opcional, avanzado):
  - En DevTools → Application → Local Storage → `user_data`, edite el JSON y deje `"name": ""` y `"lastname": ""` → recargue → debería mostrar la parte del email antes del `@`.
  - Ahora ponga `"name": " "` (solo espacios) → recargue → debería ignorar los espacios y mostrar el fallback del email. Aquí se ve el `.trim()` en acción.

**Errores esperados (ninguno):**
- Si el nombre completo no se muestra, revise en consola `JSON.parse(localStorage.getItem('user_data'))` para confirmar los campos.

# 🎯 PASO 11: src/hooks/useAuth.js

## ¿Por qué este archivo ahora?
Este Hook es el punto central de toda la autenticación en el frontend. Todas las vistas (LoginView, RegisterView, Navbar, DashboardView) dependen de él. Ya que modificamos AuthModel, jwtService y AuthController, este es el momento de alinear el Hook con la nueva arquitectura.

## 🔴 Problemas del código actual

### Problema 1: No hay auto-logout cuando el token expira
El token JWT del backend expira en 1 hora (recuerde: `exp: Math.floor(Date.now() / 1000) + (60 * 60)` en `userController.js`). Pero el Hook no verifica la expiración periódicamente. Resultado: el usuario sigue "logueado" hasta que hace una petición y el backend responde 401. UX pobre.

### Problema 2: El listener de storage solo escucha `auth_token`
```js
const handleStorageChange = (e) => {  
  if (e.key === 'auth_token') {    
    checkAuth()  
  }
}
```
Esto está bien, pero si el `user_data` cambia (por ejemplo, otro tab actualiza el perfil), no se refleja. Menor, pero mejorable.

### Problema 3: El logout no limpia el estado si falla la red
```js
const logout = async () => {  
  return new Promise((resolve, reject) => {    
    AuthController.handleLogout(      
      () => { /* limpia estado */ },      
      (error) => { reject({ success: false, error }) }    
    )  
  })
}
```
Como `AuthModel.logout` solo limpia localStorage (no hace petición al backend), no puede fallar. Pero el `reject` deja el estado inconsistente: el localStorage está limpio pero `isAuthenticated` sigue en true. Bug potencial.

### Problema 4: No se expone `hasRole()` ni `getRole()`
Con el backend real, el role del usuario es clave (admin/seller/customer/user). Las vistas necesitan saberlo. Falta un helper.

### Problema 5: `checkAuth` no espera a `jwtService` (aunque es sincrónico, conceptualmente debería ser más explícito)

---

## 🎯 ¿Qué cambios hago?
1. **Auto-logout por expiración:** un `setInterval` que verifica cada minuto si el token expiró y, si es así, cierra sesión automáticamente.
2. **Logout más robusto:** limpiar el estado siempre, incluso si algo falla.
3. **Exponer `hasRole(role)` y `getRole()`:** helpers para que las vistas validen permisos.
4. **Escuchar también `user_data`:** para reflejar cambios de perfil entre tabs.
5. **Añadir `refreshAuth()`** como alias explícito de `checkAuth()` (útil en vistas).
6. **Documentar bien** los métodos expuestos.

## ⚠️ Sobre el setInterval y React StrictMode
En modo desarrollo con React 18+ y StrictMode, los efectos se ejecutan dos veces (a propósito, para detectar bugs). El `useEffect` de auto-logout debe limpiar correctamente el `setInterval` en el `return` para evitar timers duplicados. Su `main.jsx` tiene `<React.StrictMode>` envolviendo la app, así que esto es relevante.

**Archivo completo a reemplazar:**

```js
// src/hooks/useAuth.js
import { useState, useEffect, useCallback } from 'react'
import AuthController from '../controllers/AuthController'
import jwtService from '../services/jwtService'

const useAuth = () => {
  const [isAuthenticated, setIsAuthenticated] = useState(false)
  const [currentUser, setCurrentUser] = useState(null)
  const [token, setToken] = useState(null)
  const [loading, setLoading] = useState(true)

  // Sincroniza el estado interno con el storage y verifica expiración del JWT
  const checkAuth = useCallback(async () => {
    setLoading(true)
    const authState = AuthController.getAuthState()

    setIsAuthenticated(authState.isAuthenticated)
    setCurrentUser(authState.currentUser)
    setToken(authState.currentToken)

    setLoading(false)
    return authState.isAuthenticated
  }, [])

  // Alias explícito: útil para llamar desde vistas
  const refreshAuth = useCallback(() => checkAuth(), [checkAuth])

  // Efecto inicial + escucha cambios en storage + auto-logout por expiración
  useEffect(() => {
    checkAuth()

    // 1. Escuchar cambios entre pestañas (login/logout en otro tab)
    const handleStorageChange = (e) => {
      if (e.key === 'auth_token' || e.key === 'user_data') {
        checkAuth()
      }
    }
    window.addEventListener('storage', handleStorageChange)

    // 2. Auto-logout cuando el JWT expire
    const interval = setInterval(() => {
      const currentToken = localStorage.getItem('auth_token')
      if (!currentToken) return

      const isValid = jwtService.verifyToken(currentToken)
      if (!isValid) {
        console.warn('⏰ Token expirado — cerrando sesión automáticamente')
        // Logout silencioso
        AuthController.handleLogout(
          () => {
            setIsAuthenticated(false)
            setCurrentUser(null)
            setToken(null)
            // Pequeña señal para que la UI redirija (opcional)
            window.dispatchEvent(new Event('auth:expired'))
          },
          () => {
            // Aunque falle, limpiamos estado local igual
            setIsAuthenticated(false)
            setCurrentUser(null)
            setToken(null)
          }
        )
      }
    }, 60 * 1000) // cada minuto

    return () => {
      window.removeEventListener('storage', handleStorageChange)
      clearInterval(interval)
    }
  }, [checkAuth])

  // LOGIN
  const login = async (credentials) => {
    return new Promise((resolve, reject) => {
      AuthController.handleLogin(
        credentials,
        (user, authToken) => {
          setIsAuthenticated(true)
          setCurrentUser(user)
          setToken(authToken)
          resolve({ success: true, user, token: authToken })
        },
        (error) => {
          reject({ success: false, error })
        }
      )
    })
  }

  // REGISTER
  const register = async (userData) => {
    return new Promise((resolve, reject) => {
      AuthController.handleRegister(
        userData,
        (user) => {
          resolve({ success: true, user })
        },
        (error) => {
          reject({ success: false, error })
        }
      )
    })
  }

  // LOGOUT (robusto: siempre limpia el estado)
  const logout = async () => {
    return new Promise((resolve) => {
      AuthController.handleLogout(
        () => {
          setIsAuthenticated(false)
          setCurrentUser(null)
          setToken(null)
          resolve({ success: true })
        },
        (error) => {
          // Aunque el Model falle, limpiamos estado local
          console.warn('Logout con error, limpiando estado local:', error)
          setIsAuthenticated(false)
          setCurrentUser(null)
          setToken(null)
          resolve({ success: true, warning: error })
        }
      )
    })
  }

  // HELPERS DE ROL
  const getRole = () => currentUser?.role || null

  const hasRole = (role) => {
    if (!role) return false
    if (Array.isArray(role)) {
      return role.includes(currentUser?.role)
    }
    return currentUser?.role === role
  }

  return {
    // Estado
    isAuthenticated,
    currentUser,
    token,
    loading,

    // Acciones
    login,
    register,
    logout,
    checkAuth,
    refreshAuth,

    // Helpers de autorización
    getRole,
    hasRole
  }
}

export default useAuth
```

# 📌 Notas didácticas clave para su clase

1. **useCallback para checkAuth**: al envolverlo en `useCallback`, garantizamos que la referencia de la función no cambie entre renders, evitando re-ejecuciones infinitas del `useEffect`. Es un patrón fundamental cuando una función se usa como dependencia de un efecto.
2. **Auto-logout con setInterval**: cada minuto revisa el token. Si expiró, hace logout silencioso. Este es un patrón de UX proactiva — mejor cerrar sesión antes de que el usuario haga una petición y reciba un 401 inesperado.
3. **clearInterval en el cleanup**: sin esto, cada render crearía un nuevo intervalo, produciendo memory leaks y logout repetidos. React StrictMode en desarrollo ejecuta el efecto dos veces, así que el cleanup es crítico. Este es el error #1 de los estudiantes con `setInterval` en React.
4. **Logout robusto con resolve en ambos casos**: al inicio rechazábamos en caso de error. Ahora siempre resolvemos porque el logout local es lo único que importa (el backend no tiene endpoint de logout). Si quisiéramos invalidar tokens en el servidor, esto cambiaría.
5. **Helpers `getRole()` y `hasRole()`**: estas funciones son la base de la autorización en el cliente. Ejemplo de uso:

```jsx
const { hasRole } = useAuth()
// Mostrar el botón de eliminar solo para admins
{hasRole('admin') && <button onClick={handleDelete}>Eliminar</button>}
```

⚠️ **Advertencia importante para sus estudiantes**: `hasRole()` en el cliente es solo UX. Un usuario malicioso puede editar `localStorage` y hacerse pasar por admin en el cliente, pero el backend siempre valida el token y el rol reales. Este es un principio fundamental de seguridad: la autorización real vive en el servidor.

6. **`hasRole` acepta array o string**: esto permite llamadas versátiles:

```js
hasRole('admin')              // true si es admin
hasRole(['admin', 'seller'])  // true si es admin O seller
```

7. **Detalle de API design**.
8. **Evento `auth:expired`**: cuando el token expira, disparamos un evento global. Cualquier componente puede escucharlo y reaccionar (por ejemplo, mostrar un mensaje de "Sesión expirada"). Es un patrón de comunicación desacoplada entre componentes. No lo usamos aún, pero está listo para un paso opcional posterior.
9. **Escuchar `user_data` además de `auth_token`**: si el usuario actualiza su nombre en otro tab, este hook se enterará. Mejora la sincronización.

---

## 🚨 ¿Qué pasa ahora cuando el token expira?

**Antes** (con el bug del `exp` en milisegundos vs segundos):
- El token generado con `btoa` nunca expiraba realmente.
- `isAuthenticated` siempre devolvía `true`.

**Ahora**:
1. Al hacer login, el backend genera un JWT con `exp` en segundos (+1 hora).
2. Cada minuto, el `useAuth` verifica con `jwtService.verifyToken(token)`.
3. Cuando expira, se dispara logout silencioso y `isAuthenticated = false`.
4. El `ProtectedRoute` en `App.jsx` detecta `isAuthenticated = false` y redirige a `/login`.

### Prueba sugerida para sus estudiantes:
- Reduzca el `exp` en el backend a `60 * 1` (1 minuto) para probar rápido.
- Haga login y espere 1 minuto.
- En la consola, verá `⏰ Token expirado — cerrando sesión automáticamente`.
- La app debería redirigir a `/login` en la siguiente navegación.

---

## 🔧 Ajuste opcional en `DashboardView`

Si quiere reaccionar al evento `auth:expired` para mostrar un mensaje bonito:

```jsx
// En DashboardView.jsx, agregue este useEffect:
useEffect(() => {  
  const handleExpired = () => {    
    setLogoutMessage('Tu sesión ha expirado. Por favor, inicia sesión nuevamente.')    
    setTimeout(() => navigate('/login'), 2000)  
  }  
  window.addEventListener('auth:expired', handleExpired)  
  return () => window.removeEventListener('auth:expired', handleExpired)
}, [navigate])
```

Esto es opcional — no es necesario para que la migración funcione. Pero es un detalle de UX pulido que puede mostrar en clase.

---

## ✅ Antes de continuar

Por favor:
1. Reemplace `src/hooks/useAuth.js` con el contenido de arriba.
2. Recargue la app y haga login normalmente.
3. Verifique que todo lo anterior siga funcionando:
   - Login → redirige a dashboard ✅
   - Navbar muestra el nombre del usuario ✅
   - Dashboard muestra el header con rol ✅
   - Tab "Usuarios" funciona con el CRUD ✅
   - Logout cierra sesión y redirige a login ✅
4. Prueba especial de auto-logout (opcional pero muy didáctica):
   - En el backend `backend/nodejs_express/controllers/userController.js`, cambie temporalmente:
   
   ```js
   exp: Math.floor(Date.now() / 1000) + (60 * 1)  // 1 minuto
   ```
   
   - Reinicie el backend (`npm run dev`).
   - Haga login.
   - Espere 1 minuto viendo la consola del navegador.
   - Debería ver: `⏰ Token expirado — cerrando sesión automáticamente`.
   - Al hacer clic en cualquier opción del menú, debería redirigir a `/login`.
5. No olvide revertir el cambio a `60 * 60` (1 hora) después de la prueba.

### Errores esperados:
- Si el auto-logout no se dispara, revise que `jwtService.verifyToken` esté leyendo correctamente `exp` en segundos (PASO 2).
- Si hay timers múltiples, revise el `clearInterval` en el `return` del `useEffect`.

---

## 🎉 Resumen de la migración completada

| Paso | Archivo | Cambio principal |
| :--- | :--- | :--- |
| 1 | `config/api.js` | BASE_URL + endpoints del backend Express |
| 2 | `services/jwtService.js` | Solo decodificar, no firmar; Base64URL + exp en segundos |
| 3 | `services/httpService.js` | Manejo de respuesta `{success, message, data}` |
| 4 | `models/AuthModel.js` | Login con POST real, sin trampas; extraer token de `session_token` |
| 5 | `controllers/AuthController.js` | Validaciones más limpias; priorizar mensajes del backend |
| 6 | `models/UserModel.js` | CRUD completo con endpoints reales |
| 7 | `views/dashboard/UserForm.jsx` | Campos name/lastname/phone; corregir PATCH; validaciones |
| 8 | `views/dashboard/UsersView.jsx` | Tabla con más columnas; badges de rol; fix `created_at` |
| 9 | `views/dashboard/UserDetails.jsx` | Modal completo con avatar; fix `created_at`; botón eliminar |
| 10 | `views/dashboard/DashboardHeader.jsx` | Nombre completo + badge de rol en el header |
| 11 | `hooks/useAuth.js` | Auto-logout por expiración; helpers `hasRole()` y `getRole()` |

---

## 📝 Pasos opcionales que puede hacer después (tareas sugeridas para los estudiantes)

1. Extraer `RoleBadge` como componente reutilizable (`src/views/common/RoleBadge.jsx`) — el código está duplicado en 3 archivos.
2. Normalizar `snake_case` → `camelCase` en `UserModel` para eliminar el problema `created_at` vs `createdAt`.
3. Reaccionar al evento `auth:expired` en `DashboardView` para mostrar un mensaje de sesión expirada.
4. Añadir validación de email duplicado en el backend (retornar 409 si el email ya existe).
5. Migrar a `fetch` con `AbortController` para implementar el `TIMEOUT: 10000` que está declarado en `api.js` pero no se usa.
6. Proteger rutas por rol en el frontend: por ejemplo, que solo `admin` vea el tab "Usuarios".
7. Implementar refresh token: el backend actual no lo tiene, pero es una extensión natural.
