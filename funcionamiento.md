# Documento de Funcionamiento - Fleet Tracking

## 📋 Descripción General del Proyecto

**Fleet Tracking** es una aplicación web completa de rastreo y gestión de flotas que permite:
- Geolocalización en tiempo real de usuarios
- Gestión de marcadores personalizados en mapas
- Comunicación en tiempo real entre múltiples clientes
- Seguimiento de ubicaciones GPS históricas
- Sistema de notificaciones en tiempo real

El proyecto está compuesto por dos partes principales:
1. **Frontend**: Aplicación Angular 20 con mapas interactivos
2. **Backend**: Servidor Node.js con Socket.IO y base de datos SQLite

---

## 🏗️ Arquitectura del Sistema

### Diagrama de Componentes

```
┌─────────────────────────────────────────────────────────────┐
│                        FRONTEND (Angular)                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ MapView      │  │ SocketService│  │ ApiService   │     │
│  │ Component    │◄─┤              │◄─┤              │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│         │                  │                  │             │
│         └──────────────────┼──────────────────┘             │
│                            │                                │
└────────────────────────────┼────────────────────────────────┘
                             │
                             │ HTTP REST API
                             │ WebSocket (Socket.IO)
                             │
┌────────────────────────────┼────────────────────────────────┐
│                            ▼                                │
│                   BACKEND (Node.js/Express)                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ Express      │  │ Socket.IO    │  │ SQLite DB    │     │
│  │ REST API     │  │ Server       │  │              │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│         │                  │                  │             │
│         └──────────────────┼──────────────────┘             │
│                            │                                │
└────────────────────────────┴────────────────────────────────┘
```

### Flujo de Comunicación

1. **Conexión Inicial**:
   - Cliente Angular se conecta vía Socket.IO
   - Cliente envía `user:join` para registrarse
   - Servidor guarda usuario en `usersOnline` y notifica a todos
   - Servidor envía marcadores iniciales al nuevo cliente

2. **Sincronización GPS (Flujo Mejorado)**:
   - Cliente nuevo se conecta y se registra (`usuario-conectado`)
   - Servidor envía inmediatamente todas las ubicaciones de usuarios ya conectados (`ubicaciones-usuarios-conectados`)
   - Cliente nuevo muestra marcadores de todos los usuarios existentes
   - Todos comparten ubicaciones periódicamente con `ubicacion-actual`

3. **Actualización de Ubicación (Tiempo Real)**:
   - Cliente envía `ubicacion-actual` cada vez que se actualiza la ubicación GPS
   - Servidor guarda en mapa de últimas ubicaciones y hace broadcast a otros clientes
   - Otros clientes actualizan marcadores en el mapa
   - Servidor mantiene mapa de últimas ubicaciones para enviar a nuevos usuarios

4. **Gestión de Marcadores**:
   - Cliente crea/actualiza/elimina marcador vía HTTP REST
   - Servidor guarda en base de datos y archivo (si aplica)
   - Servidor notifica cambios vía Socket.IO (`marcador:creado`, etc.)

---

## 🖥️ SERVIDOR (Backend)

### Descripción General

El servidor es una aplicación Node.js construida con:
- **Express.js**: Framework web para REST API
- **Socket.IO**: Comunicación en tiempo real bidireccional
- **SQLite (better-sqlite3)**: Base de datos ligera y rápida
- **Multer**: Manejo de carga de archivos
- **CORS**: Control de acceso entre orígenes

### Ubicación
```
server/
├── server.js          # Servidor principal
├── database.js        # Lógica de base de datos
├── utils/
│   └── fileUpload.js  # Manejo de archivos
├── storage/           # Archivos subidos
└── fleet_tracking.db  # Base de datos SQLite
```

### Configuración

El servidor se configura mediante variables de entorno (archivo `.env`):

```env
PORT=3000                                    # Puerto del servidor
CORS_ORIGINS=http://localhost:4200          # Orígenes permitidos (separados por comas)
JSON_LIMIT=10mb                             # Límite de tamaño de payload
NODE_ENV=development                        # Entorno de ejecución
```

### Estructura de la Base de Datos

#### Tabla: `usuarios`
Almacena información de los usuarios conectados:
- `id` (TEXT, PRIMARY KEY): Identificador único del usuario
- `num_usuario` (INTEGER): Número secuencial asignado
- `nombre` (TEXT): Nombre del usuario (opcional)
- `plataforma` (TEXT): Plataforma del cliente (web, mobile, etc.)
- `modelo_dispositivo` (TEXT): Modelo del dispositivo (opcional)
- `fecha_registro` (DATETIME): Fecha de primer registro
- `ultima_conexion` (DATETIME): Última vez que se conectó

#### Tabla: `marcadores`
Almacena los marcadores personalizados creados por usuarios:
- `id` (TEXT, PRIMARY KEY): UUID del marcador
- `user_id` (TEXT, FOREIGN KEY): ID del usuario que lo creó
- `lat` (REAL): Latitud
- `lng` (REAL): Longitud
- `categoria` (TEXT): 'alerta', 'peligro', 'informacion'
- `descripcion` (TEXT): Descripción del marcador (mínimo 10 caracteres)
- `archivo` (TEXT): URL del archivo adjunto
- `timestamp` (TEXT): Fecha/hora de creación
- `created_at` (TEXT): Timestamp de creación
- `updated_at` (TEXT): Timestamp de última actualización

#### Tabla: `coordenadas_gps`
Almacena el historial de coordenadas GPS de todos los usuarios:
- `id` (INTEGER, PRIMARY KEY): ID autoincremental
- `user_id` (TEXT, FOREIGN KEY): ID del usuario
- `lat` (REAL): Latitud
- `lng` (REAL): Longitud
- `accuracy` (REAL): Precisión GPS en metros (se llena automáticamente desde la API de Geolocalización)
- `timestamp` (TEXT): Fecha/hora de la coordenada
- `created_at` (TEXT): Timestamp de creación

**Nota**: El campo `accuracy` contiene la precisión del GPS en metros proporcionada por la API de Geolocalización del navegador. Valores típicos:
- GPS al aire libre: 5-20 metros
- GPS en interiores: 20-100 metros
- WiFi/Red móvil: 50-500 metros

### API REST Endpoints

