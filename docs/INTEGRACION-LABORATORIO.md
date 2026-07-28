# Integración con el laboratorio (TR LAB)

← [Índice del brain](brain/00-INDICE.md)

> Qué tiene que construir TRAFODEX para recibir resultados y los informes
> firmados del sistema de laboratorio.
>
> Documento espejo, con el plan completo del lado del laboratorio:
> `labo/docs/migracion/04-INTEGRACION-TRAFODEX.md`.
>
> **Estado: DISEÑO.** Nada de esto está implementado todavía. Corresponde a la
> fase 7 del plan de migración del laboratorio.

---

## 1. Contexto

El laboratorio de análisis de aceite (hoy una aplicación Rails de 2019, en
migración a Laravel) es el que genera las muestras que TRAFODEX diagnostica.

**Cómo funciona hoy**: el sistema del laboratorio comparte **14 tablas** con la
base `tr_app_development`, por dos vías: cinco modelos con
`establish_connection(:primary2)` explícito, y otros diez que heredan de una
clase abstracta `Primary2` sin ninguna marca visible (`Customer`, `OilType`,
`Mark`, `Country`, `ConmutationType`, `CustomerLocation`, `CustomerArea`,
`CustomerSubstation`, `ChromatographicalDuval`, `ChromatographicalDgaDiag`).

O sea: el laboratorio **no tiene clientes ni catálogos propios**. Los lee y los
escribe en la otra base.

> **`tr_app_development` es el TRAPP viejo en Ruby, no este proyecto.**
> Tabla `physicals` (acá es `fiquis`), columnas `num_hid`/`num_oxi` (acá `h2`/`o2`),
> `date_rehearsal` (acá `sample_date`), `deleted` entero (acá SoftDeletes), y
> MySQL contra PostgreSQL. Los datos de esa base ya se migraron acá con los
> `Legacy*Seeder`.
>
> **Implicancia para el despliegue de TrafoDex**: el día que este sistema
> reemplace al TRAPP viejo en producción, el laboratorio deja de funcionar —
> no pierde una integración, pierde sus clientes y sus catálogos. El desacople
> del laboratorio (su fase 1) tiene que estar hecho **antes** de ese corte.
> Vale tenerlo presente al planificar el deploy del droplet.

Riesgos que eso implica para TRAFODEX, todos reales:

- **Duplicados**: el asistente de envío no tiene idempotencia; reejecutarlo
  inserta las mismas muestras otra vez.
- **Muestras en el transformador equivocado**: empareja por número de serie con
  `find_by`, que devuelve el primero si hay repetidos.
- **Diagnóstico desactualizado**: al insertar por SQL no corre
  `HealthIndexService::evaluate`, así que el índice de salud y la caché de
  flota quedan viejos hasta que alguien ejecute `diagnose:fleet-cache`.
- **Acoplamiento de esquema**: cualquier `ALTER TABLE` de este lado puede
  romper el laboratorio sin aviso. El nombre de base `tr_app_development` está
  escrito en el código de ellos.

**Cómo va a funcionar**: API REST bajo `/api/v1`, con Sanctum y abilities, la
misma que ya usa `CustomerApiController` como patrón de referencia.

---

## 2. Reparto de responsabilidades

TR LAB emite el informe de ensayo: dice si el aceite cumple el criterio de
aceptación (IEEE C57.106, IEC 61203, etc.).

TRAFODEX diagnostica el equipo: DGAF, Duval, IEEE C57.104-2019, furanos,
fisicoquímico, factor de potencia, índice de salud, tendencias y flota.

**El motor de diagnóstico no se duplica del lado del laboratorio.** Está
acordado en su plan de migración (`05-REUSO-DEL-CORE.md` §4) y conviene
sostenerlo: dos motores garantizan dos números distintos para la misma muestra.
Si el informe de laboratorio necesita mostrar el índice de salud, lo pide por
la API.

---

## 3. Qué hay que construir de este lado

### 3.1 Controlador y rutas

`app/Http/Controllers/Api/V1/LabResultApiController.php`, clonando la
estructura de `CustomerApiController`.

```php
Route::prefix('v1')
    ->middleware(['auth:sanctum', 'throttle:api', 'plan_feature:lab_integration'])
    ->group(function () {
        Route::middleware('ability:transformers:read')->group(function () {
            Route::get('transformers/lookup', [LabResultApiController::class, 'lookup']);
        });
        Route::middleware('ability:lab:write')->group(function () {
            Route::post('lab-results',                    [LabResultApiController::class, 'store']);
            Route::post('lab-results/{id}/documents',     [LabResultApiController::class, 'attachDocument']);
        });
        Route::middleware('ability:transformers:write')->group(function () {
            Route::post('transformers', [LabResultApiController::class, 'storeTransformer']);
        });
    });
```

Abilities nuevas: `lab:write`, `transformers:read`, `transformers:write`.

### 3.2 Idempotencia

Tabla nueva:

```
idempotency_keys   id, key (uuid, único junto con token_id), token_id,
                   endpoint, request_hash, response_status, response_body json,
                   created_at, expires_at
```

