# Guorkers — Inconsistencias y Pendientes

Análisis generado el 2026-03-01 comparando MODELING.MD vs implementación actual.
Actualizar este archivo a medida que se resuelvan los ítems.

---

## 🔴 Críticos

### C-1: Concurrencia en creación de citas (Caso C2)
- **Problema:** La implementación hacía `count → check → save` sin transacción ni row lock.
  Con dos requests simultáneos se podía superar `max_requests`.
- **Solución implementada 2026-03-01:** `slot.with_lock` dentro de `ActiveRecord::Base.transaction`.
  El row lock bloquea el slot en PG hasta que la transacción confirma, garantizando
  que el count y el save sean atómicos.
- **Archivo:** `inmobiliaria_api/app/controllers/appointments_controller.rb`
- **Estado:** ✅ Resuelto

### C-2: Renombrar `admin` → `seller` en el código
- **Decisión:** `admin` pasa a llamarse `seller`. `assistant` se mantiene.
  `root` y `support` son roles de plataforma (sin company_id fijo).
  MODELING.MD actualizado a versión 3.0 con el nuevo sistema de roles.
- **Implementado 2026-03-01:**
  - `user.rb` → `ROLES = %i[root support seller assistant customer]` ✅
  - `user.rb` → validación de company usa `has_any_role?(:seller, :assistant)` ✅
  - `registrations_controller.rb` → asigna `:seller` al registrar proveedor ✅
  - `user-role.enum.ts` → enum completo: `Root`, `Support`, `Seller`, `Assistant`, `Customer` ✅
  - No requirió migración de BD: `seller` ocupa el mismo bit que `admin` en el bitmask ✅
  - Vistas `view :admin` en blueprints son identificadores internos, no roles → se dejan igual ✅
- **Estado:** ✅ Resuelto

---

## 🟠 Importantes

### I-1: Estado `inactive` falta en AppointmentSlot
- **Problema:** MODELING.MD define estados `active`, `inactive`, `suspended`.
  El AASM solo tiene `active` y `suspended`. Falta `inactive` y sus transiciones.
- **Archivo:** `inmobiliaria_api/app/models/appointment_slot.rb`
- **Estado:** ⏳ Pendiente

### I-2: Cascada al suspender Slot (Caso B3)
- **Problema:** Al suspender un slot, el modelo dice que se deben cancelar todas las citas
  activas asociadas (`seller_canceled`) y generar notificaciones.
  La implementación solo cambia el estado del slot.
- **Archivo:** `inmobiliaria_api/app/controllers/appointment_slots_controller.rb` (acción `suspend`)
- **Estado:** ⏳ Pendiente

### I-3: Cascada al inactivar Usuario (Caso A4)
- **Problema:** Al inactivar un user, se debe cancelar sus citas activas.
  Si tiene company: también cancelar citas de sus slots y pasar sus servicios/slots a `inactive`.
  Nada de esto está implementado.
- **Estado:** ⏳ Pendiente

### I-4: Cascada al inactivar Servicio (Caso E2)
- **Problema:** Al inactivar un service, se deben cancelar citas futuras asociadas
  y notificar a los usuarios.
- **Archivo:** `inmobiliaria_api/app/controllers/services_controller.rb`
- **Estado:** ⏳ Pendiente

---

## 🟡 Medios

### M-1: Validaciones faltantes en creación de cita (Caso C2)
- **Problema:** Al crear un `Appointment`, no se valida:
  - Que el `Service` esté en estado `active`
  - Que la `Company` esté en estado activo
- **Archivo:** `inmobiliaria_api/app/controllers/appointments_controller.rb`
- **Estado:** ⏳ Pendiente

### M-2: Transición `active → attended` no permitida por el modelado
- **Problema:** AASM actual permite `active → attended` directamente
  (`event :attend` tiene `from: [:active, :arrived]`).
  El modelado indica que desde `active` solo se puede ir a `arrived`, `missed`,
  `user_canceled` o `seller_canceled`.
- **Archivo:** `inmobiliaria_api/app/models/appointment.rb`
- **Estado:** ⏳ Pendiente

### M-3: Edición de Slot sin guardián de citas (Caso B2)
- **Problema:** Se puede editar un slot aunque ya tenga citas asociadas. Debería bloquearse.
- **Archivo:** `inmobiliaria_api/app/controllers/appointment_slots_controller.rb` (acción `update`)
- **Estado:** ⏳ Pendiente