#### Health Check
```
GET /api/health
```
Verifica el estado del servidor.
**Respuesta**: `{ status: 'ok', message: 'Fleet Tracking Server is running' }`

#### Marcadores

##### Obtener todos los marcadores
```
GET /api/marcadores
```
**Respuesta**: `{ success: true, data: Marcador[] }`

##### Obtener un marcador por ID
```
GET /api/marcadores/:id
```
**Respuesta**: `{ success: true, data: Marcador }`
**Error 404**: Si no se encuentra el marcador

##### Crear un nuevo marcador
```
POST /api/marcadores
Content-Type: multipart/form-data
```
**Body (FormData)**:
- `lat` (number): Latitud
- `lng` (number): Longitud
- `categoria` (string): 'alerta' | 'peligro' | 'informacion'
- `descripcion` (string): Mínimo 10 caracteres
- `archivo` (File, opcional): Archivo a adjuntar
- `user_id` (string, opcional): ID del usuario

**Respuesta**: `{ success: true, data: Marcador }`
**Error 400**: Si faltan datos o validación falla

##### Actualizar un marcador
```
PUT /api/marcadores/:id
Content-Type: multipart/form-data
```
**Body**: Similar a crear, todos los campos opcionales

##### Eliminar un marcador
```
DELETE /api/marcadores/:id
```
**Respuesta**: `{ success: true, message: 'Marcador eliminado correctamente' }`

##### Estadísticas de marcadores
```
GET /api/marcadores/stats/estadisticas
```
**Respuesta**: `{ success: true, data: [{ categoria, total, primera_fecha, ultima_fecha }] }`

#### Coordenadas GPS

##### Guardar coordenada GPS
```
POST /api/coordenadas
Content-Type: application/json
```
**Body**:
```json
{
  "lat": 11.0049,
  "lng": -74.8060,
  "accuracy": 10.5,
  "user_id": "uuid-del-usuario"
}
```

##### Obtener últimas coordenadas
```
GET /api/coordenadas?limit=100
```
**Query Params**:
- `limit` (number, opcional): Número de coordenadas a obtener (default: 100)

##### Obtener coordenadas por rango de tiempo
```
GET /api/coordenadas/rango?fechaInicio=2024-01-01&fechaFin=2024-12-31
```
**Query Params**:
- `fechaInicio` (string, requerido): Fecha de inicio (ISO string)
- `fechaFin` (string, requerido): Fecha de fin (ISO string)

### Eventos Socket.IO

#### Eventos Cliente → Servidor

##### `user:join` (Nuevo - Flujo Mejorado)
Usuario nuevo se conecta al sistema.
```javascript
socket.emit('user:join', {
  userId: 'uuid-del-usuario',
  nombre: 'Nombre Usuario'
});
```

##### `usuario-conectado` (Legacy - Compatibilidad)
Registra un usuario cuando se conecta (evento anterior, mantenido por compatibilidad).
```javascript
socket.emit('usuario-conectado', {
  id: 'uuid-del-usuario',
  nombre: 'Nombre Usuario',
  plataforma: 'web',
  modeloDispositivo: 'Desktop'
});
```

##### `gps:ready` (Nuevo - Flujo Mejorado)
Usuario nuevo confirma que su GPS está listo y obtiene ubicación por primera vez.
```javascript
socket.emit('gps:ready', {
  userId: 'uuid-del-usuario',
  lat: 11.0049,
  lng: -74.8060
});
```

##### `gps:myLocation` (Nuevo - Flujo Mejorado)
Usuario existente envía su ubicación a un usuario nuevo que la solicitó.
```javascript
socket.emit('gps:myLocation', {
  toUser: 'uuid-usuario-nuevo',
  fromUser: 'uuid-usuario-existente',
  lat: 11.0049,
  lng: -74.8060
});
```

##### `gps:share` (Nuevo - Flujo Mejorado)
Comparte ubicación GPS en tiempo real con todos los usuarios conectados.
```javascript
socket.emit('gps:share', {
  userId: 'uuid-del-usuario',
  lat: 11.0049,
  lng: -74.8060
});
```

##### `coordenada:actualizar` (Legacy - Compatibilidad)
Envía una actualización de ubicación GPS (evento anterior, mantenido por compatibilidad).
```javascript
socket.emit('coordenada:actualizar', {
  lat: 11.0049,
  lng: -74.8060,
  accuracy: 10.5,
  userId: 'uuid-del-usuario'
});
```

##### `ubicacion-actual` (Evento Principal)
Envía ubicación en tiempo real con velocidad y precisión GPS. Este es el evento principal usado para compartir ubicaciones.
```javascript
socket.emit('ubicacion-actual', {
  userId: 'uuid-del-usuario',
  lat: 11.0049,
  lng: -74.8060,
  speed: 45.5,
  accuracy: 15.2,  // Precisión GPS en metros
  timestamp: Date.now()
});
```

#### Eventos Servidor → Cliente

##### `marcadores:iniciales`
Se envía automáticamente cuando un cliente se conecta.
```javascript
socket.on('marcadores:iniciales', (marcadores) => {
  // Recibe array de todos los marcadores existentes
});
```

##### `marcador:creado`
Notifica cuando se crea un nuevo marcador.
```javascript
socket.on('marcador:creado', (marcador) => {
  // Recibe el marcador creado (excepto al creador)
});
```

##### `marcador:actualizado`
Notifica cuando se actualiza un marcador.
```javascript
socket.on('marcador:actualizado', (marcador) => {
  // Recibe el marcador actualizado (excepto al que actualizó)
});
```

##### `marcador:eliminado`
Notifica cuando se elimina un marcador.
```javascript
socket.on('marcador:eliminado', ({ id }) => {
  // Recibe el ID del marcador eliminado (excepto al que eliminó)
});
```

##### `coordenada:nueva`
Notifica nueva coordenada GPS de otro usuario.
```javascript
socket.on('coordenada:nueva', (coordenada) => {
  // { id, lat, lng, accuracy, user_id, timestamp }
});
```

##### `ubicacion-usuario`
Notifica ubicación en tiempo real de otro usuario (actualizaciones continuas).
```javascript
socket.on('ubicacion-usuario', (ubicacion) => {
  // { userId, lat, lng, speed, timestamp }
  // Recibe actualizaciones de ubicación de otros usuarios en tiempo real
});
```

