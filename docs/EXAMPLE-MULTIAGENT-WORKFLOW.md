# Guía de Referencia y Ejemplo: Flujo de Trabajo Multiagente (Worker / Auditor)

Este documento es una guía canónica para que tanto **operadores humanos** como **agentes autónomos de IA** comprendan y ejecuten el flujo de trabajo colaborativo bajo el framework **SDD-mottadev**.

---

## 1. Principio Fundamental: Separación Productor-Evaluador

En SDD-mottadev, el trabajo se desacopla rigurosamente entre dos roles especializados:

```
┌────────────────────────────────────────────────────────────────────────┐
│                          Orquestador / Planner                         │
│             (Despacha tareas, coordina y monitorea estados)            │
└───────────────────┬────────────────────────────────┬───────────────────┘
                    │                                │
                    ▼                                ▼
       ┌────────────────────────┐       ┌────────────────────────┐
       │   Constructor (Worker) │       │   Auditor (Reviewer)   │
       │   Perfil: implementer  │       │   Perfil: reviewer     │
       │   Capacidades: L0,L1,L2│       │   Capacidades: L0, L2  │
       └────────────┬───────────┘       └────────────▲───────────┘
                    │                                │
                    ▼                                │
  ┌────────────────────────────────────────────────────────────────────┐
  │                    MEMORIA COMPARTIDA (.sdd/)                      │
  │  1. tasks/PLAN-001.json (Sincronización: ACTIVE -> GATE_EVAL)      │
  │  2. design/ (Mocks y contratos visuales aprobados)                 │
  │  3. evidence/EV-PLAN-001.json (Veredicto sellado e inmutable)      │
  └────────────────────────────────────────────────────────────────────┘
```

* **Constructor (Worker):** Codifica, crea tests unitarios y emite artefactos. **Tiene prohibido autoevaluarse.**
* **Auditor (Reviewer):** Inspecciona de forma aislada, corre suites de compuertas deterministas y sella evidencias probatorias. **Tiene prohibido escribir código de solución (`src/`).**
* **Memoria Compartida Agnóstica:** Ningún agente comparte memoria RAM ni procesos en ejecución. La sincronización se realiza exclusivamente a través de los archivos estructurados en `.sdd/`.

---

## 2. Ejemplo Práctico de Extremo a Extremo

### Escenario
Se desea implementar la pantalla de Login y su servicio de autenticación en el proyecto (`ChangeRequest: CR-001`).

---

### Paso 1: Especificación y Diseño Aprobado (Humano + Planner)
Por la invariante de frontend, antes de generar tareas de código, deben existir los artefactos visuales en:
```text
.sdd/changes/CR-001/design/
├── mocks/
│   └── login-view-mock.png
└── UI-SPEC.md
```
El usuario humano valida y aprueba el diseño.

---

### Paso 2: Creación de la Tarea en Memoria Compartida
El Orquestador crea `.sdd/changes/CR-001/tasks/PLAN-001.json`:

```json
{
  "task_id": "PLAN-001",
  "title": "Implementar vista de Login y cliente de autenticación",
  "state": "UNASSIGNED",
  "assigned_actor_id": null,
  "lease_expires_at": null,
  "required_inputs": [
    {
      "logical_id": ".sdd/changes/CR-001/design/UI-SPEC.md",
      "content_hash": "a1b2c3d4e5f6..."
    }
  ],
  "target_outputs": [
    "src/components/LoginForm.tsx",
    "tests/components/LoginForm.test.tsx"
  ],
  "active_policies": ["authz-default", "risk-default"]
}
```

---

### Paso 3: El Constructor (Worker) en Acción

1. **Toma de Lease (Claim):**
   El Worker ejecuta el reclamo de la tarea:
   ```bash
   python tools/scripts/sdd.py claim --task PLAN-001 --actor "worker-agent-1" --json
   ```
   El archivo `PLAN-001.json` se actualiza automáticamente:
   ```json
   {
     "state": "ACTIVE",
     "assigned_actor_id": "worker-agent-1",
     "lease_expires_at": "2026-09-10T19:30:00Z"
   }
   ```

