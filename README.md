# Especificación Formal de Arquitectura: Framework SDD-mottadev

---

## 1. Fundamentos y Modelo Ontológico

El framework **SDD-mottadev** es un plano de control (*Control Plane*) determinista y agnóstico diseñado para gobernar el ciclo de vida de desarrollo de software bajo colaboración humano-agente o multi-agente. Establece un sistema formal de restricciones, transiciones de estado e interfaces verificables donde la computación generativa (probabilística) está subordinada a compuertas de evaluación e invariantes del sistema (deterministas).

### Jerarquía Estructural de Entidades

El metamodelo rechaza la fase o el agente como raíz. La unidad fundamental y jerárquica del sistema se modela estrictamente como:

```
Project (Raíz del Dominio)
  └── PolicySet (Políticas versionables que rigen el proyecto)
  └── ChangeRequest (Unidad de intención: feature, bugfix, refactor, config)
        └── Task (Unidad atómica y asignable de trabajo)
              └── Execution (Instancia temporal de cómputo bajo un Lease)
                    ├── Evidence (Registros probatorios inmutables append-only)
                    └── Artifact (Entregables tipados con doble identidad)

```

```
┌────────────────────────────────────────────────────────────────────────┐
│                               Project                                  │
│  ┌───────────────────────┐                  ┌───────────────────────┐  │
│  │       PolicySet       │                  │     ChangeRequest     │  │
│  └───────────────────────┘                  └───────────┬───────────┘  │
│                                                         │              │
│                                             ┌───────────▼───────────┐  │
│                                             │         Task          │  │
│                                             └───────────┬───────────┘  │
│                                                         │              │
│                                             ┌───────────▼───────────┐  │
│                                             │       Execution       │  │
│                                             └─────┬───────────┬─────┘  │
│                                                   │           │        │
│                                     ┌─────────────▼───┐   ┌───▼──────┐ │
│                                     │    Evidence     │   │ Artifact │ │
│                                     └─────────────────┘   └──────────┘ │
└────────────────────────────────────────────────────────────────────────┘

```

---

## 2. Modelo de Identidad, Capacidades y Seguridad

### 2.1 Identidad de Actores (`Actor`)

Cualquier entidad que interactúe con el framework (humano, agente autónomo, pipeline o script) es un `Actor` abstracto:

* `actor_id`: Identificador canónico (`urn:sdd:actor:<namespace>:<id>`).
* `public_key`: Clave criptográfica para firma de artefactos, evidencias y tokens.
* `role`: Rol de ejecución asignado (`planner`, `implementer`, `evaluator`, `auditor`, `operator`).

### 2.2 Niveles de Capacidad (*Privilege Levels*)

Las capacidades definen el alcance operativo del comando o herramienta, desvinculadas de la criticidad del entorno:

* **L0 (Read-Only):** Inspección de estado, lectura de especificaciones y análisis estático pasivo sin alteración de disco ni memoria persistente.
* **L1 (Scoped Workspace Mutation):** Escritura y mutación de archivos restringida exclusivamente al espacio de trabajo asignado a la tarea (`TaskWorkspace`).
* **L2 (Sandbox Execution):** Ejecución de procesos efímeros (compilación, linters, suites de pruebas) dentro de entornos confinados, sin acceso a redes externas ni persistencia fuera del workspace.
* **L3 (Privileged / External Mutation):** Alteración de infraestructura, despliegues, operaciones destructivas sobre datos o invocación de servicios externos sensibles.

### 2.3 Cálculo de Riesgo Contextual

El riesgo es una propiedad emergente calculada por el runtime antes de autorizar cualquier operación:


$$\text{RiskScore} = f(\text{Capability}, \text{ResourceImpact}, \text{EnvironmentCriticality})$$

* Si $\text{RiskScore} \ge \text{RiskThreshold}$: La operación no puede ejecutarse de manera desatendida y exige un `ApprovalToken`.

### 2.4 Token de Aprobación Criptográfica (`ApprovalToken`)

Autorización atómica, de un solo uso y vinculada a un contexto exacto para operaciones L3:

```
ApprovalToken {
    token_id: UUIDv4
    issuer: ActorId (Autoridad humana u operador autorizado)
    subject_task: TaskId
    permitted_capability: "L3"
    target_resource: ResourceURI
    expected_artifact_hash: SHA-256
    valid_from: Timestamp ISO-8601 UTC
    expires_at: Timestamp ISO-8601 UTC (TTL estricto)
    nonce: CryptographicNonce
    signature: DigitalSignature
}

```

### 2.5 Protocolo de Excepción Crítica (*Break-Glass*)

Mecanismo para omitir compuertas inmutables ante incidentes de producción o recuperación de desastres:

1. Requiere la emisión de un `EmergencyOverrideToken` firmado mediante esquema multi-firma (mínimo dos actores autorizados).
2. Desactiva condicionalmente la compuerta por un período no prorrogable ($\text{TTL} \le 120 \text{ minutos}$).
3. Genera automáticamente un `ChangeRequest` de tipo `POST_MORTEM_REMEDIATION` en estado `BLOCKED` asignado al equipo de seguridad.
4. Emite un evento de auditoría no repudiable y de difusión global.

---

## 3. Sistema de Políticas y Evaluación de Compuertas (*Gates*)

### 3.1 `Policy` como Entidad de Primer Orden

Las políticas son artefactos inmutables, versionables y evaluables que configuran el comportamiento del plano de control:

* **`GatePolicy`:** Reglas de paso, condiciones de prueba y tolerancias por fase.
* **`AuthorizationPolicy`:** Asignación entre roles de `Actor`, niveles de capacidad (L0–L3) y recursos.
* **`RiskPolicy`:** Matrices de criticidad y umbrales para requerir `ApprovalToken`.
* **`OrphanPolicy`:** Estrategias de recuperación ante expiración de leases de agentes.
* **`MergePolicy`:** Criterios de integración concurrente al tronco principal.

Toda modificación a una política requiere tramitarse formalmente a través de un `ChangeRequest`.

### 3.2 Semántica de Evaluación de Gates

Un Gate es una función booleana pura ejecutada sobre las evidencias generadas:


$$\text{GateResult} \in \{\text{PASS}, \text{FAIL}, \text{ERROR}\}$$

* **`PASS`:** La totalidad de las aserciones obligatorias (`strict` e `immutable`) se evalúan como verdaderas.
* **`FAIL`:** Una o más aserciones evaluadas son falsas (vulnerabilidad detectada, prueba unitaria fallida, contrato roto).
* **`ERROR`:** Falla de infraestructura en el evaluador (timeout, script caído, servicio inaccesible). **Bajo ninguna circunstancia un `ERROR` equivale a un `PASS**`; la transición se detiene en estado de falla operacional.

### 3.3 Aceptación Formal de Riesgo (`RiskAcceptance`)

El framework rechaza los estados de "Pass condicional". Si una compuerta genera `FAIL`, la transición solo puede desbloquearse mediante una entidad formal `RiskAcceptance`:

```
RiskAcceptance {
    acceptance_id: UUIDv4
    policy_id: PolicyId
    failing_assertion: String
    justification: String
    compensating_controls: List<String>
    owner: ActorId
    expires_at: Timestamp ISO-8601 UTC
    signature: DigitalSignature
}

```

### 3.4 Separación entre Productor y Evaluador (*Separation of Concerns*)

**Invariante Axiomático:** El actor (`Actor`) que genera un artefacto (`Producer`) tiene prohibido actuar como evaluador (`Evaluator`) del mismo en compuertas catalogadas como `strict` o `immutable`. La evaluación debe ser ejecutada por herramientas deterministas o por un actor independiente con rol de auditoría.

---

## 4. Ciclo de Vida Dual-Track y Fases del Framework

El sistema desacopla la preparación del entorno de la materialización de software mediante dos vías interconectadas:

```
[TRACK 1: CONFIGURATION]
Specs Config ──> Planning Config ──> Configuration ──> Config Testing ──> Config Security [GATE INMUTABLE]
                                                                                │
                                                                                ▼
[TRACK 2: DELIVERY]                                                    [Ambiente Certificado]
Discovery ──> Specs ──> Planning ──> Design ──> Testing ──> Implementing ──> Security [GATE INMUTABLE] ──> Deploying ──> Monitoring

```

### 4.1 Contrato Estándar de Fase (*Phase Contract*)

Toda fase (canónica o personalizada) se rige por la interfaz:

```
PhaseContract {
    id: KebabCaseIdentifier
    track: "configuration" | "delivery"
    required_inputs: Set<LogicalArtifactId>
    invariants: Set<PolicyId>
    required_outputs: Set<LogicalArtifactId>
    gate: {
        rigor: "advisory" | "strict" | "immutable"
        evaluator_ref: EvaluatorURI
    }
    hooks: {
        before: List<PhaseContract>
        after: List<PhaseContract>
    }
}

```

### 4.2 Fases del Track de Configuración (Infraestructura, Herramientas y Accesos)

1. **Specs Configuration:** Especificación de dependencias, variables, APIs, interfaces MCP, roles y permisos requeridos.
2. **Planning Configuration:** Plan para aprovisionar el MVP de configuración operativa (health checks basales, logging, autenticación mínima).
3. **Configuration:** Aprovisionamiento físico o virtual del entorno de desarrollo/ejecución.
4. **Config Testing:** Validación funcional de conectores, variables de entorno, pipelines y aislamiento de red.
5. **Config Security Analysis [COMPUERTA INMUTABLE]:** Auditoría estricta de superficies de ataque, permisos IAM, detección de secretos y análisis de CVEs en dependencias base.