##### `ubicaciones-usuarios-conectados` (Nuevo)
Se envía automáticamente a un nuevo usuario cuando se conecta, contiene todas las ubicaciones de usuarios ya conectados.
```javascript
socket.on('ubicaciones-usuarios-conectados', (ubicaciones) => {
  // Array de ubicaciones: [{ userId, lat, lng, speed, timestamp }, ...]
  // Recibido por el nuevo usuario al conectarse
  // Permite ver inmediatamente a todos los usuarios ya conectados
});
```

##### `cliente-conectado`
Notifica cuando otro cliente se conecta.
```javascript
socket.on('cliente-conectado', (data) => {
  // { userId, numUsuario, nombre, plataforma, timestamp }
});
```

##### `cliente-desconectado`
Notifica cuando otro cliente se desconecta.
```javascript
socket.on('cliente-desconectado', (data) => {
  // { userId, numUsuario, nombre, plataforma, timestamp }
});
```

##### `usuario-registrado`
Confirma el registro exitoso del usuario.
```javascript
socket.on('usuario-registrado', (data) => {
  // { success: true, userId, usuario: { num_usuario, ... } }
});
```

##### Eventos Legacy (Mantenidos por compatibilidad)
Los siguientes eventos están documentados pero ya no se usan en el flujo principal:
- `gps:ready`, `gps:ask`, `gps:myLocation`, `gps:update`, `gps:share`, `user:new`
- El sistema actual usa `ubicacion-actual` y `ubicaciones-usuarios-conectados` para un flujo más eficiente

### Gestión de Archivos

Los archivos subidos se almacenan en:
```
server/storage/
├── imagenes/     # Imágenes adjuntas
├── videos/       # Videos adjuntos
└── documentos/   # PDFs y otros documentos
```

Los archivos se sirven mediante:
```
GET /api/files/{carpeta}/{nombre-archivo}
```

### Flujo Mejorado de Sincronización GPS

El sistema implementa un flujo optimizado para sincronizar ubicaciones GPS cuando un nuevo usuario se conecta:

#### 🔵 1. Usuario Nuevo se Conecta

**Cliente envía:**
```javascript
socket.emit('usuario-conectado', { id, nombre, plataforma });
```

**Servidor procesa:**
- Guarda el usuario en `userSocketMap` (Map: userId → socketId)
- Registra usuario en base de datos
- **Envía inmediatamente todas las ubicaciones de usuarios ya conectados:**
```javascript
socket.emit('ubicaciones-usuarios-conectados', [
  { userId: 'user1', lat: 11.0049, lng: -74.8060, speed: 0, timestamp: ... },
  { userId: 'user2', lat: 11.0050, lng: -74.8061, speed: 0, timestamp: ... }
]);
```
- Notifica a otros usuarios que alguien nuevo se conectó:
```javascript
socket.broadcast.emit('cliente-conectado', { userId, numUsuario, plataforma });
```

**Resultado:** El nuevo usuario ve inmediatamente las ubicaciones de todos los usuarios ya conectados, incluso si están quietos.

#### 🟢 2. Usuario Nuevo Obtiene GPS y Envía Ubicación

**Cliente envía (cuando obtiene posición GPS por primera vez):**
```javascript
socket.emit('ubicacion-actual', {
  userId: 'uuid-usuario-nuevo',
  lat: 11.0049,
  lng: -74.8060,
  speed: 0,
  timestamp: Date.now()
});
```

**Servidor procesa:**
- Guarda ubicación en mapa `ultimasUbicaciones` (userId → {lat, lng, speed, timestamp})
- Guarda en base de datos (opcional, para historial)
- Reenvía a todos los demás usuarios:
```javascript
socket.broadcast.emit('ubicacion-usuario', {
  userId: 'uuid-usuario-nuevo',
  lat: 11.0049,
  lng: -74.8060,
  speed: 0,
  timestamp: Date.now()
});
```

**Resultado:** Todos los usuarios existentes ven la ubicación del nuevo usuario.

#### 🟣 3. Compartir Ubicaciones en Tiempo Real

**Cada cliente emite cuando su ubicación cambia:**
```javascript
socket.emit('ubicacion-actual', { userId, lat, lng, speed, timestamp });
```

**Servidor procesa:**
- Actualiza mapa `ultimasUbicaciones` con la nueva ubicación
- Guarda en base de datos (opcional, para historial)
- Hace broadcast a todos los demás usuarios:
```javascript
socket.broadcast.emit('ubicacion-usuario', { userId, lat, lng, speed, timestamp });
```

**Resultado:** Todos los usuarios reciben actualizaciones de ubicación de los demás en tiempo real.

#### 🔴 4. Usuario se Desconecta

**Servidor procesa automáticamente:**
- Remueve del mapa `userSocketMap`
- Remueve ubicación del mapa `ultimasUbicaciones`
- Notifica a otros usuarios:
```javascript
socket.broadcast.emit('cliente-desconectado', { userId, numUsuario, plataforma });
```

**Resultado:** La ubicación del usuario desconectado se elimina del mapa y otros usuarios son notificados.

### Ventajas del Flujo Mejorado

1. **Sincronización Inicial Instantánea**: El nuevo usuario obtiene todas las ubicaciones inmediatamente al conectarse
2. **Funciona con Usuarios Quietos**: Los usuarios que están quietos (velocidad 0) también se muestran
3. **Menos Tráfico de Red**: No requiere solicitudes/respuestas, el servidor mantiene estado
4. **Escalable**: El servidor mantiene un mapa eficiente de últimas ubicaciones
5. **Limpieza Automática**: Las ubicaciones se eliminan automáticamente cuando un usuario se desconecta

### Características del Servidor

1. **Gestión de Usuarios**:
   - Asignación automática de números de usuario secuenciales
   - Seguimiento de última conexión
   - Manejo de reconexiones
   - Gestión de `usersOnline` (Map de usuarios conectados)

2. **Sincronización en Tiempo Real**:
   - Los cambios en marcadores se propagan automáticamente
   - Las ubicaciones GPS se comparten en tiempo real con flujo optimizado
   - Notificaciones de conexión/desconexión
   - Sincronización inicial instantánea: nuevos usuarios ven inmediatamente ubicaciones de usuarios ya conectados
   - Mapa de últimas ubicaciones (`ultimasUbicaciones`) para envío inmediato a nuevos usuarios

