# Base App — Plataforma SaaS multi-tenant

Sistema base reutilizable para construir aplicaciones B2B SaaS. Pensado como **fundación** para crear verticales específicos (sales, inventory, healthcare, etc.) y vender acceso a múltiples empresas-cliente.

> **Estado**: 453 tests passing, módulos core listos, scaffold `make:module` validado, multi-idioma (es/en).

---

## 3 guías + carpeta `docs/`

Este README es el **mapa general**. Para detalles operativos hay dos guías hermanas:

| Guía | Para qué |
|---|---|
| **README.md** (este) | Qué es el sistema, módulos resumen, links a docs/ |
| **[README-DEV.md](README-DEV.md)** | Setup en PC nueva, comandos del día a día, troubleshooting |
| **[README-PROD.md](README-PROD.md)** | Deploy a DigitalOcean, security, backups, supervisor, monitoring |

Y la carpeta [`docs/`](docs/) tiene profundizaciones técnicas que las 3 guías referencian:

| Documento | Cuándo abrirlo |
|---|---|
| [`docs/USAGE.md`](docs/USAGE.md) | **Manuales por rol** (super/admin/user) + flujo del sistema + cada módulo en detalle |
| [`docs/MANUAL-CLIENTE.md`](docs/MANUAL-CLIENTE.md) | **Manual para el cliente final** (sin jerga técnica) — admin y workers |
| [`docs/CREATE-MODULE.md`](docs/CREATE-MODULE.md) | **Cómo crear módulos nuevos** con `php artisan make:module` |
| [`docs/AUTOMATIONS.md`](docs/AUTOMATIONS.md) | **Cómo crear automatizaciones** (triggers + data sources + acciones) |
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | Por qué se eligió cada tecnología, decisiones de diseño |
| [`docs/STRUCTURE.md`](docs/STRUCTURE.md) | Qué hay en cada carpeta del proyecto |
| [`docs/PERMISSIONS.md`](docs/PERMISSIONS.md) | Spatie, super bypass, jerarquía de roles |
| [`docs/plan-features.md`](docs/plan-features.md) | Matriz completa plan × feature, suscripciones |
| [`docs/CRONS-AND-SETTINGS.md`](docs/CRONS-AND-SETTINGS.md) | 4 capas de cron, 23 settings con su efecto, background tasks |
| [`docs/FRONTEND.md`](docs/FRONTEND.md) | Construir vistas con Inertia + Vue + Antd |
| [`docs/ENV.md`](docs/ENV.md) | Variables de entorno (todas) |
| [`docs/MAIL-SETUP.md`](docs/MAIL-SETUP.md) | **Configurar SMTP** — Gmail App Password, Mailgun, SES, Postmark paso a paso |
| [`docs/INSTALL-TOOLS.md`](docs/INSTALL-TOOLS.md) | Setup de Laragon + Postgres + DBeaver en Windows |
| [`docs/TROUBLESHOOTING.md`](docs/TROUBLESHOOTING.md) | Errores comunes y cómo resolverlos |
| [`docs/DEPLOY.md`](docs/DEPLOY.md) | Complemento técnico del README-PROD.md |
| [`docs/SENTRY.md`](docs/SENTRY.md) | Activar error tracking en producción (feature futura) |
| [`docs/HARDENING.md`](docs/HARDENING.md) | Inventario de fixes de seguridad/bugs aplicados y patrones defensivos |

---

## ¿Qué es este sistema?

Una plataforma donde:

- **Tú** (creador del sistema, rol `super`) administras la plataforma completa — creas workspaces (empresas-cliente), defines planes de suscripción, gestionas el catálogo de idiomas/países/regiones.
- **Cada cliente tuyo** tiene un **workspace** (su empresa). Cada workspace tiene su propio rol `admin` que gestiona usuarios y roles internos.
- **Los empleados del cliente** son usuarios con rol `user` + permisos custom que el admin les asigna (ej. "puede ver clientes pero no eliminarlos").
- Cada workspace paga una **suscripción** a un plan (free / basic / pro / enterprise) que **desbloquea features** (exports a PDF, imports masivos, automatizaciones, API REST).

