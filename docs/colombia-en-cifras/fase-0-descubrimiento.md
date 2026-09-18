# FASE 0 — Descubrimiento y Decisiones

**Proyecto:** Colombia en Cifras — Economic Clock
**Estado:** Entregable de FASE 0 (solo descubrimiento y decisiones). No se ha construido la aplicación ni se han integrado fuentes.
**Fecha de corte del documento:** 2026-09-18
**Regla observada:** en esta fase no se inventan cifras, endpoints, códigos de series ni identificadores de datasets. Toda verificación técnica de recursos descargables se difiere a la FASE 1.

---

## 0. Cómo leer este documento

Este documento responde exactamente a los 10 puntos exigidos por la sección 33 del prompt maestro y usa el formato de respuesta de la sección 32. No sustituye la FASE 1 (investigación de fuentes): las URLs institucionales aquí listadas son **puntos de partida a verificar**, no endpoints confirmados.

---

## 1. Confirmación de entendimiento del producto

Construiremos un **tablero económico nacional** que muestra la economía colombiana como un sistema vivo: contadores que se mueven de forma **reproducible y auditable** entre publicaciones oficiales, sin presentar una estimación como si fuera un dato oficial en tiempo real.

Puntos que se consideran entendidos y aceptados como base de diseño:

- El diferenciador NO son tarjetas estáticas, sino un **motor de estimación determinístico** que anima valores entre publicaciones oficiales.
- Todo número visible se clasifica en uno de cinco estados: `OFFICIAL`, `ESTIMATED`, `PROJECTED`, `MARKET`, `DEMO`.
- El movimiento nunca usa números aleatorios: se reproduce a partir de (valor base + tiempo transcurrido + fórmula + parámetros + versión del modelo).
- Los valores oficiales no se reescriben; las revisiones se guardan como versiones nuevas.
- La plataforma debe permitir **apagar las estimaciones** y ver solo valores oficiales.
- Zona horaria de presentación `America/Bogota`; almacenamiento interno en UTC.
- Precisión decimal (no punto flotante) para montos monetarios críticos.
- El producto es **independiente**: no usa logotipos oficiales de forma que sugiera afiliación institucional.

No se ha detectado ninguna contradicción interna en el prompt que impida arrancar. Las decisiones abiertas están en la sección 5.

---

## 2. Nombre final y arquitectura conceptual propuestos

### 2.1 Nombre
- **Nombre de producto:** *Colombia en Cifras*
- **Descriptor:** *Economic Clock*
- **Propuesta de valor:** *La economía colombiana, en movimiento y con fuentes verificables.*
- Alternativas dejadas como configurables (no como marca definitiva): *Colombia Economic Clock*, *Reloj Económico de Colombia*, *Colombia Ahora*, *Colombia en Tiempo Real*.
- **No** se registrarán dominios ni se tomarán decisiones de marca definitivas sin aprobación del propietario.

### 2.2 Arquitectura conceptual (alto nivel)

Flujo de datos en un solo sentido, con separación estricta entre cálculo económico (servidor) y animación visual (cliente):

```
Fuentes oficiales
      │
      ▼
[connectors] ─ descarga (API/CSV/XLSX/JSON/SDMX/HTML/PDF), con allowlist de dominios
      ▼
[parsers] ─ interpretan el formato crudo
      ▼
[validators] ─ esquema, unidad, escala, signo, fecha, rango plausible, saltos
      ▼
[normalizers] ─ modelo común de observación
      ▼
[reconciliation] ─ componentes vs total (p. ej. interna+externa≈total)
      ▼
[storage] ─ crudo (raw_source_files) + normalizado (observations) + versiones
      ▼
[models] ─ motor de estimación determinístico, escenarios (bajo/central/alto), intervalos
      ▼
[api] ─ contrato público del indicador (valor base + timestamp + tasa/función + límites)
      ▼
[ui] ─ dashboard que anima con requestAnimationFrame, recalculando desde el timestamp
      │
[monitoring] ─ logs de ingesta, staleness, cambios de esquema, error del modelo, alertas
```

Principio clave de animación: **el navegador nunca consulta el servidor por frame**. Recibe valor base, timestamp base y una tasa/función; recalcula desde el timestamp (evitando drift acumulativo) y sincroniza periódicamente.