3. **Seguridad**:
   - Validación de datos en todos los endpoints
   - CORS configurado para orígenes específicos
   - Límites de tamaño de archivo
   - Prepared statements para prevenir SQL injection

4. **Rendimiento**:
   - Índices en base de datos para consultas rápidas
   - Almacenamiento eficiente de archivos
   - Optimización de queries SQL

### Iniciar el Servidor

#### Desarrollo
```bash
cd server
npm install
npm run dev  # Modo watch con auto-reload
```

#### Producción
```bash
cd server
npm install
npm start
```

El servidor estará disponible en `http://localhost:3000`

---

## 🎨 FRONTEND (Angular)

### Descripción General

El frontend es una Single Page Application (SPA) desarrollada con:
- **Angular 20**: Framework frontend moderno
- **Leaflet**: Biblioteca de mapas interactivos
- **Socket.IO Client**: Cliente para comunicación en tiempo real
- **Tailwind CSS + DaisyUI**: Framework de estilos
- **TypeScript**: Lenguaje de programación tipado

### Ubicación
```
fleet-tracking/
├── src/
│   ├── app/
│   │   ├── map-view/              # Componente principal del mapa
│   │   ├── speed-dial/            # Menú radial de acciones
│   │   ├── notifications-panel/   # Panel de notificaciones
│   │   └── services/              # Servicios Angular
│   │       ├── api.service.ts     # Cliente REST API
│   │       ├── socket.service.ts  # Cliente Socket.IO
│   │       ├── geo.service.ts     # Geolocalización
│   │       ├── user.service.ts    # Gestión de usuarios
│   │       └── notification.service.ts  # Notificaciones
│   ├── environments/              # Variables de entorno
│   └── assets/                    # Recursos estáticos
└── angular.json                   # Configuración Angular
```

### Servicios Principales

#### ApiService (`api.service.ts`)
Gestiona todas las llamadas HTTP REST al servidor:
- `getMarcadores()`: Obtiene todos los marcadores
- `createMarcador()`: Crea un nuevo marcador (con soporte de archivos)
- `updateMarcador()`: Actualiza un marcador
- `deleteMarcador()`: Elimina un marcador
- `saveCoordenadaGPS()`: Guarda coordenada GPS
- `getCoordenadas()`: Obtiene coordenadas históricas

#### SocketService (`socket.service.ts`)
Gestiona la conexión Socket.IO y eventos en tiempo real:
- Conexión automática al servidor
- Registro de usuario al conectar
- Escucha de eventos: marcadores, coordenadas, conexiones
- Manejo de reconexiones automáticas
- Emisión de coordenadas GPS

#### GeoService (`geo.service.ts`)
Gestiona la geolocalización del navegador:
- `watchPosition()`: Seguimiento continuo de ubicación
- `calcularDistancia()`: Distancia entre dos puntos
- `calcularRumbo()`: Dirección entre dos puntos
- `obtenerDireccionCompass()`: Dirección cardinal (N, S, E, W)

#### UserService (`user.service.ts`)
Gestiona la identidad del usuario:
- Generación de UUID único por usuario
- Almacenamiento en localStorage
- Obtención de información del usuario

#### NotificationService (`notification.service.ts`)
Gestiona las notificaciones del sistema:
- Notificaciones de conexiones/desconexiones
- Notificaciones de marcadores nuevos
- Notificaciones de ubicaciones de otros usuarios

### Componentes Principales

#### MapViewComponent (`map-view.component.ts`)
Componente principal que contiene:
- **Mapa Leaflet**: Visualización de mapas interactivos
- **Marcadores**:
  - Marcador GPS del usuario (azul, con animación)
  - Marcador de búsqueda (rojo, arrastrable)
  - Marcadores guardados (según categoría)
  - Marcadores de otros usuarios en tiempo real con movimiento premium
- **Búsqueda de direcciones**: Integración con Nominatim
- **Modales**:
  - Modal de agregar marcador
  - Modal de gestión de marcadores
  - Modal de visualización de archivos/videos
  - Modal de imagen en pantalla grande (popups y lista)
- **Selector de capas**: 5 estilos de mapa diferentes
- **Sistema de movimiento avanzado**: Animación suave, rotación y dead-reckoning para marcadores de usuarios

#### SpeedDialComponent (`speed-dial.component.ts`)
Menú radial flotante con acciones rápidas:
- 📍 Mi ubicación
- ⚙️ Abrir filtros
- 🎨 Cambiar estilo de mapa
- 🎯 Centrar mapa
- 🚚 Lista de vehículos
- 📌 Agregar marcador
- 🗺️ Cargar marcadores

#### NotificationsPanelComponent (`notifications-panel.component.ts`)
Panel lateral para mostrar notificaciones:
- Lista de notificaciones en tiempo real
- Contador de notificaciones no leídas
- Diferentes tipos de notificaciones

### Flujo de Funcionamiento

#### 1. Inicialización de la Aplicación

```
1. Usuario carga la aplicación
2. Angular inicializa componentes
3. SocketService se conecta al servidor
4. UserService genera/obtiene userId
5. SocketService registra usuario en servidor
6. Servidor envía marcadores iniciales
7. MapViewComponent carga marcadores en el mapa
8. GeoService solicita permiso de geolocalización
9. Si se concede, se muestra marcador GPS en tiempo real
```

#### 2. Seguimiento de Ubicación GPS (Flujo Mejorado)

**Cuando un usuario nuevo se conecta:**

```
1. Usuario se conecta → SocketService emite 'usuario-conectado'
2. Servidor guarda en userSocketMap y registra en BD
3. Servidor envía inmediatamente 'ubicaciones-usuarios-conectados' con todas las ubicaciones existentes
4. Cliente nuevo recibe y muestra marcadores de todos los usuarios ya conectados
5. GeoService obtiene ubicación GPS por primera vez
6. SocketService emite 'ubicacion-actual' con primera ubicación
7. Servidor guarda en ultimasUbicaciones y reenvía a otros usuarios
8. Otros usuarios ven la ubicación del nuevo usuario
```

**Actualizaciones continuas en tiempo real:**

