# Punto 1: Entorno de Desarrollo

## 📚 Explicación

Antes de empezar a codificar, necesitamos preparar nuestro entorno de trabajo. Este proyecto utiliza **React** con **Vite** como bundler, y se conecta a un **backend Node.js + Express** con autenticación JWT.

### Tecnologías principales

| Tecnología | Versión | Propósito |
| ------------------------------ | ------ | -------------------------------------- |
| **Node.js**                    | v18+   | Entorno de ejecución JavaScript        |
| **Vite**                       | 8.0.4  | Bundler rápido para aplicaciones React |
| **React**                      | 19.2.4 | Biblioteca para interfaces de usuario  |
| **React Router DOM**           | 7.14.0 | Enrutamiento para SPA                  |
| **JWT Decode**                 | 4.0.0  | Decodificación de tokens JWT           |
| **ESLint**                     | 9.39.4 | Linter para calidad de código          |

### Requisitos previos

- ✅ Node.js instalado (v18+)
- ✅ Editor de código (VS Code recomendado)
- ✅ Conocimiento básico de terminal
- ✅ Conexión a Internet

### Estructura del proyecto al finalizar

```bash
frontend_web/
└── react_node_express/
    ├── node_modules/          # Dependencias del proyecto
    ├── public/                # Archivos estáticos
    │   └── vite.svg           # Favicon de Vite
    ├── src/                   # Código fuente (lo crearemos después)
    │   ├── App.css
    │   ├── App.jsx
    │   ├── index.css
    │   └── main.jsx
    ├── .gitignore             # Archivos ignorados por Git
    ├── eslint.config.js       # Configuración de ESLint
    ├── index.html             # Página principal HTML
    ├── package-lock.json      # Versiones exactas de dependencias
    ├── package.json           # Configuración del proyecto
    ├── README.md              # Documentación
    └── vite.config.js         # Configuración de Vite
```

---

## 📝 Paso a paso

### 1. Verificar Node.js instalado

Abre tu terminal (PowerShell, CMD o Bash) y ejecuta:

```bash
node --version
# Debe mostrar v18.x.x o superior

npm --version
# Debe mostrar v9.x.x o superior
```

