# Tablero de compras públicas desde Guatecompras (OCDS)

Toma los paquetes OCDS mensuales de Guatecompras, filtra por una entidad compradora
y produce un tablero HTML autocontenido, en español, con KPIs, series de tiempo,
rankings y el detalle de contratos.

El flujo tiene cinco fases y **cada una tiene una condición de salida**: no avances
si la anterior no cumple. La razón es económica: la fase 1 descarga ~2 GB y la 2
tarda varios minutos, así que un parámetro mal entendido cuesta caro.

## Contrato de salida

Al final debe existir:

1. Los JSON crudos en `~/Downloads/ocds_gc/` (uno por mes, `AAAA-MM_Guatecompras.json`).
2. El JSON compacto filtrado (<2 MB) en la carpeta conectada.
3. Un único archivo HTML autocontenido publicado con la herramienta Artifact.
4. Un mensaje final con las cifras clave y la ubicación de los archivos.

---

## Fase 0 — Parámetros

### 0.1 Qué necesitas antes de tocar la red

| Parámetro           | Obligatorio | Ejemplo                                              |
| ------------------- | ----------- | ---------------------------------------------------- |
| NIT del comprador   | sí          | `3440915` (MICIVI), sin guiones ni prefijo `GT-NIT-` |
| Mes inicial         | sí          | `enero 2026`                                         |
| Mes final           | sí          | `marzo 2026`                                         |
| Enfoque del tablero | no          | por defecto, el conjunto estándar de la fase 3       |

Si el mensaje del usuario ya trae NIT y ambos extremos del intervalo, **no preguntes
nada**: confirma en una línea lo que entendiste y arranca. Preguntar lo ya dicho es
la forma más rápida de resultar molesto.

### 0.2 Preguntar lo que falte — una sola llamada

Si falta algo, usa **una** llamada a AskUserQuestion con hasta tres preguntas.
No descargues nada antes de tener respuesta.

**Pregunta 1 — NIT del comprador.** Explica que la API se filtra por NIT, no por
nombre. Opciones: escribir el NIT, o "no lo sé" → en ese caso ofrece bajar **un solo
mes de muestra** y buscar el NIT por nombre dentro de `parties[]` / `buyer.name`
antes de comprometerse al resto de la descarga.

**Pregunta 2 — Mes inicial (mes y año).** Ejemplos concretos: `enero 2026`,
`octubre 2025`.

**Pregunta 3 — Mes final (mes y año).** Ejemplos: `marzo 2026`, `el mes pasado`.

El intervalo se expresa **siempre como dos extremos con mes y año**, nunca como
"año + rango de meses": el usuario puede querer octubre 2025 – marzo 2026, y un
formato de un solo año hace imposible pedirlo. Expande el intervalo a la lista
completa de meses (inclusive en ambos extremos) y enséñasela antes de descargar:
"voy a bajar 6 paquetes: 2025-10 … 2026-03".

El enfoque del tablero no se pregunta por defecto: describe en una línea lo que vas
a incluir y di que se puede ajustar después de verlo. Pregúntalo solo si el usuario
insinuó un corte no estándar (por región, por tipo de obra, comparación entre años).

### 0.3 Validar el intervalo antes de aceptarlo

- El mes inicial no puede ser posterior al final; si lo es, no adivines: repregunta.
- El **mes en curso y los futuros** normalmente no están publicados. Si el intervalo
  los incluye, avísalo y propón recortar hasta el mes anterior completo.
- Más de 12 meses: advierte el costo (~250 MB por mes en disco, varios minutos por
  mes de descarga) y pide confirmación explícita.
- **Qué significa el intervalo.** Los paquetes están segmentados por _mes de
  publicación de la convocatoria_, no por fecha de adjudicación. Consecuencias que
  debes decir en voz alta, porque cambian lo que el usuario recibe:
  - Dentro de los archivos de enero–marzo verás adjudicaciones fechadas en abril o
    después.
  - Una adjudicación de febrero cuyo concurso se publicó en 2025 **no** estará si no
    bajas también los meses de 2025.
  - Si el usuario quiere "todo lo adjudicado en enero–marzo 2026", ofrécele agregar
    3–6 meses previos a la descarga.

### 0.4 Verificar el entorno antes de bajar 2 GB

Con `device_bash`:

- Espacio libre: `df -h ~/Downloads`. Exige al menos `400 MB × nMeses + 2 GB` de
  margen. Si no alcanza, dilo y para.
- `mkdir -p ~/Downloads/ocds_gc`.
- **Reanudación:** lista lo que ya esté ahí. Los meses ya descargados y válidos se
  saltan. Si ya existe el JSON compacto de este mismo NIT e intervalo, salta directo
  a la fase 3.
- Si no tienes navegador ni `device_bash` en esta sesión, di claramente que este
  flujo requiere la app de escritorio y detente. No intentes sustitutos.

