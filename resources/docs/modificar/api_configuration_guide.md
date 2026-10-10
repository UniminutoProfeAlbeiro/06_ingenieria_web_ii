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