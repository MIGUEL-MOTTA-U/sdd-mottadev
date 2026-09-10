# Registro Histórico de Decisiones y Trazabilidad (ADR Log)

Este registro documenta de forma secuencial e inmutable las decisiones arquitectónicas, descartes de diseño y la evolución conceptual del framework **SDD-mottadev**.

---

## Índice de Registros de Decisión (ADRs)

* [ADR-001: Arquitectura Dual-Track e Invariante de Seguridad](#adr-001-arquitectura-dual-track-e-invariante-de-seguridad)
* [ADR-002: Desacoplamiento de Persistencia: Protocolo Abstracto vs. Storage Drivers](#adr-002-desacoplamiento-de-persistencia-protocolo-abstracto-vs-storage-drivers)
* [ADR-003: Taxonomía de Entidades de Primer Orden y Jerarquía Raíz](#adr-003-taxonomía-de-entidades-de-primer-orden-y-jerarquía-raíz)
* [ADR-004: Modelo de Gobernanza: Capacidad (L0-L3) vs. Riesgo Contextual](#adr-004-modelo-de-gobernanza-capacidad-l0-l3-vs-riesgo-contextual)
* [ADR-005: Descarte de TDD Universal en Core y de Refactor como Fase Temporal](#adr-005-descarte-de-tdd-universal-en-core-y-de-refactor-como-fase-temporal)
* [ADR-006: Identidad Agnóstica de Actores mediante Criptografía Asimétrica](#adr-006-identidad-agnóstica-de-actores-mediante-criptografía-asimétrica)
* [ADR-007: Desacoplamiento Ontológico: Artifact (Doble Identidad) vs. Evidence](#adr-007-desacoplamiento-ontológico-artifact-doble-identidad-vs-evidence)
* [ADR-008: Concurrencia mediante Leases, Heartbeats y Recuperación de Huérfanos](#adr-008-concurrencia-mediante-leases-heartbeats-y-recuperación-de-huérfanos)
* [ADR-009: Especificación Contractual de Interfaces de Agentes y Evaluadores](#adr-009-especificación-contractual-de-interfaces-de-agentes-y-evaluadores)

---

### ADR-001: Arquitectura Dual-Track e Invariante de Seguridad
* **Fecha:** 2026-09-08
* **Estado:** Aceptado
* **Contexto:** Al operar con agentes de software, la preparación del entorno, variables, dependencias y accesos genera confusión recurrente si se mezcla con el desarrollo de requerimientos funcionales.
* **Decisión:** Desacoplar formalmente el ciclo de vida en dos vías:
  1. *Track 1 (Configuration):* Especifica, aprovisiona y prueba el entorno y las herramientas.
  2. *Track 2 (Delivery):* Construye y entrega valor sobre un entorno previamente validado.
  Se establece que las compuertas de análisis de seguridad en ambas vías son `inmutables` y bloqueantes absolutas ante vulnerabilidades críticas.
* **Consecuencias:** Todo proyecto nuevo o cambio de infraestructura base debe certificar el Track 1 antes de admitir transiciones de tareas en el Track 2.

---

### ADR-002: Desacoplamiento de Persistencia: Protocolo Abstracto vs. Storage Drivers
* **Fecha:** 2026-09-08
* **Estado:** Aceptado
* **Contexto:** Se propuso acoplar el almacenamiento operacional a bases de datos relacionales locales (SQLite) o repositorios Markdown fijos (`stuff/`). Esto comprometía el agnosticismo y la portabilidad del plano de control.
* **Decisión:** El núcleo del framework define un protocolo de direccionamiento uniforme abstracto (`sdd://<project>/<domain>/<entity>`). Los mecanismos de almacenamiento se relegan a adaptadores concretos (*Filesystem Adapter*, *Git Object Adapter*, *Blob Storage Adapter*).
* **Descartes:** Se descarta incluir drivers nativos SQL o dependencias de motores de bases de datos dentro de la especificación core.

---

### ADR-003: Taxonomía de Entidades de Primer Orden y Jerarquía Raíz
* **Fecha:** 2026-09-09
* **Estado:** Aceptado
* **Contexto:** Las fases del SDLC y los agentes competían por ser la entidad raíz del metamodelo, lo que impedía un modelado orientado a flujos no lineales.
* **Decisión:** Establecer el objeto raíz como `Project`, derivando en la jerarquía:
  $$\text{Project} \longrightarrow \text{ChangeRequest} \longrightarrow \text{Task} \longrightarrow \text{Execution} \longrightarrow (\text{Evidence}, \text{Artifact})$$
  Las fases dejan de ser código rígido y se convierten en configuraciones del pipeline aplicadas sobre cada `ChangeRequest`.

---

### ADR-004: Modelo de Gobernanza: Capacidad (L0-L3) vs. Riesgo Contextual
* **Fecha:** 2026-09-09
* **Estado:** Aceptado
* **Contexto:** La clasificación inicial L0–L3 mezclaba el nivel de privilegios con el riesgo de la acción, provocando que acciones legítimas de baja severidad fueran bloqueadas indebidamente.
* **Decisión:**
  * L0–L3 clasifica estrictamente la **capacidad técnica / nivel de privilegio** de la operación.
  * El **riesgo** es una función emergente calculada: $\text{RiskScore} = f(\text{Capability}, \text{ResourceImpact}, \text{EnvironmentCriticality})$.
  * Toda operación de capacidad L3 exige obligatoriamente un `ApprovalToken` emitido por un operador humano autorizado, con nonce único y TTL estricto.

---

### ADR-005: Descarte de TDD Universal en Core y de Refactor como Fase Temporal
* **Fecha:** 2026-09-09
* **Estado:** Aceptado
* **Contexto:** Exigir Test-Driven Development (TDD) estricto en el núcleo inhabilitaba el framework para proyectos de infraestructura como código (IaC), picos de exploración o pipelines de datos. Tratar *Refactor* como fase generaba bucles cíclicos en la máquina de estados.
* **Decisión:**
  * TDD se extrae del núcleo y se convierte en una **política de fase configurable** (`TDDPolicy`).
  * Se elimina *Refactor* como fase temporal del ciclo de vida y se formaliza como un tipo de cambio (`ChangeType = REFACTOR`).
* **Consecuencias:** Todo refactor entra por las etapas canónicas con una especificación técnica de la deuda técnica o mejora esperada.

---

### ADR-006: Identidad Agnóstica de Actores mediante Criptografía Asimétrica
* **Fecha:** 2026-09-09
* **Estado:** Aceptado
* **Contexto:** Inicialmente se consideró clasificar a los agentes por nombre de modelo de IA (Claude, Codex, etc.), lo cual generaba acoplamiento a proveedores propietarios y rompía la auditabilidad frente a operadores humanos.
* **Decisión:** Modelar a todos los ejecutores como un `Actor` abstracto identificado por un URI canónico y una clave pública asimétrica.
* **Consecuencias:** Cualquier entidad (humano, agente CLI o bot de CI/CD) firma criptográficamente sus artefactos y evidencias bajo el mismo contrato.

---

### ADR-007: Desacoplamiento Ontológico: Artifact (Doble Identidad) vs. Evidence
* **Fecha:** 2026-09-10
* **Estado:** Aceptado
* **Contexto:** Se utilizaba el término "artefacto" indistintamente para referirse al código producido y a los resultados de los tests o linters.
* **Decisión:** Separación ontológica estricta:
  * **`Artifact`:** Es el sujeto generado por la tarea (código, spec, esquema). Posee **Doble Identidad**: una Identidad Lógica (`LogicalId`) persistente en el tiempo y una Identidad de Contenido (`ContentHash`, SHA-256) inmutable.
  * **`Evidence`:** Es el registro probatorio inmutable generado exclusivamente por un `Evaluator` (nunca por el productor del código) que demuestra conformidad o falla.

---

### ADR-008: Concurrencia mediante Leases, Heartbeats y Recuperación de Huérfanos
* **Fecha:** 2026-09-10
* **Estado:** Aceptado
* **Contexto:** La ejecución simultánea o asíncrona de múltiples agentes sobre un mismo repositorio generaba riesgos de colisión de archivos y bloqueos huérfanos ante caídas de proceso.
* **Decisión:**
  * Implementar control de concurrencia basado en arriendos (*leases*) temporales con renovación periódica mediante `Heartbeat`.
  * Si $\text{CurrentTime} - \text{LastHeartbeat} > \text{LeaseTTL}$, el runtime declara el estado `FAULT(LEASE_EXPIRED)`.
  * La recuperación ante huérfanos se delega a la política `OrphanPolicy` configurada (`ROLLBACK` automático a base estable o escalado a `BLOCKED` para intervención humana).

---

### ADR-009: Especificación Contractual de Interfaces de Agentes y Evaluadores
* **Fecha:** 2026-09-10
* **Estado:** Aceptado
* **Contexto:** La falta de esquemas JSON formales para la invocación de herramientas provocaba discrepancias en las implementaciones de los agentes y en la interpretación de los evaluadores.
* **Decisión:** Redactar y congelar la especificación de `Agent API` (vía MCP/CLI) y `Evaluator Contract`, con esquemas JSON-Schema rigurosos para `EvaluatorInput`, `EvaluatorOutput` y códigos de error normalizados (`ERR_*`).
* **Consecuencias:** Las herramientas de validación solo requieren consumir JSON por `stdin` y emitir JSON por `stdout` conforme al contrato para integrarse al framework.
