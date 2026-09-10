# Especificación Formal de Interfaces: SDD-mottadev

Este documento define los contratos de interfaz de frontera del plano de control (*Control Plane*) para SDD-mottadev:
1. **Agent API (MCP/CLI Contract):** Interfaz para la interacción entre agentes ejecutores y el runtime.
2. **Evaluator Contract:** Interfaz determinista para herramientas de validación y generación de evidencia probatoria.

---

## 1. Agent API (Protocolo MCP / CLI)

### 1.1 Transporte y Convenciones
Las herramientas se exponen mediante dos adaptadores canónicos equivalentes:
* **Transporte MCP:** Servidor local de herramientas sobre `stdio` mediante RPC (`tools/call`).
* **Transporte CLI:** Binario o script POSIX (`sdd <comando> --json`) que interactúa mediante `stdin` y emite JSON estructurado en `stdout`.

#### Códigos de Error Canónicos (`error_code`)
Toda respuesta de error debe retornar un objeto con el campo `error_code` tipado:

| Código | Significado | Transición de Estado de Tarea |
| :--- | :--- | :--- |
| `ERR_TASK_NOT_FOUND` | El `task_id` especificado no existe. | Ninguna |
| `ERR_LEASE_CONFLICT` | La tarea ya está tomada por otro `Actor` con lease vigente. | Ninguna |
| `ERR_LEASE_EXPIRED` | El lease del agente actual expiró por falta de heartbeat. | $\to$ `FAULT` |
| `ERR_INVALID_TRANSITION` | La transición solicitada viola la máquina de estados. | Ninguna |
| `ERR_CAPABILITY_DENIED` | La operación excede el nivel L0-L3 autorizado para el agente. | $\to$ `BLOCKED` |
| `ERR_APPROVAL_REQUIRED` | Operación L3 sin `ApprovalToken` válido. | $\to$ `BLOCKED` |
| `ERR_GATE_EVAL_FAILED` | Una o más aserciones de compuerta retornaron `false`. | $\to$ `FAILED` |
| `ERR_EVALUATOR_RUNTIME` | El evaluador falló operacionalmente (timeout, caída). | $\to$ `FAULT` |
| `ERR_SCHEMA_VIOLATION` | El artefacto o input no cumple con el esquema declarado. | Ninguna |

---

### 1.2 Endpoints y Herramientas

#### `sdd_sync_state`
Obtiene el contexto mínimo necesario para la sesión activa bajo el principio de divulgación progresiva.

* **Firma MCP:** `sdd_sync_state(project_id: string, task_id?: string)`
* **Equivalente CLI:** `sdd sync [--project <id>] [--task <id>]`
* **Input Schema:**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "project_id": { "type": "string" },
    "task_id": { "type": "string" }
  },
  "required": ["project_id"]
}
```

* **Output Schema:**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "project_id": { "type": "string" },
    "active_change_request": {
      "type": "object",
      "properties": {
        "change_id": { "type": "string" },
        "type": { "type": "string", "enum": ["FEATURE", "BUGFIX", "REFACTOR", "CONFIG"] },
        "current_phase": { "type": "string" },
        "rigor_level": { "type": "integer", "enum": [1, 2, 3] }
      },
      "required": ["change_id", "type", "current_phase", "rigor_level"]
    },
    "active_task": {
      "type": "object",
      "properties": {
        "task_id": { "type": "string" },
        "state": { 
          "type": "string", 
          "enum": ["UNASSIGNED", "ACTIVE", "GATE_EVAL", "COMPLETED", "FAILED", "FAULT", "BLOCKED", "CANCELLED"] 
        },
        "assigned_actor_id": { "type": ["string", "null"] },
        "lease_expires_at": { "type": ["string", "null"] },
        "required_inputs": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "logical_id": { "type": "string" },
              "content_hash": { "type": "string" },
              "uri": { "type": "string" }
            },
            "required": ["logical_id", "content_hash", "uri"]
          }
        },
        "target_outputs": { "type": "array", "items": { "type": "string" } },
        "active_policies": { "type": "array", "items": { "type": "string" } }
      },
      "required": ["task_id", "state", "required_inputs", "target_outputs", "active_policies"]
    }
  },
  "required": ["project_id", "active_change_request", "active_task"]
}
```

---

#### `sdd_claim_task`

Adquiere el derecho de ejecución (*lease*) sobre una tarea en estado `UNASSIGNED` o recuperada de `FAULT`.