Crea la lista de tareas (TaskCreate) con las fases 1–5 recién ahora.

---

## Fase 1 — Descarga por navegador

El proxy de red bloquea `ocds.guatecompras.gt` tanto en el bash de la nube como en
`device_bash`, y `javascript_tool` no corre por el CSP del sitio. La única vía es el
navegador del usuario.

1. `preview_start` en `https://ocds.guatecompras.gt/` para abrir una pestaña.
2. **Un mes a la vez**, `navigate` a `https://ocds.guatecompras.gt/file/json/{AÑO}/{MES}`
   (mes sin cero a la izquierda salvo que la URL lo exija; verifica con el primero).
   Varias navegaciones seguidas en la misma pestaña se cancelan entre sí y solo
   sobreviven algunas descargas.
3. La llamada **siempre responde "navigation denied or failed"; eso es normal** — el
   navegador lo trata como descarga, no como navegación. Ignora ese error; el único
   indicador real de éxito es el archivo en disco.
4. Antes de cada `navigate`, guarda la lista de archivos temporales existentes
   (`ls ~/Downloads/.Q6L2SF6YDW.com.anthropic.claudefordesktop.* 2>/dev/null`).
   Después, trabaja **solo con la diferencia**: así no confundes una descarga nueva
   con un resto de un intento anterior.
5. **No esperes un tiempo fijo.** Sondea cada 5 s el tamaño del archivo nuevo y
   considéralo terminado cuando no cambie en tres lecturas seguidas. Corta a los
   180 s y reintenta ese mes (máximo 2 reintentos).
6. Los temporales son **ZIP**, no JSON. Valida con `unzip -t` y descomprime en
   `~/Downloads/ocds_gc/`; el nombre interno (`AAAA-MM_Guatecompras.json`) dice a qué
   mes corresponde, así que no necesitas rastrear el orden. Un ZIP que no pasa
   `unzip -t` está truncado: bórralo y reintenta.
7. Borra el ZIP intermedio tras extraer correctamente (son ~35 MB cada uno y su
   contenido ya quedó en el JSON). Los JSON crudos **no se borran**.

**Condición de salida:** están los `AAAA-MM_Guatecompras.json` de todos los meses
pedidos, cada uno de tamaño plausible (~200–300 MB). Si falta alguno tras los
reintentos, no lo escondas: sigue con los que hay y regístralo para el pie del
tablero y para el mensaje final.

---

## Fase 2 — Extracción y normalización (en la máquina del usuario)

Corre todo con `device_bash` + `python3` sobre `~/Downloads/ocds_gc/`. **Nunca hagas
stage de archivos de 250 MB**; lo único que sube a la nube es el JSON compacto.

**Un proceso `python3` por archivo**, no un script que los recorra todos: un JSON de
250 MB se expande a varios GB en memoria, y un proceso por mes acota el consumo,
sobrevive a un fallo aislado y hace el trabajo reanudable. Cada proceso escribe
`~/Downloads/ocds_gc/rows_AAAA-MM.jsonl` con las filas ya filtradas por NIT; un paso
final une los JSONL y produce el JSON compacto.

Estructura del paquete: `{"records": [{"ocid", "compiledRelease": {...}}]}`, donde
`compiledRelease` trae `buyer.id` (`GT-NIT-<nit>`), `tender`, `parties`, y
opcionalmente `awards` y `contracts`.

Extrae **una fila por adjudicación** (`award`), recorriendo todas, no solo la primera:

| campo       | origen                                                                               |
| ----------- | ------------------------------------------------------------------------------------ |
| `nog`       | último segmento de `ocid` (`ocds-xqjsxa-28887778` → `28887778`)                      |
| `fecha`     | `award.date` (fallback `tender.datePublished`)                                       |
| `monto`     | `award.value.amount`                                                                 |
| `moneda`    | `award.value.currency`                                                               |
| `proveedor` | `award.suppliers[0].name` / `.id`                                                    |
| `unidad`    | primer `parties[]` cuyo `id` empiece con `GT-GCUC` → `name`                          |
| `categoria` | `tender.mainProcurementCategory` (goods/services/works)                              |
| `modalidad` | `tender.procurementMethodDetails`                                                    |
| `estado`    | `tender.statusDetails`                                                               |
| `titulo`    | `tender.title`                                                                       |
| contrato    | de `contracts[]` unido por `awardID`: `contractNumber`, `dateSigned`, `value.amount` |

Reglas que importan:

- **Fechas: corta la cadena ISO a `[:10]`**, no la parsees con zona horaria. Una
  fecha con `T00:00:00-06:00` interpretada en UTC se corre un día y ensucia el
  calendario diario. Si no hay fecha válida, la fila entra en los totales pero se
  excluye de las series de tiempo; cuenta cuántas fueron.
