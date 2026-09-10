# Especificación Técnica: Reconciliación Concurrente e Integración (`IntegrationContract`)

Este documento formaliza el contrato y algoritmo determinista para fusionar tareas concurrentes desarrolladas en aislamiento hacia el tronco común (`main`) dentro del plano de control de **SDD-mottadev**.

---

## 1. Esquema de Solicitud: `IntegrationRequest`

El `IntegrationRequest` es emitido por el actor ejecutor al finalizar la tarea técnica para solicitar la fusión atómica de sus artefactos al tronco principal. Se valida formalmente contra el JSON Schema ubicado en [`schemas/v1/integration-request.json`](../schemas/v1/integration-request.json).

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://sdd-mottadev.org/schemas/v1/integration-request.json",
  "title": "IntegrationRequest",
  "type": "object",
  "properties": {
    "request_id": { "type": "string", "format": "uuid" },
    "task_id": { "type": "string", "pattern": "^PLAN-[0-9]{3,4}$" },
    "change_id": { "type": "string", "pattern": "^CR-[0-9]{3,4}$" },
    "source_ref": {
      "type": "object",
      "properties": {
        "type": { "type": "string", "enum": ["git_branch", "git_commit", "patch_ref"] },
        "value": { "type": "string" }
      },
      "required": ["type", "value"]
    },
    "target_base_hash": {
      "type": "string",
      "description": "Hash SHA-256 o Git SHA del commit de main sobre el que se basó la tarea"
    },
    "modified_artifacts": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "logical_id": { "type": "string" },
          "content_hash": { "type": "string" }
        },
        "required": ["logical_id", "content_hash"]
      },
      "minItems": 1
    },
    "actor_signature": { "type": "string" }
  },
  "required": ["request_id", "task_id", "change_id", "source_ref", "target_base_hash", "modified_artifacts", "actor_signature"],
  "additionalProperties": false
}
```

---

## 2. Algoritmo de Evaluación de Reconciliación (`MergeEvaluator`)

El `MergeEvaluator` procesa la integración bajo un flujo estricto y determinista:

```
[IntegrationRequest recibido]
             │
             ▼
      ¿Base actual == target_base_hash?
             │
      ┌──────┴──────┐
      │ Sí          │ No
      │             ▼
      │      ¿MergePolicy == STRICT_LINEAR_REBASE?
      │             │
      │      ┌──────┴──────┐
      │      │ Sí          │ No
      │      ▼             ▼
      │  [Rebase local]  [Intentar Merge de 3 vías]
      │      │             │
      │      └──────┬──────┘
      │             │
      ▼             ▼
  ¿Existen conflictos de contenido / sintaxis?
             │
      ┌──────┴──────┐
      │ No          │ Sí
      ▼             ▼
[Crear Sandbox]  TaskState -> BLOCKED (Reason: MERGE_CONFLICT)
      │
      ▼
[Re-ejecutar Compuertas de Fase en Sandbox]
      │
      ├──> ¿Gate == FAIL o ERROR? ──> TaskState -> BLOCKED (Reason: INTEGRATION_GATE_FAIL)
      │
      └──> ¿Gate == PASS?
                 │
                 ▼
          [Aplicar Commit a main]
          [Generar EvidenceRecord de Integración]
          [Liberar Lease]
          [TaskState -> COMPLETED]
```

### Pasos de Ejecución Determinista:

1. **Verificación de Linaje:** El evaluador inspecciona si el commit base sobre el que operó la tarea coincide con el `HEAD` actual del tronco (`main`). Si difieren y la política activa ([`policies/merge-default.yaml`](../policies/merge-default.yaml)) es `STRICT_LINEAR_REBASE`, se intenta un rebase automático en un espacio efímero.
2. **Detección de Colisiones:** Si existen conflictos de parcheo o combinación no resolubles automáticamente, la operación se cancela de inmediato sin tocar el tronco y la tarea pasa al estado `BLOCKED(Reason: MERGE_CONFLICT)`.
3. **Validación de No-Regresión en Sandbox Efímero:** Los cambios integrados jamás se escriben directamente en el tronco. Se materializan en un workspace confinado y se re-ejecutan todas las compuertas obligatorias (`strict` e `immutable`) configuradas para la fase.
4. **Fusión Atómica y Sellado:** Si todas las compuertas en el sandbox emiten `PASS`, se realiza el fast-forward sobre el tronco principal, se registra la evidencia probatoria inmutable de integración (`EvidenceRecord`), se libera el lease y la tarea transiciona a `COMPLETED`.
