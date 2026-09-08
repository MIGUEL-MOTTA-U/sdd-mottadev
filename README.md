
# Especificación Técnica del Framework SDD-mottadev

## 1. Visión General y Filosofía

**SDD (Spec-Driven Development / Software Development Design) - mottadev** es un framework de ciclo de vida de desarrollo de software diseñado para la colaboración fluida, auditable y determinista entre ingenieros humanos y agentes de inteligencia artificial (Claude Code, Codex, Cursor, Aider, CLI agents autónomos).

### Principios Fundamentales

* **No es una camisa de fuerza:** Es un sistema de reglas modular, atómico y configurable que se adapta a proyectos nuevos (*greenfield*) o existentes (*brownfield*).
* **Seguridad Inmutable:** La seguridad es la única compuerta (*gate*) no negociable. Si una fase de análisis de seguridad falla, el avance del ciclo de vida se bloquea por completo.
* **Agnosticismo de Modelo y Herramienta:** No depende de plataformas propietarias ni de memoria de chat. El estado se persiste en archivos locales estructurados (Markdown/YAML), aplicando divulgación progresiva de contexto (*progressive disclosure*).
* **Trazabilidad Absoluta:** Cada cambio de código debe estar respaldado por un ítem de especificación (*Spec ID*), un ítem de plan (*Plan ID*), pruebas asociadas y un registro en la bitácora de auditoría.
* **TDD Estricto:** La implementación siempre sigue a la definición de pruebas ejecutables. Ningún agente puede generar código funcional sin pruebas previas.

---

## 2. Arquitectura de Ciclo de Vida: Modelo de Dos Vías (Dual-Track)

El framework opera mediante dos pipelines acoplados mediante compuertas de validación:

```
[TRACK 1: CONFIGURATION] (Requerido en inicio, onboarding o cambios de infra)
Specs Config ──> Planning Config ──> Configuration ──> Config Testing ──> Config Security Analysis [GATE INMUTABLE]
                                                                                  │
                                                                                  ▼
[TRACK 2: DELIVERY]                                                  [Entorno Operativo Listo]
Discovery ──> Specs ──> Planning ──> Design ──> Testing (TDD) ──> Implementing ──> Security Analysis [GATE INMUTABLE] ──> Deploying ──> Monitoring
    ▲                                                                                                                            │
    └───────────────────────────────────── Refactor (Loopback a Discovery o Specs) ──────────────────────────────────────────────┘

```

### Track 1: Configuration (Entorno, Herramientas e Infraestructura)

1. **Specs Configuration:** Especificación formal de dependencias, variables de entorno, APIs, MCPs, roles IAM, permisos de agentes y baseline de infraestructura.
2. **Planning Configuration:** Plan paso a paso para aprovisionar el MVP de configuración (endpoints base como `/health`, logging estructurado, observabilidad mínima, autenticación).
3. **Configuration:** Ejecución del aprovisionamiento del entorno y herramientas locales/cloud.
4. **Configuration Testing:** Pruebas de conectividad, variables, permisos y aislamiento.
5. **Config Security Analysis [COMPUERTA INMUTABLE]:** Auditoría estricta de secretos, dependencias (CVEs) y políticas de mínimo privilegio.

### Track 2: Delivery (Construcción del Producto)

1. **Discovery:** Investigación de viabilidad técnica, herramientas, supuestos, regulaciones, tratamiento de datos y modelos de amenaza preliminares.
2. **Specs:** Definición formal de requerimientos funcionales/no funcionales, alcance y criterios de aceptación.
3. **Planning:** Desglose atómico del trabajo en `PLAN.md`. Reducción al mínimo de la toma de decisiones no consultadas.
4. **Design:** Definición de arquitectura técnica, diagramas de secuencia, contratos de interfaz, modelos de datos, resiliencia y concurrencia.
5. **Testing:** Definición y escritura de suites de pruebas (unitarias, integración, concurrencia o estrés según el rigor) antes de codificar la solución.
6. **Implementing:** Codificación de la solución atada a los tests creados. Prohibido código huérfano de pruebas.
7. **Security Analysis [COMPUERTA INMUTABLE]:** Análisis SAST/DAST, escaneo de dependencias y validación contra OWASP Top 10.
8. **Deploying:** Despliegue con verificación automática de *healthcheck* y capacidad de rollback inmediato.
9. **Monitoring:** Observabilidad continua (logs, métricas, trazas, KPIs de negocio) para detección temprana de incidentes.
10. **Refactor:** Evaluación de métricas y nuevas necesidades. **Regla de retorno:** Debe volver a *Discovery* o *Specs*; queda terminantemente prohibido saltar directo a *Implementing*.