```
1. GeoService obtiene ubicación cada 500ms-2s
2. MapViewComponent actualiza marcador GPS suavemente (local)
3. SocketService emite 'ubicacion-actual' cuando la ubicación cambia (incluye accuracy)
4. Servidor actualiza ultimasUbicaciones y hace broadcast 'ubicacion-usuario' a otros clientes
5. Otros clientes reciben y actualizan marcadores de usuarios con:
   - Animación suave premium (ease-out)
   - Rotación según dirección
   - Movimiento predictivo (10% adicional)
   - Dead-reckoning si hay pérdida de señal
```

**Flujo Legacy (mantenido por compatibilidad):**

```
1. GeoService obtiene ubicación cada 500ms
2. MapViewComponent actualiza marcador GPS suavemente
3. SocketService emite 'coordenada:actualizar' al servidor
4. Servidor guarda en base de datos
5. Servidor emite 'coordenada:nueva' a otros clientes
6. Otros clientes actualizan marcadores de usuarios
```

#### 3. Creación de Marcador

```
1. Usuario hace clic en "Agregar marcador"
2. Se abre modal con formulario
3. Usuario completa: categoría, descripción, archivo (opcional)
4. ApiService crea marcador vía POST /api/marcadores
5. Servidor guarda en base de datos y archivo (si existe)
6. Servidor emite 'marcador:creado' a otros clientes
7. Todos los clientes actualizan el mapa
```

#### 4. Búsqueda de Direcciones

```
1. Usuario escribe en barra de búsqueda
2. Después de 1 segundo de inactividad (debounce)
3. Se llama a Nominatim API para geocodificación
4. Se muestran resultados en lista desplegable
5. Usuario selecciona resultado
6. Mapa se centra en ubicación
7. Marcador rojo se mueve a la ubicación
```

### Características del Frontend

1. **Geolocalización en Tiempo Real**:
   - Seguimiento continuo con alta precisión
   - Marcador con animación pulse
   - Actualización suave sin saltos

2. **Mapas Interactivos**:
   - 5 estilos de mapa diferentes
   - Zoom y pan suaves
   - Marcadores personalizados por categoría

3. **Comunicación en Tiempo Real**:
   - Actualización instantánea de marcadores
   - Visualización de otros usuarios en tiempo real
   - Notificaciones de eventos importantes
   - Sincronización inmediata: nuevos usuarios ven ubicaciones de usuarios ya conectados al instante

4. **Diseño Responsivo**:
   - Adaptado a móvil y desktop
   - Componentes DaisyUI modernos
   - Interfaz intuitiva y accesible

5. **Wake Lock (Prevención de Hibernación)**:
   - Activa automáticamente Wake Lock API al iniciar la aplicación
   - Evita que la pantalla se apague en dispositivos móviles (Android e iOS 16.4+)
   - Se reactiva automáticamente cuando la página vuelve a estar visible
   - Se desactiva automáticamente al cerrar la aplicación
   - Manejo de permisos: si se requiere interacción del usuario, se activa después de la primera interacción

6. **Sistema de Movimiento Avanzado (Tipo Uber)**:
   - Movimiento predictivo: extiende la posición un 10% adicional en la dirección del movimiento
   - Animación suave con easing ease-out usando curvas personalizadas
   - Rotación real del vehículo según la dirección de movimiento
   - Dead-reckoning: continúa el movimiento suavemente cuando hay pérdida de señal GPS
   - Vector de velocidad: calcula y mantiene la velocidad de cada marcador
   - Bucle de actualización cada 100ms para movimiento continuo
   - Sin saltos ni temblores: transiciones completamente suaves

7. **Visualización de Imágenes en Grande**:
   - Modal premium para ver imágenes en pantalla completa
   - Disponible tanto en popups de marcadores como en la lista de marcadores guardados
   - Diseño tipo Instagram/Viewer con fondo oscuro
   - Cierre con botón o clic en el backdrop
   - Miniaturas clickeables con efectos hover

8. **Función "Mi Ubicación"**:
   - Mueve el marcador draggable al centro actual del mapa sin mover la vista
   - Actualiza automáticamente la barra de búsqueda con la dirección del centro
   - Mantiene la capacidad de arrastre del marcador

### Variables de Entorno

El frontend se configura mediante `src/environments/environment.ts`:

```typescript
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000/api',
  socketUrl: 'http://localhost:3000'
};
```

### Iniciar el Frontend

#### Desarrollo
```bash
cd fleet-tracking
npm install
npm start
```

La aplicación estará disponible en `http://localhost:4200`

#### Producción
```bash
cd fleet-tracking
npm install
npm run build
# Los archivos compilados estarán en dist/
```

---

## 🔄 Flujo de Comunicación Completo

### Escenario: Usuario crea marcador y otro usuario lo ve

```
┌─────────────┐                                    ┌─────────────┐
│  Cliente A  │                                    │  Cliente B  │
└──────┬──────┘                                    └──────┬──────┘
       │                                                   │
       │ 1. POST /api/marcadores                          │
       │    { lat, lng, categoria, descripcion, archivo } │
       ├──────────────────────────────────────────────────>│
       │                                                   │
       │                                          ┌────────┴────────┐
       │                                          │   Servidor      │
       │                                          │   - Guarda BD   │
       │                                          │   - Guarda arch │
       │                                          └────────┬────────┘
       │                                                   │
       │ 2. { success: true, data: marcador }             │
       │<──────────────────────────────────────────────────┤
       │                                                   │
       │                                          ┌────────┴────────┐
       │                                          │   Servidor      │
       │                                          │   emite evento  │
       │                                          └────────┬────────┘
       │                                                   │
       │                                           3. socket.emit    │
       │                                              'marcador:creado' │
       │                                                   │
       │                                                   ▼
       │                                            ┌─────────────┐
       │                                            │  Cliente B  │
       │                                            │  recibe y   │
       │                                            │  actualiza  │
       │                                            │  mapa       │
       │                                            └─────────────┘
       │
       │ 4. Marcador aparece en mapa de Cliente A
```

### Escenario: Usuario Nuevo se Conecta y Sincroniza GPS (Flujo Mejorado)