**Si no tienes Node.js:** Descárgalo desde [https://nodejs.org/](https://nodejs.org/)

### 2. Crear la carpeta del proyecto

```bash
# Crear la estructura de carpetas
mkdir -p frontend_web/react_node_express

# Navegar a la carpeta del proyecto
cd frontend_web/react_node_express
```

**Explicación:**

- `mkdir -p` crea las carpetas padre si no existen
- En Windows funciona igual en PowerShell

### 3. Inicializar el proyecto con Vite

```bash
npm create vite@latest . -- --template react
```

**¿Qué hace este comando?**

- Crea un nuevo proyecto React con Vite
- El `.` indica que use la carpeta actual
- `--template react` usa la plantilla oficial de React

**⚠️ IMPORTANTE: Seleccionar ESLint**

Cuando aparezca el prompt, selecciona **ESLint**:

```bash
◆  Which linter to use?
│  ● Oxlint       ← Opción por defecto
│  ○ ESLint       ← ¡Selecciona esta! (flecha abajo + Enter)
│  ↑/↓ to navigate • Enter: confirm
```

**¿Por qué ESLint y no Oxlint?**

| Característica | ESLint | Oxlint |
| ------------------------------ | ------------------------ | ------------------------------- |
| Madurez                        | ✅ Maduro (2013)          | ❌ Nuevo (2024)                  |
| Plugins                        | ✅ Miles disponibles      | ❌ Limitados                     |
| Configuración                  | ✅ Altamente configurable | ❌ Menos opciones                |
| Ecosistema React               | ✅ Soporte completo       | ⚠️ En desarrollo                |
| **Educativo**                  | ✅ Estándar industrial    | ❌ No recomendado para enseñanza |

### 4. Instalar dependencias base

```bash
npm install
```

Este comando instala todas las dependencias que Vite configuró automáticamente.

### 5. Instalar dependencias adicionales

**Dependencias de producción (necesarias para la aplicación):**

```bash
npm install react@19.2.4 react-dom@19.2.4 react-router-dom@7.14.0 jwt-decode@4.0.0
```

**Dependencias de desarrollo (solo para desarrollo):**

```bash
npm install -D @vitejs/plugin-react@6.0.1 @eslint/js@9.39.4 eslint@9.39.4 eslint-plugin-react-hooks@7.0.1 eslint-plugin-react-refresh@0.5.2 globals@17.4.0 @types/react@19.2.14 @types/react-dom@19.2.3 vite@8.0.4
```

**📦 Resumen de dependencias con versiones exactas:**

| Paquete | Versión | Tipo | Propósito |
| ------------------------------- | ------- | ---- | ---------------------------------- |
| **Dependencias de producción**  |         |      |                                    |
| react                           | 19.2.4  | prod | Biblioteca principal de React      |
| react-dom                       | 19.2.4  | prod | Renderizado de React en el DOM     |
| react-router-dom                | 7.14.0  | prod | Enrutamiento en React              |
| jwt-decode                      | 4.0.0   | prod | Decodificar tokens JWT             |
| **Dependencias de desarrollo**  |         |      |                                    |
| vite                            | 8.0.4   | dev  | Bundler y servidor de desarrollo   |
| @vitejs/plugin-react            | 6.0.1   | dev  | Plugin de React para Vite          |
| eslint                          | 9.39.4  | dev  | Linter de JavaScript               |
| @eslint/js                      | 9.39.4  | dev  | Configuración base de ESLint       |
| eslint-plugin-react-hooks       | 7.0.1   | dev  | Reglas para React Hooks            |
| eslint-plugin-react-refresh     | 0.5.2   | dev  | Reglas para React Refresh          |
| globals                         | 17.4.0  | dev  | Variables globales para ESLint     |
| @types/react                    | 19.2.14 | dev  | Tipos de TypeScript para React     |
| @types/react-dom                | 19.2.3  | dev  | Tipos de TypeScript para React DOM |

### 6. Verificar el archivo package.json

Abre `package.json` y confirma que tenga estas scripts y dependencias:

```json
{
  "name": "sis-web-mvc",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "lint": "eslint .",
    "preview": "vite preview"
  },
  "dependencies": {
    "jwt-decode": "^4.0.0",
    "react": "^19.2.4",
    "react-dom": "^19.2.4",
    "react-router-dom": "^7.14.0"
  },
  "devDependencies": {
    "@eslint/js": "^9.39.4",
    "@types/react": "^19.2.14",
    "@types/react-dom": "^19.2.3",
    "@vitejs/plugin-react": "^6.0.1",
    "eslint": "^9.39.4",
    "eslint-plugin-react-hooks": "^7.0.1",
    "eslint-plugin-react-refresh": "^0.5.2",
    "globals": "^17.4.0",
    "vite": "^8.0.4"
  }
}
```

### 7. Configurar vite.config.js

Crea el archivo `vite.config.js` en la raíz del proyecto con este contenido:

```javascript
// vite.config.js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    open: true // Abre automáticamente el navegador
  },
  define: {
    'process.env': {} // Para compatibilidad con variables de entorno
  }
})
```

**Explicación de la configuración:**

| Propiedad | Valor | Explicación |
| ----------------------------- | ----------- | -------------------------------------- |
| `plugins`                     | `[react()]` | Habilita el soporte de React en Vite   |
| `server.port`                 | `5173`      | Puerto donde corre la aplicación       |
| `server.open`                 | `true`      | Abre el navegador automáticamente      |
| `define['process.env']`       | `{}`        | Evita errores con variables de entorno |

### 8. Configurar ESLint (eslint.config.js)

El archivo `eslint.config.js` ya existe después de la instalación. Verifica que tenga este contenido:

```javascript
import js from '@eslint/js'
import globals from 'globals'
import reactHooks from 'eslint-plugin-react-hooks'
import reactRefresh from 'eslint-plugin-react-refresh'
import { defineConfig, globalIgnores } from 'eslint/config'

export default defineConfig([
  globalIgnores(['dist']),
  {
    files: ['**/*.{js,jsx}'],
    extends: [
      js.configs.recommended,
      reactHooks.configs.flat.recommended,
      reactRefresh.configs.vite,
    ],
    languageOptions: {
      ecmaVersion: 2020,
      globals: globals.browser,
      parserOptions: {
        ecmaVersion: 'latest',
        ecmaFeatures: { jsx: true },
        sourceType: 'module',
      },
    },
    rules: {
      'no-unused-vars': ['error', { varsIgnorePattern: '^[A-Z_]' }],
    },
  },
])
```

**Explicación de la configuración ESLint:**

| Sección | Función |
| -------------------------- | -------------------------------------- |
| `globalIgnores(['dist'])`  | Ignora la carpeta de build             |
| `files: ['**/*.{js,jsx}']` | Aplica a archivos .js y .jsx           |
| `extends`                  | Usa configuraciones recomendadas       |
| `ecmaVersion: 2020`        | Soporta características modernas de JS |
| `globals: globals.browser` | Reconoce variables del navegador       |
| `'no-unused-vars'`         | Previene variables sin usar            |

### 9. Verificar/Actualizar index.html

Abre `index.html` y confirma que tenga este contenido:

```html
<!DOCTYPE html>
<html lang="es">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/vite.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta name="description" content="Sistema de Información Web con React + MVC + JWT" />
    <title>Sistema de Información Web</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

**Elementos importantes:**

- `lang="es"` - Idioma español
- `meta description` - SEO y descripción
- `div id="root"` - Contenedor principal de React
- `script src="/src/main.jsx"` - Punto de entrada de la aplicación

### 10. Actualizar src/main.jsx

Modifica el archivo `src/main.jsx` con este contenido:

```jsx
// src/main.jsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App.jsx'
import './index.css'

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
)
```

### 11. Actualizar src/App.jsx

Modifica el archivo `src/App.jsx` con este contenido base (lo ampliaremos después):

```jsx
// src/App.jsx
import './App.css'