---

## 3. Matriz de Rigurosidad por Niveles

Configurable en el manifiesto global o por componente:

| Nivel | Enfoque | Autonomía del Agente | Exigencia de Pruebas | Compuertas de Seguridad |
| --- | --- | --- | --- | --- |
| **Nivel 1 (MVP)** | Prototipos, validación rápida | Alta (el modelo asume decisiones de implementación no críticas). | Tests unitarios sobre lógica core. | Escaneo estático básico de secretos y dependencias. |
| **Nivel 2 (Estándar)** | Producción estándar | Media (decisiones de arquitectura e infra requieren confirmación). | Tests unitarios + tests de integración. | OWASP Top 10 básico, SAST y validación de permisos en CI/CD. |
| **Nivel 3 (Avanzado)** | Sistemas críticos, alta concurrencia | Baja (toda decisión de datos, seguridad y concurrencia se aprueba). | Unitarios, integración, concurrencia, estrés y casos de borde. | Threat modeling formal, SAST/DAST exhaustivo, cero CVEs altos/críticos. |

---

## 4. Estructura del Repositorio (`.sdd/`)

Todos los artefactos de control del framework residen en el directorio `.sdd/` en la raíz del repositorio:

```text
.sdd/
├── sdd.config.yaml              # Manifiesto principal del pipeline y reglas
├── STATE.md                     # Snapshot caliente (<30 líneas). Lo primero que lee el agente
├── PLAN.md                      # Checklist atómico de la fase activa
├── CHANGELOG.md                 # Bitácora append-only de sesiones y auditoría
├── DECISIONS.md                 # Architecture Decision Records (ADRs) ligeros
├── rules/                       # Reglas modulares inyectadas progresivamente
│   ├── core/
│   │   ├── security-inmutable.md
│   │   ├── tdd-enforcement.md
│   │   └── traceability.md
│   └── custom/                  # Reglas específicas del proyecto
│       └── api-contracts.md
└── scripts/                     # Validadores deterministas ejecutables
    ├── sdd                      # CLI agnóstico en POSIX shell o binario compilado
    ├── gate-runner.sh           # Orquestador de compuertas
    └── sec-scan.sh              # Evaluador de seguridad (SAST, secretos, CVEs)

```

---

## 5. Especificaciones de Archivos y Contratos

### A. Manifiesto Principal: `.sdd/sdd.config.yaml`

Permite extender el pipeline inyectando fases personalizadas mediante un arreglo secuencial con soporte para compuertas.

```yaml
version: "1.0"
framework: "sdd-mottadev"
settings:
  rigor_level: 2 # 1: MVP | 2: Standard | 3: Advanced
  auto_commit_on_gate: true

tracks:
  configuration:
    enabled: true
    pipeline:
      - id: "specs-config"
        gate: "strict"
      - id: "planning-config"
        gate: "strict"
      - id: "configuration"
        gate: "strict"
      - id: "config-testing"
        gate: "strict"
      - id: "config-security-analysis"
        gate: "immutable" # No bypassable

  delivery:
    pipeline:
      - id: "discovery"
        gate: "advisory"
      - id: "specs"
        gate: "strict"
      - id: "planning"
        gate: "strict"
      - id: "design"
        gate: "strict"
      # Extensión personalizada de ejemplo:
      - id: "contract-validation"
        type: "custom"
        position:
          before: "testing"
        gate: "strict"
        rules: ["rules/custom/api-contracts.md"]
        evaluator: "scripts/validate-contracts.sh"
      - id: "testing"
        gate: "strict"
        rules: ["rules/core/tdd-enforcement.md"]
      - id: "implementing"
        gate: "strict"
        rules: ["rules/core/traceability.md"]
      - id: "security-analysis"
        gate: "immutable"
        evaluator: "scripts/sec-scan.sh"
      - id: "deploying"
        gate: "strict"
      - id: "monitoring"
        gate: "advisory"

gates:
  block_on_security_violation: true
  allow_manual_override_on_security: false

```

### B. Snapshot Caliente: `.sdd/STATE.md`

Este archivo debe mantenerse pequeño (< 30 líneas). Contiene la verdad activa que cualquier agente lee al arrancar.