```
┌─────────────┐                                    ┌─────────────┐
│ Cliente A   │                                    │ Cliente B   │
│ (NUEVO)     │                                    │ (EXISTENTE) │
└──────┬──────┘                                    └──────┬──────┘
       │                                                   │
       │ 1. socket.emit('usuario-conectado', {...})      │
       ├──────────────────────────────────────────────────>│
       │                                                   │
       │                                          ┌────────┴────────┐
       │                                          │   Servidor      │
       │                                          │ - Guarda en     │
       │                                          │   userSocketMap │
       │                                          │ - Consulta      │
       │                                          │   ultimasUbicaciones│
       │                                          └────────┬────────┘
       │                                                   │
       │ 2. socket.emit('ubicaciones-usuarios-conectados')│
       │    [{ userId: 'B', lat, lng, speed, ... }]      │
       │<──────────────────────────────────────────────────┤
       │                                                   │
       │ 3. Cliente A muestra inmediatamente              │
       │    marcador de Cliente B                         │
       │                                                   │
       │ 4. socket.broadcast.emit('cliente-conectado')    │
       │                                                   ▼
       │                                            ┌─────────────┐
       │                                            │  Cliente B  │
       │                                            │  sabe que   │
       │                                            │  A conectó  │
       │                                            └─────────────┘
       │
       │ 5. Obtiene GPS por primera vez
       │
       │ 6. socket.emit('ubicacion-actual', {...})
       ├──────────────────────────────────────────────────>│
       │                                                   │
       │                                          ┌────────┴────────┐
       │                                          │   Servidor      │
       │                                          │ - Guarda en     │
       │                                          │   ultimasUbicaciones│
       │                                          │ - Broadcast     │
       │                                          └────────┬────────┘
       │                                                   │
       │                                           7. socket.broadcast │
       │                                              .emit('ubicacion-usuario')│
       │                                                   │
       │                                                   ▼
       │                                            ┌─────────────┐
       │                                            │  Cliente B  │
       │                                            │  ve ubicación│
       │                                            │  de Cliente A│
       │                                            └─────────────┘
       │
       │ ✅ Sincronización completa e instantánea
```

### Escenario: Usuario Comparte Ubicación GPS en Tiempo Real

```
┌─────────────┐                                    ┌─────────────┐
│  Cliente A  │                                    │  Cliente B  │
└──────┬──────┘                                    └──────┬──────┘
       │                                                   │
       │ 1. Geolocation API obtiene nueva ubicación       │
       │    (cada 500ms-2s)                                │
       │                                                   │
       │ 2. socket.emit('ubicacion-actual', {...})        │
       │    cuando la ubicación cambia                     │
       ├──────────────────────────────────────────────────>│
       │                                                   │
       │                                          ┌────────┴────────┐
       │                                          │   Servidor      │
       │                                          │ - Actualiza     │
       │                                          │   ultimasUbicaciones│
       │                                          │ - Opcional:     │
       │                                          │   Guarda en BD  │
       │                                          │ - Broadcast     │
       │                                          └────────┬────────┘
       │                                                   │
       │                                           3. socket.broadcast │
       │                                              .emit('ubicacion-usuario')│
       │                                                   │
       │                                                   ▼
       │                                            ┌─────────────┐
       │                                            │  Cliente B  │
       │                                            │  mueve      │
       │                                            │  marcador   │
       │                                            │  de Usuario A│
       │                                            └─────────────┘
       │
       │ 4. Marcador GPS de Cliente A se actualiza localmente
```

### Escenario: Usuario mueve su ubicación GPS (Flujo Legacy - Compatibilidad)

```
┌─────────────┐                                    ┌─────────────┐
│  Cliente A  │                                    │  Cliente B  │
└──────┬──────┘                                    └──────┬──────┘
       │                                                   │
       │ 1. Geolocation API obtiene nueva ubicación       │
       │    (cada 500ms)                                   │
       │                                                   │
       │ 2. socket.emit('coordenada:actualizar', {...})   │
       ├──────────────────────────────────────────────────>│
       │                                                   │
       │                                          ┌────────┴────────┐
       │                                          │   Servidor      │
       │                                          │   - Guarda BD   │
       │                                          │   - Emite a     │
       │                                          │     otros       │
       │                                          └────────┬────────┘
       │                                                   │
       │                                           3. socket.emit    │
       │                                              'coordenada:nueva' │
       │                                                   │
       │                                                   ▼
       │                                            ┌─────────────┐
       │                                            │  Cliente B  │
       │                                            │  mueve      │
       │                                            │  marcador   │
       │                                            │  de Usuario A│
       │                                            └─────────────┘
       │
       │ 4. Marcador GPS de Cliente A se actualiza localmente
```

---

## 📊 Estructura de Datos

### Tipo: Marcador
```typescript
interface Marcador {
  id?: string;                    // UUID único
  user_id?: string;               // ID del usuario creador
  lat: number;                    // Latitud
  lng: number;                    // Longitud
  categoria: 'alerta' | 'peligro' | 'informacion';
  descripcion: string;            // Mínimo 10 caracteres
  archivo?: string | null;        // URL del archivo adjunto
  timestamp?: string;             // ISO string
  created_at?: string;            // ISO string
  updated_at?: string;            // ISO string
  usuario_nombre?: string;        // Nombre del usuario (join)
  usuario_plataforma?: string;    // Plataforma (join)
  usuario_num?: number;           // Número de usuario (join)
}
```

### Tipo: CoordenadaGPS
```typescript
interface CoordenadaGPS {
  id?: number;                    // ID autoincremental
  user_id?: string;               // ID del usuario
  lat: number;                    // Latitud
  lng: number;                    // Longitud
  accuracy?: number;              // Precisión GPS en metros (se llena automáticamente)
  timestamp?: string;             // ISO string
  created_at?: string;            // ISO string
}
```

**Nota**: El campo `accuracy` se obtiene de la API de Geolocalización del navegador (`position.coords.accuracy`) y se guarda automáticamente en la base de datos cuando se recibe una ubicación.

### Tipo: Usuario
```typescript
interface Usuario {
  id: string;                     // UUID único
  num_usuario: number;            // Número secuencial
  nombre?: string;                // Nombre del usuario
  plataforma: string;             // 'web', 'mobile', etc.
  modelo_dispositivo?: string;    // Modelo del dispositivo
  fecha_registro: string;         // ISO string
  ultima_conexion: string;        // ISO string
}
```

### Tipo: Respuesta API
```typescript
interface ApiResponse<T> {
  success: boolean;
  data?: T;
  error?: string;
  message?: string;
}
```