### M-4: Eliminación de Slot sin guardián de citas (Caso B4)
- **Problema:** Se puede eliminar un slot aunque tenga citas. Debería bloquearse.
- **Archivo:** `inmobiliaria_api/app/controllers/appointment_slots_controller.rb` (acción `destroy`)
- **Estado:** ⏳ Pendiente

### M-5: Filtrado de búsqueda pública incompleto (Caso C1)
- **Problema:** El endpoint público `GET /services` no filtra por:
  - `Service.state == active`
  - `Company.state == active`
- **Archivo:** `inmobiliaria_api/app/controllers/services_controller.rb` (acción `index`)
- **Estado:** ⏳ Pendiente

---

## 🟢 Menores / Secciones no implementadas aún

### N-1: Cancelación de cita desde "Mis citas" (Caso C3)
- **Implementado 2026-03-01:**
  - Backend: acción `cancel` en `AppointmentsController` con `may_cancel_by_user?` + `cancel_by_user!`
  - Ruta: `PATCH /appointments/:id/cancel`
  - Frontend: `cancelAppointment(id)` en `AppointmentsService`
  - UI: botón "Cancelar" con confirm + spinner solo en citas `active`; recarga lista tras éxito
- **Estado:** ✅ Resuelto

### N-2a: Caso A5 — Seller gestiona colaboradores (Assistants)
- **Descripción:** Pantalla para que el seller busque usuarios por email, los asocie
  como `assistant` de su company o los desafilie. Requiere:
  - Backend: endpoint para asociar/desafiliar assistant
  - Frontend: componente de gestión de colaboradores
  - Restricción: no se puede asociar a un `seller`, `root` o `support`
- **Estado:** ⏳ Pendiente

### N-2b: Caso A6 — Support/Root selecciona company al login
- **Descripción:** Si el usuario es `root` o `support`, tras el login se presenta
  una segunda pantalla para seleccionar la company en la que va a operar.
  La selección contextualiza la sesión sin modificar el `company_id` en BD.
  - `support`: solo ve sus companies asignadas
  - `root`: ve todas las companies activas
- **Estado:** ⏳ Pendiente

### N-2c: Caso A7 — Root gestiona companies asignadas a Support
- **Descripción:** Pantalla (solo para `root`) para asignar/quitar companies
  a cada usuario `support`. Requiere tabla intermedia `support_companies` (user_id + company_id).
- **Estado:** ⏳ Pendiente

### N-2: Conversión de customer a proveedor (Caso A3)
- **Descripción:** Flujo para que un customer active "Quiero ofrecer servicios",
  se le crea una Company y se le agrega rol admin/seller.
- **Estado:** ⏳ Pendiente

### N-3: Recuperación de contraseña (Caso A2)
- **Descripción:** Devise lo soporta en backend. Falta flujo en frontend.
- **Estado:** ⏳ Pendiente

### N-4: Protección de campos de Servicio si existen citas (Caso E1)
- **Descripción:** Si un servicio tiene citas (pasadas o futuras), no se debe poder editar
  `title`, `description` ni imágenes.
- **Archivo:** `inmobiliaria_api/app/controllers/services_controller.rb` (acción `update`)
- **Estado:** ⏳ Pendiente

### N-5: Sistema de Rating (Sección F)
- **Descripción:** El modelo `Rating` existe en BD pero no tiene controller ni rutas.
  Solo permitido si la cita está en `finished` y el user es el dueño.
- **Estado:** ⏳ Pendiente

### N-6: Sistema de Notificaciones
- **Descripción:** El modelo `Notification` existe pero no está implementado.
  Se dispara en B3, E2, A4.
- **Estado:** ⏳ Pendiente

### N-7: Validación de overlap de slots (Caso B1)
- **Descripción:** No existe validación de que dos slots de la misma Company
  no se solapen en horario.
- **Archivo:** `inmobiliaria_api/app/models/appointment_slot.rb`
- **Estado:** ⏳ Pendiente

---

## 🧪 Tests (RSpec — Backend)

> El frontend no se testea con unit tests. La estrategia es: RSpec sólido en backend
> + E2E selectivo (Playwright/Cypress) para flujos críticos en el futuro.
> Revisar y actualizar specs existentes que puedan estar desactualizadas.

