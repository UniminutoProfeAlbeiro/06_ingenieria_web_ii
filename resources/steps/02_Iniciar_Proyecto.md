# Punto 2 — Iniciar el Proyecto
## Estructura de Carpetas y Archivos

> Documentación técnica del proyecto **React + Vite + Node.js + Express**, organizada bajo una arquitectura MVC adaptada a React.

## 📄 **PUNTO 2: INICIAR EL PROYECTO (Estructura de Carpetas y Archivos)**
```markdown
# Punto 2: Iniciar el Proyecto

## 📚 Explicación

Una vez configurado el entorno de desarrollo, el siguiente paso es **organizar la estructura de carpetas y archivos** del proyecto. Esta organización sigue el patrón **MVC (Modelo-Vista-Controlador)** adaptado a React, que separa las responsabilidades y facilita el mantenimiento del código.

### ¿Por qué MVC en React?

| Componente | Responsabilidad | Ejemplo en el proyecto |
|------------|-----------------|------------------------|
| **Model** | Lógica de datos y comunicación con API | `src/models/AuthModel.js` |
| **View** | Interfaz de usuario (componentes React) | `src/views/auth/LoginView.jsx` |
| **Controller** | Lógica de negocio y coordinación | `src/controllers/AuthController.js` |

### Estructura final de carpetas

```

frontend\_web/
└── react\_node\_express/
├── public/ # Archivos estáticos
│ └── vite.svg
├── src/ # Código fuente principal
│ ├── config/ # Configuraciones globales
│ │ ├── api.js # Configuración de API
│ │ ├── constants.js # Constantes del sistema
│ │ └── routes.js # Rutas de la aplicación
│ ├── controllers/ # Controladores (lógica de negocio)
│ │ ├── AuthController.js
│ │ ├── DashboardController.js
│ │ └── UserController.js
│ ├── hooks/ # Custom Hooks de React
│ │ ├── useAuth.js
│ │ ├── useDashboard.js
│ │ └── useUsers.js
│ ├── models/ # Modelos (comunicación con API)
│ │ ├── AuthModel.js
│ │ ├── DashboardModel.js
│ │ └── UserModel.js
│ ├── services/ # Servicios (utilidades)
│ │ ├── httpService.js
│ │ ├── jwtService.js
│ │ └── storageService.js
│ ├── styles/ # Estilos CSS
│ │ ├── common.css
│ │ ├── Login.css
│ │ ├── Navbar.css
│ │ ├── Register.css
│ │ └── Users.css
│ ├── utils/ # Utilidades
│ │ ├── helpers.js
│ │ └── validators.js
│ ├── views/ # Vistas (componentes de página)
│ │ ├── auth/
│ │ │ ├── LoginView\.jsx
│ │ │ └── RegisterView\.jsx
│ │ ├── common/
│ │ │ ├── AlertMessage.jsx
│ │ │ ├── LoadingSpinner.jsx
│ │ │ └── Navbar.jsx
│ │ ├── dashboard/
│ │ │ ├── DashboardHeader.jsx
│ │ │ ├── DashboardStats.jsx
│ │ │ ├── DashboardView\.jsx
│ │ │ ├── UserDetails.jsx
│ │ │ ├── UserForm.jsx
│ │ │ └── UsersView\.jsx
│ │ └── layouts/
│ │ └── MainLayout.jsx
│ ├── App.css
│ ├── App.jsx
│ ├── index.css
│ └── main.jsx
├── .gitignore
├── eslint.config.js
├── index.html
├── package-lock.json
├── package.json
├── README.md
└── vite.config.js
```text
---

## 📝 Paso a paso

### 1. Crear la estructura de carpetas

Desde la raíz del proyecto (`frontend_web/react_node_express/`), crea las siguientes carpetas:

```bash
# Crear todas las carpetas necesarias
mkdir -p src/config
mkdir -p src/controllers
mkdir -p src/hooks
mkdir -p src/models
mkdir -p src/services
mkdir -p src/styles
mkdir -p src/utils
mkdir -p src/views/auth
mkdir -p src/views/common
mkdir -p src/views/dashboard
mkdir -p src/views/layouts
````

**Explicación de cada carpeta:**