function App() {
  return (
    <div>
      <h1>Sistema de Información Web</h1>
      <p>Configuración inicial completada</p>
    </div>
  )
}

export default App
```

### 12. Verificar estructura de carpetas

Al finalizar este punto, tu estructura debe verse así:

```bash
frontend_web/
└── react_node_express/
    ├── node_modules/          # Dependencias (no se sube a Git)
    ├── public/                # Archivos estáticos
    │   └── vite.svg           # Favicon de Vite
    ├── src/                   # Código fuente
    │   ├── App.css
    │   ├── App.jsx
    │   ├── index.css
    │   └── main.jsx
    ├── .gitignore             # Archivos ignorados por Git
    ├── eslint.config.js       # Configuración de ESLint
    ├── index.html             # Página principal HTML
    ├── package-lock.json      # Versiones exactas de dependencias
    ├── package.json           # Configuración del proyecto
    ├── README.md              # Documentación
    └── vite.config.js         # Configuración de Vite
```

---

## ✅ Verificación

Para probar que todo está correctamente configurado:

### 1. Iniciar el servidor de desarrollo

```bash
npm run dev
```

### 2. Resultado esperado en la terminal

```bash
VITE v8.0.4  ready in 500 ms

➜  Local:   http://localhost:5173/
➜  Network: use --host to expose
➜  press h + enter to show help
```

### 3. Resultado esperado en el navegador

✅ Se abre automáticamente el navegador en `http://localhost:5173`
✅ Se muestra la página de inicio de React con Vite
✅ No hay errores en la consola del navegador (F12 → Consola)

---

## 🚨 Solución de problemas comunes

### Error: "Node.js version is too old"

**Problema:** La versión de Node.js es menor a v18

