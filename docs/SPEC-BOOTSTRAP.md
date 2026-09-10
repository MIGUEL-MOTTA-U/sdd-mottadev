# Especificación Técnica: Protocolo de Arranque (*Bootstrap / Day-Zero*)

El protocolo *Bootstrap* es la secuencia determinista obligatoria para inicializar el plano de control en un repositorio nuevo (*greenfield*) o existente (*brownfield*) dentro del framework **SDD-mottadev**.

---

## 1. Secuencia de Inicialización (`sdd init`)

La invocación del comando `sdd init` ejecuta las siguientes operaciones atómicas:

1. **Estructuración del Directorio de Control:**
   * Crea el árbol de directorios `.sdd/` (`policies/`, `changes/`, `artifacts/blobs/`, `tokens/`, `workspaces/`).
   * Configura entradas de seguridad en `.gitignore`:
     ```gitignore
     # SDD Runtime & Secrets isolation
     .sdd/workspaces/
     .sdd/tokens/
     .sdd/STATE.md.lock
     ```

2. **Aprovisionamiento del Manifiesto Base:**
   * Emite el archivo `.sdd/sdd.manifest.yaml` configurado con los evaluadores estándar disponibles en el entorno (ver plantilla en [`examples/sdd.manifest.yaml`](../examples/sdd.manifest.yaml)).

3. **Instalación del Paquete Canónico de Políticas:**
   * Escribe las cuatro políticas base en `.sdd/policies/`:
     * `authz-default.yaml` ([`policies/authz-default.yaml`](../policies/authz-default.yaml))
     * `risk-default.yaml` ([`policies/risk-default.yaml`](../policies/risk-default.yaml))
     * `orphan-default.yaml` ([`policies/orphan-default.yaml`](../policies/orphan-default.yaml))
     * `merge-default.yaml` ([`policies/merge-default.yaml`](../policies/merge-default.yaml))

4. **Instanciación del ChangeRequest Raíz (`CR-000`):**
   * Genera `.sdd/changes/CR-000/CR.md` con la tarea canónica:
     `PLAN-000: Initial Configuration Track Certification`.

---

## 2. Invariante de Certificación de Configuración Inicial

**Regla de Bloqueo Absoluto:** Un repositorio inicializado bajo **SDD-mottadev** tiene **bloqueado de forma física el Track de Delivery** hasta que el Track de Configuración emita un veredicto `PASS` en la compuerta inmutable `config-security-analysis`.

```
[sdd init]
    │
    ▼
Genera CR-000 (Initial Setup)
    │
    ▼
[EJECUCIÓN DEL TRACK 1: CONFIGURATION]
Specs Config ──> Planning Config ──> Configuration ──> Config Testing
                                                             │
                                                             ▼
                                             Config Security Analysis (Gate)
                                                             │
                              ┌──────────────────────────────┴──────────────────────────────┐
                              │ FAIL / ERROR                                                │ PASS
                              ▼                                                             ▼
                    [SISTEMA BLOQUEADO]                                           [CERTIFICACIÓN EXITOSA]
             Delivery Track Inhabilitado                                           CR-000 -> COMPLETED
             No se pueden procesar ChangeRequests                                  Delivery Track Habilitado
             Se requiere mitigación técnica                                        Sistema listo para valor
```

---

## 3. Verificación de Cierre del Bootstrap

El bootstrap concluye exitosamente cuando el archivo `.sdd/STATE.md` refleja el estado validado del sistema:

```markdown
# SDD Runtime State - Baseline Certified
- **Active Track**: none
- **Certified Baseline**: CR-000 (Commit: a1f3e79...)
- **Configuration Security Gate**: PASS
- **Status**: FREE
- **Delivery Track Ready**: TRUE
- **Last Evaluated**: 2026-09-10T14:30:00Z
```

Con la emisión de este estado, el framework queda formalmente activado, asegurando que ningún agente u operador humano escriba código funcional sin un entorno pre-validado, seguro y auditable.
