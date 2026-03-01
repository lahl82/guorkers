# 🚀 Casos de Uso del Proyecto Guorkers (Versión Consolidada 3.0)

Este documento centraliza el modelado funcional del sistema Guorkers.
Sirve como base conceptual única para el desarrollo en Rails (API) y frontend.

Los estados de las entidades se modelan utilizando **AASM**, con máquinas de estado independientes para cada modelo que lo requiera.

> **Nota de versión 3.0:** Se reestructuró completamente el sistema de roles.
> El rol `admin` del código (versiones anteriores) se renombra a `seller`.
> Se incorporan los roles `assistant`, `root` y se redefine `support`.
> Ver TODO.md ítem C-2 para el estado del cambio en el código.

---

# 🛠️ Sección A: Gestión de Usuarios, Cuenta y Seguridad

## Modelo General

- Cada **User** puede estar vinculado a **una única Company**.
- Una Company solo puede ser creada durante el proceso de conversión a proveedor
  o en el registro inicial como proveedor.
- Las Companies no pueden eliminarse ni gestionarse directamente por los usuarios.
- El email es **único globalmente**.
- No existe eliminación física (hard delete). Solo inactivación mediante cambio de estado.
- Todas las fechas se almacenan en **UTC**.

---

## Sistema de Roles

Los roles se dividen en tres grupos independientes. **No son acumulativos entre grupos.**
Cualquier usuario autenticado puede reservar citas, independientemente de su rol.

### Roles de negocio (tienen `company_id` asignado)

| Rol | Descripción |
|-----|-------------|
| `seller` | Dueño del negocio. Crea servicios, gestiona turnos, gestiona colaboradores. |
| `assistant` | Colaborador del negocio. Gestiona citas del día, confirma llegadas, registra información post-atención. Afiliado a una única company. |

Reglas:
- Un `assistant` pertenece a una sola company.
- No se puede asociar como `assistant` a un usuario que ya sea `seller`.
- Un `seller` puede desafiliar a un `assistant`, quien regresa a ser solo `customer`.

### Roles de plataforma (sin `company_id` fijo)

| Rol | Descripción |
|-----|-------------|
| `root` | Superadmin de la plataforma (solo el creador). Acceso total a todas las companies. |
| `support` | Colaborador de soporte interno. Acceso restringido a las companies que le sean asignadas explícitamente. |

Reglas:
- `root` y `support` no tienen `company_id` asignado en su perfil.
- Al iniciar sesión, si el usuario es `root` o `support`, se le solicita seleccionar
  la company en la que va a operar durante esa sesión.
- `support` solo puede seleccionar companies de su lista asignada.
- `root` puede seleccionar cualquier company.

### Rol de consumidor

| Rol | Descripción |
|-----|-------------|
| `customer` | Rol base asignado a todo usuario nuevo. Puede reservar citas en cualquier servicio activo. |

---

## ✅ Caso A1: Registro de proveedor nuevo (Seller)

Un visitante decide ofrecer servicios en la plataforma desde el inicio.

**Acción:**
- Completa formulario con datos personales y datos de negocio.

**Resultado:**
- Se crea una Company.
- Se crea un User con rol `seller`.
- Se asigna permanentemente `company_id`.

---

## ✅ Caso A2: Recuperación de credenciales

Un usuario olvidó su contraseña.

**Proceso:**
- Se genera token válido por 1 hora.
- Tras cambio exitoso:
  - El token se invalida.
  - El usuario debe reautenticarse.

---

## ✅ Caso A3: Cliente se convierte en proveedor

Un usuario con rol `customer` decide ofrecer servicios.

**Acción:**
- Activa opción “Quiero ofrecer mis servicios”.
- Completa datos del negocio.

**Resultado:**
- Se crea Company.
- Se agrega rol `seller` al usuario.
- Mantiene su historial como customer (sus citas reservadas permanecen intactas).

---

## ✅ Caso A4: Inactivación de Usuario (Proceso en Cascada)

Un usuario decide cerrar su cuenta.

**Reglas:**

1. El User pasa a estado `inactive`.
2. Todas sus citas futuras en estado `active` pasan a `user_canceled`.

Si el usuario es `seller` (posee Company):

3. Todos sus Services pasan a `inactive`.
4. Todos sus Appointment Slots pasan a `inactive`.
5. Todas las citas activas asociadas a esos slots pasan a `seller_canceled`.
6. La Company pasa a `inactive`.
7. Todos sus `assistant` vinculados pierden la afiliación y quedan como `customer`.

Este proceso se ejecuta como operación masiva en background.
No existe reversión automática.

---

## ✅ Caso A5: Seller gestiona colaboradores (Assistants)

El seller puede asociar usuarios registrados como colaboradores de su company.

**Flujo de asociación:**
1. El seller busca al usuario por email dentro de la plataforma.
2. Verifica que el usuario exista y sea solo `customer` (no puede ser `seller`).
3. Asocia al usuario: se le asigna el rol `assistant` y el `company_id` del seller.

**Flujo de desafiliación:**
1. El seller selecciona un assistant de su lista de colaboradores.
2. Confirma la desafiliación.
3. Al usuario se le retira el rol `assistant` y el `company_id`.
4. El usuario queda con rol `customer` únicamente, con su historial intacto.

**Restricciones:**
- No se puede asociar a un usuario que ya sea `seller`, `root` o `support`.
- Un `assistant` solo puede pertenecer a una company a la vez.

---

## ✅ Caso A6: Support / Root selecciona company al login

Al iniciar sesión, si el sistema detecta rol `root` o `support`, presenta un segundo paso.