**Solución:** Actualiza Node.js a v18 o superior

```bash
# Verificar versión
node --version

# Descargar actualización desde:
# https://nodejs.org/
```

### Error: "Cannot find module 'vite'"

**Problema:** Las dependencias no se instalaron correctamente

**Solución:** Reinstala las dependencias

```bash
# En Windows:
rmdir /s node_modules
del package-lock.json

# En Mac/Linux:
rm -rf node_modules package-lock.json

# Luego:
npm install
```

### Error: "ESLint configuration is invalid"

**Problema:** El archivo `eslint.config.js` tiene formato incorrecto

**Solución:** Asegúrate de usar `export default` y la sintaxis moderna

```bash
// ✅ Correcto (ES Modules)
export default defineConfig([...])

// ❌ Incorrecto (CommonJS)
module.exports = defineConfig([...])
```

### El puerto 5173 está ocupado

**Problema:** Otro proceso está usando el puerto 5173

**Solución 1:** Cambia el puerto en `vite.config.js`:

```bash
server: {
  port: 5174,  // Cambia a otro puerto
  open: true
}
```

**Solución 2:** Encontrar y cerrar el proceso que usa el puerto

```bash
# En Windows (PowerShell):
netstat -ano | findstr :5173
taskkill /PID <PID> /F

# En Mac/Linux:
lsof -i :5173
kill -9 <PID>
```

### Error al instalar dependencias con versiones exactas

**Problema:** Conflictos de versiones entre paquetes

**Solución:** Instala sin versiones exactas primero

```bash
npm install react react-dom react-router-dom jwt-decode
npm install -D @vitejs/plugin-react eslint
```

---

## 📚 Recursos adicionales

- [Documentación oficial de Vite](https://vitejs.dev/)
- [Documentación de React](https://react.dev/)
- [Guía de ESLint](https://eslint.org/docs/latest/)
- [React Router DOM](https://reactrouter.com/)
- [Node.js](https://nodejs.org/)

---

## ✅ Checklist de verificación

Marca cada elemento cuando esté completado:

- [ ] Node.js v18+ instalado y verificado
- [ ] Proyecto creado con `npm create vite@latest`
- [ ] ESLint seleccionado como linter
- [ ] Dependencias de producción instaladas
- [ ] Dependencias de desarrollo instaladas
- [ ] `vite.config.js` configurado correctamente
- [ ] `eslint.config.js` verificado
- [ ] `index.html` actualizado con lang="es" y meta tags
- [ ] `main.jsx` verificado
- [ ] `App.jsx` actualizado
- [ ] `npm run dev` funciona correctamente
- [ ] Aplicación visible en `http://localhost:5173`

---

## 📝 Resumen de comandos (cheatsheet)

```bash
# 1. Verificar Node.js
node --version
npm --version

# 2. Crear proyecto
mkdir -p frontend_web/react_node_express
cd frontend_web/react_node_express
npm create vite@latest . -- --template react
# Seleccionar: ESLint

# 3. Instalar dependencias
npm install
npm install react@19.2.4 react-dom@19.2.4 react-router-dom@7.14.0 jwt-decode@4.0.0
npm install -D @vitejs/plugin-react@6.0.1 @eslint/js@9.39.4 eslint@9.39.4 eslint-plugin-react-hooks@7.0.1 eslint-plugin-react-refresh@0.5.2 globals@17.4.0 @types/react@19.2.14 @types/react-dom@19.2.3 vite@8.0.4

# 4. Iniciar servidor
npm run dev

# 5. Si hay errores, reinstalar
rm -rf node_modules package-lock.json
npm install
```

---

## 🎯 Resultado final del Punto 1

Al completar este punto, tendrás:

#### ✅ Un proyecto React con Vite configurado
#### ✅ ESLint instalado y configurado
#### ✅ Todas las dependencias necesarias instaladas
#### ✅ El servidor de desarrollo funcionando
#### ✅ La aplicación visible en el navegador

---
**¡Estás listo para comenzar a desarrollar!**
