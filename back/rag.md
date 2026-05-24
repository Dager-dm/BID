# RAG para BID: guía completa de implementación

Este documento define qué necesita el sistema RAG de Business Idea Discovery (BID) y cómo construirlo sobre PostgreSQL con pgvector, sirviéndolo a través de PostgREST y consumiéndolo desde n8n + ChatGPT. La idea es que el RAG aporte contexto colombiano, coaching claro y evidencia recuperada antes de responder al usuario.

---

## 1. Objetivo del RAG

El RAG debe ayudar a BID a responder con contexto real, no con respuestas genéricas. Su función es recuperar información útil sobre emprendimiento, Colombia, cultura, validación de ideas, problemas frecuentes y ejemplos concretos para que la IA responda como coach.

El sistema debe estar pensado para jóvenes emprendedores, así que el contenido debe ser fácil de entender, con lenguaje simple y recomendaciones accionables.

---

## 2. Qué debe contener

### 2.1 Documentos base

Empieza con estos bloques de información:

- Contexto del proyecto BID.
- Guía de coaching y tono de respuesta.
- Glosario simple de emprendimiento.
- Contexto colombiano: economía, cultura, informalidad, inseguridad, canales y hábitos de consumo.
- Casos de referencia y aprendizajes.
- Reglas de negocio y validación.
- Preguntas frecuentes y respuestas modelo. [web:383][web:348][web:376][web:385]

### 2.2 Metadatos necesarios

Cada documento o fragmento debería guardar:

- `source_type`.
- `title`.
- `content`.
- `tags`.
- `country`.
- `region`.
- `sector`.
- `audience_level`.
- `language_style`.
- `created_at`.
- `updated_at`. [web:397][web:401][web:404][web:407]

### 2.3 Tipos de contenido

El RAG debe diferenciar entre:

- conocimiento general,
- conocimiento local colombiano,
- guía pedagógica,
- casos reales,
- reglas y restricciones,
- ejemplos prácticos. [web:383][web:348][web:405][web:428]

---

## 3. Arquitectura

El RAG tendrá cinco partes:

1. **Ingesta**: cargar documentos desde Markdown, PDF, HTML o texto.
2. **Chunking**: partir el contenido en fragmentos pequeños.
3. **Embeddings**: convertir cada fragmento en vector.
4. **Almacenamiento**: guardar texto + vector en PostgreSQL con pgvector.
5. **Recuperación**: consultar por similitud cuando el usuario haga una pregunta. [web:377][web:423][web:425][web:428]

Esto permite que PostgREST exponga la búsqueda semántica como una API consumible por n8n. [web:397][web:401][web:405][web:407]

---

## 4. Esquema de base de datos

### 4.1 Tabla de documentos

```sql
create extension if not exists vector;

create table rag_documents (
  id bigserial primary key,
  source_type text not null,
  title text not null,
  content text not null,
  tags text[] default '{}',
  country text,
  region text,
  sector text,
  audience_level text,
  language_style text,
  metadata jsonb default '{}'::jsonb,
  embedding vector(1536),
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);
```

### 4.2 Índice vectorial

```sql
create index rag_documents_embedding_idx
on rag_documents
using hnsw (embedding vector_cosine_ops);
```

Ese patrón es el más práctico para RAG con pgvector en PostgreSQL. [web:397][web:401][web:404][web:407]

### 4.3 Tabla de chunks

Si quieres más control, guarda también los fragmentos:

```sql
create table rag_chunks (
  id bigserial primary key,
  document_id bigint references rag_documents(id) on delete cascade,
  chunk_index int not null,
  chunk_text text not null,
  chunk_metadata jsonb default '{}'::jsonb,
  embedding vector(1536),
  created_at timestamptz default now()
);
```

---

## 5. Funciones RPC

### 5.1 Búsqueda semántica

```sql
create or replace function match_rag_documents(
  query_embedding vector(1536),
  match_threshold float default 0.75,
  match_count int default 5
)
returns table (
  id bigint,
  source_type text,
  title text,
  content text,
  tags text[],
  metadata jsonb,
  similarity float
)
language plpgsql
as $$
begin
  return query
  select
    d.id,
    d.source_type,
    d.title,
    d.content,
    d.tags,
    d.metadata,
    1 - (d.embedding <=> query_embedding) as similarity
  from rag_documents d
  where 1 - (d.embedding <=> query_embedding) > match_threshold
  order by d.embedding <=> query_embedding
  limit match_count;
end;
$$;
```

### 5.2 Búsqueda por filtros

Puedes crear otra función para filtrar por país, sector o audiencia:

```sql
create or replace function match_rag_documents_filtered(
  query_embedding vector(1536),
  p_country text default null,
  p_sector text default null,
  p_audience_level text default null,
  match_threshold float default 0.75,
  match_count int default 5
)
returns table (
  id bigint,
  title text,
  content text,
  similarity float,
  metadata jsonb
)
language plpgsql
as $$
begin
  return query
  select
    d.id,
    d.title,
    d.content,
    1 - (d.embedding <=> query_embedding) as similarity,
    d.metadata
  from rag_documents d
  where (p_country is null or d.country = p_country)
    and (p_sector is null or d.sector = p_sector)
    and (p_audience_level is null or d.audience_level = p_audience_level)
    and 1 - (d.embedding <=> query_embedding) > match_threshold
  order by d.embedding <=> query_embedding
  limit match_count;
end;
$$;
```

PostgREST puede exponer estas funciones como endpoints RPC. [web:401][web:405][web:407][web:411]

---

## 6. Flujo de ingestión

### 6.1 Entrada

Los documentos pueden entrar por:

- upload manual,
- carpeta compartida,
- formulario n8n,
- Google Drive,
- endpoint HTTP. [web:362][web:376][web:386][web:423]

### 6.2 Procesamiento

El flujo debe:

1. leer el archivo,
2. extraer texto,
3. dividir en chunks,
4. generar embeddings,
5. guardar documentos y chunks,
6. marcar metadatos,
7. habilitar búsqueda. [web:377][web:423][web:426][web:428]

### 6.3 Recomendación de chunking

- chunk size: entre 400 y 800 tokens,
- overlap: entre 50 y 120 tokens,
- guardar siempre el documento original y el índice del chunk. [web:422][web:423][web:426]

---

## 7. Flujo de consulta

Cuando un usuario mande una pregunta o una idea:

1. n8n recibe el texto.
2. Se genera el embedding de la consulta.
3. Se llama a `match_rag_documents` por PostgREST.
4. Se obtienen los fragmentos más relevantes.
5. Se arma un contexto compacto.
6. ChatGPT responde usando ese contexto.
7. Se devuelve un JSON con análisis y coaching. [web:377][web:382][web:405][web:423]

---

## 8. Qué documentos subir primero

### Prioridad alta

- Documento de visión BID.
- Documento de coaching.
- Glosario básico.
- Documento de contexto colombiano.
- Reglas de validación de idea.

### Prioridad media

- Casos de emprendimiento.
- FAQs.
- Ejemplos de respuestas.

### Prioridad baja

- Material de apoyo o documentación secundaria. [web:383][web:348][web:385][web:376]

---

## 9. Cómo conectarlo con n8n

### Nodo 1: Trigger

- Webhook o form submit.

### Nodo 2: Embedding

- Generar vector de la consulta.

### Nodo 3: PostgREST

- Llamar RPC `match_rag_documents`.

### Nodo 4: Prompt builder

- Unir pregunta + fragmentos + contexto del usuario.

### Nodo 5: ChatGPT

- Generar respuesta final.

### Nodo 6: Guardado

- Registrar query, chunks usados y salida. [web:377][web:423][web:424][web:426]

---

## 10. Reglas de calidad

- No guardar documentos duplicados.
- No usar documentos sin metadatos mínimos.
- No mezclar contenido genérico con local sin marcarlo.
- No responder con fragmentos sin relevancia suficiente.
- Priorizar fuentes claras, cortas y actualizadas. [web:383][web:385][web:399][web:405]

---

## 11. Formato de salida del RAG

La salida ideal para n8n debería ser algo así:

```json
{
  "query": "",
  "top_matches": [
    {
      "title": "",
      "similarity": 0.0,
      "content": "",
      "source_type": "",
      "tags": []
    }
  ],
  "context_summary": "",
  "answer_ready_context": "",
  "confidence": 0.0
}
```

---

## 12. Recomendación de implementación

Para BID, la ruta más simple y sólida es:

- PostgreSQL + pgvector,
- PostgREST para exponer RPC,
- n8n para orquestar,
- ChatGPT para redactar,
- documentos en Markdown como base del conocimiento. [web:397][web:401][web:405][web:423]

Así no dependes de un vector DB aparte y mantienes todo dentro de tu stack principal. [web:402][web:405][web:408]

---

## 13. Checklist de construcción

- [ ] Crear tablas `rag_documents` y `rag_chunks`.
- [ ] Activar `pgvector`.
- [ ] Crear índice vectorial.
- [ ] Crear función RPC de búsqueda.
- [ ] Preparar documentos base.
- [ ] Construir flujo n8n de ingestión.
- [ ] Construir flujo n8n de consulta.
- [ ] Definir prompt final para ChatGPT.
- [ ] Probar con casos reales de jóvenes emprendedores.
- [ ] Guardar feedback para mejorar el RAG. [web:377][web:423][web:426][web:428]

---

## 14. Resultado esperado

Con este RAG, BID podrá responder como un coach con memoria: entenderá la idea, recuperará contexto útil, explicará en lenguaje claro y sugerirá pasos realistas para jóvenes emprendedores en Colombia. [web:348][web:383][web:405][web:411]
