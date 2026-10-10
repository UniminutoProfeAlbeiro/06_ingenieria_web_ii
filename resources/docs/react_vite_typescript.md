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

### 🎯 PASO 1: src/config/api.js

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