2. **Verificación de Precondición:**
   El Worker comprueba que `.sdd/changes/CR-001/design/mocks/` contenga los mocks aprobados. Como existen, procede a codificar.

3. **Codificación:**
   Escribe el código en `src/components/LoginForm.tsx` y las pruebas en `tests/components/LoginForm.test.tsx`.

4. **Emisión de Artefactos al Almacén Inmutable (CAS):**
   ```bash
   python tools/scripts/sdd.py emit --task PLAN-001 --logical-id "src/components/LoginForm.tsx" --path "src/components/LoginForm.tsx" --json
   python tools/scripts/sdd.py emit --task PLAN-001 --logical-id "tests/components/LoginForm.test.tsx" --path "tests/components/LoginForm.test.tsx" --json
   ```

5. **Transición a Evaluación (`GATE_EVAL`):**
   El Worker actualiza el estado de la tarea en `PLAN-001.json`:
   ```json
   {
     "state": "GATE_EVAL"
   }
   ```
   El Worker notifica: *"PLAN-001 lista para evaluación de compuertas."* y pasa a estado inactivo (`idle`).

---

### Paso 4: El Auditor (Reviewer) en Acción

1. **Detección del Relevo:**
   El Auditor detecta que `PLAN-001.json` tiene `state: "GATE_EVAL"`.

2. **Ejecución Local de Pruebas (L2 en aislamiento):**
   ```bash
   npm test tests/components/LoginForm.test.tsx
   ```

3. **Ejecución Determinista de Compuerta:**
   ```bash
   python tools/scripts/sdd.py eval-gate --task PLAN-001 --json
   ```
   El motor valida:
   * Linters de sintaxis (`syntax-linter`).
   * Cero secretos en código (`security-scanner`).
   * Trazabilidad y gobernanza (`vault-governance`).

4. **Emisión de Evidencia Probatoria:**
   El sistema sella `.sdd/changes/CR-001/evidence/EV-PLAN-001-audit.json`:
   ```json
   {
     "evidence_id": "EV-PLAN-001-audit",
     "task_id": "PLAN-001",
     "evaluator": "reviewer-agent-1",
     "verdict": "PASS",
     "assertions_passed": 12,
     "assertions_failed": 0,
     "timestamp": "2026-09-10T19:15:00Z"
   }
   ```

5. **Sellado de Tarea:**
   La tarea se marca como aprobada en `PLAN-001.json`:
   ```json
   {
     "state": "COMPLETED"
   }
   ```
   El Auditor reporta al Orquestador y al operador humano: *"PLAN-001 aprobada y sellada."*

---

## 3. ¿Cómo Monitorear el Estado en Tiempo Real?

### A. Para Humanos u Orquestadores (Inspección de Subagentes)
Usa la herramienta `manage_subagents(Action: 'list')`. La salida reportará:
* **Worker:** `state: "idle"` o `"running"` (en qué tool call se encuentra).
* **Auditor:** `state: "running"` (evaluando) o `"idle"` (esperando el siguiente lote).

### B. Para Inspección en Disco
* Abre `.sdd/changes/<cr_id>/tasks/PLAN-XXX.json` para ver el estado del ciclo (`ACTIVE`, `GATE_EVAL`, `COMPLETED`, `FAILED`, `BLOCKED`).
* Abre `memory/01_PROJECTS/<project_id>/SESSION-LOG.md` para el resumen humano compacto de la sesión.

---

## 4. Matriz Rápida de Comandos para Agentes

| Acción | Rol | Comando |
| :--- | :--- | :--- |
| Reclamar tarea | Worker | `python tools/scripts/sdd.py claim --task <ID> --actor <NAME>` |
| Emitir artefacto | Worker | `python tools/scripts/sdd.py emit --task <ID> --logical-id <PATH> --path <PATH>` |
| Enviar a revisión | Worker | Modificar `"state": "GATE_EVAL"` en `tasks/<ID>.json` |
| Auditar compuertas | Auditor | `python tools/scripts/sdd.py eval-gate --task <ID>` |
| Consultar estado | Todos | `python tools/scripts/sdd.py sync --project <ID>` |