### T-1: Modelo `User` — roles y bitmask
- Que `seller?`, `assistant?`, `customer?`, `root?`, `support?` funcionen correctamente
- Que `has_any_role?` sea correcto con combinaciones
- Que la validación de `company` se exija solo para `seller` y `assistant`
- Que `assign_default_role` asigne `customer` a un usuario nuevo
- Que `add_role` y `remove_role` manipulen el bitmask correctamente
- **Archivo:** `spec/models/user_spec.rb`
- **Estado:** ⏳ Pendiente

### T-2: Modelo `Appointment` — máquina de estados AASM
- Que el estado inicial sea `active`
- Transiciones válidas: `active → arrived`, `active → missed`, `active → user_canceled`, `active → seller_canceled`
- Transiciones inválidas: `active → attended` directo (inconsistencia M-2 pendiente de corregir)
- Estados terminales no tienen transiciones salientes
- **Archivo:** `spec/models/appointment_spec.rb`
- **Estado:** ⏳ Pendiente

### T-3: Modelo `AppointmentSlot` — validaciones y estados
- Que `starting` deba ser futuro
- Que `starting` no pueda ser mayor a 30 días
- Que `max_requests` y `duration` sean obligatorios
- Estados AASM: `active → suspended → active`
- **Archivo:** `spec/models/appointment_slot_spec.rb`
- **Estado:** ⏳ Pendiente

### T-4: `AppointmentsController#create` — casos de negocio críticos
- ✅ Crea cita correctamente con datos válidos
- ❌ Falla si el slot no existe
- ❌ Falla si el slot está suspendido
- ❌ Falla si el slot ya pasó
- ❌ Falla si el cupo está lleno (`max_requests` superado)
- ❌ Falla si el usuario ya tiene una cita activa en ese slot
- ❌ Falla si no hay sesión (401)
- 🔒 Concurrencia: dos requests simultáneos no superan `max_requests`
- **Archivo:** `spec/requests/appointments_spec.rb`
- **Estado:** ⏳ Pendiente

### T-5: `AppointmentsController#cancel` — cancelación
- ✅ Cancela correctamente una cita `active` del usuario
- ❌ Falla si la cita no pertenece al usuario
- ❌ Falla si la cita no está en estado `active` (ej: ya `user_canceled`)
- ❌ Falla si no hay sesión (401)
- **Archivo:** `spec/requests/appointments_spec.rb`
- **Estado:** ⏳ Pendiente

### T-6: `AppointmentsController#index` — listado de citas
- ✅ Devuelve solo las citas del usuario autenticado
- ✅ No devuelve citas de otros usuarios
- ❌ Falla si no hay sesión (401)
- **Archivo:** `spec/requests/appointments_spec.rb`
- **Estado:** ⏳ Pendiente

### T-7: `ServicesController#appointment_slots` — slots disponibles para un servicio
- ✅ Devuelve solo slots activos y futuros vinculados al servicio
- ✅ Incluye `current_requests` en la respuesta
- ✅ No devuelve slots pasados ni suspendidos
- **Archivo:** `spec/requests/services_spec.rb`
- **Estado:** ⏳ Pendiente

### T-8: `AppointmentSlotsController` — guardanes de edición y eliminación (M-3, M-4)
- ❌ No permite editar un slot que ya tiene citas asociadas
- ❌ No permite eliminar un slot que ya tiene citas asociadas
- ✅ Permite editar/eliminar si no tiene citas
- **Archivo:** `spec/requests/appointment_slots_spec.rb`
- **Estado:** ⏳ Pendiente (implementar guardanes primero — ver M-3 y M-4)

---

## ✅ Resueltos

### N-1: Cancelación de cita desde "Mis citas" (2026-03-01)
Botón "Cancelar" solo para citas `active`. Backend con row lock implícito en AASM. Frontend con confirm + spinner.

### C-1: Concurrencia en creación de citas (2026-03-01)
`slot.with_lock` dentro de `ActiveRecord::Base.transaction`. Atómico a nivel PG.

### C-2: Roles — renombrar `admin` → `seller` (2026-03-01)
`seller` reemplaza a `admin` en el mismo bit del bitmask. Sin migración de BD.
Enum del frontend completado con los 5 roles definitivos.
