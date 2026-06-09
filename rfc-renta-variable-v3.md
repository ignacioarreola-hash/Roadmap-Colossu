# RFC: Renta Variable / Renta Mínima Garantizada (RMG)

**Estado:** Listo para implementación
**Autor:** Nacho (PM Pod-SaaS) · aportaciones de Alfonso (PRD + review técnico)
**Implementación:** Gabriel (FE), Josue (BE), Dante (Tech Lead), Vicente (Supabase)
**Fecha:** Junio 2026 · **v5**
**Módulo:** Colossu · Cobranza + Contratos
**Confluence:** subir bajo *Módulo de Cobranza* (parent `542277635`, espacio SDKB)

---

## Para quien implementa

Este RFC se lee así:
- **§1–§4**: contexto, objetivos, alcance. Léelos para entender el porqué.
- **§5.1–§5.6**: arquitectura y lógica del módulo. Núcleo conceptual.
- **§5.7–§5.8**: ⭐ **specs de UI para Gabriel**. Componentes, estados, transiciones, layouts.
- **§6**: modelo de datos para Josue/Vicente.
- **§11**: contrato de API para Josue.
- **§15.1**: prototipos JSX que reflejan exactamente lo que hay que construir.

---

## 1. Resumen ejecutivo

Construir un subflujo de **Renta Variable** dentro de Contratos + Cobranza de Colossu, que permita al desarrollador inmobiliario:

1. Configurar contratos como **Fija**, **Mínima garantizada (RMG)** o **Variable pura** desde el wizard.
2. Capturar el plan completo de cobranza del contrato a lo largo de toda su vigencia, incluyendo cambios planificados año por año.
3. Permitir **addendums** que modifiquen reglas después de la firma, sin romper el histórico ya facturado.
4. Cobrar la renta base/RMG como cargo normal mensual.
5. Recibir reportes mensuales de ventas (captura interna o link tokenizado para inquilino).
6. Soportar reportes especiales bajo demanda y auditorías con cargos correctivos.
7. Generar un cargo separado de **Diferencial RV** ligado al reporte de ventas, auditable y trazable.

**Concepto clave (v4):** un contrato no tiene "una regla de cobranza". Tiene un **lifecycle de reglas** con versiones planificadas desde el inicio + addendums posteriores. El motor de cobranza selecciona la versión vigente automáticamente según el periodo de cálculo.

**Cambio operativo clave (v5):** la UI separa con disciplina los **3 universos de información** del contrato:
- **Información General** → fuente única de TODA la configuración (datos del contrato, financieros, cargos recurrentes, moratorios, incrementos). Solo lectura — para editar abre el wizard.
- **Cargos** → tabla plana operativa con cargos individuales (idéntica al dashboard global de Cobranza).
- **Ventas** → pantalla operativa propia con su ciclo de estados, edición inline, quick actions y link tokenizado para inquilino.

No existe ya "Reglas del contrato" como pestaña separada. Su contenido vive en Información General.

---

## 2. Motivación y contexto

### 2.1 Estado actual

Colossu administra contratos retail, oficinas, industrial y terrenos, pero la renta variable no está modelada como flujo propio:

- El OCR detecta `extracted_data.property_rent_type: variable` en contratos como Smart Fit, Inditex y Mammut Pizza, pero el contrato operativo queda como `rent_type: fixed`.
- `billing_setting.details` tiene indicios como `variable_rent_basis: sales`, pero no hay campos normalizados.
- El Estado de Cuenta no distingue "pendiente de reporte" de "pendiente de pago".
- No hay forma de capturar contratos con esquema escalonado.
- No hay distinción entre "regla planificada en contrato" y "addendum posterior" — todo cambio sería tratado como excepción.

### 2.2 Caso validador: Mammut Pizza (Paseo Altozano, Morelia)

| Año | Vigencia | Esquema | Configuración |
|---|---|---|---|
| 1 | Abr 2025 — Mar 2026 | Variable pura | 8% sobre ventas. Sin RMG. |
| 2 | Abr 2026 — Mar 2027 | RMG | $50,000 + 6% si supera |
| 3+ | Abr 2027 — Nov 2029 | RMG | $60,000 + 7% si supera. INPC desde Ene 2029. |

Adicionalmente:
- Mantenimiento = **15% de la Renta Mensual fija**.
- Publicidad = **5% de la Renta Mensual fija**.
- Reportes mensuales dentro de los 5 días hábiles del mes vencido.
- Reportes especiales bajo demanda con 5 días hábiles para entregar.
- Cláusula de auditoría: diferencia >5% entre ventas reportadas y reales → cargo correctivo + honorarios.

Este contrato no admite ser modelado como "una sola regla de cobranza". Es 3 reglas distintas planificadas desde la firma, más posibles addendums futuros.

### 2.3 Riesgos no resueltos

| Riesgo | Manifestación hoy |
|---|---|
| Operativo | El equipo no sabe quién falta de reportar ventas |
| Financiero | Saldos por cobrar pueden mezclarse con montos no confirmados |
| Auditoría | No hay trazabilidad: contrato → ventas → fórmula → aprobación → cargo |
| Versionado | No se puede capturar un esquema escalonado al crear el contrato |
| Histórico | Si una regla cambia, los cargos ya emitidos podrían recalcularse incorrectamente |
| Addendums | No hay flujo nativo para capturar cambios pactados después de la firma |
| UX | Estados de reportes y estados de cargos se confunden, generan errores operativos |

### 2.4 Valor de negocio

- **Mercado:** RMG es el caso más común en retail mexicano. Permite a desarrolladores comerciales (Inditex, Petco, Starbucks, Mammut) operar sin Excel paralelo.
- **Estratégico:** habilita expandir el SaaS a centros comerciales y plazas, segmento donde hoy no podemos competir con Yardi/MRI.
- **Operativo:** elimina conciliación manual mensual; el motor genera, calcula y factura.
- **Diferencial vs. competencia:** capturar un contrato escalonado en menos de 5 minutos.

---

## 3. Objetivos

### 3.1 Objetivos de producto

1. Configurar un contrato RMG/Variable con su esquema completo en menos de 5 minutos después del OCR.
2. Permitir capturar el lifecycle completo de reglas planificadas sin requerir addendums posteriores artificiales.
3. Que el reporte de ventas no bloquee el cobro de la renta base.
4. Que el motor de cobranza genere siempre un cargo de Diferencial RV (con monto $0 si no supera).
5. Que cualquier cargo generado sea auditable: trazable a la versión exacta de regla que lo originó, congelado en snapshot inmutable.
6. Que la regla aplicable a cada periodo se resuelva determinísticamente.
7. Capturar **addendums posteriores** como eventos formales con documento y fecha de firma.
8. **UX operativa: capturar ventas y aprobarlas en menos de 30 segundos por reporte**, sin abrir modales ni perder contexto.

