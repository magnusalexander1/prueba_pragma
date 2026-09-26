# Pipeline de micro-batches con estadísticas incrementales

Solución a la prueba técnica de **Ingeniero de Datos**. El pipeline carga archivos CSV en PostgreSQL por lotes y actualiza, después de cada fila insertada, las estadísticas del campo `price` (conteo, promedio, mínimo y máximo), **sin tener nunca más de un archivo en memoria** y sin recalcular sobre los datos ya cargados.

**Stack:** Python 3 · pandas · psycopg 3 · PostgreSQL (Neon) · Google Colab

---

## Contenido del repositorio

```
├── README.md
├── prueba_pipeline_v2.ipynb          # Notebook con el pipeline documentado paso a paso
├── Documentacion_Pipeline_Pragma.docx # Documentación detallada del proceso
└── data/
    ├── 2012-1.csv
    ├── 2012-2.csv
    ├── 2012-3.csv
    ├── 2012-4.csv
    ├── 2012-5.csv
    └── validation.csv
```

Cada CSV tiene tres campos: `timestamp`, `price`, `user_id`.

---

## Cómo funciona

```
 CSV (uno a la vez) ──► read_csv(chunksize=10) ──► lote de 10 filas
                                                        │
                                  ┌─────────── transacción del lote ───────────┐
                                  │  1. INSERT de cada fila en transactions     │
                                  │  2. actualizar stats (por fila, en O(1))    │
                                  │  3. guardar stats en pipeline_stats         │
                                  │  4. registrar avance en load_log            │
                                  └─────────────────────────────────────────────┘
                                         │ OK → COMMIT → siguiente lote
                                         │ error → ROLLBACK (base y memoria)
```

### 1. Memoria: nunca más de un archivo a la vez

- Los archivos se procesan **en secuencia**: `cargar_csv` termina un archivo antes de empezar el siguiente.
- Cada archivo se lee **por lotes** con `pd.read_csv(..., chunksize=10)`, así que ni siquiera un archivo completo está en memoria: solo el lote actual.
- Los archivos nunca se concatenan ni se guardan en listas.

En cualquier momento, en memoria hay **como máximo un lote de 10 filas** y los 5 acumuladores de las estadísticas. El tamaño del lote se ajusta con `TAMANO_LOTE`.

### 2. Estadísticas incrementales

Se mantienen 5 acumuladores que se actualizan con cada fila:

| Acumulador | Uso |
|---|---|
| `count` | Filas cargadas (incluye precios vacíos) |
| `n_precios` | Filas con precio |
| `suma` | Suma de precios |
| `min` / `max` | Precio mínimo y máximo |

El promedio se calcula como `suma / n_precios`. **No se usa `SELECT AVG(...)` para actualizar las estadísticas**; la consulta SQL solo se usa al final para verificar.

Los 4 registros con precio vacío se cargan como `NULL`: cuentan como filas cargadas, pero no entran en el promedio, el mínimo ni el máximo (igual que `AVG`, `MIN` y `MAX` en SQL).

### 3. Transacciones: todo o nada por lote

Cada lote se guarda en una sola transacción que incluye las filas, las estadísticas y el avance. Si algo falla, se revierte el lote completo en la base, y las estadísticas en memoria vuelven a su valor anterior. Nunca queda un lote a medias.

### 4. Reanudación e idempotencia

| Tabla | Para qué sirve |
|---|---|
| `pipeline_stats` | Guarda los acumuladores: sobreviven a un reinicio del notebook |
| `load_log` | Guarda por archivo su estado (`EN_PROCESO` / `COMPLETADO`) y las filas leídas |

- Si el proceso falla, al reanudar continúa **desde el último lote confirmado**.
- Si se ejecuta de nuevo, los archivos `COMPLETADO` se omiten: **no se duplican datos**.

---

## Modelo de datos

**`transactions`**

| Columna | Tipo | Descripción |
|---|---|---|
| `id` | SERIAL PK | Identificador |
| `event_date` | DATE | Fecha del evento |
| `price` | NUMERIC (nulo) | Precio |
| `user_id` | INTEGER | Usuario |
| `source_file` | TEXT | Archivo de origen |

**`pipeline_stats`**: una fila con `total_filas`, `n_precios`, `suma`, `minimo`, `maximo`.

**`load_log`**: `archivo` (PK), `estado`, `filas_leidas`.

---

## Cómo ejecutarlo

No requiere instalar nada localmente.

1. **Base de datos:** crea un proyecto gratuito en [Neon](https://neon.tech) y copia la *Connection string*. Sirve cualquier PostgreSQL.
2. **Notebook:** abre [Google Colab](https://colab.research.google.com) → *Archivo → Subir notebook* → `prueba_pipeline_v2.ipynb`.
3. **Conexión:** ejecuta la sección 1. Pega la cadena de conexión en el cuadro que aparece; se pide con `getpass`, así que no queda guardada en el notebook.
4. **Datos:** en la sección 2, sube los 6 archivos de la carpeta `data/`.
5. **Ejecución:** corre las celdas en orden o usa *Entorno de ejecución → Ejecutar todas*.

| Situación | Celdas a ejecutar |
|---|---|
| Empezar de cero | Conexión → subir CSV → **crear tablas** → stats → `cargar_csv` → carga |
| Reanudar tras un fallo o reinicio | Conexión → subir CSV → stats → `cargar_csv` → carga (**sin** crear tablas) |

> ⚠️ La celda de crear tablas ejecuta `DROP TABLE`: borra todo lo cargado.

---

## Resultados

| Métrica | Tras 2012-1 a 2012-5 | Tras validation.csv |
|---|---|---|
| Filas cargadas | 143 | 151 |
| Filas con precio | 139 | 147 |
| Promedio | 57.8849 | 57.0068 |
| Mínimo | 10 | 10 |
| Máximo | 100 | 100 |

Las estadísticas en ejecución coinciden exactamente con la consulta SQL:

```sql
SELECT COUNT(*), ROUND(AVG(price), 4), MIN(price), MAX(price) FROM transactions;
```

Filas por archivo: 2012-1 (22), 2012-2 (29), 2012-3 (31), 2012-4 (30), 2012-5 (31), validation (8).

El notebook incluye además una **prueba de robustez**: provoca un error en la fila 37, simula un reinicio, recupera las estadísticas desde la base y reanuda. El resultado final es idéntico al de una carga sin fallos, y una segunda ejecución no inserta ninguna fila.

---

## Posibles mejoras

- Insertar cada lote con `executemany` o `COPY` para mayor velocidad.
- Métricas adicionales (varianza con el algoritmo de Welford, estadísticas por `user_id`).
- Orquestación con Airflow o despliegue en AWS (S3 → Lambda → RDS).
- Mover las funciones a un módulo `.py` con pruebas unitarias.

---

**Autor:** Fabian Ramírez
