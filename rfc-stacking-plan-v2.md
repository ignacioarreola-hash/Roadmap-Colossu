# [RFC][WIP][SWARCH] Stacking Plan interactivo — vista espacial del Activo (Colossu)

**Authors:**

- @nacho (Product — Pod-SaaS)

**Insumos:** PRD-completo y PRD-mini de @alfonso · Prototipos de producto v1–v6 (referencia visual congelada: `stacking_plan_v6_navegable.jsx`)

---

## 1 Executive Summary

Proponemos una **tercera pestaña en el detalle del Activo** (junto a `Información` y `Espacios`) que convierte el plano de cada piso en un **mapa espacial 2D navegable**: el cliente entra, ve sus locales bien geometrizados sobre su propio plano, y **se desliza por él** (pan + zoom) como en un lienzo tipo Miro. Cada local es un polígono coloreado por ocupación; al hacer clic se abre su resumen ejecutivo. El usuario es el **cliente de Colossu** (desarrollador / administrador de portafolio).

El producto se apoya en tres pilares:

1. **El mapa es el protagonista.** No es un tablero con filtros: es una vista espacial que se explora. Una sola lectura — **Ocupación** — para que el foco esté en navegar y entender el inmueble, no en configurar. (Lentes adicionales y línea de tiempo se posponen; ver §6.)
2. **Geometrización autogenerable por IA.** La promesa al cliente es *"sube tu plano y ve tus locales mapeados"*, sin trazado manual. La detección y vectorización de los locales a partir de la imagen del plano la hace un proceso de IA; el trazado a mano es el fallback, no el camino principal. **Este es el corazón del proyecto** y lo que lo hace escalable a cientos de activos.
3. **Base para la capa propietaria.** La geometría limpia habilita, más adelante, cruzar el mapa con **Cobranza** (renta variable / RMG) — un diferenciador que ningún competidor modela porque la renta variable mexicana es específica de nuestro mercado. Se deja como fase posterior para no bloquear el arranque.

Se construye de forma **aditiva**: no toca `Información`, `Espacios` ni la lógica de contratos.

## 2 Motivation

Hoy la capa espacial del inmueble vive fuera de Colossu. El detalle del Activo es tabla (`Espacios`) y ficha (`Información`); no hay forma de ver el plano, representar cada local con su forma real, ni recorrer el inmueble. Los planos viven como archivos sueltos (PDF/DWG/JPG) fuera del sistema.

El valor para el cliente B2B:

- **Reconocimiento inmediato:** el cliente ve *su* inmueble como lo conoce — su plano, sus formas — no una cuadrícula abstracta. Eso genera confianza en que el sistema "tiene" su realidad.
- **Eficiencia operativa:** leer el estado de un piso completo deslizándose por el mapa, en vez de cruzar filas de una tabla.
- **Escalabilidad de la promesa:** si geometrizar dependiera de trazado manual, no escala más allá de una demo. La autogeneración por IA es lo que vuelve esto un producto y no un servicio.
- **Tesis estratégica de Spot2:** materializa "sistema operativo indispensable", no "repositorio de contratos", y sienta la base para cruzar lo espacial con lo financiero (Cobranza).

## 3 Proposed Implementation

> **Nota de alcance.** Este RFC define el **qué**, las **consideraciones técnicas** (arquitectura, modelo de datos, stack candidato) y, donde aplica, **recomendaciones de enfoque**. Las recomendaciones son eso — un punto de partida para la discusión, no una instrucción. El **cómo** es decisión del equipo.

### 3.1 Principio arquitectónico (no negociable)

Tres capas estrictamente independientes:

```
Geometría   →   Espacio (Asset)   →   Contrato
 (forma)         (identidad)          (lo comercial)
```

- El **contrato se ancla al `asset_id`**, nunca a la geometría. La forma puede cambiar, regenerarse o rehacerse sin afectar el contrato.
- La **geometría es un atributo opcional** del espacio. Un espacio con `geometry: null` es válido (existe sin estar dibujado).
- Cuando un espacio cambia de forma de manera estructural (dividir/juntar), **no se muta**: se archiva y nace uno nuevo con su `origin` (linaje).

