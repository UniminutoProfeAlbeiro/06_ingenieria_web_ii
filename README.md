# Curso: Sistema de Inventario con React + Bootstrap + MySQL

Curso practico, clase por clase, para construir el **frontend de un sistema de
inventario** desde cero: login real contra una base de datos MySQL (con
contraseña encriptada) y un dashboard con gestion CRUD (crear, editar,
activar/desactivar) de **usuarios, categorias, subcategorias y productos**
usando datos simulados en JavaScript.

## Stack usado

| Parte                         | Tecnologia                                   |
|--------------------------------|-----------------------------------------------|
| Frontend                       | React 19 + Vite + Bootstrap 5 + React Router  |
| Backend (solo para el login)   | Node.js + Express + JWT                        |
| Base de datos                  | MySQL (XAMPP) - base `inventario_react`        |
| Encriptacion de contraseñas    | bcrypt (`bcryptjs`)                            |
| Gestiones (usuarios/categorias/subcategorias/productos) | Datos simulados en JS (`useState` + `localStorage`) |
| Automatizacion                 | npm scripts + `concurrently`                   |

## Por que el login es real y las gestiones son simuladas

Este curso separa a proposito dos cosas:

1. **El login** (`frontend/src/pages/LoginPage.jsx`) SI habla con una base de
   datos MySQL real a traves de un backend en Node/Express. Se valida el
   usuario y se compara la contraseña encriptada con bcrypt.
2. **Las pantallas de gestion** dentro del dashboard (Usuarios, Categorias,
   Subcategorias, Productos) trabajan con **datos simulados en JavaScript**
   (arreglos en `frontend/src/data/`, persistidos en `localStorage`), porque
   el foco del curso es practicar React/Bootstrap en el frontend sin
   necesitar una API completa para cada modulo.

## Estructura del repositorio

```
REACT/
├── backend/            API en Node/Express que valida el login contra MySQL
├── frontend/           Aplicacion React (Vite) con Bootstrap
├── database/           Script SQL para crear la base de datos inventario_react
├── clases/             El curso en si, dividido en 8 clases
│   ├── clase-01-entorno-y-bootstrap/
│   ├── clase-02-base-datos-y-backend/
│   ├── clase-03-login-funcional/
│   ├── clase-04-dashboard-y-rutas/
│   ├── clase-05-crud-usuarios/
│   ├── clase-06-crud-categorias-subcategorias/
│   ├── clase-07-crud-productos/
│   └── clase-08-integracion-final/
└── package.json        Comandos de automatizacion para correr todo junto
```

Cada carpeta de `clases/` contiene:
- `README.md`: que se construye en esa clase y por que, con explicacion linea
  a linea de los archivos reales del proyecto (todos los archivos de
  `backend/` y `frontend/` estan comentados).
- `ejercicios/EJERCICIOS.md`: ejercicios de practica para reforzar el tema.

## Requisitos previos