* **Firma MCP:** `sdd_claim_task(task_id: string, actor_id: string, lease_ttl_seconds: integer)`
* **Equivalente CLI:** `sdd claim --task <id> --actor <id> --ttl <seconds>`
* **Input Schema:**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "task_id": { "type": "string" },
    "actor_id": { "type": "string" },
    "lease_ttl_seconds": { "type": "integer", "minimum": 60, "maximum": 3600 }
  },
  "required": ["task_id", "actor_id", "lease_ttl_seconds"]
}
```

* **Output Schema:**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "task_id": { "type": "string" },
    "status": { "type": "string", "enum": ["ACTIVE"] },
    "lease_granted": {
      "type": "object",
      "properties": {
        "actor_id": { "type": "string" },
        "granted_at": { "type": "string" },
        "expires_at": { "type": "string" },
        "heartbeat_interval_seconds": { "type": "integer" }
      },
      "required": ["actor_id", "granted_at", "expires_at", "heartbeat_interval_seconds"]
    },
    "workspace_ref": { "type": "string" }
  },
  "required": ["task_id", "status", "lease_granted", "workspace_ref"]
}
```

---

#### `sdd_heartbeat`

Renueva el lease de una tarea en estado `ACTIVE` para evitar su marcado como huérfana.

* **Firma MCP:** `sdd_heartbeat(task_id: string, actor_id: string)`
* **Equivalente CLI:** `sdd heartbeat --task <id> --actor <id>`
* **Input Schema:**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "task_id": { "type": "string" },
    "actor_id": { "type": "string" }
  },
  "required": ["task_id", "actor_id"]
}
```

* **Output Schema:**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "task_id": { "type": "string" },
    "status": { "type": "string", "enum": ["ACTIVE"] },
    "extended_expires_at": { "type": "string" }
  },
  "required": ["task_id", "status", "extended_expires_at"]
}
```

---

#### `sdd_emit_artifact`

Registra un nuevo entregable generado por la ejecución, calculando su doble identidad e insertándolo en el árbol de linaje.

* **Firma MCP:** `sdd_emit_artifact(task_id: string, logical_id: string, source_path: string, schema_version: string, parent_hashes?: string[])`
* **Equivalente CLI:** `sdd emit --task <id> --logical-id <id> --path <path> --schema-ver <semver>`
* **Input Schema:**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "task_id": { "type": "string" },
    "logical_id": { "type": "string" },
    "source_path": { "type": "string" },
    "schema_version": { "type": "string" },
    "parent_hashes": {
      "type": "array",
      "items": { "type": "string" }
    }
  },
  "required": ["task_id", "logical_id", "source_path", "schema_version"]
}
```

* **Output Schema:**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "logical_id": { "type": "string" },
    "content_hash": { "type": "string" },
    "canonical_uri": { "type": "string" },
    "registered_at": { "type": "string" }
  },
  "required": ["logical_id", "content_hash", "canonical_uri", "registered_at"]
}
```

---

#### `sdd_request_gate_eval`

Finaliza la computación técnica del agente y solicita al runtime la evaluación determinista de las compuertas de la fase activa.

* **Firma MCP:** `sdd_request_gate_eval(task_id: string)`
* **Equivalente CLI:** `sdd eval-gate --task <id>`
* **Input Schema:**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "task_id": { "type": "string" }
  },
  "required": ["task_id"]
}
```

* **Output Schema:**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "task_id": { "type": "string" },
    "overall_result": { "type": "string", "enum": ["PASS", "FAIL", "ERROR"] },
    "current_state": { "type": "string", "enum": ["COMPLETED", "FAILED", "FAULT"] },
    "evidence_summary": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "evaluator_id": { "type": "string" },
          "status": { "type": "string", "enum": ["PASS", "FAIL", "ERROR"] },
          "exit_code": { "type": "integer" },
          "failed_assertions": { "type": "array", "items": { "type": "string" } },
          "evidence_ref": { "type": "string" }
        },
        "required": ["evaluator_id", "status", "exit_code", "failed_assertions", "evidence_ref"]
      }
    }
  },
  "required": ["task_id", "overall_result", "current_state", "evidence_summary"]
}
```

---

#### `sdd_request_approval`

Genera una solicitud formal de autorización humana para operaciones que requieren capacidad `L3`.