**Multi-todo**:
- **Multi-tenant**: cada workspace ve solo sus propios datos (aislado por `tenant_id`).
- **Multi-idioma**: UI en español e inglés (es/en).
- **Multi-país**: cada usuario tiene su país + timezone (fechas en su huso local).
- **Multi-permiso**: roles del sistema + roles custom + permisos individuales (Spatie).
- **Multi-plan**: features desbloqueables por tier de suscripción.

Detalle del flujo completo (de la primera vez al uso diario): [`docs/USAGE.md`](docs/USAGE.md).

---

## Stack técnico

| Capa | Tecnología |
|---|---|
| Backend | Laravel 13 · PHP 8.3 |
| BD | PostgreSQL 16 (con `unaccent`) · MySQL 8 funciona con fallback |
| Frontend | Inertia.js v2 · Vue 3 (Composition API) · Vite · Tailwind 4 · Ant Design Vue 4 |
| Auth | Laravel Sanctum · Spatie Permission |
| Queue | `database` driver (sin Redis) |
| Storage | `local` disk (sin S3 — solo logos + fotos perfil + imports) |
| Mail | `log` en dev, SMTP en prod (Gmail / Mailgun / SES / Postmark) |
| Tests | PHPUnit (453 tests, 19 skipped justificados) |