- **Procesos sin `awards`** entran igual, con `monto: 0`, para que el conteo de
  procesos y el filtro de estado sean correctos.
- **Adjudicaciones canceladas o desiertas** (`award.status` en `cancelled` /
  `unsuccessful`) no suman monto. Cuéntalas aparte.
- **Moneda distinta de GTQ:** no la conviertas ni la mezcles. Exclúyela del monto y
  reporta cuántas filas fueron.
- **Deduplica** por `(nog, fecha, monto, titulo)`. La fuente repite adjudicaciones
  cuando el mismo proveedor aparece con dos grafías o dos NIT; en la corrida del
  MICIVI esto infló el total en Q 990 M. Riesgo conocido en el otro sentido: un
  concurso con varios renglones adjudicados el mismo día por el mismo monto se
  colapsa indebidamente — mitígalo agregando `contractNumber` a la llave cuando
  exista. Guarda el número de filas eliminadas para el pie.
- **Serializa con pools de strings**: `unidad`, `proveedor`, `categoria`, `modalidad`
  y `estado` como índices enteros, y las filas como arrays posicionales. Eso baja de
  ~1.2 MB a ~370 KB y permite incrustar los datos dentro del HTML.

Forma exacta del JSON compacto (el tablero no debe inventar otra):

```json
{
  "meta": {
    "nit": "3440915",
    "comprador": "…",
    "mesesSolicitados": ["2026-01", "2026-02", "2026-03"],
    "mesesPresentes": ["2026-01", "2026-02", "2026-03"],
    "fechaMin": "2026-01-03",
    "fechaMax": "2026-07-18",
    "filas": 4821,
    "procesos": 3990,
    "duplicadosEliminados": 214,
    "sinFecha": 7,
    "monedaNoGTQ": 0,
    "canceladas": 33,
    "generado": "2026-09-09"
  },
  "pools": {
    "unidades": [],
    "proveedores": [],
    "categorias": [],
    "modalidades": [],
    "estados": []
  },
  "cols": [
    "nog",
    "fecha",
    "monto",
    "prov",
    "unid",
    "cat",
    "mod",
    "est",
    "titulo",
    "contrato",
    "fechaContrato",
    "montoContrato"
  ],
  "rows": [
    [
      "28887778",
      "2026-02-14",
      1250000.0,
      12,
      3,
      0,
      1,
      2,
      "…",
      "C-2026-441",
      "2026-03-01",
      1250000.0
    ]
  ]
}
```

Copia el JSON compacto a la carpeta conectada, `device_stage_files` y sigue en la nube.

**Condición de salida:** `meta.filas > 0`. Si el NIT no aparece en ningún mes, no
construyas un tablero vacío: dilo y ofrece buscar el NIT correcto en los datos ya
descargados.

---

## Fase 3 — El tablero

Un solo archivo HTML autocontenido, **en español**, publicado con la herramienta
Artifact. Carga antes las skills `artifact-design` y `dataviz`.

### Filtros

Fila superior, combinados con **Y** (AND), recalculando todo en vivo: desde, hasta,
unidad ejecutora, proveedor, categoría, estado, modalidad, y botón Limpiar.

Los filtros de lista **deben ser comboboxes con búsqueda interna**, no `<select>`:
con cientos de proveedores el scroll nativo es inservible. Cada uno necesita:

- input que filtra en vivo, **insensible a tildes y mayúsculas**
  (`s.toLowerCase().normalize("NFD").replace(/[\u0300-\u036f]/g,"")`), con la
  coincidencia resaltada;
- opciones ordenadas por monto descendente y calculadas sobre las filas que pasan
  _los demás_ filtros activos, para que nunca se ofrezca una opción que daría cero;
- hint a la derecha (`n adjudicaciones · Q X M`) para que lo relevante salga primero
  sin escribir nada;
- teclado: ↑/↓, Enter, Esc; una × para limpiar sin abrir la lista;
- `role="combobox"`, `aria-expanded`, `aria-activedescendant` y foco visible.

Con ~20 000 filas, recalcular en cada tecla se siente lento: aplica un debounce de
~120 ms al texto y precomputa índices por columna.

### Contenido

- **KPIs** — definiciones fijas, porque las filas con monto 0 las distorsionan:
  monto adjudicado (suma de montos > 0, solo GTQ, sin canceladas) · procesos (NOG
  únicos, incluidos los sin adjudicación) · adjudicaciones (filas con monto > 0) ·
  proveedores únicos · unidades únicas · ticket promedio (monto ÷ adjudicaciones).
- **Barras de monto por mes**, con tooltip (monto y conteo). Incluye los meses sin
  gasto dentro del rango observado, o la serie miente por omisión. Elige las marcas
  del eje Y de una lista de múltiplos redondos
  (`[1,1.25,1.5,2,2.5,3,4,5,6,8,10] × 10^k`), no con `ceil(max/(step/2))`, que
  produce ejes feos tipo "Q 1.9 MM".
