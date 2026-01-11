# Proyecto Inmobiliaria / Guorkers

## Descripción general
Proyecto compuesto por:
- Frontend: Angular
- Backend: Rails API
- Base de datos: PostgreSQL

Inicialmente todo se configuró para ejecutarse con Docker,
pero en producción (AWS) se decidió correr los servicios
directamente en el host por razones de performance.

---

## Arquitectura DEV (sin Docker)

    Móvil / PC
       |
    Angular dev-server (LAN)
    http://192.168.2.103:4200
       |
    /services-api
       |
    Proxy Angular
       |
    Rails API
    http://127.0.0.1:3001

El móvil nunca accede directamente a Rails.

---

## Arquitectura Docker (referencia)

Servicios:
- db: PostgreSQL 14
- backend: Rails (3000)
- frontend: Angular + Nginx (80)

Accesos:
- Frontend: http://localhost:8080
- Backend interno: http://backend-host:3000

---

## docker-compose.yml

Ubicado en la carpeta padre de frontend y backend.

Define:
- Red bridge propia
- Alias de red (db-host, backend-host)
- Volumen persistente para PostgreSQL

---

## Uso recomendado actualmente

Desarrollo:
- Angular con ng serve (LAN)
- Rails sin Docker
- Proxy Angular activo

Producción:
- Servicios instalados directamente en el host
- Docker solo como referencia o entorno local

---

## Comandos rápidos para ejecucion de la apliacion (DEV)

Backend:

    rails s

Frontend:

    npx ng serve --host 0.0.0.0 --port 4200 --disable-host-check

Abrir desde navegador de la PC:
- http://localhost:4200/

Desde el navegador del teléfono
- http://[IP_DEL_PC]:4200

NOTA: en este último escenario el teléfono y la PC de desarrollo deben estar en la misma subred.