### 3.2 Métricas de éxito

- **Adopción Fase 1:** ≥3 cuentas beta migran al menos 1 contrato a RMG o Variable.
- **Calidad:** <5% de reportes requieren corrección post-aprobación.
- **Operación:** 100% de los diferenciales tienen `source_id` apuntando a reporte válido + `rule_version_id` apuntando a versión exacta.
- **UX:** captura inline + aprobación toma <30 segundos por reporte.
- **Versionado:** ≥80% de contratos retail con escalación se configuran de una sola vez al crear el contrato.
- **Inmutabilidad:** 0% de cargos generados se afectan retroactivamente cuando se agrega un addendum.

---

## 4. Alcance

### 4.1 Dentro de Fase 1

- Configuración del tipo de renta desde el wizard del contrato (Fija / RMG / Variable pura).
- Captura de múltiples versiones planificadas en el wizard.
- Captura de addendums posteriores como eventos formales que generan nuevas versiones.
- Motor de resolución de regla vigente (`get_active_rule`) según `reference_period`.
- Snapshot inmutable de la regla en cada cargo generado.
- Generación mensual de periodos esperados de reporte (tipo `regular`).
- Reportes especiales **on-demand** y de **auditoría** con cargo correctivo.
- Captura de ventas **inline** desde la pestaña Ventas (sin modal).
- **Quick actions** ✓/✕ inline para aprobar/rechazar reportes.
- **Menú ⋯ por reporte** con acciones extra (evidencia, notas, auditoría).
- Link tokenizado público para captura del inquilino.
- Cargo de Diferencial RV siempre se genera (con monto $0 cuando no supera).
- Cargos derivados (`derived_pct_of_rent`).
- Vista de contrato con 4 tabs: Información General | Histórico | Cobranza | Documentos.
- Pestaña Cobranza con 3 sub-tabs: Línea de tiempo | Cargos | Ventas.
- Conciliación de cargos desde el menú ⋯ de cada fila.
- Dashboard KPIs nuevos.

### 4.2 Fuera de Fase 1

- Integración POS/terminales.
- OCR automático de evidencia.
- Reglas por categoría de producto o escalones intra-mes.
- Bandeja central multi-contrato para plazas comerciales.
- Recordatorios automáticos al inquilino.
- Reapertura controlada de periodos cerrados.
- Conciliación bancaria automática.
- Diferenciales parciales / notas de crédito ligadas.
- Prorrateo automático cuando un addendum entra a mitad de mes.

### 4.3 Anti-patterns

- Sub-tab "Reglas del contrato" en Cobranza. **Eliminado en v5** — su contenido vive en Información General.
- Tabs de RV en contratos `scheme = fixed`.
- Cargos manuales de `variable_rent_overage` sin reporte ligado.
- Mezclar estados de reporte (`pending_submission`, `pending_review`) con estados de cargo (`due_soon`, `paid`).
- Modales para captura/aprobación cuando inline es suficiente.
- Replicar info de Información General en Cobranza.
- Permitir modificar una versión cuyo periodo de vigencia ya generó cargos emitidos.
- Crear versiones de regla sin trazabilidad de origen.
- Recalcular cargos ya facturados cuando la regla cambia.

---

## 5. Propuesta detallada

### 5.1 Tres conceptos que deben separarse

| # | Concepto | Naturaleza | Tabla |
|---|---|---|---|
| 1 | **Obligación fija** | Compromiso del contrato. Facturable de inmediato. | `billing_records` con `concept_code = rent_base` |
| 2 | **Reporte operativo** | Input del inquilino (ventas + evidencia). No es saldo. | `sales_report_periods` + `sales_report_submissions` |
| 3 | **Cargo derivado** | Diferencial calculado y aprobado. Factura aparte. | `billing_records` con `concept_code = variable_rent_overage` |

Un reporte pendiente **NO es deuda**. Solo entra a saldos cuando el cargo derivado se aprueba y genera.

### 5.2 Lógica de cálculo

```
renta_calculada_periodo = ventas_netas_periodo × variable_rate_vigente
diferencial             = max(0, renta_calculada_periodo − RMG_vigente)
```

Donde `variable_rate_vigente` y `RMG_vigente` provienen de la versión activa para ese `reference_period` (resolver con motor §5.4.3).

| Tipo | Comportamiento | Fase 1 |
|---|---|---|
| Fija | Monto fijo mensual. Sin variabilidad. | ✅ |
| RMG | `max(RMG, ventas × %)`. RMG siempre se cobra. Si supera, genera Diferencial separado. | ✅ |
| Variable pura | `ventas × %` sin piso. Cargo = diferencial completo (RMG=$0). | ✅ |

**El diferencial siempre se genera** como BillingRecord. Si no supera, monto = $0 y status = `closed_no_overage`.

### 5.3 Workflow del reporte (estados internos y visibles)

```
[1] Sistema genera periodo esperado            status_interno = pending_submission
       ↓                                        status_ventas  = Pendiente
[2] Inquilino reporta ventas o equipo captura  status_interno = pending_review
                                                status_ventas  = Reportado
       ↓
[3] Admin click en ✓ inline                    status_interno = charge_generated o closed_no_overage
                                                status_ventas  = Aprobado
                                                + cargo aparece en pestaña Cargos
       ↓
    (alternativa: click en ✕)                  status_interno = rejected
                                                status_ventas  = Rechazado
                                                → vuelve a Pendiente
```

**Estados internos del periodo** (`sales_report_periods.status`):

| Estado interno | Cuándo |
|---|---|
| `pending_submission` | Esperando captura del inquilino o admin |
| `submitted` | Captura completa, esperando revisión interna |
| `pending_review` | Listo para aprobar/rechazar |
| `approved` | Aprobado, sin generar cargo aún |
| `charge_generated` | Cargo de Diferencial RV emitido |
| `closed_no_overage` | Aprobado sin diferencial (ventas no superan RMG) |
| `rejected` | Admin pidió corrección |
| `cancelled` | Periodo cancelado por addendum |

**Estados visibles en UI de Ventas** (mapeo simplificado, ver §5.8.3):

| Estado visible | Color | Mapea a estados internos |
|---|---|---|
| **Pendiente** | Ámbar `#B26500` | `pending_submission` |
| **Reportado** | Amarillo `#92400E` | `pending_review`, `submitted` |
| **Aprobado** | Verde `#1F8546` | `approved`, `charge_generated`, `closed_no_overage` |
| **Rechazado** | Rojo `#C62828` | `rejected` |

---

### 5.4 ⭐ Arquitectura de versionado de reglas (rule lifecycle)

#### 5.4.1 Conceptos fundamentales

Un contrato tiene **N versiones de regla de cobranza** (`contract_rent_terms`) a lo largo de su vida. Cada versión define el esquema completo (`scheme`, `RMG`, `%`, días para reportar, etc.) y tiene una **vigencia temporal** (`effective_from` / `effective_to`).