---

## 🔐 Seguridad y Validaciones

### Servidor

1. **Validación de Datos**:
   - Latitud/longitud válidas
   - Categoría en lista permitida
   - Descripción mínimo 10 caracteres
   - Archivos con límite de tamaño

2. **CORS**:
   - Solo orígenes permitidos
   - Métodos HTTP específicos
   - Headers controlados

3. **SQL Injection**:
   - Prepared statements en todas las queries
   - Validación de parámetros

4. **Archivos**:
   - Validación de tipo MIME
   - Límite de tamaño (configurable)
   - Almacenamiento seguro en carpetas específicas

### Frontend

1. **Validación de Formularios**:
   - Validación en tiempo real
   - Mensajes de error claros
   - Prevención de envío inválido

2. **Sanitización**:
   - Angular sanitiza automáticamente
   - No ejecuta código malicioso

3. **Almacenamiento**:
   - localStorage para datos locales
   - UUIDs para identificación única

---

## 🚀 Instalación y Ejecución

### Requisitos
- Node.js 18+ y npm
- Angular CLI 20.3.3+ (para frontend)

### Instalación Completa

#### 1. Servidor
```bash
cd server
npm install
cp .env.example .env  # Configurar variables de entorno
npm run dev           # Desarrollo con auto-reload
# o
npm start            # Producción
```

#### 2. Frontend
```bash
cd fleet-tracking
npm install
# Editar src/environments/environment.ts con URLs del servidor
npm start
```

### Ejecutar Ambos en Desarrollo

El proyecto incluye scripts de ayuda:
- `start-dev.sh` (Linux/Mac)
- `start-dev.bat` (Windows)

O manualmente:
1. Terminal 1: `cd server && npm run dev`
2. Terminal 2: `cd fleet-tracking && npm start`

---

## 📝 Notas Técnicas

### Optimizaciones

1. **Base de Datos**:
   - Índices en campos frecuentemente consultados
   - Queries optimizadas con JOINs
   - Foreign keys para integridad

2. **Socket.IO**:
   - Reconexión automática
   - Emisión selectiva (solo a usuarios relevantes)
   - Manejo eficiente de múltiples conexiones

3. **Frontend**:
   - Debounce en búsquedas
   - Animaciones suaves con requestAnimationFrame
   - Lazy loading cuando sea posible

### Manejo de Errores

1. **Servidor**:
   - Try-catch en todas las operaciones
   - Logging de errores
   - Respuestas HTTP apropiadas

2. **Frontend**:
   - Manejo de errores en observables
   - Mensajes de error al usuario
   - Fallbacks cuando sea necesario

### Escalabilidad

- SQLite es adecuado para aplicaciones pequeñas/medianas
- Para mayor escala, considerar migrar a PostgreSQL/MySQL
- Socket.IO soporta múltiples instancias con Redis adapter
- Frontend puede optimizarse con lazy loading de rutas

---

## 🔮 Funcionalidades Futuras

- Autenticación y autorización de usuarios
- Roles y permisos
- Historial de rutas
- Análisis y reportes
- Exportación de datos
- Notificaciones push
- Modo offline
- Optimización de rutas
- Zonas geográficas (geofencing)

---

## 📞 Soporte y Mantenimiento

Para problemas o preguntas:
1. Revisar logs del servidor (consola)
2. Revisar consola del navegador (F12)
3. Verificar configuración de variables de entorno
4. Verificar conexión a base de datos
5. Verificar permisos de geolocalización

---

---

## 📱 Funcionalidades Móviles

### Wake Lock (Prevención de Hibernación)

La aplicación implementa la API de Wake Lock para evitar que la pantalla se apague en dispositivos móviles mientras el usuario está usando la aplicación.

#### Características:
- **Activación Automática**: Se activa automáticamente cuando el componente del mapa se inicializa
- **Compatibilidad**:
  - ✅ Android Chrome (soporte completo)
  - ✅ iOS Safari 16.4+ (soporte completo)
  - ⚠️ Navegadores sin soporte: se registra un aviso en consola sin afectar funcionalidad
- **Reactivación Automática**: Si el Wake Lock se libera (por ejemplo, al cambiar de pestaña), se reactiva automáticamente cuando la página vuelve a estar visible
- **Manejo de Permisos**: Si el navegador requiere interacción del usuario, se intenta activar después de la primera interacción
- **Limpieza Automática**: Se desactiva automáticamente cuando el componente se destruye o la aplicación se cierra

#### Implementación Técnica:
```typescript
// En map-view.component.ts
private async activarWakeLock(): Promise<void> {
  if (!navigator.wakeLock) return;
  
  this.wakeLock = await navigator.wakeLock.request('screen');
  // La pantalla permanecerá encendida
}
```

---

## 🚗 Sistema de Movimiento Avanzado (Tipo Uber)

### Descripción General

El sistema implementa un motor de movimiento premium para los marcadores de usuarios en tiempo real, proporcionando una experiencia fluida y profesional similar a aplicaciones como Uber o Google Maps.

### Características Principales

#### 1. Movimiento Predictivo
- **Función**: `getPredictedPosition(oldPos, newPos)`
- Extiende la nueva posición un 10% adicional en la dirección del movimiento
- Evita frenazos visuales y proporciona movimiento más natural
- Calcula la dirección del vector de movimiento y proyecta hacia adelante

#### 2. Animación Suave con Easing
- **Función**: `animateMarkerPremium(marker, start, end, duration = 350)`
- Usa curva ease-out personalizada: `t = (elapsed / duration)^0.7`
- Implementado con `requestAnimationFrame` para 60 FPS
- Duración configurable (por defecto 350ms)
- Interpolación suave entre posiciones

#### 3. Rotación Real del Vehículo
- **Función**: `getAngle(oldPos, newPos)`
- Calcula el ángulo de dirección entre dos posiciones en grados (0-360°)
- **Función**: `rotateMarker(marker, angle)`
- Aplica rotación usando `transform: rotate()` con transición CSS suave
- El ícono del vehículo apunta siempre en la dirección del movimiento

#### 4. Vector de Velocidad
- **Almacenamiento**: Para cada marcador se guarda:
  - `velocityVector`: { lat, lng } - Velocidad en grados por milisegundo
  - `lastUpdateTime`: Timestamp de la última actualización
  - `estimatedPosition`: Posición estimada actual
  - `lastRealPosition`: Última posición real recibida