**Flujo:**
1. Usuario ingresa credenciales normalmente.
2. El sistema detecta rol `root` o `support`.
3. Se presenta una pantalla de selección de company:
   - `root`: ve todas las companies activas.
   - `support`: ve solo las companies que le han sido asignadas.
4. El usuario selecciona la company.
5. La sesión queda contextualizada a esa company (equivalente a tener `company_id` temporal en sesión).

**Resultado:**
- Los endpoints `mine` y de gestión funcionan en el contexto de la company seleccionada.
- No modifica el `company_id` del usuario en la base de datos.

---

## ✅ Caso A7: Root gestiona companies asignadas a Support

El `root` puede administrar qué companies puede ver cada usuario `support`.

**Flujo:**
1. Root accede a la gestión de usuarios de soporte.
2. Selecciona un usuario `support`.
3. Puede:
   - Asignar companies de la lista global.
   - Quitar companies previamente asignadas.

**Modelo de datos:**
- Tabla intermedia `support_companies` (o `user_companies`): `user_id` + `company_id`.
- Solo aplica a usuarios con rol `support`.
- `root` no necesita esta tabla (acceso irrestricto).

---

# 📅 Sección B: Gestión de Turnos (Appointment Slots)

Todos los horarios se almacenan en UTC y se presentan en zona horaria local del usuario.

Los estados de AppointmentSlot se gestionan con AASM.

---

## Estados posibles de Slot

- `active`
- `inactive`
- `suspended`

---

## ✅ Caso B1: Creación de disponibilidad

El proveedor define disponibilidad.

**Reglas:**

- No puede existir overlap de horarios dentro de la misma Company.
- Se define:
  - `starting` (UTC)
  - `duration`
  - `max_requests`

---

## ✅ Caso B2: Edición de Slot

Permitido únicamente si:

- No existen citas asociadas (pasadas o futuras).

Si existen citas:
- No se permite edición.
- Solo puede ejecutarse suspensión.

---

## ✅ Caso B3: Suspensión de Slot con Citas

Si un slot tiene citas agendadas:

**Acción:**
- El proveedor suspende el slot.

**Resultado:**

1. Slot pasa a `suspended`.
2. Todas las citas activas asociadas pasan a:
   - `seller_canceled`.
3. Se generan notificaciones a los usuarios afectados.

No se puede revertir la suspensión.

---

## ✅ Caso B4: Eliminación de Slot

Solo permitido si:

- No existen citas asociadas (ni pasadas ni futuras).

---

# 🛒 Sección C: Directorio y Agendamiento

---

## ✅ Caso C1: Búsqueda pública

Se listan únicamente:

- Services activos
- De Companies activas
- Con Appointment Slots activos

---

## ✅ Caso C2: Creación de Appointment

Un usuario reserva un turno.

**Validaciones obligatorias:**

- Slot activo.
- Service activo.
- Company activa.
- No se ha superado `max_requests`.

Debe existir control de concurrencia mediante transacción y bloqueo de fila para evitar sobre-reservas.

Si no hay cupo disponible → la operación falla.

---

## ✅ Caso C3: Cancelación de Cita

Una cita no puede eliminarse.

Puede cancelarse:

- Por el cliente → `user_canceled`.
- Por el proveedor → `seller_canceled`.

---

## ✅ Caso C4: Restricción de edición de Cita

No existe edición de citas.

Si se requiere cambiar horario:
- Se cancela la cita actual.
- Se crea una nueva.

---

# 🩺 Sección D: Flujo de Estados de Appointment

Estados posibles:

- `active`
- `arrived`
- `attended`
- `with_pending_info`
- `finished`
- `missed`
- `user_canceled`
- `seller_canceled`

---

## Transiciones Permitidas

| Estado actual       | Puede pasar a                                 |
|--------------------|-----------------------------------------------|
| active             | arrived, missed, user_canceled, seller_canceled |
| arrived            | attended, missed                              |
| attended           | with_pending_info, finished                   |
| with_pending_info  | finished                                      |

Estados terminales:

- finished
- missed
- user_canceled
- seller_canceled

Desde estados terminales no se permiten transiciones adicionales.

---

# 🏆 Sección E: Servicios y Protección de Integridad

---

## ✅ Caso E1: Edición de Servicio

Permitido siempre:

- Cambiar precio.
- Activar / desactivar.

Bloqueado si existe cualquier cita creada (pasada o futura):

- title
- description
- imágenes

Esto protege la integridad histórica del marketplace.

---

## ✅ Caso E2: Inactivación de Servicio

Si un Service pasa a `inactive`:

- No puede recibir nuevas citas.
- Los slots asociados deben inactivarse.
- Citas futuras pasan a `seller_canceled`.
- Se generan notificaciones correspondientes.

---

# ⭐ Sección F: Sistema de Rating

Tabla `ratings`:

- user_id
- appointment_id
- score
- description
- state

---

## ✅ Caso F1: Creación de Rating

Permitido únicamente si:

- La cita está en estado `finished`.
- El usuario que califica es el dueño de la cita.

Reglas:

- Una sola calificación por cita.
- Score obligatorio.
- No editable tras su creación.
- Puede ser inactivado por soporte.

---

## ✅ Caso F2: Moderación por Soporte

Un usuario con rol `support` puede:

- Inactivar rating.
- Inactivar servicio.
- Inactivar usuario.

Sin restricciones de Company.

---

# 🔒 Sección G: Reglas Globales del Sistema

1. No existe eliminación física de:
   - Users
   - Companies
   - Services
   - Appointment Slots
   - Appointments
2. Email único global.
3. Roles acumulativos.
4. Todas las fechas almacenadas en UTC.
5. Control obligatorio de concurrencia en creación de citas.
6. Suspensiones y cancelaciones masivas no son reversibles.
7. Los estados de cada entidad se gestionan mediante AASM.