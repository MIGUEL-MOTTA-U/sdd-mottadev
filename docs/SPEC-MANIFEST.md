# Especificación Técnica: Manifiesto de Proyecto (`sdd.manifest.yaml`)

El archivo `sdd.manifest.yaml` reside en la raíz del repositorio o dentro de `.sdd/` y actúa como la especificación declarativa del plano de control para el framework **SDD-mottadev**. Define la identidad del proyecto, las políticas activas, el catálogo de evaluadores y el grafo de fases con soporte para extensiones (*hooks*).

---

## 1. Esquema Declarativo y Componentes

El manifiesto se valida estrictamente contra el JSON Schema Draft-07 ubicado en [`schemas/v1/manifest.json`](../schemas/v1/manifest.json).

### 1.1 `project` (Identidad y Rigor Basal)
* **`id`**: Identificador canónico del proyecto con formato `urn:sdd:project:<nombre>`.
* **`name`**: Nombre descriptivo del sistema o servicio.
* **`default_rigor`**: Nivel de rigor predeterminado para las compuertas de evaluación:
  * `1`: MVP (compuertas básicas orientadas a prototipado rápido).
  * `2`: Standard (rigor recomendado para software productivo).
  * `3`: High-Criticality (máxima rigidez; aserciones extendidas y trazabilidad exhaustiva).

### 1.2 `workspaces` (Límites de Mutación en Disco)
Define las fronteras de mutación permitidas para tareas ejecutadas bajo nivel de privilegio **L1**:
* **`root`**: Directorio raíz del proyecto (típicamente `.` o ruta relativa).
* **`allowed_mutations`**: Lista de patrones glob de archivos y directorios que los agentes implementadores tienen permitido alterar (ej. `src/**`, `tests/**`, `docs/**`).
* **`denied_mutations`**: Lista de patrones glob estrictamente protegidos cuya mutación está vedada bajo L1 (ej. `.git/**`, `.sdd/policies/**`, `infra/production/**`).

### 1.3 `budgets` (Presupuestos Operacionales y Resiliencia)
* **`task_max_eval_attempts`**: Límite de reintentos de evaluación determinista de compuertas antes de que una tarea sea escalada a estado `BLOCKED` (default: 3).
* **`lease_default_ttl_seconds`**: Ventana de tiempo en segundos (TTL) otorgada a un actor al tomar una tarea (default: 900s). Exige renovación periódica mediante `Heartbeat`.
* **`circuit_breaker_error_threshold`**: Umbral consecutivo de errores operacionales (`FAULT` / `ERROR`) tras el cual se interrumpe la asignación automática de tareas (default: 5).

### 1.4 `evaluators` (Catálogo de Evaluadores Registrados)
Diccionario de evaluadores independientes que implementan el [Evaluator Contract](SPEC-INTERFACES.md#2-evaluator-contract-interfaz-estándar-de-validación):
* **`command`**: Ruta o comando ejecutable (ej. `./scripts/evaluators/run-tests.sh`).
* **`timeout_seconds`**: Límite estricto de tiempo de ejecución antes de considerar la evaluación como falla operacional (`ERROR`).
* **`environment`**: Variables de entorno opcionales inyectadas al evaluador.

### 1.5 `tracks` (Definición de Pipelines y Compuertas)
Estructuración de las dos vías de ejecución:
* **`configuration`**: Pipeline de preparación y certificación del entorno:
  * `specs-config` $\to$ `planning-config` $\to$ `configuration` $\to$ `config-testing` $\to$ `config-security-analysis` [COMPUERTA INMUTABLE].
* **`delivery`**: Pipeline de materialización de valor de negocio:
  * Fases canónicas: `discovery`, `specs`, `planning`, `design`, `testing`, `implementing`, `security-analysis` [COMPUERTA INMUTABLE], `deploying`, `monitoring`.
  * **Fases personalizadas (`type: custom`)**: Permiten inyectar validaciones de dominio intermedias especificando su posición relativa mediante `position.before` o `position.after`.

---

## 2. Ejemplo Canónico de Manifiesto

Un ejemplo completo y funcional se encuentra versionado en [`examples/sdd.manifest.yaml`](../examples/sdd.manifest.yaml):

```yaml
version: "1.0.0"
project:
  id: "urn:sdd:project:auth-gateway"
  name: "Authentication Gateway Service"
  default_rigor: 2 # 1: MVP | 2: Standard | 3: High-Criticality

workspaces:
  root: "."
  allowed_mutations:
    - "src/**"
    - "tests/**"
    - "docs/**"
  denied_mutations:
    - ".git/**"
    - ".sdd/policies/**"
    - "infra/production/**"

budgets:
  task_max_eval_attempts: 3
  lease_default_ttl_seconds: 900
  circuit_breaker_error_threshold: 5

evaluators:
  linter-runner:
    command: "./scripts/evaluators/run-linter.sh"
    timeout_seconds: 120
  unit-test-runner:
    command: "./scripts/evaluators/run-tests.sh"
    timeout_seconds: 300
  security-scanner:
    command: "./scripts/evaluators/run-security.sh"
    timeout_seconds: 600

tracks:
  configuration:
    enabled: true
    pipeline:
      - id: "specs-config"
        gate:
          rigor: "strict"
          evaluator_ref: "linter-runner"
      - id: "planning-config"
        gate:
          rigor: "strict"
          evaluator_ref: "linter-runner"
      - id: "configuration"
        gate:
          rigor: "strict"
          evaluator_ref: "unit-test-runner"
      - id: "config-testing"
        gate:
          rigor: "strict"
          evaluator_ref: "unit-test-runner"
      - id: "config-security-analysis"
        gate:
          rigor: "immutable"
          evaluator_ref: "security-scanner"

  delivery:
    pipeline:
      - id: "discovery"
        gate:
          rigor: "advisory"
      - id: "specs"
        gate:
          rigor: "strict"
          evaluator_ref: "linter-runner"
      - id: "planning"
        gate:
          rigor: "strict"
          evaluator_ref: "linter-runner"
      - id: "design"
        gate:
          rigor: "strict"
          evaluator_ref: "linter-runner"
      - id: "api-contract-lint" # Fase personalizada inyectada
        type: "custom"
        position:
          before: "testing"
        gate:
          rigor: "strict"
          evaluator_ref: "linter-runner"
      - id: "testing"
        gate:
          rigor: "strict"
          evaluator_ref: "unit-test-runner"
      - id: "implementing"
        gate:
          rigor: "strict"
          evaluator_ref: "unit-test-runner"
      - id: "security-analysis"
        gate:
          rigor: "immutable"
          evaluator_ref: "security-scanner"
      - id: "deploying"
        gate:
          rigor: "strict"
          evaluator_ref: "unit-test-runner"
      - id: "monitoring"
        gate:
          rigor: "advisory"
```

---

## 3. Esquema de Validación JSON Schema

El esquema formal con todas las reglas de validación sintáctica, patrones regex y definiciones de fase se encuentra en [`schemas/v1/manifest.json`](../schemas/v1/manifest.json).
