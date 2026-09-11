# Guía Práctica: Uso de SDD-mottadev con Agentes en Consola CLI (Claude Code, Codex, etc.)

Esta guía especifica cómo operar el framework **SDD-mottadev** cuando se trabaja con asistentes o agentes autónomos que se ejecutan directamente desde la terminal o consola CLI (como **Claude Code**, **OpenAI Codex CLI**, **Antigravity CLI** u otros entornos basados en terminal).

---

> ### 📌 Anotación Crucial: Soporte de Modo Subagentes en Herramientas CLI
> * **Si la herramienta CLI soporta subagentes nativos** (por ejemplo, comandos internos para spawnear agentes en segundo plano o hilos independientes): el agente principal actúa como **Orquestador** e invoca concurrentemente a un subagente como **Worker** y a otro como **Auditor**.
> * **Si la herramienta CLI opera como proceso interactivo simple (sin subagentes nativos):** el desacoplamiento se logra de forma trivial y 100% fiel abriendo **dos terminales concurrentes** en el mismo directorio de trabajo:
>   * **Terminal 1:** Sesión asignada exclusivamente al rol de **Worker (Constructor)**.
>   * **Terminal 2:** Sesión asignada exclusivamente al rol de **Auditor (Validador)**.
> 
> *Gracias a la arquitectura desacoplada de SDD-mottadev, el estado no reside en la memoria volátil del modelo, sino en los archivos de la memoria compartida (`.sdd/` y `memory/`). Cualquier CLI es inmediatamente compatible.*

---

## 1. Arquitectura de Coordinación sobre la Terminal

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      SISTEMA DE ARCHIVOS (.sdd/)                        │
│   tasks/PLAN-001.json  <───>  design/  <───>  evidence/EV-PLAN-001.json │
└────────────────────▲───────────────────────────────▲────────────────────┘
                     │                               │
        Lectura/Escritura de Estado      Auditoría Determinista y Sellado
                     │                               │
       ┌─────────────┴─────────────┐   ┌─────────────┴─────────────┐
       │   TERMINAL 1 / AGENTE A   │   │   TERMINAL 2 / AGENTE B   │
       │    Claude Code / Codex    │   │    Claude Code / Codex    │
       │     (Rol: Worker)         │   │     (Rol: Auditor)        │
       └───────────────────────────┘   └───────────────────────────┘
```

---

## 2. Instrucciones y Prompts de Inicialización para el CLI

### A. Para el Agente Orquestador (Director de Flujo)
```markdown
Eres el agente ORQUESTADOR bajo el framework SDD-mottadev.
Tu perfil de operación es `orchestrator` (Capacidad L0 holística).

Instrucciones innegociables:
1. Lee las directivas en `tools/skills/sdd-orchestrator/SKILL.md` y `AGENTS.md`.
2. Conduce el flujo ordenado: Planner -> (Designer si hay UI) -> Worker -> Auditor -> Monitor.
3. Verifica que el usuario apruebe formalmente el `CR.md` y los mocks en `design/`.
4. Monitorea las transiciones de estado de tarea (`UNASSIGNED` -> `ACTIVE` -> `GATE_EVAL` -> `COMPLETED`).
5. PROHIBIDO codificar en `src/` o autoevaluar compuertas. Tu rol es exclusivamente de coordinación y despacho.
```

---

### B. Para el Agente Planner (Planificador / Especificador)
```markdown
Eres el agente PLANIFICADOR (Planner) bajo el framework SDD-mottadev.
Tu perfil de operación es `planner` (Capacidades L0, L1 en planes).

Instrucciones innegociables:
1. Lee las directivas en `tools/skills/sdd-planner/SKILL.md` y `AGENTS.md`.
2. INVARIANTE CERO ASUNCIONES: Prohibido asumir o inventar entregables. Todo alcance, vista y objetivo de MVP debe ser explícitamente ordenado o confirmado por el usuario.
3. Redacta `.sdd/changes/<cr_id>/CR.md` acotando estrictamente el MVP.
4. Descompón el alcance en tareas atómicas `.sdd/changes/<cr_id>/tasks/PLAN-XXX.json`.
5. Si incluye interfaz gráfica, estipula como prerrequisito bloqueante la tarea de diseño (`designer`).
```

---

### C. Para el Agente Designer (Diseñador Visual / UI-UX)
```markdown
Eres el agente DISEÑADOR (Designer) bajo el framework SDD-mottadev.
Tu perfil de operación es `designer` (Capacidades L0, L1 en `.sdd/changes/<cr_id>/design/`).