Las versiones pueden originarse de dos maneras:

| Origen | Cuándo se crea | `source` | UX de captura |
|---|---|---|---|
| **Planificada** | Al firmar el contrato. Parte del acuerdo original. | `planned_initial` | Wizard de creación (modo "agregar versión futura") |
| **Addendum** | Después de la firma. Cambio pactado entre las partes. | `amendment` | Flujo "Registrar addendum" con documento adjunto |

Ambas viven en `contract_rent_terms`. Su `source` las distingue.

#### 5.4.2 Invariantes del sistema

1. **Cobertura completa:** entre `effective_from` de la primera versión y `effective_to` de la última, no debe haber huecos.
2. **No solape:** dos versiones no pueden tener vigencia que se cruce.
3. **Determinismo:** dado un `reference_period`, existe exactamente una versión activa.
4. **Inmutabilidad de cargos emitidos:** si un cargo ya se generó usando la versión X, conserva su `calculation_snapshot` aunque X después se modifique.
5. **No retroactividad oculta:** modificar una versión cuyo rango incluye periodos con cargos generados está prohibido. Para corregir → addendum + cargos compensatorios.
6. **Cierre de día:** `effective_from` y `effective_to` se manejan al inicio del día (00:00). No mid-day en Fase 1.

#### 5.4.3 Motor de resolución de regla vigente

```python
def get_active_rule(contract_id, reference_period):
    """
    Dado un contrato y un periodo de referencia (ej. "2027-04"),
    devuelve la versión única de regla aplicable.
    """
    rules = ContractRentTerms.objects.filter(
        contract_id=contract_id,
        status='active'
    ).order_by('effective_from')

    for rule in rules:
        starts_on_or_before = rule.effective_from <= reference_period
        ends_after_or_open  = rule.effective_to is None or rule.effective_to >= reference_period
        if starts_on_or_before and ends_after_or_open:
            return rule

    raise NoRuleForPeriodError(contract_id, reference_period)
```

El motor de cobranza llama a esta función al generar cada `sales_report_period` y al recalcular previews. Una vez aprobado el cálculo, la versión queda snapshotteada en el cargo (§5.4.5).

#### 5.4.4 Captura de versiones planificadas (wizard del contrato)

Al crear un contrato, el usuario captura todas las versiones planificadas. UI propuesta para el wizard:

```
┌─ Tipo de renta y vigencia ─────────────────────────────────────────┐
│                                                                    │
│   Versión 1 · planificada                          [editar] [↑] [↓]│
│   ▸ Variable pura · 8%                                             │
│   ▸ Vigencia: 01 Abr 2025 → 31 Mar 2026                            │
│                                                                    │
│   Versión 2 · planificada                          [editar] [↑] [↓]│
│   ▸ RMG $50,000 + 6%                                               │
│   ▸ Vigencia: 01 Abr 2026 → 31 Mar 2027                            │
│                                                                    │
│   Versión 3 · planificada                          [editar] [↑] [↓]│
│   ▸ RMG $60,000 + 7%                                               │
│   ▸ Vigencia: 01 Abr 2027 → 30 Nov 2029                            │
│                                                                    │
│   [ + Agregar versión planificada ]                                │
│                                                                    │
│   ✓ Cobertura completa desde 01 Abr 2025 hasta 30 Nov 2029         │
│   ✓ Sin solapes                                                    │
└────────────────────────────────────────────────────────────────────┘
```

**Validaciones al guardar:** primera versión inicia en o después de fecha de inicio de cobro. Última versión termina en o antes de fin del contrato. Sin huecos. Sin solapes.

#### 5.4.5 Snapshot inmutable en cada cargo

Cuando se genera un cargo derivado (ej. Diferencial RV tras aprobar reporte), el cargo guarda un snapshot completo:

```json
billing_records.calculation_snapshot = {
  "rule_id": "uuid-of-contract_rent_terms",
  "rule_version_number": 2,
  "rule_source": "planned_initial",
  "scheme": "minimum_guaranteed",
  "rmg_amount": 50000,
  "variable_rate": 0.06,
  "sales_amount_captured": 920000,
  "calculated_variable_amount": 55200,
  "calculated_overage_amount": 5200,
  "calculated_at": "2026-05-08T14:30:00Z",
  "approved_by_user_id": "...",
  "approved_at": "2026-05-08T15:12:00Z"
}
```

El cargo nunca cambia aunque la regla se modifique después. Correcciones → cargo complementario (ver §5.6).

#### 5.4.6 Addendums

Después de la firma, las partes pueden pactar cambios. El sistema modela esto con `contract_amendments`:

| Campo | Tipo |
|---|---|
| `id` | uuid |
| `contract_id` | FK |
| `amendment_number` | int |
| `signed_at` | date |
| `effective_from` | date |
| `reason` | text |
| `document_id` | FK nullable → PDF del addendum firmado |
| `created_by` | FK user |

Al registrar un addendum, el sistema:

1. Recorta `effective_to` de la versión vigente al día anterior al `effective_from` del addendum.
2. Crea una nueva versión en `contract_rent_terms` con `source = amendment` y `triggered_by_amendment_id = ID del addendum`.
3. No toca cargos ya emitidos.

Ejemplo: Mammut tiene 3 versiones planificadas. En Sep 2026 se pacta addendum que sube RMG de $50K a $55K a partir de Oct 2026:

```
Antes:
  V1 (planned)  Abr 2025 → Mar 2026 · Variable 8%
  V2 (planned)  Abr 2026 → Mar 2027 · RMG $50K + 6%
  V3 (planned)  Abr 2027 → Nov 2029 · RMG $60K + 7%

Después del addendum:
  V1 (planned)             Abr 2025 → Mar 2026 · Variable 8%
  V2 (planned, recortada)  Abr 2026 → Sep 2026 · RMG $50K + 6%
  V4 (amendment #1)        Oct 2026 → Mar 2027 · RMG $55K + 6%
  V3 (planned)             Abr 2027 → Nov 2029 · RMG $60K + 7%
```

Cargos Abr–Sep 2026 ya generados con V2 conservan su snapshot. Oct 2026 en adelante usa V4.

### 5.5 Cargos derivados (mantenimiento, publicidad como % de renta)

Mammut: Mantenimiento = 15% de Renta, Publicidad = 5% de Renta.

| Tipo de cargo recurrente | Cálculo |
|---|---|
| `fixed_amount` | Monto absoluto fijo |
| `derived_pct_of_rent` | `% × renta_base_del_periodo` |

**Edge case: Variable pura + cargo derivado.** Si la versión vigente es Variable pura (RMG = $0), un derivado daría $0. Estrategias configurables:

- `use_zero` — derivado = $0 durante Variable pura (default).
- `use_minimum_reference` — capturar una "renta mínima de referencia" para cálculo de derivados.
- `use_fallback_amount` — definir un monto fijo aplicable solo durante Variable pura.

