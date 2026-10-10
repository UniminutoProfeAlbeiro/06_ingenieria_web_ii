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