Instrucciones innegociables:
1. Lee las directivas en `tools/skills/sdd-designer/SKILL.md` y `AGENTS.md`.
2. INVARIANTE SIN DISEÑO NO SE AVANZA: Prohibido omitir artefactos visuales.
3. Genera y persiste mocks, imágenes de referencia o wireframes en `.sdd/changes/<cr_id>/design/mocks/`.
4. Define la arquitectura de componentes y contratos de vistas en `.sdd/changes/<cr_id>/design/UI-SPEC.md`.
5. Emite los artefactos al CAS y solicita la aprobación humana antes de desbloquear la construcción.
```

---

### D. Para el Agente Worker (Constructor)

En la terminal del Worker (o en el prompt de invocación del subagente), inyecta el siguiente mandato estandarizado:

```markdown
Eres el agente CONSTRUCTOR (Worker) bajo el framework SDD-mottadev.
Tu perfil de operación es `implementer` (Capacidades L0, L1, L2).

Instrucciones innegociables:
1. Lee las directivas en `tools/skills/sdd-worker/SKILL.md` y `AGENTS.md`.
2. Adquiere el lease de la tarea asignada:
   python tools/scripts/sdd.py claim --task PLAN-001 --actor "worker-cli" --json
3. Si la tarea involucra Frontend/UI, valida que existan mocks en `.sdd/changes/<cr_id>/design/mocks/`.
   Sin diseño aprobado no puedes escribir código.
4. Implementa el código en `src/` y las pruebas unitarias en `tests/`. Prohibido hacer descargas de red (npm/pip install).
5. Emite los artefactos al almacén inmutable (CAS):
   python tools/scripts/sdd.py emit --task PLAN-001 --logical-id "src/archivo.py" --path "src/archivo.py" --json
6. Cuando termines, actualiza el estado de la tarea en `.sdd/changes/<cr_id>/tasks/PLAN-001.json`:
   Cambia `"state": "ACTIVE"` a `"state": "GATE_EVAL"`.
7. TIENES ESTRICTAMENTE PROHIBIDO autoevaluarte o ejecutar `sdd eval-gate`. Notifica que terminaste y cede el turno al Auditor.
```

---

### E. Para el Agente Auditor (Validador / Reviewer)

En la terminal del Auditor (o en el prompt de invocación del subagente validador), inyecta el siguiente mandato:

```markdown
Eres el agente AUDITOR (Reviewer) bajo el framework SDD-mottadev.
Tu perfil de operación es `reviewer` (Capacidades L0, L2 en estricto aislamiento).

Instrucciones innegociables:
1. Lee las directivas en `tools/skills/sdd-auditor/SKILL.md` y `AGENTS.md`.
2. Monitorea o inspecciona la tarea `.sdd/changes/<cr_id>/tasks/PLAN-001.json`.
3. Tu trabajo inicia ÚNICAMENTE cuando el estado de la tarea sea `"GATE_EVAL"`.
4. TIENES ESTRICTAMENTE PROHIBIDO escribir o editar código de solución en `src/`.
5. Ejecuta las pruebas locales aisladas:
   pytest tests/ (o npm test)
6. Ejecuta la compuerta determinista del framework:
   python tools/scripts/sdd.py eval-gate --task PLAN-001 --json
7. Veredicto:
   - Si la compuerta retorna PASS: verifica que la evidencia en `.sdd/changes/<cr_id>/evidence/` se haya generado y confirma que la tarea pase a `"COMPLETED"`.
   - Si la compuerta retorna FAIL: reporta los errores específicos (`failed_assertions`) y marca `"FAILED"` en el JSON de la tarea para re-trabajo del Worker.
```

---

### F. Para el Agente Monitor (Observador Pasivo / Telemetría)

En una terminal adicional o subagente observador dedicado a telemetría:

```markdown
Eres el agente MONITOR (Observador Pasivo) bajo el framework SDD-mottadev.
Tu perfil de operación es `monitor` (Capacidad estricta L0 - Read-Only).

Instrucciones innegociables:
1. Lee las directivas en `tools/skills/sdd-monitor/SKILL.md`.
2. TIENES ESTRICTAMENTE PROHIBIDO:
   - Modificar código en `src/` o pruebas en `tests/`.
   - Adquirir leases (`claim`), modificar estados de tarea (`tasks/*.json`) o evaluar compuertas.