`POST /api/v1/lab-results` **exige** la cabecera `Idempotency-Key`. Si la clave
ya existe para ese token, devuelve `200` con la respuesta guardada, sin crear
nada. Si existe con otro `request_hash`, devuelve `409`.

Purga por `expires_at` (30 días), enganchada en `config/purge.php`.

Esto es lo único que impide que un reintento del laboratorio duplique muestras.
Es requisito de la puerta de salida de su fase 7.

### 3.3 Ingreso de resultados

```http
POST /api/v1/lab-results
Idempotency-Key: 6b1f...
```

El cuerpo trae el transformador (slug, o serie + tag + cliente), los datos del
informe de laboratorio, y una lista de `tests` con `kind`
(`chromatography` | `physicochemical` | `furanos` | `power_factor`),
`measured_at` y `values` **por código de analito** (`h2`, `ch4`, `acid`, `rig`,
`ten`, `wat`, `pot`, `fal`…), más `methods` opcional con la norma y las
condiciones de ensayo.

El controlador:

1. Resuelve el transformador. Si es ambiguo o no existe → `422` con el detalle;
   **no** crea nada ni adivina.
2. Mapea códigos de analito a las columnas de `chromatographicals` / `fiquis` /
   `furanos` / `fpots`. El mapa vive en `config/lab_integration.php`, no en el
   controlador.
3. Inserta en transacción, con `tenant_id` tomado del token.
4. Ejecuta el mismo recálculo que la UI: `HealthIndexService::evaluate()` y la
   caché de flota (`fault_type`, `gassing_rate`, `paper_dp`, `paper_life_years`,
   `ieee_condition`).
5. Devuelve `201` con los ids creados, el DGAF de la muestra y el índice de
   salud actualizado del transformador.

> **`methods` importa.** El `CLAUDE.md` de este proyecto tiene anotado que los
> umbrales de `rig` vienen del Ruby viejo sin registro del gap y están
> rotulados 2.0 mm, pendiente de confirmar con el laboratorio. A partir de esta
> integración el gap llega explícito por muestra: guardarlo (columna
> `rig_method` / `pot_method` o un JSON `methods` en la muestra) cierra ese
> pendiente hacia adelante. Lo histórico sigue como está.

### 3.4 Documentos del laboratorio

Tabla nueva, polimórfica:

```
sample_documents   id, slug, tenant_id
                   documentable_type, documentable_id   (chromatographical | fiqui | furano | fpot)
                   kind (lab_report), file_path, file_name, mime, size, sha256
                   report_number, issued_at, source ('lab'), verify_url
                   created_by, timestamps, softDeletes
```

Registrar en `config/polymorphic.php` y en `config/purge.php`.

Se recibe **solo el PDF emitido y firmado**, nunca el Word editable ni el
borrador: el Word no puede sostener la promesa de autenticidad, criterio ya
establecido para el informe editable propio.

Se muestra en la ficha del transformador como "Informe de laboratorio", con el
número, la fecha de emisión y el enlace de verificación del laboratorio.

### 3.5 Búsqueda de transformador

```http
GET /api/v1/transformers/lookup?serial=...&tag=...&customer=...
```

Devuelve `[]`, uno o varios candidatos con slug, serie, tag, cliente y
subestación. Nunca elige por el laboratorio: la desambiguación es de ellos, con
una persona.

### 3.6 Feature de plan

`lab_integration` en `config/features.php`, tier `enterprise` (junto con
`api_access`). Un workspace sin la feature recibe `402` en estos endpoints.

---

## 4. Lo que NO cambia

- Los motores de diagnóstico, sus datos y su verificación. La API es una
  puerta de entrada; el diagnóstico corre igual que si la muestra se hubiera
  cargado por la interfaz.
- El módulo Customers y su API, que sigue siendo el patrón de referencia.
- El flujo de aprobación y firmas de los informes de TRAFODEX. El informe del
  laboratorio es un adjunto, no reemplaza al informe consolidado.

---

## 5. Retorno hacia el laboratorio — fuera de alcance por ahora

Un webhook que avise al laboratorio cuando TRAFODEX recalcula el índice de
salud sería útil, pero los webhooks están cerrados como funcionalidad premium
futura (ver [backlog](brain/backlog-decisiones.md)). No se adelantan por esta
integración. Si el laboratorio necesita el índice de salud, lo consulta.

---

## 6. Resumen de cambios

| # | Cambio | Tipo |
|---|---|---|
| 1 | `LabResultApiController` + rutas en `routes/api.php` | código |
| 2 | Abilities `lab:write`, `transformers:read`, `transformers:write` | código |
| 3 | Tabla `idempotency_keys` + middleware | migración + código |
| 4 | Tabla `sample_documents` polimórfica | migración |
| 5 | `config/lab_integration.php` (mapa analito → columna) | config |
| 6 | Guardar el método/gap que informa el laboratorio | migración |
| 7 | Feature `lab_integration` en planes | config + seeder |
| 8 | Ficha del transformador: sección "Informes de laboratorio" | Vue |
| 9 | Purga y polimórficos registrados | config |