### 5.6 Reportes on-demand y auditoría

#### On-demand
Admin crea un `sales_report_period` con `type = on_demand` y `reference_period` específico. Flujo idéntico al regular. No genera cargo automático (el cargo regular ya existe).

#### Auditoría con cargo correctivo
Admin detecta diferencia entre ventas reportadas y reales. Crea `sales_report_period` con `type = audit` ligado al regular original (`parent_period_id`). Al aprobar:
- Si genera diferencial adicional → `billing_record` con `concept_code = variable_rent_audit_adjustment`.
- Si contrato lo permite (Mammut: diferencia >5%) → cargo manual `concept_code = audit_fees`.

R7 + R15: el reporte regular original y su cargo NO se mutan.

---

### 5.7 ⭐ UI: Vista del contrato — Información General (fuente única de verdad)

**Esta es la pestaña principal del contrato. NO se replica en otras pestañas.**

#### 5.7.1 Estructura general

Tab principal con scroll vertical. Header del tab incluye:
- Banda superior: resumen ejecutivo (4 datos clave) con dividers verticales
- 9 secciones consecutivas, cada una con título + subtitle opcional + contenido

#### 5.7.2 Resumen ejecutivo (banda superior)

4 items con dividers verticales. **Tipo de renta y % van primero**:

```
┌────────────────────┬────────────────────┬────────────────────┬────────────────────┐
│ TIPO DE RENTA      │ % SOBRE VENTAS     │ RENTA BASE MENSUAL │ VALOR DEL CONTRATO │
│ Mínima garantizada │ 5%                 │ $180,000           │ $6.91M             │
│ RMG + 5% sobre vtas│ aplicado a vtas net│ MXN · sin IVA      │ 36 meses · 245 d   │
└────────────────────┴────────────────────┴────────────────────┴────────────────────┘
```

#### 5.7.3 Secciones (en orden vertical)

1. **Datos del inquilino** — razón social, RFC (con badge OCR si aplica), tipo de persona, representante legal, contacto operativo.
2. **Espacio rentado** — nombre del espacio, m², tipo, complejo, dirección.
3. **Vigencia** — fecha firma, inicio cobro, fin, plazo en meses, días restantes (con progress bar visual).
4. **Términos financieros del esquema de renta** — Moneda, Tipo de renta (con tag), RMG, % sobre ventas, Días para reportar, Cobro del diferencial.
5. **Cargos recurrentes activos** ⭐ (nueva sección, contenido migrado de la antigua "Reglas del contrato") — tabla con columnas:
   - Concepto (con badge OCR%)
   - Monto (o "variable" si es derivado)
   - Frecuencia (Mensual / Anual)
   - Día de cobro
   - **Origen** (badge): `Contrato` (azul) · `Auto (RV)` (índigo) · `Derivado` (azul) · `Manual` (morado)
   - Texto aclaratorio arriba: *"Para modificarlos, usa Editar contrato arriba"*
6. **Intereses moratorios** ⭐ (nueva sección) — Día de facturación, Tipo de interés, Base, Modo, Desde cuándo, Días de gracia.
7. **Incrementos y garantías** — Tipo (INPC), Periodicidad, Próximo incremento, Depósito, Meses adelantados, Meses de gracia.
8. **Valor proyectado del contrato** — totales calculados.
9. **Documentos clave** — links a documentos asociados.

#### 5.7.4 Acciones en el header de Información General

- Botón principal **"Editar contrato"** → abre el wizard de configuración (`configuracion_cobranza_contrato.jsx`).
- NO hay botón "Editar cobranza" separado. Toda configuración vive en el wizard único.

---

### 5.8 ⭐ UI: Vista del contrato — pestaña Cobranza

#### 5.8.1 Header de Cobranza — KPIs operativos

4 KPIs en banda superior:
- Total cobrado (verde si >0)
- Pendiente por vencer (azul)
- Saldo en mora (rojo si >0)
- Diferenciales por aprobar (ámbar si >0)

**No se repiten** Renta base, Incremento anual, Depósito ni Valor del contrato — viven en Información General.

#### 5.8.2 Sub-tabs de Cobranza

Solo 3:

```
┌─────────────────────┬────────────────┬─────────────────┐
│ 📅 Línea de tiempo  │ 📄 Cargos       │ 💰 Ventas  [N]  │
└─────────────────────┴────────────────┴─────────────────┘
```

El badge `[N]` en Ventas muestra la suma de "Pendientes" + "Por aprobar" cuando >0.

#### 5.8.3 Sub-tab "Línea de tiempo"

Vista agrupada por concepto, columnas = meses. Una fila por concepto:
- Renta
- Mantenimiento (derivado o fijo)
- Estacionamiento u otros recurrentes
- **Diferencial RV** (con tag `AUTO` visible)
- Depósito

Comportamiento del Diferencial RV en su fila:
- Mes con ventas superando RMG → celda índigo con monto
- Mes con ventas no superando → celda gris discreta con `$0`
- Mes pendiente de reporte → celda dashed naranja con `?`
- Mes pendiente de aprobación → celda ámbar con monto preview

#### 5.8.4 Sub-tab "Cargos" — tabla plana

⭐ **Idéntica al dashboard global de Cobranza.** Sin agrupación.

**Columnas (en este orden exacto):**

| # | Columna | Contenido |
|---|---|---|
| 1 | ☐ checkbox | Selección masiva |
| 2 | Espacio | Badge azul pastel redondeado: `Torre Majo Test · P1-01` |
| 3 | Categoría | Texto: Renta / Mantenimiento / Diferencial RV (con tag `AUTO`) / Estacionamiento / Depósito |
| 4 | Fecha vencimiento | `1 jun 2026` |
| 5 | Total | `$208,800 MXN` |
| 6 | Pagado | `$0 MXN` (gris si 0, verde si >0) |
| 7 | Pendiente | `$208,800 MXN` |
| 8 | Cobranza | Progress bar horizontal + `%` debajo (gris 0% / naranja parcial / verde 100%) |
| 9 | Estatus | Badge según `STATUS` (Pagado, Por vencer, En mora, Parcial, etc.) |
| 10 | Acciones | Botón `⋯` |

**Filtros (sub-row arriba de la tabla):**
- Todos · Pendientes · **En mora** (rojo) · Parciales · Pagados
- Dropdown a la derecha: `Categoría: [Todas ▼]` con todas las categorías disponibles

**No aparecen aquí** cargos con `status_interno` = `pending_submission`, `pending_review` o `rejected` — esos viven en Ventas.

**Menú `⋯` por cargo:**
- ✓ Conciliar (pago total) → marca como pagado al 100%
- ◐ Registrar pago parcial → marca al monto capturado
- 📎 Adjuntar comprobante
- 👁 Ver detalle