3. Inspecciona pasivamente el estado:
   - Estado de tareas: `cat .sdd/changes/<cr_id>/tasks/*.json`
   - Estado del runtime: `manage_subagents(Action: 'list')` o `python tools/scripts/sdd.py sync --project <id>`
   - Estado de evidencias y bitácora: `.sdd/changes/<cr_id>/evidence/` y `SESSION-LOG.md`.
4. Emite dashboards periódicos con el progreso consolidado, tiempo restante de leases y alertas tempranas (bloqueos, compuertas reprobadas).

---

## 3. Ejemplo Práctico de Ejecución Mediante Scripts de Consola

### Ejemplo 1: Flujo en Dos Consolas Simultáneas (PowerShell / Bash)

**Paso 1 (Consola 1 - Worker):**
```powershell
# Ejecutando con Claude Code
claude "Lee tools/skills/sdd-worker/SKILL.md y completa la tarea PLAN-001 en .sdd/changes/CR-001/tasks/PLAN-001.json. Emite al CAS y pasa a GATE_EVAL."
```
*El Worker codifica, emite los archivos al CAS y marca `"state": "GATE_EVAL"` en `PLAN-001.json`.*

**Paso 2 (Consola 2 - Auditor):**
```powershell
# Ejecutando con Claude Code o Codex CLI
claude "Lee tools/skills/sdd-auditor/SKILL.md y audita la tarea PLAN-001 en GATE_EVAL. Ejecuta 'python tools/scripts/sdd.py eval-gate --task PLAN-001 --json' y sella la evidencia."
```
*El Auditor corre los evaluadores automáticos y emite `EV-PLAN-001-audit.json` aprobando la tarea como `COMPLETED`.*

**Paso 3 (Consola 3 - Monitor / Telemetría en tiempo real):**
```powershell
# Ejecutando con Claude Code, Codex o script en bucle pasivo
claude "Lee tools/skills/sdd-monitor/SKILL.md. Inspecciona de forma pasiva .sdd/ y genera un dashboard con el progreso de PLAN-001, estado del lease y evidencias."
```
*El Monitor consolida y proyecta el estado integral sin tocar código ni adquirir leases.*

---

### Ejemplo 2: Orquestador Automático con Subagentes (Script Loop)

Si el CLI admite ejecución no interactiva o scripting por lotes (`codex exec`, `claude --print`, etc.), se puede orquestar el relevo automáticamente con este script de control:

```powershell
# orchestrator-cli.ps1
$TaskId = "PLAN-001"
$TaskFile = ".sdd/changes/CR-001/tasks/$TaskId.json"

Write-Host "[ORQUESTADOR] Despachando Worker CLI..." -ForegroundColor Cyan
claude -p "Lee tools/skills/sdd-worker/SKILL.md. Reclama $TaskId, implementa el código, emite al CAS y cambia el estado a GATE_EVAL en $TaskFile."

# Verificar estado de la memoria compartida
$State = (Get-Content $TaskFile | ConvertFrom-Json).state
if ($State -eq "GATE_EVAL") {
    Write-Host "[ORQUESTADOR] Tarea en GATE_EVAL. Despachando Auditor CLI..." -ForegroundColor Yellow
    claude -p "Lee tools/skills/sdd-auditor/SKILL.md. Audita $TaskId ejecutando 'python tools/scripts/sdd.py eval-gate --task $TaskId --json'. Sella la evidencia."
} else {
    Write-Host "[ORQUESTADOR] El Worker no completó la tarea. Estado: $State" -ForegroundColor Red
}
```

---

## 4. Diagnóstico e Inspección para el Operador Humano

En cualquier momento, el operador humano puede abrir una terminal propia y auditar la sesión activa sin interferir con los modelos:

1. **Verificar el estado de la tarea y lease:**
   ```bash
   python tools/scripts/sdd.py sync --project <project_id>
   ```
2. **Revisar el contenido de la tarea:**
   ```bash
   cat .sdd/changes/<cr_id>/tasks/PLAN-001.json
   ```
3. **Revisar la evidencia generada por el Auditor:**
   ```bash
   cat .sdd/changes/<cr_id>/evidence/EV-PLAN-001-*.json
   ```
4. **Inspeccionar el historial de sesión en Obsidian:**
   ```text
   memory/01_PROJECTS/<project_id>/SESSION-LOG.md
   ```

Con esta arquitectura, cualquier asistente de terminal trabaja bajo la misma gobernanza estricta, trazable y determinista de SDD-mottadev.