Decisiones de diseño detalladas en [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

---

## Roles del sistema

```
┌─ super ─────────────────────────────────────────────────────────────┐
│  Tú. Creador y dueño de la plataforma. tenant_id = NULL             │
│  Ve TODO cross-workspace. Invisible para los clientes.              │
└─────────────────────────────────────────────────────────────────────┘
                                ↓ administra
┌─ admin ─────────────────────────────────────────────────────────────┐
│  Dueño de un workspace (1 por tenant, obligatorio).                 │
│  Gestiona usuarios, roles custom y permisos DE SU WORKSPACE.        │
└─────────────────────────────────────────────────────────────────────┘
                                ↓ administra
┌─ user (rol base) + roles custom ────────────────────────────────────┐
│  Workers del workspace. Acceso definido por permisos custom         │
│  asignados por el admin (ej. "Sales Editor", "Sales Viewer").       │
└─────────────────────────────────────────────────────────────────────┘
```

Manuales detallados por rol: [`docs/USAGE.md`](docs/USAGE.md#2-manual-del-super).

Detalle técnico de permisos, super bypass, Spatie: [`docs/PERMISSIONS.md`](docs/PERMISSIONS.md).

---

## Módulos del sistema (resumen)

| Grupo | Quién lo usa | Módulos |
|---|---|---|
| **System Management** | Solo super | Workspaces, Planes, Idiomas, Locales, Países, Regiones, Módulos del sistema, Configuración |
| **User Management** | Super + admin | Usuarios, Perfiles (Roles) |
| **Business Management** | Super + admin + workers (con permisos) | Customers + módulos creados con `make:module` |
| **Communication** | Inbox: todos · Mensajes: solo super | Inbox, Mensajes (con editor TipTap) |
| **Automation Management** | Super + admin con plan pro+ | Automatizaciones |
| **System Logs** | Super + admin con feature `audit_log_view` | Audit Logs |

Detalle de cada módulo (URL, propósito, capacidades): [`docs/USAGE.md`](docs/USAGE.md#5-módulos-del-sistema-en-detalle).

---

## Planes y suscripciones (resumen)

4 tiers:
- **free** (1 user, sin features avanzadas)
- **basic** (5 users, exports CSV/Excel, saved views)
- **pro** (25 users, automations, imports, custom roles, exports avanzados)
- **enterprise** (ilimitado + API REST + branded exports + priority support)

**Importante**: NO hay columna `tenants.plan`. El plan se **deriva en runtime** de la suscripción vigente del tenant. Sin suscripción vigente → plan `free` (piso).

Matriz completa plan × feature: [`docs/plan-features.md`](docs/plan-features.md).

---

## Capas de control de acceso

Cada acción pasa por 4 capas en orden:

1. **Rol** (`role:super|admin`) — quién puede entrar al módulo
2. **Plan** (`plan_feature:X`) — qué features tiene el workspace según su suscripción
3. **Permiso** (`permission:X.action`) — qué acción específica permite el rol custom
4. **Tenant** (automático, vía `BelongsToTenant` trait) — solo se ven datos del propio workspace

El rol `super` bypasea las capas 2, 3 y 4. Detalle con ejemplos en [`docs/PERMISSIONS.md`](docs/PERMISSIONS.md) y [`docs/plan-features.md`](docs/plan-features.md).

---

## Background tasks

Exports a Excel/PDF/Word, imports masivos, bulk operations, envío de emails y automation runs corren en **queue** (`php artisan queue:work`).

Hay 4 schedules internos que dispara `php artisan schedule:run` (1 cron del SO cada minuto):
- `app:cleanup-expired-downloads` — cada hora
- `app:purge-soft-deleted` — diario
- `subscriptions:check-expirations` — diario
- `automations:tick` — cada minuto

Detalle de cómo se conectan estas capas y los 23 settings que las controlan: [`docs/CRONS-AND-SETTINGS.md`](docs/CRONS-AND-SETTINGS.md).

---

## Crear un módulo nuevo

```bash
php artisan make:module Product --group=BusinessManagement
```

Genera ~51 archivos clonando el master template **Customers**. Después hay que ajustar columnas del dominio + agregar al sidebar manualmente.

Guía completa con todos los pasos: [`docs/CREATE-MODULE.md`](docs/CREATE-MODULE.md).

---

## API REST

`/api/v1/*` con Sanctum bearer tokens. Plan gating: solo `enterprise` (`plan_feature:api_access`).

Hoy hay un endpoint expuesto como patrón de referencia. Documentación interactiva en `/docs` (Scribe), accesible para super/admin logueados.

---

## Hardening aplicado (2026-05)

Ver [`docs/HARDENING.md`](docs/HARDENING.md) para el detalle completo. Resumen:

- **Cross-tenant IDOR**: `BelongsToTenant` trait fuerza `tenant_id` del actor en cada `creating` event para non-super (defense-in-depth contra mass assignment).
- **XSS stored**: `App\Support\HtmlSanitizer` aplicado en `StoreMessageRequest`, `UpdateMessageRequest`, `StoreReplyRequest` (whitelist de tags + strip de `on*` y `javascript:`).
- **Sanctum tokens**: `abilities` ahora es `required` en `TenantController::createToken` y `AuthController::login` — no más default `['*']`.
- **Password reset**: `throttle:5,1` en `password/email` y `password/reset`.
- **Bulk jobs**: `ShouldBeUnique` con lock por hash(userId+action+ids) → retries del worker no duplican notificaciones ni audit log.
- **Bulk services**: `DB::transaction` en los 30 métodos bulk → rollback parcial si un delete/update falla mid-foreach.
- **Exports masivos**: 33 jobs Excel/PDF/Word pasaron de `->get()` a `->cursor()` con count separado → ~10× menos memoria con datasets grandes.
- **OAuth Google**: nuevo user se crea con `tenant_id=null` (antes tenant 1 hardcoded).
- **Automations**: cron expressions respetan `trigger_config.timezone` (antes corrían en TZ del servidor).
- **AutomationService**: single `save()` en create/update (antes 2 saves, dejaba `next_run_at=null` si el 2do fallaba).
- **Foreign keys**: `customers`, `automations`, `automation_runs` ahora tienen FK constraint en `tenant_id` con `ON DELETE SET NULL` (no más registros huérfanos). Tabla `tenant_countries` eliminada (era código muerto sin modelo ni uso).
- **Email unique**: scope por tenant + `whereNull(deleted_at)` (antes era global).
- **LIKE wildcards**: nuevo `App\Support\LikeQuery::contains()` escapa `%`/`_` del input usuario en 8 modelos.
- **forceDelete**: `lockForUpdate()` dentro de la transacción → cero race entre force-delete + restore concurrente.

---

## Licencia

[MIT](LICENSE) © 2026 Carlos Morales.

Puedes usar, modificar y distribuir el código siempre que mantengas el aviso de copyright y la licencia. Sin garantías — úsalo bajo tu propio riesgo.