| **CarpetaPropósito**   |                                                   |
| ---------------------- | ------------------------------------------------- |
| `src/config/`          | Configuraciones globales (API, rutas, constantes) |
| `src/controllers/`     | Controladores con lógica de negocio               |
| `src/hooks/`           | Custom Hooks de React para reutilizar lógica      |
| `src/models/`          | Modelos para comunicación con API                 |
| `src/services/`        | Servicios utilitarios (HTTP, JWT, Storage)        |
| `src/styles/`          | Archivos CSS globales y de componentes            |
| `src/utils/`           | Funciones helper y validadores                    |
| `src/views/auth/`      | Vistas de autenticación (Login, Register)         |
| `src/views/common/`    | Componentes comunes (Navbar, Alertas)             |
| `src/views/dashboard/` | Vistas del panel de administración                |
| `src/views/layouts/`   | Layouts o plantillas de página                    |

---

### 2. Crear archivos de configuración

#### 2.1 Configuración de API (`src/config/api.js`)
```javascript
// src/config/api.js
const API_CONFIG = {
  BASE_URL: 'http://10.1.196.46:3000/api',
  TIMEOUT: 10000,
  ENDPOINTS: {
    LOGIN: '/users/login',
    REGISTER: '/users/create',
    USERS: '/users',
    USER_BY_ID: '/users/:id',
    USER_UPDATE: '/users/:id',
    USER_PATCH: '/users/:id',
    USER_DELETE: '/users/delete/:id'
  }
}

export default API_CONFIG
```

**Explicación:**

- `BASE_URL`: URL base del backend
- `TIMEOUT`: Tiempo máximo de espera para peticiones
- `ENDPOINTS`: Rutas de la API organizadas por recurso

#### 2.2 Configuración de rutas (`src/config/routes.js`)
```javascript
// src/config/routes.js
export const ROUTES = {
  LOGIN: '/login',
  DASHBOARD: '/dashboard',
  HOME: '/'
}

export const PROTECTED_ROUTES = [ROUTES.DASHBOARD]
export const PUBLIC_ROUTES = [ROUTES.LOGIN, ROUTES.HOME]
```

**Explicación:**

- `ROUTES`: Definición de rutas de la aplicación
- `PROTECTED_ROUTES`: Rutas que requieren autenticación
- `PUBLIC_ROUTES`: Rutas públicas (sin autenticación)

---

### 3. Crear servicios (`src/services/`)

#### 3.1 Servicio HTTP (`src/services/httpService.js`)
```javascript
// src/services/httpService.js
import API_CONFIG from '../config/api'

class HttpService {
  constructor() {
    this.baseURL = API_CONFIG.BASE_URL
  }

  getToken() {
    return localStorage.getItem('auth_token')
  }

  getHeaders(includeAuth = true) {
    const headers = {
      'Content-Type': 'application/json'
    }
    
    if (includeAuth) {
      const token = this.getToken()
      if (token) {
        headers['Authorization'] = `Bearer ${token}`
      }
    }
    
    return headers
  }

  async handleResponse(response) {
    let data = {}
    try {
      data = await response.json()
    } catch (e) {
      console.error('Error al parsear JSON:', e)
    }
    
    if (!response.ok) {
      const errorMessage = data.message || data.error || `Error HTTP: ${response.status}`
      throw new Error(errorMessage)
    }
    
    return data
  }

  async post(endpoint, data, includeAuth = true) {
    try {
      const url = `${this.baseURL}${endpoint}`
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

  async get(endpoint, includeAuth = true) {
    try {
      const url = `${this.baseURL}${endpoint}`
      const response = await fetch(url, {
        method: 'GET',
        headers: this.getHeaders(includeAuth)
      })
      return await this.handleResponse(response)
    } catch (error) {
      console.error(`❌ GET ${endpoint} error:`, error)
      throw error
    }
  }

  async put(endpoint, data, includeAuth = true) {
    try {
      const url = `${this.baseURL}${endpoint}`
      const response = await fetch(url, {
        method: 'PUT',
        headers: this.getHeaders(includeAuth),
        body: JSON.stringify(data)
      })
      return await this.handleResponse(response)
    } catch (error) {
      console.error(`❌ PUT ${endpoint} error:`, error)
      throw error
    }
  }

  async patch(endpoint, data, includeAuth = true) {
    try {
      const url = `${this.baseURL}${endpoint}`
      const response = await fetch(url, {
        method: 'PATCH',
        headers: this.getHeaders(includeAuth),
        body: JSON.stringify(data)
      })
      return await this.handleResponse(response)
    } catch (error) {
      console.error(`❌ PATCH ${endpoint} error:`, error)
      throw error
    }
  }

  async delete(endpoint, includeAuth = true) {
    try {
      const url = `${this.baseURL}${endpoint}`
      const response = await fetch(url, {
        method: 'DELETE',
        headers: this.getHeaders(includeAuth)
      })
      return await this.handleResponse(response)
    } catch (error) {
      console.error(`❌ DELETE ${endpoint} error:`, error)
      throw error
    }
  }
}

export default new HttpService()
```

