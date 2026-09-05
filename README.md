# PUNTO 1: CONFIGURACIÓN DEL ENTORNO DE DESARROLLO

## 📚 Explicación

Antes de empezar a codificar, necesitamos preparar nuestro entorno de trabajo. Este proyecto utiliza **React** con **Vite** como bundler, y se conecta a un **backend Node.js + Express** con autenticación JWT.

### Tecnologías principales

| Tecnología | Versión | Propósito |
|------------|---------|-----------|
| **Node.js** | v18+ | Entorno de ejecución JavaScript |
| **Vite** | 8.0.4 | Bundler rápido para aplicaciones React |
| **React** | 19.2.4 | Biblioteca para interfaces de usuario |
| **React Router DOM** | 7.14.0 | Enrutamiento para SPA |
| **JWT Decode** | 4.0.0 | Decodificación de tokens JWT |
| **ESLint** | 9.39.4 | Linter para calidad de código |

### Requisitos previos

- ✅ Node.js instalado (v18+)
- ✅ Editor de código (VS Code recomendado)
- ✅ Conocimiento básico de terminal
- ✅ Conexión a Internet

### Estructura del proyecto al finalizar
```bash
frontend_web/
└── react_node_express/
├── node_modules/ # Dependencias del proyecto
├── public/ # Archivos estáticos
│ └── vite.svg # Favicon de Vite
├── src/ # Código fuente (lo crearemos después)
│ ├── App.css
│ ├── App.jsx
│ ├── index.css
│ └── main.jsx
├── .gitignore # Archivos ignorados por Git
├── eslint.config.js # Configuración de ESLint
├── index.html # Página principal HTML
├── package-lock.json # Versiones exactas de dependencias
├── package.json # Configuración del proyecto
├── README.md # Documentación
└── vite.config.js # Configuración de Vite
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
Si no tienes Node.js: Descárgalo desde https://nodejs.org/