#### 5.8.5 Sub-tab "Ventas" — pantalla operativa principal de RV

⭐ Es la **pantalla operativa diaria** del módulo. Captura, revisa y aprueba reportes sin abrir modales.

##### Estructura vertical

**1. Banner del link tokenizado** (hasta arriba, prominente):

```
┌─ 🔗 Link de reporte para inquilino · [Tokenizado · Revocable] ───────┐
│                                                                       │
│ Comparte este link con el inquilino para que reporte sus ventas       │
│ mensuales. Pantalla de una sola tarea con evidencia obligatoria.      │
│                                                                       │
│ ┌───────────────────────────────────────────────────────────────────┐ │
│ │ https://app.colossu.com/r/c2024-INDI04-pUbLi-tok3n-xxxx           │ │
│ └───────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│                          [📋 Copiar link] [✉ Email] [🔄 Regenerar]   │
└───────────────────────────────────────────────────────────────────────┘
```

**2. Header con acciones:**
- Título "Reportes de ventas"
- Botones: `⬇ Exportar CSV` · `+ Solicitar reporte especial`

**3. KPIs de ventas** (4 items):
- Ventas reportadas YTD
- Promedio mensual
- Diferenciales generados YTD (con sub: "N periodos superaron RMG")
- Por aprobar (con sub: "Requieren tu revisión")

**4. Filtros por estado:**
- Todos · Pendientes (ámbar) · Por aprobar (ámbar) · Aprobados · Rechazados

**5. Tabla con edición inline + quick actions:**

| Columna | Contenido | Editable inline |
|---|---|---|
| Periodo | `Jun 2026` | No |
| Fecha límite | `2026-07-05` | No |
| Ventas netas | `$4,000,000` o "Capturar ventas →" | ✅ Click → input |
| % aplicado | `5% = $200,000` (monoespacio) | Calculado en vivo |
| RMG vigente | `$180,000` (monoespacio) | No (viene de regla activa) |
| Diferencial | `$20,000 ↗` (índigo) o `$0` (gris) | Calculado en vivo |
| **Estado / acción** | Badge + quick actions ✓/✕ inline | Acciones según estado |
| ⋯ | Menú de acciones extra | — |

##### Comportamientos clave

**Edición inline del monto:**
- Click en celda "Ventas netas" de fila Pendiente → input `<number>` aparece
- Borde del input cambia de color: azul (vacío) → verde (no supera) → índigo (supera)
- Debajo del input: pista visual `Enter guardar · Esc cancelar`
- En vivo, las celdas "% aplicado" y "Diferencial" se actualizan
- `Enter` → guarda, el reporte pasa a "Reportado"
- `Esc` → cancela
- `blur` → cancela (timeout 150ms para permitir click en otros lugares)

**Quick actions en columna Estado** (solo para filas "Reportado"):
- Badge "Reportado" + botón **✓ verde** + botón **✕ blanco con borde rojo**
- Click en ✓ → reporte pasa a "Aprobado" + cargo de Diferencial RV aparece en Cargos automáticamente
- Click en ✕ → reporte vuelve a "Pendiente", monto limpiado, status pasa a `rejected` brevemente y se reactiva

**Menú `⋯` (siempre disponible):**
- 📎 Adjuntar evidencia
- 👁 Ver evidencia adjunta
- 📝 Agregar nota
- 🔗 Ver cargo generado (solo si "Aprobado")
- 🚨 Marcar para auditoría (si "Reportado" o "Aprobado")
- ✉ Recordar al inquilino (solo si "Pendiente")

##### Estado visual del fondo de fila

- "Pendiente" → fondo ámbar muy claro (`rgba(255, 244, 229, .2)`)
- "Reportado" → fondo amarillo muy claro (`rgba(254, 243, 199, .25)`)
- "Aprobado" / "Rechazado" → fondo blanco

##### Transición de estado en frontend al aprobar

1. Click en ✓ inline
2. PATCH al endpoint `/sales-reports/:id/approve`
3. Backend genera el `billing_record` correspondiente con `concept_code = variable_rent_overage`
4. Backend devuelve el periodo actualizado + el nuevo billing_record
5. Frontend actualiza `billing` state: la fila del reporte cambia a "Aprobado", y aparece el nuevo cargo en la lista
6. Si el usuario va a Cargos, ve el cargo nuevo automáticamente

---

## 6. Modelo de datos

### 6.1 `contract_rent_terms` — versiones de regla

| Campo | Tipo | Nota |
|---|---|---|
| `id` | uuid | PK |
| `contract_id` | FK | — |
| `version_number` | int | Único por contrato. Orden cronológico. |
| `source` | enum | `planned_initial`, `amendment`, `migration`, `correction` |
| `triggered_by_amendment_id` | FK nullable | → `contract_amendments` si source=amendment |
| `scheme` | enum | `fixed`, `minimum_guaranteed`, `variable` |
| `minimum_guaranteed_amount` | decimal | $0 si Variable pura, requerido si RMG |
| `variable_rate` | decimal | 0.06 = 6%. Requerido si RMG o Variable |
| `reporting_lag_days` | int | Default 15 |
| `derived_charge_strategy` | enum | `use_zero`, `use_minimum_reference`, `use_fallback_amount` |
| `derived_minimum_reference` | decimal nullable | Si use_minimum_reference |
| `derived_fallback_amount` | decimal nullable | Si use_fallback_amount |
| `effective_from` | date | Inclusivo |
| `effective_to` | date nullable | Inclusivo. Null si es última versión abierta. |
| `status` | enum | `active`, `superseded`, `cancelled` |
| `created_at` | datetime | — |
| `created_by` | FK user | — |

**Reglas de integridad:**
- (R-DB-1) Para un mismo `contract_id`, rangos `[effective_from, effective_to]` de versiones con `status=active` no pueden solaparse.
- (R-DB-2) La unión de rangos activos debe cubrir desde inicio de cobro hasta fin del contrato, sin huecos.
- (R-DB-3) Modificar una versión cuyo rango incluye `reference_period` con cargos emitidos lanza error.

### 6.2 `contract_amendments` — eventos de addendum

| Campo | Tipo |
|---|---|
| `id` | uuid |
| `contract_id` | FK |
| `amendment_number` | int |
| `signed_at` | date |
| `effective_from` | date |
| `reason` | text |
| `document_id` | FK nullable |
| `previous_rule_version_id` | FK | Versión que recortó |
| `new_rule_version_id` | FK | Versión que creó |
| `created_by` | FK user |
| `created_at` | datetime |

### 6.3 `sales_report_periods` — un registro por contrato × periodo