### 2.3 Stack recomendado (a fijar versiones en FASE 2, no ahora)
- Next.js (App Router) + TypeScript estricto.
- Tailwind CSS + componentes accesibles (shadcn/ui donde ayude).
- Visualización con Apache ECharts o Recharts (decisión final en FASE 2).
- PostgreSQL/Supabase con migraciones reproducibles; RLS si se usa Supabase.
- Zod para validación de contratos.
- Aritmética decimal para dinero (biblioteca decimal, no `number`).
- Jobs programados para ingesta y estimación; almacenamiento de archivos originales.
- GitHub + GitHub Actions (CI). Hosting a documentar en FASE 2.

> No se fijan versiones de paquetes en FASE 0. Se generará lockfile y se documentará el entorno cuando se inicialice el proyecto (FASE 3).

---

## 3. Definición precisa de la serie de deuda que encabeza el dashboard

**Indicador principal propuesto:**
**Deuda bruta del Gobierno Nacional Central (GNC), expresada en pesos colombianos corrientes (COP).**

- `slug` propuesto: `deuda-bruta-gnc`
- Tipo económico: **stock** (saldo a una fecha de referencia).
- Cobertura: Gobierno Nacional Central **únicamente** (no Sector Público No Financiero, no sector público consolidado).
- Fuente primaria candidata: **Ministerio de Hacienda y Crédito Público (MinHacienda)**.
- Unidad: COP corrientes; escala a confirmar (millones/billones según publicación).

**Restricción crítica:** la nomenclatura exacta de la serie ("deuda bruta GNC") y su escala deben **confirmarse contra la publicación oficial de MinHacienda en la FASE 1** antes de cualquier lanzamiento. No se combinará ni sumará con deuda del SPNF, deuda externa total del país u otras coberturas sin verificar compatibilidad de definición, fecha y anualización.

Vistas separadas (secundarias, no encabezan el dashboard, se implementan por fases): deuda neta GNC, deuda del SPNF, deuda externa total de Colombia, deuda pública externa, deuda interna. Cada una mostrará su definición y cobertura junto al valor.

---

## 4. Los 12 indicadores del MVP

Se adopta el MVP recomendado por el prompt (sección 8), con su clasificación matemática, que determina el modelo de estimación:

| # | Indicador | slug propuesto | Tipo | Fuente primaria candidata (a verificar F1) | Estimación entre publicaciones |
|---|-----------|----------------|------|--------------------------------------------|--------------------------------|
| 1 | Deuda bruta GNC | `deuda-bruta-gnc` | Stock | MinHacienda | Sí (modelo de deuda §11) |
| 2 | Deuda por habitante | `deuda-por-habitante` | Derivado (stock/stock) | Deuda GNC ÷ Población DANE | Sí (derivado) |
| 3 | Deuda/PIB | `deuda-pib` | Tasa/proporción | Deuda GNC ÷ PIB nominal (bases compatibles) | Con cautela (bases compatibles) |
| 4 | Intereses estimados de deuda | `intereses-deuda` | Flujo acumulativo | MinHacienda (ejecución/servicio) | Sí (flujo acumulado) |
| 5 | PIB nominal estimado | `pib-nominal` | Stock/flujo (según presentación) | DANE (cuentas nacionales) | Nowcast documentado |
| 6 | Recaudo tributario acumulado | `recaudo-tributario` | Flujo acumulativo | DIAN | Sí (perfil estacional del año fiscal) |
| 7 | Gasto público acumulado | `gasto-publico` | Flujo acumulativo | MinHacienda (ejecución presupuestal) | Sí (flujo acumulado) |
| 8 | Población estimada | `poblacion` | Stock | DANE (proyecciones de población) | Interpolación entre puntos oficiales |
| 9 | Inflación anual | `inflacion-anual` | Tasa/proporción | DANE (IPC) | Suave; baja precisión intramensual |
| 10 | Desempleo | `desempleo` | Tasa/proporción | DANE (GEIH) | Serie temporal con estacionalidad |
| 11 | TRM | `trm` | Precio de mercado | Fuente certificadora (a confirmar) | Escalón diario; sin ruido aleatorio |
| 12 | Reservas internacionales | `reservas-internacionales` | Stock | Banco de la República | Según frecuencia oficial |

