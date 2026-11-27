# Contrato de API — Proyecto 09 · Reservas de Laboratorios

**Versión:** 1.0 (borrador para aprobación)
**Responsable:** Miguel Angel Galvis Torres
**Revisores:** Juan Diego Isaza Ramírez · Samuel Gómez Giraldo
**Última actualización:** 2026-11-27
**Estado:** Para aprobar en PR antes de implementar

---

## 1. Alcance

Este documento define el contrato entre:

- La interfaz web (Samuel).
- La API Flask (Miguel).
- La infraestructura AWS (Juan Diego).

Cubre las historias **HU-01 a HU-07** y los servicios de apoyo para **HU-05** (notificaciones), **HU-06** (concurrencia) y **HU-08** (despliegue). Si la implementación necesita cambiar algo, se actualiza este archivo en el mismo PR.

**Base URL:**

- Local: `http://localhost:8000`
- AWS: `http://<alb-dns>` (más adelante, HTTPS)
- Health: `/health` y `/ready`
- API: todo bajo `/api`

---

## 2. Convenciones

| Tema | Convención |
|---|---|
| Formato | JSON UTF-8 |
| Content-Type | `application/json` en POST/PUT/PATCH |
| IDs | Enteros |
| Fechas | ISO 8601 con offset, ejemplo `2026-12-01T08:00:00-05:00` |
| Fechas de filtro | `YYYY-MM-DD` |
| Zona horaria | `America/Bogota` (UTC-5, sin horario de verano) |
| Rangos | `[inicio, fin)`: el fin no se solapa con el inicio de la siguiente reserva |
| Nombres | `snake_case` |
| Autenticación | Cookie de sesión + token CSRF |
| Errores | Envelope `{ "error": { "code", "message", "details" } }` |
| Listas | `{ "items": [...], "total": n, "limit": n, "offset": n }` |

### 2.1 Códigos HTTP

| Código | Uso |
|---|---|
| `200` | OK |
| `201` | Recurso creado |
| `204` | Sin contenido (logout, borrado lógico) |
| `400` | JSON malformado, campo faltante, tipo incorrecto o fecha inválida |
| `401` | No autenticado |
| `403` | Sin permiso por rol, propiedad o CSRF inválido |
| `404` | Sala o reserva inexistente |
| `409` | Conflicto: solapamiento o código de sala duplicado |
| `422` | Regla de negocio incumplida: mínimo, capacidad, horario, duración, plazo |
| `500` | Error interno; no exponer stack trace |

> **Importante:** `400` es para requests malformados. `422` es para requests bien formados que incumplen una regla de negocio. Esto cumple el entregable #7: “un grupo menor al mínimo responde 422”.

### 2.2 Formato de error

```json
{
  "error": {
    "code": "attendees_below_minimum",
    "message": "La sala exige un mínimo de 10 estudiantes.",
    "details": {
      "room_id": 1,
      "min_students": 10,
      "attendees": 8
    }
  }
}
```

- `code`: identificador estable para la interfaz.
- `message`: texto en español para mostrar al usuario.
- `details`: opcional, datos útiles para depurar.

---

## 3. Autenticación y roles

### 3.1 Roles

| Rol | Puede |
|---|---|
| `estudiante` | Consultar salas y disponibilidad. No reserva ni administra. |
| `docente` | Reservar, cancelar y mover sus propias reservas. Consultar disponibilidad. |
| `admin` | Todo lo del docente, administrar salas, ver todas las reservas y ver notificaciones. |

### 3.2 Sesión y CSRF

- El login crea una cookie de sesión `session` (`HttpOnly`, `SameSite=Lax`).
- Todo `POST`, `PUT`, `PATCH` y `DELETE` requiere el header `X-CSRFToken`.
- El token se obtiene en `GET /api/auth/csrf` o en la respuesta de `GET /api/auth/me`.
- En AWS con HTTPS, la cookie debe llevar `Secure`.
- La API es same-origin; no se configuran headers CORS en v1.