Este principio es lo que permite que la **IA regenere o ajuste geometría sin riesgo**: como el contrato nunca vive en la forma, una re-detección de la IA no puede romper nada comercial.

### 3.2 La pieza central — geometrización autogenerable por IA

El flujo que proponemos como experiencia principal:

```
Cliente sube plano (PDF/JPG/DWG)
        │
        ▼
  Proceso de IA detecta locales  ──►  propone polígonos + atributos (área, etiqueta tentativa)
        │
        ▼
  Vista de revisión: el cliente confirma / corrige / fusiona
        │
        ▼
  Geometría persistida (coords normalizadas 0–1) ──► vinculada a cada asset_id
```

**Lo que recomendaría (no cómo hacerlo):**

- Tratar la salida de la IA como **propuesta, no verdad**. El paso de revisión humana no es opcional: la IA acelera, el cliente (o Supply en onboarding white-glove) valida. Esto protege la confianza y evita que un mal trazado contamine datos.
- Diseñar el proceso para que sea **idempotente y re-ejecutable**: volver a correr la detección sobre un plano actualizado no debería destruir las confirmaciones previas ni romper vínculos a contratos (de ahí la capa de linaje de §3.1).
- Empezar por la **forma más simple que ya da valor** — detección de locales como regiones cerradas sobre el plano — antes de perseguir reconocimiento fino (texto, cotas, mobiliario). La barra de entrada es "el cliente reconoce sus locales", no "CAD perfecto".
- Guardar **siempre el plano original** como capa de fondo y la geometría como capa encima; nunca "quemar" la detección sobre la imagen. Así el cliente puede recortar, reintentar o ajustar sin perder el insumo.
- Mantener un **fallback de trazado manual** (arrastrar polígonos) para planos que la IA no resuelva bien. La IA es el camino principal; el trazado manual, la red de seguridad.

La elección de modelo/servicio de visión, el pipeline y el dónde corre (síncrono vs. trabajo en background) son decisión del equipo. Solo recomendaría que la latencia de la detección **no bloquee** la carga del plano: el cliente sube, ve su plano de inmediato, y los polígonos aparecen cuando la IA termina.

### 3.3 Modelo de datos (aditivo, sin entidades nuevas en Fase 1)

**`assets.geometry`** (JSON, nullable) — polígono en coordenadas **normalizadas 0–1** respecto a la imagen del piso, con metadata de procedencia:

```json
{
  "id": "geo_8f2a",
  "version": 1,
  "shape": "polygon",
  "points": [[0.10,0.12],[0.45,0.12],[0.45,0.40],[0.30,0.40],[0.30,0.55],[0.10,0.55]],
  "source": "ai_detected",        // ai_detected | ai_confirmed | manual
  "confidence": 0.86,             // si viene de IA, para priorizar revisión
  "created_at": "2026-06-03T12:00:00Z"
}
```

> El campo `source`/`confidence` es lo que hace operable la autogeneración: permite ordenar la cola de revisión por lo que la IA detectó con menor certeza, y distinguir lo confirmado de lo propuesto.

**`assets.origin`** (JSON, nullable) — linaje: `{ "op": "split"|"merge"|null, "parent_ids": [...] }`.

**`floors`** — `plan_image` (referencia a la imagen del plano, reusando la infra de imágenes existente) y `scale` (opcional, m² del plano para derivar áreas).

Decisión: la geometría se guarda como **atributo del Asset**, pero diseñada como objeto autocontenido (`id`, `version`, `source`, `created_at`) para poder migrar a tabla versionada sin rediseño si aparece un requisito de versionado auditado.

### 3.4 La vista — el mapa como protagonista