| Campo | Tipo |
|---|---|
| `id` | uuid |
| `contract_id` | FK |
| `rule_version_id` | FK → `contract_rent_terms` |
| `reference_period` | date/month |
| `submission_due_date` | date |
| `type` | enum (`regular`, `on_demand`, `audit`) |
| `parent_period_id` | FK nullable → reporte original si type=audit |
| `status` | enum (8 estados internos, ver §5.3) |
| `expected_base_charge_id` | FK nullable |
| `generated_overage_charge_id` | FK nullable |
| `latest_submission_id` | FK nullable |
| `closed_at` | datetime nullable |
| `created_by` | FK user nullable |

### 6.4 `sales_report_submissions`

| Campo | Tipo |
|---|---|
| `id` | uuid |
| `sales_report_period_id` | FK |
| `sales_amount` | decimal |
| `currency` | enum |
| `submitted_by_type` | enum (`tenant`, `user`, `system`) |
| `submitted_by_name` | string |
| `submitted_by_email` | string nullable |
| `submitted_at` | datetime |
| `declaration_accepted` | bool |
| `notes` | text nullable |

### 6.5 `variable_rent_calculations` — snapshot inmutable

| Campo | Tipo |
|---|---|
| `id` | uuid |
| `sales_report_period_id` | FK |
| `submission_id` | FK |
| `rule_version_id` | FK → `contract_rent_terms` (CRÍTICO) |
| `rule_snapshot_json` | JSON | Copia completa de la regla aplicada |
| `base_amount_snapshot` | decimal |
| `variable_rate_snapshot` | decimal |
| `sales_amount_snapshot` | decimal |
| `calculated_variable_amount` | decimal |
| `calculated_overage_amount` | decimal |
| `formula_version` | string |
| `calculated_at` | datetime |
| `approved_by` | FK nullable |
| `approved_at` | datetime nullable |

### 6.6 `sales_report_files`

| Campo | Tipo |
|---|---|
| `id` | uuid |
| `sales_report_submission_id` | FK |
| `file_id / s3_path` | FK / string |
| `file_type` | string |
| `uploaded_by_type` | enum (`tenant`, `user`) |

### 6.7 `contract_public_links` — token de inquilino

| Campo | Tipo |
|---|---|
| `id` | uuid |
| `contract_id` | FK |
| `token_hash` | string |
| `purpose` | enum (`sales_reporting`) |
| `status` | enum (`active`, `revoked`, `expired`) |
| `expires_at` | datetime nullable |
| `last_used_at` | datetime nullable |

Guardar **hash** del token, no el token plano.

### 6.8 `recurring_charges` — cargos recurrentes del contrato

| Campo | Tipo |
|---|---|
| `id` | uuid |
| `contract_id` | FK |
| `concept` | string |
| `concept_code` | enum (`rent_base`, `maintenance`, `advertising`, `parking`, `other`) |
| `calculation_type` | enum (`fixed_amount`, `derived_pct_of_rent`) |
| `amount` | decimal nullable |
| `pct_of_rent` | decimal nullable |
| `frequency` | enum (`monthly`, `quarterly`, `annual`) |
| `due_day` | int |
| `origin` | enum (`contract`, `manual_recurring`, `auto_variable`) |
| `effective_from` | date |
| `effective_to` | date nullable |
| `active` | bool |

### 6.9 Cambios en `billing_records`

| Campo | Uso |
|---|---|
| `concept_code` | `rent_base`, `variable_rent_overage`, `variable_rent_audit_adjustment`, `audit_fees`, `maintenance`, `advertising`, `deposit`, `other` |
| `space_id` | FK → espacio (para columna "Espacio" en tabla Cargos) |
| `reference_period` | Periodo del cargo |
| `source_type` | `contract`, `sales_report_period`, `manual_adjustment`, `audit` |
| `source_id` | id de origen |
| `parent_billing_record_id` | Liga diferencial ↔ renta base |
| `affects_ar` | bool |
| `rule_version_id` | FK → `contract_rent_terms` |
| `calculation_snapshot` | JSON completo |

---

## 7. Reglas de negocio P0

| # | Regla |
|---|---|
| R1 | La renta fija/RMG se cobra aunque no haya reporte de ventas |
| R2 | El diferencial nunca bloquea la renta base |
| R3 | Los reportes pendientes NO incrementan saldos por cobrar (`affects_ar=false`) |
| R4 | El diferencial se calcula contra la RMG del periodo, no contra lo efectivamente pagado |
| R5 | El porcentaje se congela por periodo al momento de calcular |
| R6 | La RMG puede subir por INPC; el porcentaje no |
| R7 | El cálculo aprobado no se muta — usar cargo complementario |
| R8 | El cargo de diferencial debe tener origen obligatorio en un reporte de ventas |
| R9 | Máximo un diferencial activo por contrato-periodo-tipo |
| R10 | Cualquier override post-aprobación requiere motivo |
| R11 | El cargo de Diferencial RV se genera siempre (monto $0 si no supera) |
| R12 | El IVA se calcula por factura. El depósito siempre va sin IVA |
| R13 | Las versiones de `contract_rent_terms` no pueden solaparse |
| R14 | Los cargos derivados se recalculan automáticamente cuando la renta base cambia por versión o INPC |
| R15 | Reportes `audit` no mutan al `regular` original |
| R16 | Cada periodo de cálculo se resuelve a exactamente una versión de regla |
| R17 | Cargos emitidos son inmutables. La versión usada queda snapshotteada |
| R18 | Una versión cuyo rango incluye periodos con cargos emitidos no se puede modificar |
| R19 | Toda nueva versión post-firma debe estar ligada a un `contract_amendment` |
| R20 | El motor de resolución (`get_active_rule`) es la única fuente para determinar qué regla aplica |

---

## 8. Casos límite

| Caso | Comportamiento |
|---|---|
| No reporta en plazo | Estado `pending_submission` continúa. Alerta. Sin moratorio. |
| Reporta $0 en ventas | Aceptar con evidencia. Diferencial $0 → `closed_no_overage`. |
| Reporte tardío | Capturar y calcular. Marcar tardío. |
| Renta base no pagada | Saldos separados. Diferencial vs RMG completa. |
| Esquema escalonado planificado | Capturar todas las versiones al crear el contrato. Motor selecciona vigente. |
| Addendum que recorta versión planificada | Recortar `effective_to`. Crear versión nueva con `source=amendment`. |
| Hueco temporal entre versiones | Validación bloquea el guardado. |
| Solape entre versiones | Validación bloquea el guardado. |
| Intento de modificar versión con cargos emitidos | Bloqueado. Sugerir crear addendum. |
| Reporte especial bajo demanda | Admin crea con `type=on_demand`. Flujo normal. No genera cargo. |
| Auditoría con diferencia detectada | Admin crea `type=audit`. Si genera diferencial → cargo `variable_rent_audit_adjustment`. |
| Corrección post-factura | Reversal / nota de crédito / cargo complementario. Nunca mutar. |
| Evidencia inválida | Rechazar submission, solicitar corrección. |
| Duplicado por link público | Una submission activa. Nuevas crean revisión. |
| Variable pura sin RMG | Cargo de Diferencial RV = `ventas × %`. No hay cargo de renta base. |
| Cargo derivado durante Variable pura | Aplicar `derived_charge_strategy` configurada. |
| Addendum retroactivo | Permitido solo si no hay cargos emitidos en el rango retroactivo. |