### 3.3 Endpoints de autenticación

#### `GET /api/auth/csrf`

Público. Devuelve un token CSRF y prepara la sesión.

**200**

```json
{
  "csrf_token": "IjoxNzY0..."
}
```

#### `POST /api/auth/login`

Público. Requiere `X-CSRFToken`.

**Body**

```json
{
  "email": "docente@example.com",
  "password": "Docente1234!"
}
```

**200**

```json
{
  "user": {
    "id": 2,
    "email": "docente@example.com",
    "full_name": "Docente de prueba",
    "role": "docente"
  },
  "csrf_token": "IjoxNzY0..."
}
```

**401** si las credenciales son inválidas.

#### `POST /api/auth/logout`

Requiere sesión y CSRF. **204** sin cuerpo.

#### `GET /api/auth/me`

Requiere sesión.

**200**

```json
{
  "user": {
    "id": 2,
    "email": "docente@example.com",
    "full_name": "Docente de prueba",
    "role": "docente"
  },
  "csrf_token": "IjoxNzY0..."
}
```

**401** si no hay sesión.

---

## 4. Modelos de datos

### 4.1 `User`

```json
{
  "id": 2,
  "email": "docente@example.com",
  "full_name": "Docente de prueba",
  "role": "docente"
}
```

### 4.2 `Room`

```json
{
  "id": 1,
  "code": "LAB-101",
  "name": "Laboratorio de Redes",
  "building": "Bloque A",
  "capacity": 15,
  "min_students": 10,
  "opens_at": "06:00",
  "closes_at": "22:00",
  "slot_minutes": 120,
  "max_hours": 4,
  "active": true
}
```

### 4.3 `Reservation`

```json
{
  "id": 42,
  "room": {
    "id": 1,
    "code": "LAB-101",
    "name": "Laboratorio de Redes",
    "building": "Bloque A",
    "capacity": 15
  },
  "requester": {
    "id": 2,
    "full_name": "Docente de prueba",
    "email": "docente@example.com"
  },
  "start": "2026-12-01T08:00:00-05:00",
  "end": "2026-12-01T10:00:00-05:00",
  "attendees": 12,
  "purpose": "Práctica de redes",
  "status": "active",
  "created_at": "2026-11-27T10:00:00-05:00",
  "updated_at": "2026-11-27T10:00:00-05:00",
  "cancelled_at": null,
  "cancel_reason": null,
  "can_cancel": true,
  "can_move": true
}
```

- `status`: `active` o `cancelled`.
- `can_cancel` y `can_move`: calculados por el servidor según rol, propiedad y plazo.
- Una reserva cancelada se conserva para auditoría; no se borra.

### 4.4 `Notification`

```json
{
  "id": 7,
  "reservation_id": 42,
  "kind": "last_minute",
  "recipient": "admin@example.com",
  "status": "sent",
  "sent_at": "2026-11-27T10:00:05-05:00",
  "detail": null
}
```

- `kind`: `last_minute`.
- `status`: `pending`, `sent` o `failed`.

---

## 5. Endpoints

### 5.1 Resumen

| Método | Ruta | Auth | Descripción |
|---|---|---|---|
| GET | `/health` | Público | Liveness |
| GET | `/ready` | Público | Readiness + base de datos |
| GET | `/api/auth/csrf` | Público | Token CSRF |
| POST | `/api/auth/login` | Público | Iniciar sesión |
| POST | `/api/auth/logout` | Sesión | Cerrar sesión |
| GET | `/api/auth/me` | Sesión | Usuario actual |
| GET | `/api/rooms` | Público | Listar salas activas |
| GET | `/api/rooms/{id}` | Público | Detalle de sala |
| POST | `/api/rooms` | Admin | Crear sala |
| PUT | `/api/rooms/{id}` | Admin | Editar sala |
| DELETE | `/api/rooms/{id}` | Admin | Borrado lógico |
| GET | `/api/availability` | Público | Disponibilidad semanal |
| GET | `/api/reservations` | Docente/Admin | Listar reservas |
| GET | `/api/reservations/{id}` | Dueño/Admin | Detalle |
| POST | `/api/reservations` | Docente/Admin | Crear reserva |
| PATCH | `/api/reservations/{id}/cancel` | Dueño/Admin | Cancelar |
| PATCH | `/api/reservations/{id}/move` | Dueño/Admin | Mover |
| GET | `/api/rooms/suggest` | Docente/Admin | Sugerir sala |
| GET | `/api/admin/notifications` | Admin | Historial de alertas |