* **Firma MCP:** `sdd_request_approval(task_id: string, capability: string, target_resource: string, justification: string)`
* **Equivalente CLI:** `sdd request-approval --task <id> --cap <L3> --resource <uri> --justification <text>`
* **Input Schema:**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "task_id": { "type": "string" },
    "capability": { "type": "string", "enum": ["L3"] },
    "target_resource": { "type": "string" },
    "justification": { "type": "string" }
  },
  "required": ["task_id", "capability", "target_resource", "justification"]
}
```

* **Output Schema:**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "approval_request_id": { "type": "string" },
    "task_status": { "type": "string", "enum": ["BLOCKED"] },
    "pending_capability": { "type": "string", "enum": ["L3"] },
    "message": { "type": "string" }
  },
  "required": ["approval_request_id", "task_status", "pending_capability", "message"]
}
```

---

## 2. Evaluator Contract (Interfaz Estándar de Validación)

El *Evaluator Contract* desacopla el plano de control de las herramientas reales de validación (linters, compiladores, suites de pruebas, SAST, DAST o auditores).

### 2.1 Invocación y Aislamiento

* **Invocación:** Subproceso ejecutado de forma desatendida mediante CLI, contenedor aislado o RPC.
* **Canales:**
  * `stdin`: Recibe el documento JSON `EvaluatorInput`.
  * `stdout`: Emite exclusivamente el documento JSON `EvaluatorOutput`.
  * `stderr`: Logs brutos de diagnóstico del motor subyacente.

* **Códigos de Salida POSIX:**
  * `0`: Ejecución finalizada correctamente (evaluación completada; aserciones pueden ser `true` o `false`).
  * `1`: Falla de aserción directa.
  * `2` o superior: Error operacional o caída de infraestructura (`ERROR` en el runtime).

---

### 2.2 Esquema de Entrada: `EvaluatorInput`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "evaluator_id": { "type": "string" },
    "task_id": { "type": "string" },
    "target_workspace": { "type": "string" },
    "target_artifacts": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "logical_id": { "type": "string" },
          "content_hash": { "type": "string" },
          "path": { "type": "string" }
        },
        "required": ["logical_id", "content_hash", "path"]
      }
    },
    "configuration": {
      "type": "object",
      "additionalProperties": true
    },
    "timeout_seconds": { "type": "integer", "default": 300 }
  },
  "required": ["evaluator_id", "task_id", "target_workspace", "target_artifacts"]
}
```

---

### 2.3 Esquema de Salida: `EvaluatorOutput`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "evaluator_id": { "type": "string" },
    "execution_status": {
      "type": "string",
      "enum": ["PASS", "FAIL", "ERROR"]
    },
    "duration_ms": { "type": "integer" },
    "assertions": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "assertion_id": { "type": "string" },
          "passed": { "type": "boolean" },
          "severity": { "type": "string", "enum": ["LOW", "MEDIUM", "HIGH", "CRITICAL"] },
          "message": { "type": "string" },
          "location": {
            "type": "object",
            "properties": {
              "file": { "type": "string" },
              "line": { "type": "integer" }
            }
          }
        },
        "required": ["assertion_id", "passed", "severity", "message"]
      }
    },
    "telemetry": {
      "type": "object",
      "properties": {
        "metrics_evaluated": { "type": "integer" },
        "coverage_percentage": { "type": "number" }
      },
      "additionalProperties": true
    },
    "raw_output_ref": { "type": "string" }
  },
  "required": ["evaluator_id", "execution_status", "duration_ms", "assertions"]
}
```

---

### 2.4 Lógica de Decisión Determinista del Gate

El runtime procesa el `EvaluatorOutput` bajo las siguientes reglas ordenadas:

1. **Evaluación de `ERROR`:**
Si el proceso finaliza con código POSIX $\ge 2$, timeout o `execution_status == "ERROR"`:

$$\text{GateResult} \leftarrow \text{ERROR} \implies \text{TaskState} \leftarrow \text{FAULT}$$

2. **Evaluación de `FAIL`:**
Si existe al menos una aserción no satisfecha con severidad restrictiva:

$$\exists \, a \in \text{assertions} : a.\text{passed} == \text{false} \land (\text{GateRigor} == \text{"strict"} \lor a.\text{severity} \in \{\text{"HIGH"}, \text{"CRITICAL"}\})$$

$$\text{GateResult} \leftarrow \text{FAIL} \implies \text{TaskState} \leftarrow \text{FAILED}$$

3. **Evaluación de `PASS`:**
Si todas las aserciones obligatorias son verdaderas:

$$\forall \, a \in \text{assertions} : a.\text{passed} == \text{true}$$

$$\text{GateResult} \leftarrow \text{PASS} \implies \text{TaskState} \leftarrow \text{COMPLETED}$$