**Explicación de métodos HTTP:**

| **MétodoUsoEndpoint ejemplo** |                               |                     |
| ----------------------------- | ----------------------------- | ------------------- |
| `GET`                         | Obtener datos                 | `/users`            |
| `POST`                        | Crear recursos                | `/users/create`     |
| `PUT`                         | Actualizar todo el recurso    | `/users/:id`        |
| `PATCH`                       | Actualizar campos específicos | `/users/:id`        |
| `DELETE`                      | Eliminar recursos             | `/users/delete/:id` |

#### 3.2 Servicio JWT (`src/services/jwtService.js`)
```javascript
// src/services/jwtService.js
class JWTService {
  decodeToken(token) {
    try {
      const parts = token.split('.')
      if (parts.length !== 3) return null
      const payload = JSON.parse(atob(parts[1]))
      return payload
    } catch (error) {
      console.error('Error al decodificar token:', error)
      return null
    }
  }

  verifyToken(token) {
    try {
      if (!token) return false
      
      const payload = this.decodeToken(token)
      if (!payload) return false
      
      // Verificar expiración con tolerancia de 5 minutos
      if (payload.exp) {
        const now = Date.now()
        const expTime = payload.exp * 1000
        const tolerance = 5 * 60 * 1000
        
        if (expTime + tolerance < now) {
          console.warn('Token expirado')
          return false
        }
      }
      
      return true
    } catch (error) {
      console.error('Error al verificar token:', error)
      return false
    }
  }

  getTokenRemainingTime(token) {
    try {
      const payload = this.decodeToken(token)
      if (!payload || !payload.exp) return 0
      const remainingTime = (payload.exp * 1000) - Date.now()
      return remainingTime > 0 ? remainingTime : 0
    } catch (error) {
      return 0
    }
  }
}

export default new JWTService()
```

**Explicación:**

- `decodeToken`: Decodifica el payload del JWT
- `verifyToken`: Verifica que el token sea válido y no haya expirado
- `getTokenRemainingTime`: Calcula el tiempo restante del token

#### 3.3 Servicio de almacenamiento (`src/services/storageService.js`)
```javascript
// src/services/storageService.js
class StorageService {
  constructor(storageType = 'localStorage') {
    this.storage = storageType === 'localStorage' ? localStorage : sessionStorage
    this.tokenKey = 'auth_token'
    this.userKey = 'user_data'
    this.roleKey = 'user_role'
  }

  setToken(token) {
    return this.setItem(this.tokenKey, token)
  }

  getToken() {
    return this.getItem(this.tokenKey)
  }

  removeToken() {
    return this.removeItem(this.tokenKey)
  }

  setUser(user) {
    return this.setItem(this.userKey, JSON.stringify(user))
  }

  getUser() {
    const user = this.getItem(this.userKey)
    return user ? JSON.parse(user) : null
  }

  removeUser() {
    return this.removeItem(this.userKey)
  }

  setUserRole(role) {
    return this.setItem(this.roleKey, role)
  }

  getUserRole() {
    return this.getItem(this.roleKey)
  }

  clearSession() {
    this.removeToken()
    this.removeUser()
    this.removeUserRole()
  }

  setItem(key, value) {
    try {
      this.storage.setItem(key, value)
      return true
    } catch (error) {
      console.error('Error al guardar en storage:', error)
      return false
    }
  }

  getItem(key) {
    try {
      return this.storage.getItem(key)
    } catch (error) {
      console.error('Error al obtener del storage:', error)
      return null
    }
  }

  removeItem(key) {
    try {
      this.storage.removeItem(key)
      return true
    } catch (error) {
      console.error('Error al eliminar del storage:', error)
      return false
    }
  }

  clear() {
    try {
      this.storage.clear()
      return true
    } catch (error) {
      console.error('Error al limpiar storage:', error)
      return false
    }
  }

  hasItem(key) {
    return this.getItem(key) !== null
  }
}

export default new StorageService('localStorage')
```

