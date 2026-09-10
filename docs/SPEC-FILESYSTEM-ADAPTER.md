# Especificación Técnica: Mapeo del Adaptador de Sistema de Archivos (`Filesystem Adapter Mapping`)

Este documento formaliza la traducción determinista entre el protocolo abstracto de direccionamiento `sdd://` y el sistema de archivos local cuando el plano de control opera sobre un repositorio con control de versiones Git.

---

## 1. Jerarquía Física del Directorio de Control (`.sdd/`)

Toda la metadata, políticas, tareas, evidencias y estado de ejecución residen de forma confinada en el directorio `.sdd/` en la raíz del repositorio:

```text
<repository_root>/
├── .sdd/
│   ├── sdd.manifest.yaml                # sdd://<project>/manifest
│   ├── STATE.md                         # Estado de sesión y lock activo (runtime local)
│   ├── policies/                        # sdd://<project>/policies/<policy_id>
│   │   ├── authz-default.yaml
│   │   ├── risk-default.yaml
│   │   ├── orphan-default.yaml
│   │   └── merge-default.yaml
│   ├── changes/                         # sdd://<project>/changes/<change_id>
│   │   └── CR-001/
│   │       ├── CR.md                    # Metadatos del ChangeRequest
│   │       ├── design/                  # sdd://<project>/changes/<change_id>/design
│   │       │   ├── mocks/               # Mocks, imágenes, capturas y muestras visuales
│   │       │   └── UI-SPEC.md           # Especificación de vistas y contratos de UI
│   │       ├── tasks/                   # sdd://<project>/tasks/<task_id>
│   │       │   ├── PLAN-001.md
│   │       │   └── PLAN-002.md
│   │       └── evidence/                # Evidencias vinculadas al ChangeRequest
│   │           └── EV-PLAN-001-01.json
│   ├── artifacts/
│   │   ├── registry.json                # Mapeo: LogicalId -> ContentHash
│   │   └── blobs/                       # Almacenamiento local CAS (Content Addressable Storage)
│   │       └── e3/
│   │           └── b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
│   ├── research/                        # sdd://<project>/research/artifacts/<id>
│   │   └── RES-REQ-001.md
│   ├── tokens/                          # sdd://<project>/tokens/<token_id>
│   │   └── APP-UUID-001.json
│   └── workspaces/                      # Workspaces efímeros para tareas L1/L2 (en .gitignore)
│       └── PLAN-001/
```

---

## 2. Tabla de Traducción de URIs Canónicas

| URI Lógica (`sdd://`) | Ruta en Sistema de Archivos | Tipo de Persistencia | Propósito |
| :--- | :--- | :--- | :--- |
| `sdd://project/manifest` | `.sdd/sdd.manifest.yaml` | Versionado en Git | Configuración declarativa del plano de control |
| `sdd://project/policy/<id>` | `.sdd/policies/<id>.yaml` | Versionado en Git | Políticas activas e inmutables del proyecto |
| `sdd://project/change/<id>` | `.sdd/changes/<id>/CR.md` | Versionado en Git | Unidad de intención de cambio y contexto |
| `sdd://project/change/<id>/design/<path>` | `.sdd/changes/<id>/design/<path>` | Versionado en Git | Mocks, imágenes, wireframes y contratos de UI |
| `sdd://project/task/<id>` | `.sdd/changes/<cr_id>/tasks/<id>.md` | Versionado en Git | Unidad atómica asignable de trabajo |
| `sdd://project/artifact/logical/<path>` | `<path>` (Ruta real en repo, ej: `src/auth.ts`) | Versionado en Git | Código fuente, configuración y documentación |
| `sdd://project/artifact/blob/<hash>` | `.sdd/artifacts/blobs/<hash[:2]>/<hash[2:]>` | Append-only / CAS | Almacén de contenido inmutable direccionable |
| `sdd://project/evidence/<id>` | `.sdd/changes/<cr_id>/evidence/<id>.json` | Versionado en Git / Inmutable | Veredictos y aserciones de evaluadores |
| `sdd://project/token/<id>` | `.sdd/tokens/<id>.json` | Efímero / Audit log | Tokens de aprobación L3 emitidos |
| `sdd://project/workspace/<task_id>` | `.sdd/workspaces/<task_id>/` | Efímero / Ignorado en Git | Espacio de trabajo aislado para tareas activas |

---

## 3. Modelo de Almacenamiento Direccionable por Contenido (CAS)

Para garantizar la inmutabilidad y deduplicación de artefactos intermedios y reportes:
1. El identificador de contenido es el hash SHA-256 de los bytes del archivo.
2. Los primeros dos caracteres hexadecimales conforman el subdirectorio (ej. `e3/`).
3. Los restantes caracteres conforman el nombre del archivo dentro del subdirectorio (ej. `b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`).
4. El archivo [`registry.json`](file:///C:/Users/migue_7m/Desktop/Documentos%20Miguel/tmp/tmp/SDD---mottadev/.sdd/artifacts/registry.json) mantiene el mapeo bidireccional entre la identidad lógica (`LogicalId`) y el hash actual del contenido.
