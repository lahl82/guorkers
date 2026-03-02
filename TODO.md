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
> Estimación total: ~280–350 ejemplos para alcanzar 70–80 % de cobertura.

---

### 🔧 Specs existentes desactualizadas — corregir antes de ejecutar

**`spec/models/user_spec.rb`**
- `have_many(:requests)` → debe ser `have_many(:appointments)` (modelo renombrado)
- `build(:user, role: 1)` → interfaz obsoleta (`role` ya no es columna enum; usar `role_mask` o factory con `add_role`) → reemplazar o eliminar
- Sin tests de AASM ni de métodos de bitmask (cubiertos en T-1)

**`spec/models/service_spec.rb`**
- `belong_to(:user)` → debe ser `belong_to(:company)` (asociación actual del modelo)
- `have_many(:requests)` → debe ser `have_many(:appointment_slot_services)` (relación actual)

**`spec/models/rating_spec.rb`**
- `RSpec.describe Request` → debe ser `RSpec.describe Rating` (clase incorrecta)
- `belong_to(:service)` → debe ser `belong_to(:appointment)` (asociación actual del modelo)
- Falta validación de `score` (obligatorio, entero, 1–5); sin tests AASM

**`spec/models/question_spec.rb`**
- `RSpec.describe Request` → debe ser `RSpec.describe Question` (clase incorrecta)
- Sin tests AASM (`visible → hidden → visible`)

**`spec/models/request_spec.rb`**
- Corresponde al modelo `Request` que ya no existe en el proyecto → **eliminar**

---

### T-1: Modelo `User` — roles y bitmask
- Que `seller?`, `assistant?`, `customer?`, `root?`, `support?` funcionen correctamente
- Que `has_any_role?` sea correcto con combinaciones de roles
- Que `roles=` sobreescriba el bitmask completo
- Que `add_role` agregue un bit sin afectar los otros
- Que `remove_role` quite un bit sin afectar los otros
- Que `roles` getter devuelva el array correcto a partir del bitmask
- Que `User.with_role(:seller)` scope filtre correctamente
- Que `assign_default_role` asigne `:customer` a un usuario nuevo (`after_initialize`)
- Que la validación de `company` se exija solo para `seller` y `assistant`, no para otros roles
- AASM: estado inicial `created`; transiciones `created → active`, `active → suspended`, `suspended → active`
- **Archivo:** `spec/models/user_spec.rb`
- **Estado:** ⏳ Pendiente

### T-2: Modelo `Appointment` — máquina de estados AASM
- Estado inicial `active`
- Transiciones válidas: `active → arrived`, `active → missed`, `active → user_canceled`, `active → seller_canceled`
- Transición `arrived → attended`, `arrived → missed`
- Transición `attended → with_pending_info`, `attended → finished`
- Transición `with_pending_info → finished`
- Transición inválida: `active → attended` directo debe fallar (inconsistencia M-2 — corregir primero)
- Estados terminales (`finished`, `missed`, `user_canceled`, `seller_canceled`) no tienen transiciones salientes
- Asociaciones: `belongs_to :customer` (User), `belongs_to :appointment_slot_service`, `has_many :ratings`, `has_many :notifications`
- **Archivo:** `spec/models/appointment_spec.rb`
- **Estado:** ⏳ Pendiente

### T-3: Modelo `AppointmentSlot` — validaciones y estados
- Que `starting`, `duration`, `max_requests` sean obligatorios
- Que `starting` en el pasado genere error `:not_in_future`
- Que `starting` a más de 30 días genere error `:too_far_in_future`
- Que `starting` válido (futuro ≤ 30 días) no genere errores
- AASM: estado inicial `active`; `active → suspended`, `suspended → active`
- Asociaciones: `belongs_to :company`, `has_many :appointment_slot_services`, `has_many :services through:`, `has_many :appointments`
- **Archivo:** `spec/models/appointment_slot_spec.rb`
- **Estado:** ⏳ Pendiente

### T-4: `AppointmentsController#create` — casos de negocio críticos
- ✅ Crea cita correctamente con slot activo, service activo, cupo disponible
- ❌ Falla si la combinación `appointment_slot_id + service_id` no existe (404)
- ❌ Falla si el slot está suspendido (422)
- ❌ Falla si el slot ya pasó (422)
- ❌ Falla si el cupo está lleno (`current_requests >= max_requests`) (422)
- ❌ Falla si el usuario ya tiene una cita `active` en ese slot (422, mensaje distinto)
- ❌ Falla si no hay sesión (401)
- 🔒 Concurrencia: dos threads simultáneos no superan `max_requests` (usar threads reales o `allow_any_instance_of` + lock test)
- **Archivo:** `spec/requests/appointments_spec.rb`
- **Estado:** ⏳ Pendiente