**Explicación:**

- `localStorage`: Almacenamiento persistente (cerrar navegador no borra)
- `sessionStorage`: Almacenamiento temporal (cerrar pestaña borra)
- Métodos para guardar/obtener/eliminar token, usuario y rol

---

### 4. Crear archivos de utilidades

#### 4.1 Validadores (`src/utils/validators.js`)
```javascript
// src/utils/validators.js

/**
 * Valida que el email tenga formato correcto
 */
export const isValidEmail = (email) => {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
  return emailRegex.test(email)
}

/**
 * Valida que la contraseña tenga al menos 6 caracteres
 */
export const isValidPassword = (password) => {
  return password && password.length >= 6
}

/**
 * Valida que dos contraseñas coincidan
 */
export const doPasswordsMatch = (password, confirmPassword) => {
  return password === confirmPassword
}

/**
 * Valida que un campo no esté vacío
 */
export const isRequired = (value) => {
  return value && value.trim().length > 0
}

/**
 * Valida que el rol sea válido
 */
export const isValidRole = (role) => {
  const validRoles = ['admin', 'seller', 'customer', 'user']
  return validRoles.includes(role)
}
```

#### 4.2 Helpers (`src/utils/helpers.js`)
```javascript
// src/utils/helpers.js

/**
 * Formatea una fecha a string legible
 */
export const formatDate = (dateString) => {
  if (!dateString) return 'N/A'
  try {
    return new Date(dateString).toLocaleString('es-ES')
  } catch {
    return 'N/A'
  }
}

/**
 * Obtiene el nombre completo del usuario
 */
export const getFullName = (user) => {
  if (!user) return 'Usuario'
  return `${user.name} ${user.lastname || ''}`.trim() || user.email
}

/**
 * Capitaliza la primera letra de un string
 */
export const capitalize = (str) => {
  if (!str) return ''
  return str.charAt(0).toUpperCase() + str.slice(1).toLowerCase()
}

/**
 * Trunca un texto a una longitud máxima
 */
export const truncateText = (text, maxLength = 50) => {
  if (!text) return ''
  if (text.length <= maxLength) return text
  return text.substring(0, maxLength) + '...'
}
```

---

### 5. Crear estilos globales

#### 5.1 Estilos comunes (`src/styles/common.css`)
```css
/* src/styles/common.css */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', 'Oxygen',
    'Ubuntu', 'Cantarell', 'Fira Sans', 'Droid Sans', 'Helvetica Neue',
    sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  background: #f5f7fa;
}

.dashboard-container {
  min-height: 100vh;
  background: #f5f7fa;
}

.dashboard-main {
  max-width: 1200px;
  margin: 0 auto;
  padding: 30px;
}

/* Estilos para tarjetas de información */
.info-card {
  background: white;
  border-radius: 15px;
  padding: 25px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.05);
}

.info-card h3 {
  color: #333;
  font-size: 18px;
  margin-bottom: 15px;
  padding-bottom: 10px;
  border-bottom: 2px solid #667eea;
  display: inline-block;
}

/* Spinner de carga */
.spinner-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 200px;
}

.spinner {
  width: 40px;
  height: 40px;
  border: 4px solid #f3f3f3;
  border-top: 4px solid #667eea;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

/* Alertas */
.alert {
  padding: 15px 20px;
  border-radius: 10px;
  margin-bottom: 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.alert-error {
  background: #fee2e2;
  color: #dc2626;
  border: 1px solid #fecaca;
}

.alert-success {
  background: #dcfce7;
  color: #16a34a;
  border: 1px solid #bbf7d0;
}

.alert-close {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
  color: inherit;
}
```

---

### 6. Crear componentes comunes