### 4.3 Fases del Track de Delivery (Construcción de Solución)

1. **Discovery:** Investigación de viabilidad técnica, benchmarking, evaluación de dependencias y modelos de amenaza preliminares.
2. **Specs:** Formalización de requerimientos funcionales, no funcionales y criterios de aceptación.
3. **Planning:** Desglose del alcance en unidades de trabajo atómicas (`Task`), mitigando ambigüedades.
4. **Design:** Definición de arquitectura, diagramas de interacción, contratos de interfaz, resiliencia y modelo de datos.
5. **Testing:** Materialización de suites de pruebas ejecutables que codifican los contratos antes de la implementación funcional.
6. **Implementing:** Construcción del código que satisface las suites de pruebas definidas, vinculando cada cambio al identificador de tarea correspondiente.
7. **Security Analysis [COMPUERTA INMUTABLE]:** Análisis SAST/DAST, verificación estricta de OWASP y escaneo de vulnerabilidades sobre los artefactos producidos.
8. **Deploying:** Despliegue hacia el entorno objetivo con verificación de salud operativa y capacidad de rollback atómico.
9. **Monitoring:** Observabilidad continua (métricas, trazas, logs de auditoría) para detección temprana de degradación o brechas.

---

## 5. Máquina de Estados de Tarea y Concurrencia

### 5.1 Estados Formales de la Tarea (`TaskState`)

```
       ┌──────────────┐
       │  UNASSIGNED  │
       └──────┬───────┘
              │ Lease otorgado (Actor + TTL)
              ▼
       ┌──────────────┐      Falla Operacional (Timeout / Crash)
       │    ACTIVE    ├─────────────────────────────┐
       └──────┬───────┘                             │
              │ Ejecución finalizada                │
              ▼                                     ▼
       ┌──────────────┐                      ┌─────────────┐
       │  GATE_EVAL   │                      │    FAULT    │
       └──────┬───────┘                      └──────┬──────┘
              │                                     │
      ┌───────┴───────────────┐                     │
      │ PASS                  │ FAIL                │
      ▼                       ▼                     │
┌───────────┐           ┌───────────┐               │
│ COMPLETED │           │  FAILED   │               │
└───────────┘           └─────┬─────┘               │
                              │ Requiere aprobación │
                              ▼                     │
                        ┌───────────┐               │
                        │  BLOCKED  │ <─────────────┘
                        └─────┬─────┘
                              │
                              ▼
                        ┌───────────┐
                        │ CANCELLED │
                        └───────────┘

```

* **`UNASSIGNED`:** Tarea creada en el plan global sin ejecutor asignado.
* **`ACTIVE`:** Arriendo (*lease*) tomado por un `Actor` con un `Heartbeat` vigente.
* **`GATE_EVAL`:** Cómputo concluido; compuertas en ejecución y evaluación determinista.
* **`COMPLETED`:** 100% de compuertas aprobadas; artefactos y evidencias sellados.
* **`FAILED`:** Rechazo determinista de compuertas (pruebas fallidas, vulnerabilidad detectada). Exige re-trabajo técnico del ejecutor.
* **`FAULT`:** Error operacional del sistema (expiración de lease, timeout de ejecución, caída del runtime).
* **`BLOCKED`:** Ejecución suspendida en espera de una acción externa (emisión de `ApprovalToken`, resolución de dependencia o arbitraje humano).
* **`CANCELLED`:** Estado terminal forzado por el operador o por revocación de la tarea padre.

### 5.2 Protocolo de Arriendo (*Lease*) y Recuperación de Huérfanos

1. **Adquisición:** Un `Actor` reclama una tarea pasando su estado a `ACTIVE` fijando un $\text{LeaseTTL}$.
2. **Mantenimiento:** El actor debe emitir señales periódicas de `Heartbeat`.
3. **Detección de Abandono:**

$$\text{CurrentTimestamp} - \text{LastHeartbeat} > \text{LeaseTTL} \implies \text{TaskState} \leftarrow \text{FAULT}(\text{Reason: LEASE\_EXPIRED})$$


4. **Aislamiento y Recuperación:**
* El workspace efímero del actor desconectado se congela de inmediato.
* Se extrae el árbol de diferencias (*diff*) como `Evidence` de auditoría.
* Según la `OrphanPolicy`: se realiza un `ROLLBACK` atómico liberando la tarea a `UNASSIGNED`, o se escala a `BLOCKED` para arbitraje manual.



### 5.3 Reconciliación Concurrente e Integración