- **Una sola lectura: Ocupación.** Polígonos coloreados por estado + urgencia de vencimiento (ocupado / vence 4–9m / vence ≤3m / vacante). Sin selector de lentes ni línea de tiempo en esta versión: el foco es la navegación, no la configuración.
- **Lienzo navegable.** Pan (arrastrar), zoom (rueda / pinch / botones) centrado en el cursor, ajuste a pantalla. El plano se explora; no cabe forzado en un recuadro fijo.
- **Interacción limpia.** Hover resalta el local; clic abre el panel de detalle. Arrastrar mueve el mapa sin disparar selección.
- **Formas reales, no cajas.** El render respeta la geometría detectada (polígonos en L, ángulos, pasillos), porque la fidelidad de la forma es lo que hace que el cliente reconozca su inmueble.

### 3.5 Panel de detalle (resumen ejecutivo del local)

Al hacer clic: **Giro/Estado**, **Contrato** (inicio + vencimiento), **Salud de cobranza** (estado de pago), y **Renta** — que se adapta: si tiene renta variable, desglosa RMG, ventas, variable, cobro = `max(RMG, variable)` y excedente; si es fija o vacante, el estado correspondiente. El botón **"Abrir expediente del local"** navega a la ficha 360° existente. El mapa es para entender de un vistazo; el expediente, para operar.

### 3.6 Stack candidato (a validar por el equipo contra el repo)

| Pieza | Candidato | Por qué |
| :-- | :-- | :-- |
| Lienzo navegable, pan/zoom, hover, selección | **Konva.js** | La experiencia de "deslizarse por el plano" con sombras y decenas de polígonos **excede lo que SVG sostiene con fluidez**; Konva (canvas) está hecho para esto, es crisp a cualquier zoom y escala a miles de shapes. El giro de mapa-protagonista convierte esto de *opción* a *requisito*. |
| Corte / unión de polígonos (fase de edición) | **turf.js** | Split/merge reales de formas irregulares. |
| Detección de geometría | *(decisión del equipo)* | Servicio/modelo de visión sobre la imagen del plano. Recomendación de enfoque en §3.2, no de herramienta. |
| Coordenadas | **Normalizadas 0–1** | Independientes de resolución; la geometría sigue válida si cambia la imagen o el zoom. |

> El prototipo de producto (`v6_navegable.jsx`) está en SVG con transform manual solo para validar UX. La versión navegable de producción es Konva.

### 3.7 Fases propuestas

| Fase | Alcance | Estimación (direccional, semanas-dev) |
| :-- | :-- | :-- |
| **0 — Viewer + geometría sembrada** | Migración `geometry`/`origin` + `plan_image`; render navegable (Konva) coloreado por Ocupación; geometría sembrada por IA para 1 activo real; clic abre el espacio. | ~1.5–2 |
| **1 — Autogeneración por IA (núcleo)** | Pipeline de detección sobre plano subido → propuesta de polígonos → vista de revisión/confirmación → persistencia. Fallback de trazado manual. | a dimensionar por el equipo (pieza nueva) |
| **2 — Edición** | Mover vértices/cuerpo, dividir/juntar con `origin`, bloqueo de edición concurrente. | ~2.5–3.5 |
| **3 — Capa propietaria + extras** | Lente **Salud comercial** (cruce con Cobranza), línea de tiempo, pestañas Planos/Documentos, jerarquía complejo/sitio, render por tipología. | ~4–6 |

Calendario real ≈ 1.5–2× por review/QA/deploy. Las estimaciones de Fases 0/2/3 vienen de @alfonso y **no asumen conocimiento del repo**. **La Fase 1 (IA) es pieza nueva y debe dimensionarla el equipo** tras un spike.

## 4 Metrics & Dashboards

- **Calidad de la autogeneración:** % de locales detectados por IA que el cliente confirma sin corregir (señal directa de si la promesa se cumple); confianza promedio de la detección.
- **Adopción:** % de activos con geometría persistida; nº de sesiones que entran a la pestaña por cuenta activa (acción operativa, no login).
- **Recurrencia:** cohortes W+4 de uso de la pestaña (red flag <30%, consistente con KR2 Q2).
- **Técnicas:** latencia de detección de IA; latencia de render/navegación del plano (objetivo: pan/zoom fluido con cientos de polígonos); tasa de llenado de `assets.geometry`.

## 5 Drawbacks