#### 6.1 Navbar (`src/views/common/Navbar.jsx`)
```jsx
// src/views/common/Navbar.jsx
import { Link, useNavigate, useLocation } from 'react-router-dom'
import useAuth from '../../hooks/useAuth'
import '../../styles/Navbar.css'

const Navbar = () => {
  const { isAuthenticated, logout, currentUser } = useAuth()
  const navigate = useNavigate()
  const location = useLocation()

  const handleLogout = async () => {
    await logout()
    navigate('/login')
  }

  // No mostrar navbar en dashboard
  if (location.pathname.includes('/dashboard')) {
    return null
  }

  return (
    <nav className="navbar">
      <div className="navbar-container">
        <div className="navbar-logo">
          <Link to="/">
            <h2>Sistema Web</h2>
          </Link>
        </div>

        <div className="navbar-menu">
          {!isAuthenticated ? (
            <>
              <Link to="/login" className="nav-link">
                Iniciar Sesión
              </Link>
              <Link to="/register" className="nav-link btn-primary">
                Registrarse
              </Link>
            </>
          ) : (
            <>
              <span className="user-welcome">
                Hola, {currentUser?.name || currentUser?.email}
              </span>
              <Link to="/dashboard" className="nav-link">
                Dashboard
              </Link>
              <button onClick={handleLogout} className="nav-link btn-logout">
                Cerrar Sesión
              </button>
            </>
          )}
        </div>
      </div>
    </nav>
  )
}

export default Navbar
```

#### 6.2 Estilos del Navbar (`src/styles/Navbar.css`)
```css
/* src/styles/Navbar.css */
.navbar {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  position: sticky;
  top: 0;
  z-index: 1000;
}

.navbar-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 1rem 2rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.navbar-logo a {
  color: white;
  text-decoration: none;
}

.navbar-logo h2 {
  font-size: 1.5rem;
  margin: 0;
}

.navbar-menu {
  display: flex;
  gap: 1.5rem;
  align-items: center;
}

.nav-link {
  color: white;
  text-decoration: none;
  padding: 0.5rem 1rem;
  border-radius: 8px;
  transition: all 0.3s ease;
  font-weight: 500;
}

.nav-link:hover {
  background: rgba(255, 255, 255, 0.2);
  transform: translateY(-2px);
}

.btn-primary {
  background: white;
  color: #667eea !important;
}

.btn-primary:hover {
  background: #f0f0f0;
}

.btn-logout {
  background: #ef4444;
  border: none;
  cursor: pointer;
  font-size: 1rem;
}

.btn-logout:hover {
  background: #dc2626;
}

.user-welcome {
  color: white;
  font-weight: 500;
  padding: 0.5rem 1rem;
  background: rgba(255, 255, 255, 0.2);
  border-radius: 8px;
}
```

#### 6.3 AlertMessage (`src/views/common/AlertMessage.jsx`)
```jsx
// src/views/common/AlertMessage.jsx
import { useEffect, useState } from 'react'

const AlertMessage = ({ type, message, onClose, duration = 5000 }) => {
  const [isVisible, setIsVisible] = useState(true)

  useEffect(() => {
    if (duration > 0) {
      const timer = setTimeout(() => {
        setIsVisible(false)
        if (onClose) setTimeout(onClose, 300)
      }, duration)
      
      return () => clearTimeout(timer)
    }
  }, [duration, onClose])

  if (!isVisible) return null

  return (
    <div className={`alert alert-${type}`}>
      <span className="alert-message">{message}</span>
      {onClose && (
        <button className="alert-close" onClick={() => {
          setIsVisible(false)
          setTimeout(onClose, 300)
        }}>
          ×
        </button>
      )}
    </div>
  )
}

export default AlertMessage
```

#### 6.4 LoadingSpinner (`src/views/common/LoadingSpinner.jsx`)
```jsx
// src/views/common/LoadingSpinner.jsx
const LoadingSpinner = () => {
  return (
    <div className="spinner-container">
      <div className="spinner"></div>
      <p>Cargando...</p>
    </div>
  )
}

export default LoadingSpinner
```

---

