La respuesta directa y rotunda es: **No, no es un antipatrón. De hecho, almacenar texto no estructurado, chunks y embeddings en formato Apache Parquet es el estándar técnico dominante en la industria para el entrenamiento de LLMs y pipelines de RAG a escala.**

Existe una confusión común: pensar que «tabular» equivale a «relacional» (números, fechas y textos cortos estilo SQL tradicional). En realidad, **Apache Parquet es un formato de almacenamiento columnar y semiestructurado**. Admite de forma nativa tipos de datos complejos como arrays anidados (`LIST`), mapas clave-valor (`MAP`), estructuras jerárquicas (`STRUCT`), cadenas de texto de longitud arbitraria (`STRING/BYTE_ARRAY`) y vectores numéricos (`LIST<FLOAT32>`).

A continuación tienes las referencias técnicas, la justificación de ingeniería de datos y los ejemplos reales de la industria que respaldan esta arquitectura.

---

### 1. Referencias Reales de la Industria que usan Parquet para Texto y RAG

#### A. Hugging Face Datasets (El repositorio global de datos para IA)
* **Referencia / Documentación Oficial:** Hugging Face adoptó **Apache Parquet como el formato de almacenamiento y streaming por defecto** para prácticamente todos los datasets de texto no estructurado, NLP y RAG en su plataforma a través de la librería `datasets` (respaldada por Apache Arrow).
* **Ejemplo Real:** Los datasets más grandes de texto plano y documentos del mundo (como **FineWeb**, con 15 billones de tokens de texto web crudo, o **RedPajama**) se distribuyen, almacenan y consultan exclusivamente en archivos `.parquet`.

#### B. Apache Spark, Databricks y Ray (Pipelines de Ingesta para RAG)
* **Referencia:** En la arquitectura de referencia de **Databricks para RAG empresarial (Mosaic AI)** y en frameworks de computación distribuida para IA como **Ray Data / Anyscale**:
* **Caso de Uso:** Cuando procesas millones de PDFs o páginas web, la extracción de texto (OCR, parsers como Unstructured o Docling) vuelca los resultados a un Data Lakehouse en formato **Delta Lake / Parquet**.
  * ¿Por qué? Porque permite paralelizar la siguiente fase (la generación de embeddings con GPUs) leyendo lotes de texto en streaming desde Parquet a velocidades de memoria mediante Apache Arrow sin cargar todo el dataset en RAM.

#### C. LanceDB (Base de datos vectorial construida sobre principios columnares)
* **Referencia:** Motores vectoriales de nueva generación como **LanceDB** o formatos como **Lance** (diseñados específicamente para IA multimodal y texto) basan su diseño de bajo nivel en las especificaciones columnares de Parquet/Arrow, demostrando que el acceso columnar es el más eficiente para recuperar tanto fragmentos de texto como vectores densos.

---

### 2. Justificación Técnica: ¿Por qué Parquet y no JSON, JSONL o Blob Storage suelto?

Si tienes 5 millones de páginas de documentos procesados, comparémoslo contra las alternativas:

| Criterio | Archivos individuales (.txt / .json) en S3 | JSONL (.jsonlines comprimido en gzip) | Apache Parquet (con compresión ZSTD o Snappy) |
| :--- | :--- | :--- | :--- |
| **Problema de los archivos pequeños (*Small Files Problem*)** | **Crítico.** Millones de llamadas API `GET` a S3 disparan costos y saturan los límites de transacciones por segundo. | Mitigado (se agrupan en archivos grandes). | **Resuelto.** Miles de documentos se empaquetan en archivos óptimos (ej. de 128 MB a 512 MB). |
| **I/O y Lectura Parcial (*Projection Pushdown*)** | Tienes que descargar todo el archivo para leer cualquier atributo. | Tienes que leer y deserializar toda la línea para ver un solo campo. | **Óptimo.** Si solo quieres leer la columna `clean_text` para generar embeddings y omitir `metadata_json`, Spark/DuckDB solo lee los bytes de esa columna en disco. |
| **Rendimiento de Deserialización** | Lento (parsing de cadenas JSON en CPU). | Lento (parsing intensivo de texto a memoria). | **Casi instantáneo.** Mapeo directo a memoria con Apache Arrow en memoria contigua (zero-copy reads). |
| **Soporte Nativo de Embeddings** | Almacenados como texto `"[0.12, 0.45, ...]"`, ineficiente en tamaño. | Texto en arrays JSON (pesado y lento de parsear). | **Arrays binarios nativos** (`fixed_len_byte_array` o `list<float32>`), ocupando 4 veces menos espacio. |

---

### 3. Ejemplo Concreto: Esquema Parquet en la Capa Silver / Gold

Un archivo Parquet para RAG no fuerza una tabla relacional rígida; aprovecha tipos complejos:

```sql
-- Representación lógica del schema en un archivo Parquet (ej. consultado con DuckDB o Athena)
CREATE TABLE documents_silver (
    document_id VARCHAR,
    source_uri VARCHAR,
    extracted_at TIMESTAMP,
    -- Estructura jerárquica para metadatos variables
    metadata STRUCT<
        author VARCHAR,
        department VARCHAR,
        page_count INT,
        security_tags ARRAY<VARCHAR>
    >,
    -- Array de chunks de texto (no un único texto plano)
    chunks ARRAY<
        STRUCT<
            chunk_id VARCHAR,
            page_number INT,
            text_content VARCHAR,
            -- El vector denso guardado en el mismo registro en Gold
            embedding ARRAY<FLOAT> 
        >
    >
);
```

### 4. ¿Cuándo SÍ sería un antipatrón?

Para mantener el rigor arquitectónico, usar Parquet **sí es un error** en dos escenarios:

1. **Para servir búsquedas en tiempo real (*Low-Latency Online Serving*):** 
   * Parquet es un formato **OLAP / Batch**, no OLTP ni un motor de búsqueda.
   * **Antipatrón:** Que el agente o microservicio consulte directamente un archivo Parquet con SQL en S3 para responder a un usuario final en menos de 200 ms. 
   * **Patrón Correcto:** El pipeline de datos escribe en Parquet en el lago (como repositorio maestro inmutable), y luego un proceso de sincronización *empuja* esos datos hacia el motor de búsqueda en tiempo real (un índice HNSW en Postgres `pgvector`, Qdrant o AWS OpenSearch).

2. **Como reemplazo del documento fuente original (Bronze):**
   * El PDF, imagen o documento Word original debe preservarse intacto en S3/Blob por razones legales, de auditoría y para posibles re-extracciones futuras con mejores modelos multimodales. Parquet es para el texto derivado y procesado.

### Resumen
Guardar texto procesado, chunks y vectores en Parquet dentro de la capa Silver/Gold de tu lago de datos en AWS (S3) no solo es una práctica recomendada, sino que es **el estándar utilizado por las plataformas de ingeniería de datos para IA más avanzadas del mercado** para procesar, versionar y reprocesar conocimiento antes de proyectarlo al motor de búsqueda final.