---

### 5.2 Health

#### `GET /health`

**200**

```json
{
  "status": "ok",
  "service": "gtn-reservas",
  "version": "1.0"
}
```

#### `GET /ready`

Comprueba la conexión a PostgreSQL.

**200**

```json
{
  "status": "ready",
  "database": "ok"
}
```

**503** si la base de datos no responde.

---

### 5.3 Salas

#### `GET /api/rooms`

Público. Devuelve solo salas activas. El admin puede usar `?include_inactive=1`.

**200**

```json
{
  "items": [
    {
      "id": 1,
      "code": "LAB-101",
      "name": "Laboratorio de Redes",
      "building": "Bloque A",
      "capacity": 15,
      "min_students": 10,
      "opens_at": "06:00",
      "closes_at": "22:00",
      "slot_minutes": 120,
      "max_hours": 4,
      "active": true
    }
  ],
  "total": 1
}
```

#### `GET /api/rooms/{id}`

**200** con el objeto `Room`. **404** si no existe.

#### `POST /api/rooms`

Solo admin. Requiere CSRF.

**Body**

```json
{
  "code": "LAB-104",
  "name": "Laboratorio de Software",
  "building": "Bloque B",
  "capacity": 25,
  "min_students": 10,
  "opens_at": "06:00",
  "closes_at": "22:00",
  "slot_minutes": 120,
  "max_hours": 4,
  "active": true
}
```

**201** con `Room` y header `Location: /api/rooms/4`.

**409** si el `code` ya existe.

**422** si `min_students > capacity`, horario inválido o duración inválida.

#### `PUT /api/rooms/{id}`

Solo admin. Reemplaza los campos editables de la sala. Mismas validaciones que `POST`.

**200** con `Room`.

#### `DELETE /api/rooms/{id}`

Solo admin. Borrado lógico: `active = false`. No elimina reservas históricas.

**204** sin cuerpo.

---

### 5.4 Disponibilidad

#### `GET /api/availability`

Público. Devuelve la grilla de la semana por sala.

**Query params**

| Parámetro | Requerido | Ejemplo | Descripción |
|---|---|---|---|
| `from` | Sí | `2026-12-01` | Fecha inicial |
| `to` | Sí | `2026-12-07` | Fecha final; máximo 14 días |
| `room_id` | No | `1` | Filtrar una sala |
| `building` | No | `Bloque A` | Filtrar por edificio |
| `min_capacity` | No | `15` | Capacidad mínima |

**200**

```json
{
  "from": "2026-12-01",
  "to": "2026-12-02",
  "timezone": "America/Bogota",
  "items": [
    {
      "room": {
        "id": 1,
        "code": "LAB-101",
        "name": "Laboratorio de Redes",
        "building": "Bloque A",
        "capacity": 15,
        "min_students": 10
      },
      "days": [
        {
          "date": "2026-12-01",
          "slots": [
            {
              "start": "2026-12-01T06:00:00-05:00",
              "end": "2026-12-01T08:00:00-05:00",
              "status": "free"
            },
            {
              "start": "2026-12-01T08:00:00-05:00",
              "end": "2026-12-01T10:00:00-05:00",
              "status": "occupied"
            }
          ]
        }
      ]
    }
  ]
}
```

Reglas:

- No expone nombres, correos ni motivos de quien reservó.
- Una reserva cancelada no ocupa la franja.
- Las franjas se generan con `slot_minutes` desde `opens_at`.
- Si una reserva cubre varias franjas, todas se marcan `occupied`.
- `400` si `from`/`to` son inválidas o el rango supera 14 días.

---

### 5.5 Reservas

#### `GET /api/reservations`

Requiere sesión de docente o admin.

**Query params**

| Parámetro | Default | Descripción |
|---|---|---|
| `status` | `active` | `active`, `cancelled` o `all` |
| `scope` | `own` | `own`; el admin puede usar `all` |
| `from` | — | Fecha inicial `YYYY-MM-DD` |
| `to` | — | Fecha final `YYYY-MM-DD` |
| `room_id` | — | Filtrar por sala |
| `limit` | `50` | Máximo 100 |
| `offset` | `0` | Paginación |

Un docente solo ve sus reservas. Un admin puede ver todas con `scope=all`.

**200**

```json
{
  "items": [ { "id": 42, "status": "active" } ],
  "total": 1,
  "limit": 50,
  "offset": 0
}
```

**403** si un docente intenta `scope=all` o un estudiante accede.

#### `GET /api/reservations/{id}`

Dueño o admin. **200** con `Reservation`. **403** si no es dueño. **404** si no existe.

#### `POST /api/reservations`

Docente o admin. Requiere CSRF. El solicitante es el usuario autenticado.

**Body**

```json
{
  "room_id": 1,
  "start": "2026-12-01T08:00:00-05:00",
  "end": "2026-12-01T10:00:00-05:00",
  "attendees": 12,
  "purpose": "Práctica de redes"
}
```

**201** con `Reservation` y `Location: /api/reservations/42`.

**400** si falta un campo, el tipo es incorrecto o la fecha no tiene formato ISO.

**422** si:

- `attendees < min_students`.
- `attendees > capacity`.
- La sala está inactiva.
- El inicio es pasado.
- Está fuera de `opens_at`/`closes_at`.
- No está alineada con `slot_minutes`.
- La duración supera `max_hours`.

**409** si se solapa con otra reserva activa de la misma sala.

**403** si el usuario es estudiante.

#### `PATCH /api/reservations/{id}/cancel`

Dueño o admin. Requiere CSRF.

**Body**

```json
{
  "reason": "Cambio de clase"
}
```

**200** con `Reservation` actualizada (`status: "cancelled"`, `cancelled_at`, `cancel_reason`).

**403** si no es dueño ni admin.

**422** si el docente intenta cancelar con menos de `CANCEL_DEADLINE_HOURS` (2 h) de anticipación. El admin puede cancelar siempre.

#### `PATCH /api/reservations/{id}/move`

Dueño o admin. Requiere CSRF. Mueve la misma reserva; conserva el ID y registra un evento interno.

**Body**

```json
{
  "start": "2026-12-02T10:00:00-05:00",
  "end": "2026-12-02T12:00:00-05:00"
}
```

**200** con `Reservation` actualizada.

**409** si la franja destino se solapa; la reserva original queda intacta.

**422** si la nueva franja incumple horario, duración, pasado o plazo.

**403** si no es dueño ni admin.

---

### 5.6 Sugeridor de sala

#### `GET /api/rooms/suggest`

Docente o admin. No crea la reserva; solo propone.

**Query params**

| Parámetro | Requerido | Ejemplo |
|---|---|---|
| `start` | Sí | `2026-12-01T08:00:00-05:00` |
| `end` | Sí | `2026-12-01T10:00:00-05:00` |
| `attendees` | Sí | `12` |

**200**

```json
{
  "items": [
    {
      "room": {
        "id": 1,
        "code": "LAB-101",
        "name": "Laboratorio de Redes",
        "building": "Bloque A",
        "capacity": 15,
        "min_students": 10
      },
      "score": 3,
      "reason": "Capacidad más ajustada: 15 puestos para 12 asistentes."
    }
  ],
  "total": 1
}
```

