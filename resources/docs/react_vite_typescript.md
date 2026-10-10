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