- **La calidad de la IA es el riesgo dominante.** Si la detección no es buena, la promesa central ("ve tus locales mapeados") se cae. Mitigación: revisión humana obligatoria, fallback manual, y empezar por detección simple bien hecha antes que detección fina mediocre.
- **Pieza nueva = incertidumbre de estimación.** El pipeline de IA no tiene precedente en el repo; necesita un spike antes de comprometer fecha.
- **Navegación fluida no es gratis.** El giro a mapa-protagonista sube la barra de rendimiento; por eso Konva pasa a requisito.
- **Bordes compartidos** (muro común entre dos locales): se resuelve con snapping, no topología real, en esta etapa.

## 6 Alternatives

- **Trazado 100% manual (sin IA).** Es el fallback, no el producto: no escala más allá de una demo. Lo descartamos como camino principal precisamente por esto.
- **Mantener lentes + línea de tiempo desde v1** (como en prototipos v4/v5). Se pospone: diluyen el foco en la navegación. Vuelven en Fase 3 cuando el mapa ya es sólido y hay datos de Cobranza.
- **Isométrico / 3D** (prototipos v3/v4). Descartado: costo desproporcionado al valor sobre un plano 2D navegable.
- **Comprar herramienta de la categoría (VTS, Buildout).** Pricing enterprise, gringo, sin renta variable mexicana ni autogeneración sobre planos del cliente — perderíamos el diferenciador.

## 7 Potential Impact and Dependencies

- **Infra de imágenes (crítico):** supuesto clave de @alfonso — *que la subida de imágenes existente se pueda enganchar a nivel `floor`*. **Confirmar antes de fijar fechas.** Es lo que más mueve la estimación de Fase 0 y el insumo del pipeline de IA.
- **Servicio de IA / visión:** nueva dependencia. Definir si reusa infraestructura de IA existente (AI Pod / ContractsAI ya hacen visión sobre documentos) o requiere algo nuevo. **Recomendaría explorar primero el reuso** antes de incorporar un servicio externo.
- **Cobranza:** la capa propietaria (Salud comercial) consume ventas + RMG de Cobranza Fase 1. Dependencia diferida a Fase 3.
- **Espacios / Contratos:** el mapa lee de su mismo origen de datos; "Abrir expediente" navega a la ficha existente. No los modifica.
- **Permisos / FGA:** edición y regeneración de geometría deben estar gated. Encaja con la jerarquía Cuenta→Complejo→Piso→Espacio→Contrato y con JIT permissions para acciones destructivas.
- **Seguridad:** validar polígonos en el endpoint (≥3 puntos, sin auto-intersección, rango 0–1), no solo en cliente; tratar la imagen subida como input no confiable.

## 8 Unresolved questions

1. **¿La infra de imágenes engancha a nivel `floor`** sin trabajo extra? (Bloquea Fase 0 y alimenta la IA.)
2. **¿La detección de IA reusa AI Pod / ContractsAI** o es un servicio nuevo? Recomendación: spike de reuso primero.
3. **¿Detección síncrona o en background?** Recomendación: no bloquear la carga del plano con la detección.
4. **¿Dónde vive la revisión/confirmación** de lo que detecta la IA — en la misma pestaña o en un flujo de onboarding (Supply)? 
5. **Geometría: ¿atributo o entidad versionada?** Arrancamos con atributo; versionado solo si lo exige cumplimiento.

## 9 Conclusion

El Stacking Plan cierra la brecha espacial de Colossu de forma aditiva y de bajo riesgo estructural, con un protagonista claro: **un mapa navegable de los locales del cliente, geometrizados automáticamente por IA a partir de su propio plano**. La autogeneración es lo que vuelve esto un producto escalable y no un servicio manual, y la arquitectura de tres capas garantiza que la IA pueda regenerar geometría sin tocar nunca lo comercial. Recomendamos: aprobar el arranque por **Fase 0** sobre un activo real, **hacer un spike de la IA de detección** (reusando AI Pod si es viable) para dimensionar la Fase 1, validar el supuesto de la infra de imágenes, y revisar este RFC en SWARCH antes de comprometer las fases de IA y edición.