- Orden: menor desperdicio de capacidad; desempate por menor carga del día.
- Máximo 3 sugerencias.
- **200** con `items: []` si no hay salas disponibles.

---

### 5.7 Notificaciones

#### `GET /api/admin/notifications`

Solo admin.

**Query params**

| Parámetro | Default | Descripción |
|---|---|---|
| `status` | `all` | `pending`, `sent`, `failed` o `all` |
| `limit` | `50` | Máximo 100 |
| `offset` | `0` | Paginación |

**200**

```json
{
  "items": [
    {
      "id": 7,
      "reservation_id": 42,
      "kind": "last_minute",
      "recipient": "admin@example.com",
      "status": "sent",
      "sent_at": "2026-11-27T10:00:05-05:00",
      "detail": null
    }
  ],
  "total": 1,
  "limit": 50,
  "offset": 0
}
```

La app crea las notificaciones; este endpoint solo las consulta.

---

## 6. Reglas de negocio

| Regla | HTTP | `error.code` |
|---|---|---|
| JSON malformado o campo faltante | 400 | `validation_error` |
| Fecha con formato inválido | 400 | `invalid_date` |
| Sin sesión | 401 | `unauthorized` |
| Rol sin permiso | 403 | `forbidden` |
| CSRF ausente o inválido | 403 | `csrf_failed` |
| Sala inexistente | 404 | `room_not_found` |
| Reserva inexistente | 404 | `reservation_not_found` |
| Código de sala duplicado | 409 | `room_code_exists` |
| Solapamiento de reserva activa | 409 | `reservation_overlap` |
| Sala inactiva | 422 | `room_inactive` |
| `start >= end` | 422 | `invalid_time_range` |
| Inicio en el pasado | 422 | `reservation_in_past` |
| Fuera del horario de la sala | 422 | `outside_opening_hours` |
| Franja no alineada con `slot_minutes` | 422 | `slot_misaligned` |
| Duración mayor a `max_hours` | 422 | `duration_exceeds_max` |
| Asistentes menores al mínimo | 422 | `attendees_below_minimum` |
| Asistentes mayores a la capacidad | 422 | `attendees_above_capacity` |
| Cancelación después del plazo | 422 | `cancel_deadline_passed` |
| Error interno | 500 | `internal_error` |

### 6.1 Reglas de reserva

1. La sala debe existir y estar activa.
2. `start < end`.
3. La reserva debe ser futura.
4. Debe estar dentro de `opens_at` y `closes_at`.
5. Debe empezar en un múltiplo de `slot_minutes` contado desde `opens_at`.
6. La duración no puede superar `max_hours`.
7. `min_students <= attendees <= capacity`.
8. No puede solaparse con otra reserva `active` de la misma sala.
9. Solo el dueño o un admin puede cancelar o mover.
10. Un docente solo puede cancelar/mover hasta `CANCEL_DEADLINE_HOURS` antes; el admin no tiene ese límite.
11. Si la reserva empieza en menos de `LAST_MINUTE_HOURS`, se genera una notificación `last_minute` para el admin.
12. Una reserva cancelada libera la franja.

### 6.2 Datos semilla de referencia

| Código | Capacidad | Mínimo | Horario | Franja | Máx. |
|---|---:|---:|---|---:|---:|
| `LAB-101` | 15 | 10 | 06:00–22:00 | 120 min | 4 h |
| `LAB-102` | 20 | 10 | 06:00–22:00 | 120 min | 4 h |
| `LAB-103` | 30 | 10 | 06:00–22:00 | 120 min | 4 h |

Usuarios de demo: `admin@example.com`, `docente@example.com`, `estudiante@example.com`. Las contraseñas van en `.env.example` solo para local; en AWS se cambian.

---

## 7. Concurrencia y no doble reserva

La garantía no depende de un `SELECT` previo en Python. Está en PostgreSQL:

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

ALTER TABLE reservations
  ADD CONSTRAINT no_overlap
  EXCLUDE USING gist (
    room_id WITH =,
    during  WITH &&
  )
  WHERE (status = 'active');