---

## 9. Migración de datos existentes

1. Agregar versión inicial `contract_rent_terms` con `scheme=fixed`, `source=migration` a todos los contratos.
2. Detectar candidatos por OCR.
3. Marcar como `requires_variable_rent_review`. No convertir automáticamente.
4. Para revisados, en el wizard:
   - Capturar versión inicial.
   - Capturar versiones futuras si aplica.
5. Identificar cargos derivados y convertir a `recurring_charges` con `calculation_type=derived_pct_of_rent`.
6. Generar `sales_report_periods` desde el mes de activación.
7. Clasificar cargos existentes por `concept_code`.
8. Excluir placeholders de saldos con `affects_ar=false`.

---

## 10. Impacto en módulos existentes

### 10.1 Wizard del contrato

- Sección "Parámetros de cobro" con Tipo de renta + bloqueos según el tipo.
- Subsección "Plan de versiones" con UI de timeline — botón "+ Agregar versión planificada".
- Sección "Cargos derivados" — toggle "Calcular como % de la renta" para Mantenimiento/Publicidad.
- Sección "Estrategia para cargos derivados durante Variable pura" — visible solo si alguna versión es Variable.

### 10.2 Vista del contrato — estructura definitiva

**4 tabs principales:**

```
┌──────────────────────┬────────────┬──────────┬──────────────┐
│ ⓘ Información General│ 📋 Histórico│ 💰 Cobranza│ 📎 Documentos │
└──────────────────────┴────────────┴──────────┴──────────────┘
```

- **Información General** → fuente única (datos, financiero, cargos recurrentes, moratorios, incrementos). Botón "Editar contrato" arriba.
- **Histórico** → cadena de revisiones. Click en revisión abre detalle. NO duplica info.
- **Cobranza** → 3 sub-tabs:
  - **Línea de tiempo** — agrupada por concepto, columnas = meses
  - **Cargos** — tabla plana idéntica al dashboard global
  - **Ventas** — pantalla operativa con link tokenizado arriba + KPIs + tabla con inline editing + quick actions + menú ⋯
- **Documentos** → repositorio de archivos del contrato.

### 10.3 Dashboard general de Cobranza

- KPIs nuevos: `Pendientes de reporte`, `Diferenciales por aprobar`, `Diferenciales generados YTD`.
- Filtros nuevos: `Tipo de renta`, `Estado de ventas`.

### 10.4 Alta de cargos

- Bloquear `variable_rent_overage` y `variable_rent_audit_adjustment` sin reporte ligado.
- `audit_fees` requiere ligar a periodo de tipo `audit`.

### 10.5 Flujo nuevo: Registrar addendum

Modal con:
- Fecha de firma
- Fecha efectiva (`effective_from`)
- Razón / descripción
- Documento PDF
- Nueva regla (mismo formulario que en el wizard)
- Preview: muestra cómo queda el lifecycle antes de confirmar
- Botón "Registrar y aplicar"

### 10.6 OCR / ContractsAI

- Mapear `extracted_data.property_rent_type` → `scheme`.
- Detectar `%`, RMG por año.
- Detectar esquemas escalonados (Año 1, Año 2) y proponer versiones planificadas.
- Detectar cargos derivados.

---

## 11. API mínima Fase 1

```
# Configuración de regla y versiones
GET    /contracts/:id/rent-terms                      ← lista de versiones
POST   /contracts/:id/rent-terms                      ← agregar versión planificada (solo al crear)
PATCH  /contracts/:id/rent-terms/:vid                 ← editar versión sin cargos emitidos
DELETE /contracts/:id/rent-terms/:vid                 ← eliminar versión planificada futura

# Addendums
POST   /contracts/:id/amendments                      ← registrar addendum (crea versión nueva)
GET    /contracts/:id/amendments                      ← histórico

# Reportes de ventas
GET    /contracts/:id/sales-reports
POST   /contracts/:id/sales-reports/:period/capture   ← captura inline el monto, pasa a "Reportado"
POST   /sales-reports/:id/approve                     ← quick action ✓, genera cargo
POST   /sales-reports/:id/reject                      ← quick action ✕

POST   /contracts/:id/sales-reports/on-demand
POST   /contracts/:id/sales-reports/audit

# Link público
POST   /contracts/:id/sales-report-link               ← genera token
GET    /public/sales-report/:token                    ← inquilino ve form
POST   /public/sales-report/:token                    ← inquilino reporta
POST   /sales-reports/:id/files/presigned-url         ← subir evidencia (desde menú ⋯)

# Cargos recurrentes
GET    /contracts/:id/recurring-charges
POST   /contracts/:id/recurring-charges
PATCH  /recurring-charges/:id
DELETE /recurring-charges/:id

# Cargos: conciliar desde menú ⋯
POST   /billing-records/:id/conciliate                ← marca como pagado total
POST   /billing-records/:id/partial-payment           ← registrar pago parcial

# Motor de resolución (interno + debug)
GET    /contracts/:id/active-rule?period=YYYY-MM
```

---

## 12. Fases de implementación

### Fase 1 — MVP (alcance de este RFC)

- Configuración versionada con versiones planificadas + addendums.
- Tipos: Fija + RMG + Variable pura.
- Motor de resolución y snapshot inmutable.
- Captura inline de ventas + quick actions ✓/✕.
- Link público tokenizado.
- Workflow capturar → aprobar → generar cargo.
- Diferencial siempre generado.
- Reportes on-demand y auditoría.
- Cargos derivados.
- Vista del contrato: 4 tabs + 3 sub-tabs en Cobranza.
- Conciliación desde menú ⋯ en Cargos.
- Dashboard KPIs.

### Fase 2 — Operación avanzada

- Recordatorios automáticos al inquilino.
- Prorrateo automático mid-month en addendums.
- Reapertura controlada de periodos cerrados.
- Diferenciales parciales / notas de crédito vinculadas.
- OCR de evidencia automática.
- Bandeja multi-contrato para plazas.
- Export CSV/Excel completo.

### Fase 3 — Integraciones

- Integración POS.
- Reglas por categoría / escalones intra-mes.
- Portal completo de inquilino.
- Conciliación bancaria automática.

---

## 13. Riesgos y mitigaciones