### T-5: `AppointmentsController#cancel` — cancelación por usuario
- ✅ Cancela correctamente una cita `active` propia → estado pasa a `user_canceled`
- ❌ Falla con 404 si la cita no pertenece al usuario autenticado
- ❌ Falla con 422 si la cita no está en estado `active` (ej: `user_canceled`, `missed`)
- ❌ Falla con 401 si no hay sesión
- **Archivo:** `spec/requests/appointments_spec.rb`
- **Estado:** ⏳ Pendiente

### T-6: `AppointmentsController#index` — listado de citas del usuario
- ✅ Devuelve solo las citas del usuario autenticado (no las de otros)
- ✅ Respuesta incluye `service_name`, `starting`, `duration`, `state`
- ❌ Falla con 401 si no hay sesión
- **Archivo:** `spec/requests/appointments_spec.rb`
- **Estado:** ⏳ Pendiente

### T-7: `ServicesController#appointment_slots` — slots disponibles para un servicio
- ✅ Devuelve slots activos y futuros vinculados al servicio, ordenados por `starting`
- ✅ Cada slot incluye `current_requests`
- ✅ No devuelve slots con `state: suspended`
- ✅ No devuelve slots cuyo `starting` ya pasó
- ✅ Endpoint es público (no requiere autenticación)
- **Archivo:** `spec/requests/services_spec.rb`
- **Estado:** ⏳ Pendiente

### T-8: `AppointmentSlotsController` — guardanes de edición y eliminación (M-3, M-4)
- ❌ No permite editar un slot que ya tiene citas asociadas (422)
- ❌ No permite eliminar un slot que ya tiene citas asociadas (422)
- ✅ Permite editar si no tiene citas
- ✅ Permite eliminar si no tiene citas
- **Archivo:** `spec/requests/appointment_slots_spec.rb`
- **Estado:** ⏳ Pendiente (implementar guardanes primero — ver M-3 y M-4)

---

### T-9: Modelo `Company` — AASM y asociaciones
- Estado inicial `created`
- Transiciones: `created → active`, `active → suspended`, `suspended → active`, cualquier estado → `archived`
- Transición inválida: `archived → active` debe fallar
- Asociaciones: `has_many :users`, `has_many :appointment_slots`, `has_many :services`
- **Archivo:** `spec/models/company_spec.rb`
- **Estado:** ⏳ Pendiente

### T-10: Modelo `Service` — AASM, validaciones, `name_format`
- Validaciones: `title` (presencia, max 255, `name_format: true`), `description` (presencia, max 1000), `price` (presencia, numérico ≥ 0)
- `name_format`: títulos con caracteres especiales inválidos son rechazados
- AASM: `created → active`, `active → disabled`, `disabled → active`, `active → suspended`, `suspended → active`, `created → rejected`
- Transición inválida: `rejected → active` debe fallar
- Asociaciones correctas: `belongs_to :company`, `belongs_to :service_type`, `has_many :appointment_slot_services`, `has_many :questions`
- **Archivo:** `spec/models/service_spec.rb` (corregir asociaciones erróneas primero)
- **Estado:** ⏳ Pendiente

### T-11: Modelo `Rating` — AASM, validaciones, asociaciones
- Validaciones: `description` (presencia), `score` (presencia, entero, 1–5)
- AASM: estado inicial `visible`; `visible → hidden`, `hidden → visible`
- Asociaciones: `belongs_to :user`, `belongs_to :appointment` (NO `service`)
- Factory válida
- **Archivo:** `spec/models/rating_spec.rb` (corregir clase y asociaciones primero)
- **Estado:** ⏳ Pendiente

### T-12: Modelo `Question` — AASM, validaciones
- Validación: `description` presencia
- AASM: estado inicial `visible`; `visible → hidden`, `hidden → visible`
- Asociaciones: `belongs_to :user`, `belongs_to :service`
- **Archivo:** `spec/models/question_spec.rb` (corregir clase `RSpec.describe Request` primero)
- **Estado:** ⏳ Pendiente

### T-13: Modelo `Notification` — AASM, validaciones
- Validaciones: `description` y `sent_at` obligatorios
- AASM: estado inicial `pending`; `pending → sent`, `pending → failed`, `sent → failed`
- Asociación: `belongs_to :appointment`
- **Archivo:** `spec/models/notification_spec.rb` (crear)
- **Estado:** ⏳ Pendiente

### T-14: Modelo `AppointmentSlotService` — asociaciones
- `belongs_to :appointment_slot`
- `belongs_to :service`
- `has_many :appointments`
- Factory válida
- **Archivo:** `spec/models/appointment_slot_service_spec.rb`
- **Estado:** ⏳ Pendiente

---

### T-15: `SessionsController` — login y logout
- `POST /users/sign_in` con credenciales válidas → 200, devuelve `user` + JWT en header
- `POST /users/sign_in` con contraseña incorrecta → 401
- `POST /users/sign_in` con email inexistente → 401
- `DELETE /users/sign_out` con token válido → 200
- `DELETE /users/sign_out` con token expirado → 401
- Respuesta de login incluye `company` cuando el usuario es seller
- **Archivo:** `spec/requests/users/sessions_controller_spec.rb`
- **Estado:** ⏳ Pendiente