```

- `during` es un `TSTZRANGE` con límites `[inicio, fin)`.
- Si dos transacciones intentan la misma sala y se solapan, una inserta y la otra recibe `ExclusionViolation` (SQLSTATE `23P01`).
- La API captura esa excepción y responde **409** con `error.code = "reservation_overlap"`.
- La prueba oficial son dos `POST /api/reservations` simultáneos: exactamente uno debe devolver **201** y el otro **409**.
- La evidencia debe incluir la salida del script y una consulta SQL con `count(*) = 1` para la franja.
- El script de concurrencia se implementa en el entregable #16 (`tests/concurrencia.sh`).

---

## 8. Seguridad

- Cookie de sesión `HttpOnly` y `SameSite=Lax`; `Secure` cuando haya HTTPS.
- CSRF obligatorio en métodos de escritura.
- Validación y autorización en el servidor; no confiar en la UI.
- Consultas SQL parametrizadas; nunca concatenar entrada del usuario.
- Contraseñas con hash (`werkzeug.security`); nunca texto plano.
- No exponer datos personales en disponibilidad pública.
- No devolver stack traces ni secretos en errores `500`.
- No incluir credenciales AWS en `.env`; en AWS se usa el rol de instancia.
- La API es same-origin; no se necesita CORS en v1.

---

## 9. Ejemplos de uso

### 9.1 Login con curl

```bash
# 1. Obtener token CSRF y cookie
curl -c cookies.txt http://localhost:8000/api/auth/csrf

# 2. Login (reemplazar <TOKEN>)
curl -b cookies.txt -c cookies.txt \
  -H "Content-Type: application/json" \
  -H "X-CSRFToken: <TOKEN>" \
  -d '{"email":"docente@example.com","password":"Docente1234!"}' \
  http://localhost:8000/api/auth/login
```

### 9.2 Consultar disponibilidad

```bash
curl "http://localhost:8000/api/availability?from=2026-12-01&to=2026-12-07&min_capacity=15"
```

### 9.3 Crear reserva

```bash
curl -b cookies.txt \
  -H "Content-Type: application/json" \
  -H "X-CSRFToken: <TOKEN>" \
  -d '{
    "room_id": 1,
    "start": "2026-12-01T08:00:00-05:00",
    "end": "2026-12-01T10:00:00-05:00",
    "attendees": 12,
    "purpose": "Práctica de redes"
  }' \
  http://localhost:8000/api/reservations
```

### 9.4 Prueba de concurrencia

```bash
# Dos peticiones simultáneas a la misma sala y franja.
# Resultado esperado: [201, 409] y count(*) = 1 en la base de datos.
tests/concurrencia.sh
```

---

## 10. Decisiones pendientes de confirmar

Estas decisiones se proponen para no bloquear la implementación; si el equipo las cambia, se actualiza este documento en el mismo PR:

- **Mover:** se actualiza la misma reserva y se conserva el ID; se registra un evento interno. Alternativa descartada: crear una reserva nueva y marcar la anterior como `moved`.
- **Disponibilidad pública:** no requiere login y no muestra datos personales.
- **Plazo de cancelación:** 2 horas antes para docentes; admin sin límite.
- **Última hora:** 24 horas antes.
- **Borrado de salas:** lógico (`active = false`), nunca físico si tiene reservas.
- **Zona horaria:** `America/Bogota`; el servidor almacena `timestamptz`.
- **Errores de negocio:** 422, no 400.

---

## 11. Checklist para aprobar el PR

- [ ] Todos los endpoints que necesita la UI están documentados.
- [ ] Los códigos HTTP y `error.code` están acordados.
- [ ] El contrato de concurrencia coincide con el entregable #7.
- [ ] `.env.example` no contiene secretos reales.
- [ ] Miguel, Juan Diego y Samuel aprueban el PR.
- [ ] Cualquier cambio posterior se hace en este mismo archivo.