| Riesgo | Mitigación |
|---|---|
| Reglas mal versionadas con huecos o solapes | Validaciones de integridad (R-DB-1, R-DB-2). Tests E2E. |
| Cargos emitidos modificados retroactivamente | R17 + R18 + snapshot inmutable. |
| Motor de resolución inconsistente entre módulos | R20: única función pública `get_active_rule`. Tests unitarios exhaustivos. |
| Addendum mal capturado borra histórico | Recortar `effective_to` en lugar de borrar. Status `superseded` conserva la fila. |
| Cargos derivados durante Variable pura sin estrategia | `derived_charge_strategy` obligatoria en config. Default `use_zero`. |
| Replicación de info entre tabs | Información General es fuente única. |
| Confundir estado de reporte con estado de cargo | UI separa universos: Ventas tiene 4 estados propios, Cargos no muestra reportes en proceso. |
| Captura lenta de ventas | Inline editing + quick actions sin modal. Objetivo <30s por reporte. |
| Conciliación enterrada en sub-menús | Acción `⋯ → Conciliar` accesible desde cada fila de Cargos. |

---

## 14. Preguntas abiertas

1. ~~Variable pura en Fase 1~~ → ✅ Resuelto: dentro.
2. ~~Múltiples versiones futuras en wizard~~ → ✅ Resuelto: dentro.
3. ~~Reportes on-demand y auditoría~~ → ✅ Resuelto: dentro.
4. **Para Dante:** ¿`concept_code` campo nuevo en `billing_records` o enum existente?
5. **Para Tatiana:** ¿texto LFPDPPP en checkbox del link público?
6. **Para Vicente:** ¿`calculation_snapshot` como JSON completo o solo campos críticos?
7. ~~Cargos derivados durante Variable pura~~ → ✅ Resuelto: `derived_charge_strategy` configurable.
8. **Para Adrián:** ¿cobertura mínima de tests E2E con 8 estados internos × 3 tipos de reporte × N versiones?
9. **Para Dante/Vicente:** ¿`audit_period.parent_period_id` 1:1 o N:1?
10. **Para Alfonso/Dante:** cuando un addendum extiende su `effective_to` más allá de versiones planificadas futuras, ¿cancelamos automáticamente o pedimos confirmación explícita?
11. **Para Dante:** el endpoint `GET /contracts/:id/active-rule?period=YYYY-MM` ¿se expone público o queda interno?
12. **Para Tatiana / Legal:** documento PDF del addendum, ¿requiere firma electrónica certificada o basta archivo subido + audit log?

---

## 15. Anexos

### 15.1 Prototipos JSX (entregables para Gabriel)

- **`contrato_cobranza_v5.jsx`** — Vista del contrato completa. Implementa toda la §5.7 y §5.8. Incluye:
  - 4 tabs principales (Información General, Histórico, Cobranza, Documentos)
  - Información General con resumen ejecutivo + 9 secciones consolidadas
  - Cobranza con 3 sub-tabs y todos los comportamientos descritos
  - Edición inline + quick actions + menú ⋯ en Ventas
  - Conciliación desde menú ⋯ en Cargos
  - Datos mock que cubren todos los estados visuales
- **`configuracion_cobranza_contrato.jsx`** — Modal "Editar contrato" sub-tab "Condiciones de pago". Implementa los campos básicos del wizard (Tipo de renta, RMG, %, días para reportar, mantenimiento, moratorios, incrementos, depósito, gracia).

### 15.2 Documentos de referencia

- `renta_variable_modulo_prd.md` — PRD inicial de Alfonso.
- `colosso_cobranza_renta_variable_review.md` — Review técnico.
- **`PIZZAS_MAMMUT_-_Juan_Pablo-135.pdf`** — Contrato real validador del esquema escalonado y cargos derivados.

### 15.3 Caso validador: Mammut Pizza — configuración completa esperada

**Versiones planificadas (`contract_rent_terms`, `source=planned_initial`):**

| V# | Vigencia | scheme | RMG | % | Reporting days |
|---|---|---|---|---|---|
| 1 | 2025-04-01 → 2026-03-31 | `variable` | $0 | 8% | 5 (días hábiles) |
| 2 | 2026-04-01 → 2027-03-31 | `minimum_guaranteed` | $50,000 | 6% | 5 |
| 3 | 2027-04-01 → 2029-11-30 | `minimum_guaranteed` | $60,000 | 7% | 5 |

`derived_charge_strategy` = `use_minimum_reference` con valor `$50,000`.

**`recurring_charges`:**

| Concepto | calculation_type | Valor |
|---|---|---|
| Renta | (auto desde `contract_rent_terms`) | — |
| Mantenimiento | `derived_pct_of_rent` | 15% |
| Publicidad | `derived_pct_of_rent` | 5% |
| Depósito | `fixed_amount` (one-time) | $120,000 |

### 15.4 Glosario

| Término | Significado |
|---|---|
| RMG | Renta Mínima Garantizada |
| Diferencial | Cargo separado cuando ventas × % > RMG |
| Variable pura | Renta = ventas × % sin piso |
| Versión planificada | Regla capturada al crear el contrato. `source=planned_initial`. |
| Addendum | Cambio pactado después de la firma. Crea versión con `source=amendment`. |
| Lifecycle de reglas | Secuencia de versiones de regla a lo largo del contrato |
| Snapshot | Copia inmutable de la regla al momento del cálculo |
| Motor de resolución | Función `get_active_rule` que devuelve la versión vigente para un periodo |
| Cargo derivado | Cargo recurrente como % de otro concepto |
| Reporte on-demand | Reporte fuera del ciclo regular |
| Reporte de auditoría | Reporte que valida un periodo regular y puede generar cargo correctivo |
| Quick action | Botón inline en tabla (✓/✕) que ejecuta acción sin abrir modal |
| Menú ⋯ | Menú contextual por fila con acciones extra (evidencia, notas, auditoría) |

---

## 16. Changelog

| Versión | Fecha | Cambios |
|---|---|---|
| v1 | May 2026 | Inicial. "Excedente". Sin workflow de aprobación. |
| v2 | Jun 2026 | "Diferencial". Workflow capturar → aprobar. Link tokenizado. Variable pura por feature flag. |
| v3 | Jun 2026 | Variable pura activa. Múltiples versiones futuras. Reportes on-demand y auditoría. Cargos derivados. Caso Mammut documentado. |
| v4 | Jun 2026 | Rule lifecycle como ciudadano de primera clase. Tabla `contract_amendments`. Motor `get_active_rule`. Snapshot inmutable. Reglas R16–R20. |
| **v5 (este RFC)** | **Jun 2026** | **Specs de UI consolidadas para implementación.** Información General como fuente única (absorbe contenido de la antigua "Reglas del contrato"). Cobranza con 3 sub-tabs (eliminada "Reglas del contrato"). Cargos como tabla plana idéntica al dashboard global con columnas Espacio + Categoría. Ventas con estados únicos (Pendiente/Reportado/Aprobado/Rechazado), edición inline, quick actions ✓/✕, menú ⋯ con acciones extra. Link tokenizado movido a Ventas hasta arriba. Sin modal de aprobación. Conciliación desde menú ⋯ en Cargos. Listo para Gabriel. |