### T-16: `RegistrationsController` — registro customer y seller
- `POST /users` con datos válidos de customer → 201, usuario con rol `customer`, sin company
- `POST /users` con datos válidos de seller (`is_seller: true`, datos de company) → 201, usuario con rol `seller`, company creada
- `POST /users` sin email → 422
- `POST /users` con email duplicado → 422
- `POST /users` con contraseña muy corta → 422
- `POST /users` con nombre inválido (caracteres no permitidos) → 422
- **Archivo:** `spec/requests/users/registrations_controller_spec.rb`
- **Estado:** ⏳ Pendiente

### T-17: `ServicesController` — acciones del catálogo
- `GET /services` (público): devuelve lista paginada con `pagination`; sin autenticación funciona
- `GET /services/:id` (público): devuelve detalle del servicio con fotos
- `POST /services` (seller): crea servicio → 201
- `POST /services` sin autenticación → 401
- `POST /services` con datos inválidos → 422 con detalles
- `GET /services/mine` (seller): devuelve solo servicios de la company del seller
- `GET /services/mine` para usuario sin company (customer) → 422
- `GET /services/basic_mine` (seller): devuelve servicios sin paginación
- **Archivo:** `spec/requests/services_spec.rb`
- **Estado:** ⏳ Pendiente

### T-18: `AppointmentSlotsController` — CRUD completo del seller
- `GET /appointment_slots`: devuelve slots de la company del seller autenticado
- `GET /appointment_slots/for_month`: filtra por año y mes; parámetros inválidos → 400
- `GET /appointment_slots/:id`: devuelve slot de la company; 404 si no pertenece
- `POST /appointment_slots`: crea slot válido → 201; datos inválidos → 422
- `PATCH /appointment_slots/:id`: actualiza slot sin citas → 200 *(ver T-8)*
- `PATCH /appointment_slots/:id/suspend`: suspende slot activo → 200; ya suspendido → 422
- `PATCH /appointment_slots/:id/resume`: reanuda slot suspendido → 200
- `DELETE /appointment_slots/:id`: elimina slot sin citas → 200 *(ver T-8)*
- `PATCH /appointment_slots/:id/update_services`: actualiza servicios del slot → 201
- Todas las acciones sin autenticación → 401
- **Archivo:** `spec/requests/appointment_slots_spec.rb`
- **Estado:** ⏳ Pendiente

### T-19: `ServiceTypesController` — catálogo de tipos de servicio
- `GET /service_types`: devuelve lista de tipos; endpoint público
- Respuesta incluye `id` y `name`
- **Archivo:** `spec/requests/service_types_spec.rb`
- **Estado:** ⏳ Pendiente

### T-20: `UsersController` — servicios de un usuario público
- `GET /users/:id/services`: devuelve servicios paginados del usuario dado
- `GET /users/:id/basic_services`: devuelve servicios sin paginación
- **Nota:** `appointment_slots` usa `user_id` en slots pero el modelo usa `company_id`; verificar si esta acción funciona y agregar spec de regresión
- **Archivo:** `spec/requests/users_spec.rb`
- **Estado:** ⏳ Pendiente

---

### T-21: Concern `ApiResponseHandler` — formato estándar de respuestas
- `render_success`: respuesta incluye `{ success: true, message:, data: }` con el código HTTP correcto
- `render_error`: respuesta incluye `{ success: false, message:, details: }` con el código HTTP correcto
- Código HTTP por defecto: 200 para success, 422 para error
- **Archivo:** `spec/concerns/api_response_handler_spec.rb` (crear)
- **Estado:** ⏳ Pendiente

### T-22: `NameFormatValidator` — validador personalizado de nombres
- Acepta: letras, espacios, guiones y apóstrofes (`O'Reilly`, `Jean-Pierre`)
- Rechaza: caracteres especiales (`@`, `#`, dígitos en posición inválida)
- Se usa en `Service#title` — probar vía modelo
- **Archivo:** `spec/validators/name_format_validator_spec.rb` (crear)
- **Estado:** ⏳ Pendiente

---

## ✅ Resueltos

### N-1: Cancelación de cita desde "Mis citas" (2026-03-01)
Botón "Cancelar" solo para citas `active`. Backend con row lock implícito en AASM. Frontend con confirm + spinner.

### C-1: Concurrencia en creación de citas (2026-03-01)
`slot.with_lock` dentro de `ActiveRecord::Base.transaction`. Atómico a nivel PG.

### C-2: Roles — renombrar `admin` → `seller` (2026-03-01)
`seller` reemplaza a `admin` en el mismo bit del bitmask. Sin migración de BD.
Enum del frontend completado con los 5 roles definitivos.