```markdown
# SDD Runtime State
- **Active Track**: delivery
- **Current Phase**: testing
- **Rigorous Level**: 2
- **Agent/Model**: Claude Code (Session-104)
- **Status**: IN_PROGRESS
- **Active Task**: P-004
- **Gate Status**: PENDING_VERIFICATION
- **Last Updated**: 2026-09-08T11:20:00Z

```

### C. Plan Atómico: `.sdd/PLAN.md`

Desglosa tareas bajo el principio de responsabilidad única (SRP):

```markdown
# Implementation Plan - Phase: Testing & Implementing

- [x] **P-001**: Definir interfaz de autenticación JWT [Spec: S-02] (Tests requeridos: Unit).
- [ ] **P-002**: Implementar suite de pruebas unitarias para validación de tokens [Spec: S-02] (Status: IN_PROGRESS).
- [ ] **P-003**: Implementar middleware de autenticación [Spec: S-02] (Status: TODO).

```

### D. Definición de Reglas Modulares (`.sdd/rules/core/*.md`)

Cada regla utiliza frontmatter YAML estructurado para guiar a los agentes y referenciar el evaluador determinista:

```markdown
---
id: "RULE-SEC-01"
name: "Immutable Security Gate"
severity: "FATAL"
evaluator: "scripts/sec-scan.sh"
applies_to: ["config-security-analysis", "security-analysis"]
---

# Regla Innegociable de Seguridad

1. No se permiten credenciales, API keys o tokens en texto plano dentro del código o documentación.
2. Todas las dependencias deben estar libres de vulnerabilidades con severidad HIGH o CRITICAL.
3. Si el script `sec-scan.sh` retorna código de salida != 0, la tarea se marca inmediatamente como BLOCKED.

```

---

## 6. Especificación de Herramientas y CLI (`sdd`)

El framework cuenta con un CLI ejecutable (implementable en Shell POSIX o Go) que expone los siguientes comandos para ser invocados por scripts, agentes o hooks de Git:

* `sdd status`: Imprime el contenido de `.sdd/STATE.md` y valida la integridad de la sesión.
* `sdd plan [--add|--update|--close]`: Manipula atómicamente `.sdd/PLAN.md`.
* `sdd check-gate`: Ejecuta el script evaluador configurado para la fase actual. Devuelve `exit 0` si aprueba, o `exit 1` con reporte de fallas.
* `sdd next-phase`: Verifica que la compuerta esté en estado superado (`exit 0`), registra el cambio en `.sdd/CHANGELOG.md` y avanza el cursor de fase en `.sdd/STATE.md`.
* `sdd log --session`: Permite agregar una entrada estructurada al final de `.sdd/CHANGELOG.md`.

### Integraciones Recomendadas

* **Escaneo de Secretos:** Gitleaks o Trufflehog integrados en `scripts/sec-scan.sh`.
* **Análisis Estático (SAST):** Semgrep (OSS) ejecutando reglas OWASP Top 10.
* **Escaneo de Dependencias:** Trivy u OSV-Scanner.
* **Model Context Protocol (MCP):** Un servidor local `sdd-mcp-server` que exponga estas acciones como herramientas directas (`tools`) para agentes que soportan MCP.

---

## 7. Protocolo de Ejecución para Agentes de IA

Cada agente que interactúe con el repositorio debe seguir este algoritmo:

1. **Fase 0 (Sincronización):** Leer `.sdd/STATE.md`. Determinar el *track* activo, la *fase* actual y el nivel de rigor.
2. **Fase 1 (Carga de Contexto Mínimo):** Cargar únicamente las reglas especificadas en `.sdd/sdd.config.yaml` para la fase activa.
3. **Fase 2 (Planificación):** Si la fase actual requiere tareas, leer `.sdd/PLAN.md` y tomar el ítem de trabajo correspondiente.
4. **Fase 3 (Ejecución y Verificación):** Modificar el código o pruebas. Ejecutar suites locales.
5. **Fase 4 (Validación de Compuerta):** Correr `.sdd/scripts/sdd check-gate`.
* Si retorna error (`exit != 0`): Corregir la falla. No intentar avanzar.
* Si aprueba (`exit 0`): Ejecutar `sdd next-phase`.


6. **Fase 5 (Handoff y Cierre):** Registrar la entrada de auditoría en `.sdd/CHANGELOG.md`, marcar el estado como `FREE` en `.sdd/STATE.md` y generar el commit convencional vinculando el ID de tarea (`[PLAN:P-XXX]`).