Notas:
- Indicadores 2 y 3 son **derivados**: su corrección depende de la compatibilidad de las series base (fecha, cobertura, anualización). Se documenta explícitamente.
- Indicador 5 (PIB) y 11 (TRM) requieren decisiones abiertas (ver §5).
- Para cumplir el criterio de aceptación "al menos 5 conectores oficiales verificados", los 5 objetivos prioritarios de conector real (FASE 4) son: **Deuda, Población, IPC/Inflación, Empleo/Desempleo y TRM/tasa cambiaria oficial**.

---

## 5. Decisiones que necesitan validación del propietario (BLOQUEANTES o SEMI-BLOQUEANTES)

Estas requieren tu respuesta antes o durante la FASE 1:

1. **Fuente certificadora de la TRM.** El prompt menciona tanto Banco de la República como Superintendencia Financiera. Debemos fijar **una** fuente certificadora primaria. *Recomendación:* usar la fuente oficial certificadora de la TRM y documentar la otra como respaldo. — **Requiere tu confirmación de cuál será primaria.**
2. **Escala y nomenclatura de la deuda GNC.** ¿Confirmas que el encabezado sea "Deuda bruta del GNC en COP corrientes"? La nomenclatura exacta se verifica en F1, pero la **definición de cobertura** la fijas tú ahora.
3. **PIB del MVP: ¿mostrar un contador animado o solo el último oficial?** El PIB nominal "estimado en vivo" es delicado (riesgo de percepción). *Recomendación:* en el MVP mostrar PIB nominal **oficial** con nowcast claramente separado y opcional, no un contador principal agresivo. — **Requiere tu decisión.**
4. **Idioma del MVP.** ES primero, con EN "preparado" (i18n en estructura pero no traducido completo). ¿Correcto?
5. **Moneda de visualización.** Selector COP/USD: para el MVP, ¿USD es solo conversión visual con TRM del día (marcada como conversión, no oficial)? *Recomendación:* sí.
6. **Alcance del panel administrativo y roles en el MVP.** ¿El MVP incluye panel admin mínimo (estado de fuentes + ejecución manual + ver errores) o se difiere a FASE 7? *Recomendación:* incluir un admin mínimo protegido desde FASE 4.
7. **Hosting/infra e integraciones de terceros** (Vercel/otro, Sentry, analítica): decisión en FASE 2, pero necesitamos saber si hay restricciones corporativas.
8. **Licencias.** Antes de exponer datos de cada fuente (especialmente Banco de la República, XM, mercados) hay que confirmar términos de reutilización. Algunos datos pueden requerir atribución específica o prohibir redistribución. — **Riesgo legal, ver §6.**

---

## 6. Plan de investigación de fuentes (preparación de la FASE 1)

Para cada indicador del MVP se producirá una **ficha de descubrimiento** (sección 6 del prompt) antes de programar cualquier conector. La ficha incluirá: institución, indicador, nombre exacto de la serie, definición, cobertura, unidad, frecuencia, URL de página oficial, URL directa del recurso, formato, método de acceso, autenticación (sí/no), licencia/términos, última actualización observada, campos relevantes, ejemplo de registro, riesgos, estrategia de respaldo y estado (descubierto/verificado/integrado/suspendido).

Jerarquía de adquisición (en orden): (1) API oficial documentada → (2) API de datos abiertos → (3) CSV oficial → (4) JSON/XML/SDMX oficial → (5) XLSX/XLS oficial → (6) tabla HTML estable → (7) PDF con extracción controlada + revisión humana → (8) ingreso manual con evidencia y doble validación.

**Portales de partida a verificar (NO endpoints confirmados):**

| Institución | Rol | Portal de partida | Indicadores MVP asociados |
|-------------|-----|-------------------|---------------------------|
| DANE | Estadística primaria | dane.gov.co | PIB, IPC/Inflación, Desempleo (GEIH), Población |
| Banco de la República | Monetaria/cambiaria/externa | banrep.gov.co ; suameca.banrep.gov.co | Reservas internacionales, (TRM candidata) |
| MinHacienda | Fiscal y deuda | minhacienda.gov.co | Deuda GNC, Intereses, Gasto público |
| DIAN | Tributaria | dian.gov.co | Recaudo tributario |
| Superintendencia Financiera | Financiera | superfinanciera.gov.co | (TRM candidata) |
| Datos Abiertos | Canal estructurado | datos.gov.co | Respaldo/descubrimiento de datasets |
| Fuentes internacionales | Respaldo/comparación | Banco Mundial, FMI, CEPALSTAT | Solo respaldo, nunca reemplazo silencioso |