- [Node.js](https://nodejs.org/) 18 o superior instalado.
- [XAMPP](https://www.apachefriends.org/) instalado, con **MySQL** encendido
  desde el Panel de Control de XAMPP (no hace falta Apache ni PHP).
- Un editor de codigo (recomendado: VS Code).

## Puesta en marcha rapida

```bash
# 1) Enciende MySQL desde el Panel de Control de XAMPP

# 2) Crea la base de datos y la tabla "usuarios"
#    Opcion A: importa database/inventario_react.sql desde phpMyAdmin
#    Opcion B: por linea de comandos
"C:\xampp\mysql\bin\mysql.exe" -u root < database/inventario_react.sql

# 3) Instala las dependencias del backend y del frontend
npm run install:all

# 4) Copia backend/.env.example como backend/.env y ajustalo si tu
#    configuracion de MySQL no es la de XAMPP por defecto (root sin clave)

# 5) Crea el usuario administrador de prueba (contraseña encriptada con bcrypt)
npm run seed

# 6) Levanta el backend y el frontend juntos con un solo comando
npm run dev
```

Esto abre:
- Backend: `http://localhost:4000`
- Frontend: `http://localhost:5173`

### Credenciales de prueba (creadas por `npm run seed`)

```
Correo:     admin@inventario.com
Contraseña: Admin123!
```

## Como seguir el curso (como estudiante)

Empieza por [`clases/clase-01-entorno-y-bootstrap/README.md`](clases/clase-01-entorno-y-bootstrap/README.md)
y avanza en orden. Cada clase se apoya en el codigo real que ya esta
funcionando en `backend/` y `frontend/`, con comentarios en cada archivo
explicando que hace cada funcion y las lineas no evidentes. Cada clase
tiene su seccion "Paso a paso" (que archivos crear y en que orden) y su
carpeta `ejercicios/` con practica para reforzar el tema.

Para una vista rapida de **que archivo crear o editar en cada clase**, sin
tener que abrir los 8 README uno por uno, consulta el
[`MANUAL_DE_IMPLEMENTACION.md`](MANUAL_DE_IMPLEMENTACION.md).

---

## Guia para el instructor: como dictar cada clase

Esta seccion es para quien va a **enseñar** el curso (en vivo, grabado, o
como material de autoestudio guiado). Explica la metodologia sugerida por
clase, el cronograma completo y como empaquetar/vender el curso.

### 1. Cronograma completo (8 clases)

| # | Clase | Duracion sugerida | Objetivo de la sesion | Entregable / actividad final |
|---|-------|-------------------|------------------------|-------------------------------|
| 01 | Entorno y Bootstrap | 1.5 - 2 h | Dejar el entorno listo (Vite + Bootstrap) | Pagina de bienvenida con una tarjeta Bootstrap propia (ejercicio 1) |
| 02 | Base de datos y backend | 2 - 2.5 h | Crear `inventario_react` y la API de login en Node/Express | Login validado con curl/Postman + usuario admin creado con `npm run seed` |
| 03 | Login funcional | 2 h | Conectar el formulario de React con la API | Login end-to-end funcionando en el navegador |
| 04 | Dashboard y rutas protegidas | 2 h | Sidebar + Topbar + rutas privadas | Navegar el dashboard sin poder entrar sin sesion |
| 05 | CRUD de usuarios | 2.5 h | Primer modulo de gestion simulada + hook reutilizable | Modulo de usuarios completo (crear/editar/desactivar) |
| 06 | CRUD de categorias y subcategorias | 2.5 h | Reutilizar el hook + simular una relacion entre datos | Modulo relacional completo y funcionando |
| 07 | CRUD de productos | 2 h | Formato de moneda, campos numericos, alertas de stock | Modulo de productos completo |
| 08 | Integracion final | 2 - 3 h | Repaso general + proyecto final integrador | Modulo "Marcas" construido por el estudiante desde cero |

**Total: 16-19 horas de contenido**, ideal para dictar en:
- **Formato semanal**: 8 sesiones de una semana cada una (2 meses de curso).
- **Formato bootcamp**: 3-4 dias intensivos (2 clases por dia).
- **Formato autoestudio**: el estudiante avanza a su ritmo con los README y
  ejercicios como guia, sin sesiones en vivo.

### 2. Estructura recomendada de CADA clase (~2 horas)

Usa siempre el mismo esqueleto; los estudiantes aprenden mas rapido cuando la
rutina es predecible:

1. **Repaso y dudas (10 min)** - revisar en conjunto los ejercicios de la
   clase anterior. Pide a 1-2 estudiantes que muestren su solucion.
2. **Teoria breve (15-20 min)** - explica el "por que" antes del "como".
   Usa el README de la clase como guion: cada seccion numerada (ej. "3. Como
   se encripta la contraseña") es un bloque de explicacion de pizarra.
3. **Demo en vivo / live coding (30-40 min)** - construye junto a los
   estudiantes los archivos de la seccion "Paso a paso: que hacer y que
   archivos crear" del README de esa clase, en el mismo orden. No copies y
   pegues: escribe el codigo en vivo para que vean el proceso, no solo el
   resultado.
4. **Practica guiada (30 min)** - los estudiantes replican lo mismo en su
   propio computador mientras circulas resolviendo dudas.
5. **Ejercicios para la semana (asignar, no resolver en clase)** - la
   carpeta `ejercicios/EJERCICIOS.md` de cada clase. Se revisan al inicio de
   la siguiente sesion (paso 1).
6. **Cierre (10 min)** - resume en 3 frases que se construyo hoy y por que
   importa para el proyecto completo. Motiva mostrando como esa pieza encaja
   en el sistema final.

### 3. Como evaluar el progreso

- **Por clase**: revisa que el estudiante haya creado los archivos listados
  en la seccion "Paso a paso" del README y que la app siga funcionando
  (`npm run dev` sin errores en consola).
- **Proyecto final**: el ejercicio integrador de la Clase 08 (modulo
  "Marcas" construido sin ayuda, repitiendo el patron de la Clase 06) es la
  evaluacion final: si el estudiante lo logra solo, domino el patron
  completo del curso.
- **Portafolio**: al terminar, el estudiante tiene un proyecto real y
  presentable (login contra MySQL + dashboard CRUD) para mostrar en
  entrevistas o en su GitHub.

---

## Como empaquetar y vender este curso

Ideas concretas para convertir este material en un curso pago. Son
sugerencias de estructura comercial, no garantias de resultados: ajusta
precios y canal segun tu audiencia.

### Formatos de venta

| Formato | Que incluye | Publico ideal |
|---------|-------------|----------------|
| **Curso grabado (on-demand)** | Videos de cada clase + este repositorio + ejercicios | Autoestudio, precio de entrada bajo, alto volumen |
| **Cohorte en vivo** | Sesiones en vivo por Zoom/Meet siguiendo el cronograma, revision de ejercicios, cupo limitado | Precio mas alto, mayor compromiso y tasa de finalizacion |
| **Membresia / comunidad** | Acceso al curso + canal de Discord/WhatsApp para dudas + nuevas clases futuras | Ingreso recurrente (suscripcion mensual) |
| **Mentoria 1:1 (upsell)** | Sesiones individuales de revision de codigo despues del curso | Ticket alto, para quienes quieren feedback personalizado |

### Canales sugeridos

- **Plataformas de curso**: Hotmart, Teachable, Thinkific, Udemy (menor
  margen pero mayor descubribilidad).
- **Venta directa**: Gumroad o un checkout propio + entrega del repositorio
  en un ZIP o acceso a un repo privado de GitHub.
- **Distribucion gratuita como gancho**: publica la Clase 01 completa gratis
  (en YouTube o el blog) para demostrar la calidad de la explicacion y el
  nivel de detalle de los comentarios en el codigo; el resto del curso queda
  de pago.

### Que resaltar en el marketing (diferenciadores reales de este material)

- **Codigo 100% comentado, linea por linea**, en español: ideal para
  estudiantes que recien empiezan y necesitan entender el "por que", no solo
  copiar y pegar.
- **Login real contra MySQL con contraseñas encriptadas (bcrypt)**: no es un
  curso "de juguete", conecta con una base de datos de verdad.
- **Proyecto completo de portafolio**: al terminar, el estudiante tiene un
  sistema de inventario funcionando que puede mostrar en entrevistas.
- **Cada clase indica exactamente que archivos crear**: reduce la friccion
  de "no se por donde empezar", uno de los mayores puntos de abandono en
  cursos de programacion.

### Bonos que aumentan el valor percibido

- Certificado de finalizacion al completar el ejercicio integrador de la
  Clase 08.
- Codigo fuente completo descargable (este repositorio).
- Sesion de preguntas y respuestas en vivo (grabada, disponible para
  compradores del curso on-demand tambien).
- Plantilla resuelta del modulo "Marcas" (el ejercicio final) como
  referencia, para quienes se traban.

### Material para la pagina de ventas

- Un video corto (60-90s) mostrando el flujo completo funcionando: login
  real -> dashboard -> crear/editar/desactivar un producto. Es el mejor
  "gancho" porque demuestra que el resultado final es un sistema real, no
  solo diapositivas.
- Capturas de pantalla del dashboard (Sidebar + tabla de productos) y del
  codigo comentado (por ejemplo `UsuariosPage.jsx`) para mostrar el nivel de
  detalle de las explicaciones.
- El indice de las 8 clases (tabla de la seccion 1 de esta guia) como
  temario/programa del curso.