- Se actualiza con cada nueva coordenada recibida

#### 5. Dead-Reckoning (Navegación por Estimación)
- **Función**: `deadReckoning(marker, userId, deltaTime = 100)`
- **Activación**: Se activa cuando pasan más de 300ms sin recibir datos GPS
- **Damping**: Calcula un factor de amortiguación basado en el tiempo transcurrido
  ```javascript
  damping = Math.max(0.85 - timeSinceLast / 5000, 0.50)
  ```
- Continúa moviendo el marcador suavemente basado en el vector de velocidad
- Aplica damping a la velocidad para simular fricción natural
- Usa `animateMarkerPremium()` para mantener la animación suave

#### 6. Bucle de Actualización
- **Función**: `startDeadReckoningLoop()`
- Ejecuta `deadReckoning()` para todos los marcadores cada 100ms
- Se inicia automáticamente en `ngAfterViewInit()`
- Se detiene y limpia en `ngOnDestroy()`

### Flujo de Actualización

```
1. Cliente recibe nueva ubicación vía Socket.IO ('ubicacion-usuario')
2. Se calcula vector de velocidad: (newPos - oldPos) / deltaTime
3. Se calcula posición predicha: newPos + (vector * 0.1)
4. Se calcula ángulo de rotación: atan2(deltaLng, deltaLat)
5. Se ejecuta animateMarkerPremium(marker, oldPos, predictedEnd)
6. Se aplica rotateMarker(marker, angle)
7. Se actualizan datos de movimiento (velocityVector, lastUpdateTime, etc.)
8. Bucle de dead-reckoning continúa moviendo si hay pérdida de señal
```

### Ventajas del Sistema

1. **Movimiento Tipo Uber**: Suave, fluido y profesional
2. **Sin Saltos**: Las animaciones eliminan completamente los saltos bruscos
3. **Rotación Real**: Los vehículos apuntan en la dirección correcta
4. **Resistente a Pérdida de Señal**: Dead-reckoning mantiene el movimiento natural
5. **Corrección Suave**: Cuando vuelve el GPS, la corrección es imperceptible
6. **Predicción Natural**: El movimiento se siente anticipado y fluido
7. **Sin Pérdida de Precisión**: Los datos reales siempre se respetan

### Implementación Técnica

```typescript
// Estructura de datos por marcador
markerData[userId] = {
  velocityVector: { lat: number, lng: number },
  lastUpdateTime: number,
  estimatedPosition: L.LatLng,
  lastRealPosition: L.LatLng,
  animationFrame?: number
}

// Bucle de actualización
setInterval(() => {
  for (const userId in markers) {
    deadReckoning(markers[userId], userId, 100);
  }
}, 100);
```

---

## 🖼️ Sistema de Visualización de Imágenes

### Modal de Imagen en Pantalla Grande

La aplicación incluye un sistema premium para visualizar imágenes en pantalla completa, disponible en dos contextos:

#### 1. En Popups de Marcadores del Mapa
- Las miniaturas en los popups de Leaflet son clickeables
- Al hacer clic, se abre el modal con la imagen en grande
- Implementado mediante CustomEvent para comunicación entre Leaflet y Angular

#### 2. En Lista de Marcadores Guardados
- Las miniaturas en la lista de marcadores guardados son clickeables
- Mismo modal unificado para consistencia de experiencia

### Características del Modal

- **Diseño Tipo Instagram/Viewer**:
  - Fondo oscuro con blur (`bg-black/90 backdrop-blur-sm`)
  - Imagen centrada con `object-contain`
  - Altura máxima: `max-h-[80vh]`
  - Ancho máximo: `max-w-5xl`

- **Interacción**:
  - Botón de cerrar en esquina superior derecha
  - Cierre al hacer clic en el backdrop
  - Transiciones suaves

- **Miniaturas**:
  - Estilos Tailwind: `rounded-md`, `shadow-md`
  - Efecto hover: `hover:scale-105`
  - Cursor pointer para indicar interactividad

### Implementación

```typescript
// Variables
selectedImage: string | null = null;
showImageModal: boolean = false;

// Funciones
openImageModal(imageUrl: string): void
closeImageModal(): void

// Listener para popups de Leaflet
window.addEventListener('openImageFromPopup', (event) => {
  openImageModal(event.detail);
});
```

---

## 📍 Función "Mi Ubicación"

### Descripción

Función que mueve el marcador draggable al centro actual del mapa sin mover la vista del mapa.

### Comportamiento

1. **Obtiene el centro del mapa**: `map.getCenter()`
2. **Mueve el marcador draggable**: `searchMarker.setLatLng(center)`
3. **Actualiza la barra de búsqueda**: Geocodificación inversa del centro
4. **Mantiene capacidad de arrastre**: El marcador sigue siendo draggable

### Implementación

```typescript
moverMarcadorAlCentro(): void {
  const center = this.map.getCenter();
  if (this.searchMarker) {
    this.searchMarker.setLatLng([center.lat, center.lng]);
  } else {
    this.setupSearchMarker([center.lat, center.lng]);
  }
  this.actualizarBarraBusquedaDesdeCoordenadas(center.lat, center.lng);
}
```

---

## 🛠️ Mejoras en Manejo de Errores

### Geolocalización

- **Timeouts (código 3)**: Se manejan con `console.debug` en lugar de `console.error`
- **Mensajes informativos**: No intrusivos, el seguimiento continúa automáticamente
- **Throttling**: Solo se muestra el mismo error cada 30 segundos

### WebSocket de Vite HMR

- **Filtrado automático**: Script en `index.html` silencia errores de WebSocket de Vite
- **No afecta funcionalidad**: Estos errores son normales en desarrollo remoto
- **Solo desarrollo**: No aparecen en builds de producción

### Campo Accuracy en GPS

- **Incluido en eventos**: El campo `accuracy` ahora se envía en todos los eventos `ubicacion-actual`
- **Guardado en BD**: Se almacena correctamente en la tabla `coordenadas_gps`
- **Tipo actualizado**: `getCurrentCoords()` retorna `accuracy` en su tipo

---

**Última actualización**: Diciembre 2024
**Versión del Proyecto**: 2.2.0
