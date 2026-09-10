# Especificación Técnica: Protocolo de Investigación (`Research Protocol`)

Este documento formaliza el contrato y arquitectura del **Research Protocol** del plano de control de **SDD-mottadev**.

---

## 1. Propósito y Arquitectura de Contexto

El **Research Protocol** resuelve el problema de la degradación del razonamiento (*context drift*) y el consumo excesivo de tokens en agentes de desarrollo. Desacopla la navegación, el análisis exploratorio y la síntesis de fuentes externas del hilo de trabajo principal.

```
┌────────────────────────────────────────────────────────┐
│               Main Implementation Agent                │
└───────────────────────────┬────────────────────────────┘
                            │
                            │ Invocación: ResearchQuery
                            ▼
┌────────────────────────────────────────────────────────┐
│                   Research MCP Worker                  │
│       (Búsqueda, Recuperación, Análisis, Síntesis)     │
└─────────────┬────────────────────────────┬─────────────┘
              │                            │
              │ Persiste corpus crudo      │ Emite payload compacto (<500 tokens)
              ▼                            ▼
┌──────────────────────────┐  ┌──────────────────────────┐
│  Artifact Storage (Disk) │  │   ResearchPayload JSON   │
│  sdd://.../RES-001.md    │  │ (Evidencia, Citas, Score)│
└──────────────────────────┘  └──────────────────────────┘
```

---

## 2. Principios Operativos

1. **Aislamiento Estricto:** El agente principal jamás recibe páginas web completas, HTML crudo, volcados de PDF o transcripciones masivas.
2. **Payload Compacto:** La síntesis retornada al hilo activo debe ser estrictamente menor a 500 tokens (< 3000 caracteres).
3. **Persistencia como Artefacto:** El corpus extendido de la investigación se almacena en el sistema de almacenamiento bajo doble identidad (`LogicalId` + `ContentHash`) y queda referenciado en el payload.
4. **Verificabilidad:** Toda afirmación clave (*key finding*) debe mapearse a una cita textual, una URL de origen y un puntaje de confiabilidad asignado por el proceso de investigación.

---

## 3. Esquema de Entrada: `ResearchQuery`

El agente principal solicita una investigación estructurando los siguientes parámetros (validado contra [`schemas/v1/research-query.json`](../schemas/v1/research-query.json)):

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://sdd-mottadev.org/schemas/v1/research-query.json",
  "title": "ResearchQuery",
  "type": "object",
  "properties": {
    "query_id": {
      "type": "string",
      "pattern": "^REQ-[0-9]{3,4}$"
    },
    "task_id": {
      "type": "string",
      "pattern": "^PLAN-[0-9]{3,4}$"
    },
    "topic": {
      "type": "string",
      "minLength": 5,
      "maxLength": 200
    },
    "depth": {
      "type": "string",
      "enum": ["shallow", "standard", "deep"],
      "default": "standard"
    },
    "focus_areas": {
      "type": "array",
      "items": { "type": "string" },
      "minItems": 1
    },
    "allowed_domains": {
      "type": "array",
      "items": { "type": "string" }
    },
    "blocked_domains": {
      "type": "array",
      "items": { "type": "string" }
    },
    "max_sources": {
      "type": "integer",
      "minimum": 1,
      "maximum": 30,
      "default": 10
    }
  },
  "required": ["query_id", "task_id", "topic", "focus_areas"],
  "additionalProperties": false
}
```

---

## 4. Esquema de Salida: `ResearchPayload`

El worker de investigación procesa las fuentes, guarda el corpus en disco y devuelve al agente exclusivamente esta estructura compacta (validada contra [`schemas/v1/research-payload.json`](../schemas/v1/research-payload.json)):

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://sdd-mottadev.org/schemas/v1/research-payload.json",
  "title": "ResearchPayload",
  "type": "object",
  "properties": {
    "query_id": { "type": "string" },
    "topic": { "type": "string" },
    "summary": {
      "type": "string",
      "description": "Resumen ejecutivo denso y accionable (<500 tokens)",
      "maxLength": 3000
    },
    "key_findings": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "claim": { "type": "string" },
          "evidence_snippet": { "type": "string" },
          "source_id": { "type": "string" },
          "confidence": {
            "type": "number",
            "minimum": 0.0,
            "maximum": 1.0
          }
        },
        "required": ["claim", "evidence_snippet", "source_id", "confidence"],
        "additionalProperties": false
      }
    },
    "sources": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "source_id": { "type": "string" },
          "title": { "type": "string" },
          "url": { "type": "string", "format": "uri" },
          "reliability_score": {
            "type": "number",
            "minimum": 0.0,
            "maximum": 1.0
          }
        },
        "required": ["source_id", "title", "url", "reliability_score"],
        "additionalProperties": false
      }
    },
    "contradictions": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Conflictos encontrados entre distintas fuentes analizadas"
    },
    "corpus_artifact": {
      "type": "object",
      "properties": {
        "logical_id": { "type": "string" },
        "content_hash": { "type": "string" },
        "canonical_uri": { "type": "string" }
      },
      "required": ["logical_id", "content_hash", "canonical_uri"],
      "additionalProperties": false
    }
  },
  "required": ["query_id", "topic", "summary", "key_findings", "sources", "corpus_artifact"],
  "additionalProperties": false
}
```

---

## 5. Estructura del Artefacto Persistido (`Corpus Artifact`)

El corpus crudo no se devuelve en la respuesta RPC/MCP. Se escribe directamente en el adaptador de almacenamiento en la ruta lógica correspondiente a la investigación:

* **URI Canónica:** `sdd://<project-id>/research/artifacts/RES-<query_id>.md`
* **Formato del archivo:**

```markdown
---
logical_id: "urn:sdd:artifact:research/RES-001"
content_hash: "sha256:d41d8cd98f00b204e9800998ecf8427e..."
query_id: "REQ-001"
task_id: "PLAN-002"
created_at: "2026-09-10T12:00:00Z"
evaluator_signature: "ed25519:..."
---

# Corpus de Investigación: [Título del Tema]

## 1. Metodología de Recuperación
- Proveedores utilizados: [Exa / Gemini API / Herramientas locales]
- Filtros de búsqueda aplicados.
- Total de fuentes inspeccionadas.

## 2. Fuentes Primarias Extraídas (Texto Crudo Filtrado)
### Fuente [SRC-1]: [Título]
- URL: [Enlace]
- Fecha de captura: [Timestamp]
- Contenido relevante extraído:
> [Texto textual extenso de la fuente...]

## 3. Matriz de Deduplicación y Cruce de Datos
[Tablas comparativas, discrepancias técnicas detectadas entre versiones de librerías, estándares o autores.]
```

Con este diseño, el agente de código solo procesa el `ResearchPayload` (que cabe en menos de 1 KB en su contexto activo) mientras que la trazabilidad completa, citas y auditoría quedan preservadas de forma inmutable en el repositorio de artefactos.