- **Calendario de gasto** — una fila por mes presente (no 12 fijas) × 31 columnas de
  día, coloreado por **septiles del gasto diario**; una escala logarítmica satura
  casi todo. Calcula los septiles solo sobre días con gasto > 0; los días en cero van
  en color neutro y las casillas inexistentes (30 de febrero) quedan vacías. Incluye
  leyenda con los siete umbrales.
- **Top 10 unidades ejecutoras** y **top 10 proveedores** por monto, con barra y conteo.
- **Tabla de top 20 contratos**: monto, NOG enlazado a
  `guatecompras.gt/concursos/consultaConcurso.aspx?nog=…`, descripción, proveedor,
  unidad, fecha y número de contrato. Scroll horizontal en pantallas angostas.
- **Estado vacío**: si los filtros dejan 0 filas, muestra un mensaje claro y un botón
  Limpiar, no gráficas rotas.
- **Pie** generado desde `meta`, nunca escrito a mano: fuente, meses cubiertos, meses
  faltantes si los hubo, la nota de segmentación por mes de convocatoria, cuántos
  duplicados se eliminaron y cuántas filas quedaron fuera por fecha o moneda.

### Formato y color

- `Intl.NumberFormat("es-GT")`. Ojo: es-GT usa **coma para miles y punto para
  decimales** (`Q 1,234.56`), no al revés.
- Abreviaturas explícitas: `Q X K` = miles, `Q X M` = millones, `Q X MM` = miles de
  millones. Úsalas de forma consistente en ejes, hints y KPIs.
- Tokens de color para tema claro y oscuro: define **todos** en `:root` y redefínelos
  en `@media (prefers-color-scheme: dark)` y en `:root[data-theme="dark"]`, incluidos
  los siete pasos de la escala del calendario.

---

## Fase 4 — Verificación antes de publicar

Renderiza el archivo con Playwright (`NODE_PATH=~/.npm-global/lib/node_modules`,
chromium ya instalado), captura los errores de consola y **revisa la captura una vez**.
Lista de chequeo:

- [ ] Suma de `monto` de las filas = KPI de monto adjudicado (reconciliación numérica).
- [ ] `meta.mesesPresentes` coincide con lo pedido, o el pie lo declara.
- [ ] Ningún valor de KPI cortado ni desbordado.
- [ ] Marcas del eje Y legibles y redondas.
- [ ] Calendario con contraste real en claro y en oscuro.
- [ ] Dropdown abierto sin solaparse con lo de abajo; búsqueda sin tildes funciona.
- [ ] Un NOG de la tabla abre el concurso correcto en guatecompras.gt.
- [ ] Consola sin errores.

Un solo ciclo de corrección, luego publica. Si algo sigue mal después de esa
corrección, publícalo y di qué quedó pendiente en lugar de seguir iterando en silencio.

---

## Fase 5 — Entrega

En el mensaje final, en prosa breve:

- total adjudicado, número de adjudicaciones y de procesos (NOG únicos);
- intervalo de meses cubierto y cualquier mes faltante;
- duplicados eliminados;
- que los JSON crudos quedaron en `~/Downloads/ocds_gc/` (~250 MB por mes, ~2 GB en
  total) y que no se borraron por falta de permiso.

Nunca rellenes una cifra que no salió de los datos. Si un número no se pudo calcular,
dilo; un hueco declarado es más útil que un número inventado.

---

## Apéndice — Trampas conocidas

| Síntoma                               | Causa                                                  | Qué hacer                                      |
| ------------------------------------- | ------------------------------------------------------ | ---------------------------------------------- |
| "navigation denied or failed"         | el navegador trata la URL como descarga                | ignorar; verificar por archivo en disco        |
| Faltan meses tras varias navegaciones | navegaciones concurrentes se cancelan                  | un mes a la vez, con sondeo                    |
| El archivo temporal no es JSON        | es un ZIP con el JSON adentro                          | `unzip -t` y extraer                           |
| El total no cuadra con lo esperado    | adjudicaciones duplicadas por grafía/NIT del proveedor | deduplicar por `(nog, fecha, monto, titulo)`   |
| Aparecen fechas fuera del intervalo   | paquetes segmentados por mes de convocatoria           | explicarlo en el pie, no filtrarlo en silencio |
| El calendario corre todo un día       | fecha ISO parseada con zona horaria                    | usar `award.date[:10]`                         |
| `python3` muere sin mensaje           | 250 MB de JSON expandidos en memoria                   | un proceso por archivo, salida a JSONL         |
| Eje Y tipo "Q 1.9 MM"                 | `ceil(max/(step/2))`                                   | marcas de la lista de múltiplos redondos       |