* **Aislamiento:** Cada tarea en estado `ACTIVE` opera en un espacio de trabajo desacoplado (rama git efímera o contenedor aislado).
* **`IntegrationRequest`:** Solicitud formal emitida al completar una tarea para fusionar los cambios al tronco común.
* **`MergeEvaluator`:** Validador que simula la integración en un entorno efímero y re-ejecuta las compuertas sobre el artefacto unificado.
* **`MergePolicy`:**
* *Linear Strict:* Exige que la rama esté rebasada sobre el último commit del tronco antes de evaluar compuertas.
* *Arbitrated:* En caso de conflicto semántico o de contenido, transiciona a `BLOCKED` requiriendo resolución humana.



---

## 6. Modelo de Artefactos, Evidencia y Linaje

### 6.1 Separación de Identidad de Artefactos

Para evitar colisiones entre la función de un archivo y su estado temporal, todo artefacto implementa doble identidad:

* **Identidad Lógica (`LogicalId`):** Inmutable en el tiempo; define el rol del recurso (`urn:sdd:artifact:specs/auth-service`).
* **Identidad de Contenido (`ContentHash`):** Criptográficamente unívoca para cada versión del archivo ($\text{SHA-256}$).

```
ArtifactReference {
    logical_id: LogicalId
    content_hash: SHA256Hash
    parent_hashes: List<SHA256Hash>       # Linaje formal (DAG)
    derived_from_task: TaskId
    schema_version: SemanticVersion
    producer_signature: DigitalSignature
}

```

### 6.2 Evidencia Probatoria Inmutable (`Evidence`)

La evidencia es un registro inmutable generado por un `Evaluator` que demuestra objetivamente el cumplimiento o incumplimiento de una regla o compuerta:

* La evidencia **nunca** es creada por el productor del código.
* Es de naturaleza estrictamente *append-only*.
* Estructura:

```
EvidenceRecord {
    evidence_id: UUIDv4
    task_id: TaskId
    evaluator_id: ActorId
    evaluated_artifact_hash: SHA256Hash
    assertion_results: Map<String, Boolean>
    raw_output_ref: ContentURI
    exit_code: Integer
    timestamp: Timestamp ISO-8601 UTC
    evaluator_signature: DigitalSignature
}

```

---

## 7. Protocolo de Contexto y Abstracción de Almacenamiento

### 7.1 Esquema Abstracto de Direccionamiento

El núcleo del framework es independiente de sistemas de archivos locales, APIs en la nube o bases de datos relacionales. Toda entidad se referencia mediante el protocolo abstracto `sdd://`:

$$\text{sdd://}\langle \text{project-id} \rangle / \langle \text{domain} \rangle / \langle \text{entity-type} \rangle / \langle \text{entity-id} \rangle$$

### 7.2 Capa de Adaptadores de Almacenamiento (*Storage Adapters*)

El runtime materializa el protocolo abstracto a través de adaptadores especializados según el entorno de ejecución:

| Adaptador | Dominio de Uso | Mecanismo de Persistencia |
| --- | --- | --- |
| **Filesystem Adapter** | Desarrollo local y agentes CLI | Mapeo determinista en directorio `.sdd/` del repositorio. |
| **Git Object Adapter** | Trazabilidad distribuida | Almacenamiento de artefactos y evidencias como blobs y tags de Git. |
| **Blob Storage Adapter** | CI/CD y nubes públicas | Persistencia inmutable en buckets S3/GCS compatibles. |

### 7.3 Divulgación Progresiva de Contexto (*Progressive Disclosure*)

Para evitar la degradación del razonamiento de modelos y agentes por saturación de contexto:

1. **Filtro de Entrada:** Un actor solo recibe los esquemas y contratos de las fases activas y los `ArtifactReference` listados como `required_inputs` en el `PhaseContract`.
2. **Aislamiento de Documentación Pesada:** Las investigaciones extensas, logs de compilación masivos o volcados de datos permanecen en el subsistema de almacenamiento. Al actor únicamente se le inyectan payloads de resumen estructurado (`Summary`, `EvidenceList`, `ConfidenceScore`).
3. **Inyección Dinámica de Políticas:** Las políticas se suministran bajo demanda únicamente cuando la tarea activa entra en contacto con el recurso o capacidad correspondiente.

---

## 8. Verificación de Integridad y Trazabilidad

Todo cambio aplicado al sistema debe verificar la cadena de custodia completa:

$$\text{Project} \longleftarrow \text{ChangeRequest} \longleftarrow \text{Task} \longleftarrow \text{Execution} \longleftarrow \text{Evidence} \Longrightarrow \text{Artifact}$$

El estado global del proyecto en cualquier instante de tiempo $T$ es matemáticamente reproducible a partir del historial secuencial de evidencias selladas, firmas criptográficas y árboles de dependencias de artefactos, garantizando auditabilidad absoluta e independencia frente al ejecutor subyacente.