### 7. Verificar estructura final de carpetas
```bash
frontend_web/
└── react_node_express/
    ├── public/
    │   └── vite.svg
    ├── src/
    │   ├── config/
    │   │   ├── api.js
    │   │   ├── constants.js
    │   │   └── routes.js
    │   ├── controllers/
    │   │   ├── AuthController.js
    │   │   ├── DashboardController.js
    │   │   └── UserController.js
    │   ├── hooks/
    │   │   ├── useAuth.js
    │   │   ├── useDashboard.js
    │   │   └── useUsers.js
    │   ├── models/
    │   │   ├── AuthModel.js
    │   │   ├── DashboardModel.js
    │   │   └── UserModel.js
    │   ├── services/
    │   │   ├── httpService.js
    │   │   ├── jwtService.js
    │   │   └── storageService.js
    │   ├── styles/
    │   │   ├── common.css
    │   │   ├── Login.css
    │   │   ├── Navbar.css
    │   │   ├── Register.css
    │   │   └── Users.css
    │   ├── utils/
    │   │   ├── helpers.js
    │   │   └── validators.js
    │   ├── views/
    │   │   ├── auth/
    │   │   │   ├── LoginView.jsx
    │   │   │   └── RegisterView.jsx
    │   │   ├── common/
    │   │   │   ├── AlertMessage.jsx
    │   │   │   ├── LoadingSpinner.jsx
    │   │   │   └── Navbar.jsx
    │   │   ├── dashboard/
    │   │   │   ├── DashboardHeader.jsx
    │   │   │   ├── DashboardStats.jsx
    │   │   │   ├── DashboardView.jsx
    │   │   │   ├── UserDetails.jsx
    │   │   │   ├── UserForm.jsx
    │   │   │   └── UsersView.jsx
    │   │   └── layouts/
    │   │       └── MainLayout.jsx
    │   ├── App.css
    │   ├── App.jsx
    │   ├── index.css
    │   └── main.jsx
    ├── .gitignore
    ├── eslint.config.js
    ├── index.html
    ├── package-lock.json
    ├── package.json
    ├── README.md
    └── vite.config.js
```

---

## ✅ Verificación

### 1. Verificar que todas las carpetas existen
```bash
# En Windows (PowerShell)
tree src /F

# En Mac/Linux
find src -type f
```

### 2. Verificar que la aplicación sigue funcionando
```bash
npm run dev
```

**Resultado esperado:** ✅ La aplicación se inicia sin errores en `http://localhost:5173`

---

## 🚨 Solución de problemas comunes

### Error: "Cannot find module './config/api'"

**Problema:** La ruta de importación es incorrecta

**Solución:** Verifica que la ruta relativa sea correcta (usa `../` según la profundidad)

### Error: "Failed to resolve import"

**Problema:** El archivo no existe o la extensión es incorrecta

**Solución:** Asegúrate de que todos los archivos tengan la extensión `.js` o `.jsx`

### Error: "Cannot find module 'react'"

**Problema:** Las dependencias no se instalaron

**Solución:** Ejecuta `npm install` nuevamente

---

## 📚 Recursos adicionales

- [Documentación de React sobre Hooks](https://react.dev/reference/react)
- [Guía de estructura de proyectos React](https://react.dev/learn/thinking-in-react)
- [Patrón MVC en React](https://es.react.dev/learn/thinking-in-react)

---

## ✅ Checklist de verificación

- □ 

  Todas las carpetas creadas (`config`, `controllers`, `hooks`, `models`, `services`, `styles`, `utils`, `views`)
- □ 

  Archivo `api.js` creado en `config/`
- □ 

  Archivo `routes.js` creado en `config/`
- □ 

  Servicio HTTP (`httpService.js`) creado
- □ 

  Servicio JWT (`jwtService.js`) creado
- □ 

  Servicio Storage (`storageService.js`) creado
- □ 

  Validadores (`validators.js`) creados
- □ 

  Helpers (`helpers.js`) creados
- □ 

  Estilos comunes (`common.css`) creados
- □ 

  Navbar (`Navbar.jsx`) y sus estilos creados
- □ 

  AlertMessage y LoadingSpinner creados
- □ 

  `npm run dev` funciona sin errores

---

## 📝 Resumen de comandos (cheatsheet)
```bash
# Crear todas las carpetas
mkdir -p src/config src/controllers src/hooks src/models src/services src/styles src/utils src/views/auth src/views/common src/views/dashboard src/views/layouts

# Verificar estructura
tree src /F   # Windows
find src -type f   # Mac/Linux

# Iniciar aplicación
npm run dev
```

---

## 🎯 Resultado final del Punto 2

Al completar este punto, tendrás:

#### ✅ Estructura de carpetas MVC organizada

#### ✅ Servicios HTTP, JWT y Storage implementados

#### ✅ Configuraciones de API y rutas definidas

#### ✅ Componentes comunes (Navbar, Alertas, Spinner) creados

#### ✅ La aplicación sigue funcionando correctamente

---

**¡Excelente! La base del proyecto está lista. En el siguiente punto comenzaremos a implementar la autenticación.**