**Entregable de la FASE 1** (cuando se apruebe): matriz indicador/fuente con URL de página, URL de recurso verificable, formato, frecuencia, unidad, cobertura, licencia, estado de verificación y riesgos. Si para algún indicador no existe fuente estructurada, se dirá explícitamente y se propondrá el nivel más bajo viable de la jerarquía.

---

## 7. Riesgos (legales, estadísticos, técnicos, de percepción pública)

### 7.1 Legales / licenciamiento
- **R-L1:** Términos de reutilización de datos por fuente pueden prohibir redistribución o exigir atribución específica (Banco de la República, XM, mercados). *Mitigación:* verificar licencia en ficha de descubrimiento antes de integrar; inventario de licencias y atribuciones; no publicar lo que no esté permitido.
- **R-L2:** Uso de logotipos/marca oficial que sugiera afiliación. *Mitigación:* aviso de independencia visible; no usar logos oficiales; atribución textual con enlace a la fuente.
- **R-L3:** No copiar marca/código/diseño de usdebtclock.org. *Mitigación:* diseño y copy propios; solo inspiración conceptual de utilidad.

### 7.2 Estadísticos / metodológicos
- **R-E1:** Mezclar definiciones incompatibles (deuda GNC vs SPNF; FOB vs CIF; corriente vs constante; desestacionalizado vs original). *Mitigación:* metadatos de cobertura por serie; reglas de reconciliación; prohibición de sumar/comparar sin verificación.
- **R-E2:** *Leakage* en backtesting (usar datos futuros). *Mitigación:* simular el conocimiento disponible en cada fecha; métricas por horizonte.
- **R-E3:** Presentar interpolaciones (población, pobreza) como conteos observados. *Mitigación:* marcar `ESTIMATED`, bandas de incertidumbre, no interpolar pobreza automáticamente entre años.
- **R-E4:** Sobreajuste a modelos complejos. *Mitigación:* preferir el modelo más simple robusto; aprobación para cambios metodológicos materiales.

### 7.3 Técnicos
- **R-T1:** Fuentes frágiles / cambios de esquema silenciosos. *Mitigación:* validación de esquema, fallo seguro del conector, circuit breaker, conservar último dato validado + alerta de staleness.
- **R-T2:** Drift en la animación de contadores. *Mitigación:* recalcular desde timestamp base, no acumular incrementos; sincronización periódica.
- **R-T3:** Precisión monetaria con punto flotante. *Mitigación:* aritmética decimal; no `float` para montos críticos.
- **R-T4:** Exposición de secretos/API keys en el navegador. *Mitigación:* secretos solo en servidor; funciones server-side; `.env.example` sin valores.
- **R-T5:** SSRF en URLs administrativas / conectores. *Mitigación:* allowlist de dominios oficiales; validación de URLs.

### 7.4 Percepción pública
- **R-P1:** Que una estimación se perciba como "dato oficial en tiempo real". *Mitigación:* ver §8 (estrategia dedicada).
- **R-P2:** Interpretación política/mediática de cifras sensibles (deuda, pobreza, desempleo). *Mitigación:* metodología pública auditable, fuente y fecha en cada tarjeta, política de correcciones, contacto para reportar errores.

---

## 8. Cómo evitaremos que una estimación parezca un dato oficial

Estrategia multicapa, obligatoria en todo componente que muestre un número:

1. **Estado explícito por número:** cada valor lleva su etiqueta visible `OFFICIAL` / `ESTIMATED` / `PROJECTED` / `MARKET` / `DEMO`, no solo por color (para accesibilidad).
2. **Codificación visual consistente:** ámbar reservado para estimaciones/advertencias; el estado se comunica con texto + icono + color (nunca solo color).
3. **Procedencia en la tarjeta:** último valor oficial, fecha de referencia oficial, fecha de publicación, fuente con enlace y "cómo se calcula".
4. **Interruptor "Estimaciones en vivo":** apagarlo detiene el movimiento y muestra únicamente el último valor oficial.
5. **Sin falsa precisión:** el número mostrado se redondea a una unidad legible acorde a la incertidumbre del modelo; se muestran bandas (bajo/central/alto) cuando aplica.
6. **Aviso global de transparencia** (texto de la sección 27 del prompt) visible en el sitio.
7. **Página de metodología por indicador** con fórmula, parámetros, versión del modelo, fecha de corte y precisión histórica.
8. **Reproducibilidad:** todo movimiento es recalculable desde (valor base, timestamp, fórmula, parámetros, versión). Nada aleatorio.
9. **Recalibración transparente:** al llegar un dato oficial nuevo se muestra estimación anterior, nuevo oficial, diferencia y error; sin reescribir predicciones pasadas.

---

## 9. Criterios de éxito del MVP (propuestos)

Se adoptan los 18 criterios de la sección 29 del prompt como definición de "terminado", resumidos como puertas de aceptación:

1. 12 indicadores funcionales.
2. ≥5 provienen de conectores oficiales verificados (Deuda, Población, IPC, Empleo, TRM).
3. Ningún componente muestra un número *hardcoded* como si fuera real.
4. La deuda se anima desde un modelo reproducible.
5. El usuario puede apagar estimaciones.
6. Cada tarjeta muestra fuente y fecha.
7. Cada estimación tiene metodología.
8. Existe una gráfica oficial vs. estimado.
9. Se guardan datos crudos y normalizados.
10. Existen logs de ingesta.
11. Existen alertas de staleness.
12. Las fórmulas tienen pruebas.
13. Interfaz responsive.
14. Accesibilidad esencial (WCAG AA en contraste, teclado, no solo color).
15. Pasa lint, typecheck, tests y build.
16. Documentación para ejecutar localmente y desplegar.
17. Inventario de licencias y atribuciones.
18. Sin secretos en Git.

---

## 10. Plan de fases (hoja de ruta) y solicitud de aprobación

### Hoja de ruta
- **FASE 0 — Descubrimiento y decisiones** ← *este entregable.*
- **FASE 1 — Investigación de fuentes:** matriz indicador/fuente verificada, fichas de descubrimiento, licencias. Sin inventar endpoints.
- **FASE 2 — Arquitectura y diseño:** ADRs, diagrama, modelo de datos (SQL), contratos API (Zod/OpenAPI), estrategia de jobs y caché, wireframes, sistema visual, plan de seguridad y de pruebas.
- **FASE 3 — Prototipo con DEMO:** frontend con fixtures marcados `DEMO` (nunca presentados como reales); tarjetas, navegación, gráficas, escenarios, modo oficial/estimado.
- **FASE 4 — Primeros conectores reales:** Deuda, Población, IPC/Inflación, Empleo/Desempleo, TRM.
- **FASE 5 — Motor de estimación:** modelos base, backtesting sin leakage, escenarios, intervalos, recalibración, historial de precisión.
- **FASE 6 — Ampliación:** PIB, Recaudo, Gasto, Reservas, comercio exterior, remesas, energía.
- **FASE 7 — QA y producción:** seguridad, rendimiento, accesibilidad, SEO, observabilidad, documentación, deploy, runbooks.

Al final de cada fase: detener, entregar evidencia y esperar aprobación antes de decisiones materiales.

### Documentación que se producirá (según sección 31)
`README.md`, `docs/product-requirements.md`, `docs/architecture/*`, `docs/sources/*`, `docs/methodologies/*`, `docs/security/threat-model.md`, `docs/runbooks/*`, `docs/decisions/` (ADRs), `.env.example`, y OpenAPI o equivalente. Se crearán en las fases correspondientes.

### Solicitud concreta de aprobación
Para pasar a la **FASE 1 — Investigación de fuentes**, necesito tu aprobación y, en lo posible, respuesta a las decisiones bloqueantes de la §5 (especialmente: fuente certificadora de TRM #1, definición/cobertura de la deuda #2, y tratamiento del PIB en el MVP #3). Con eso, en la FASE 1 entregaré la matriz de fuentes verificadas sin construir conectores todavía.
