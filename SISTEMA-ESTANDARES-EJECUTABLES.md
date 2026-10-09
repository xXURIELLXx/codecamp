# Sistema de Estándares Ejecutables + Autocorrección

**Fecha**: 2026-10-09  
**Versión**: 2.0 (Enforcement-first)  
**Ámbito**: Validación automática, estandarización, corrección en tiempo real

---

## Resumen Ejecutivo

Tu estructura actual ya tiene buena intención, pero el problema real es que no existe una capa ejecutable que impida que los agentes mezclen proyectos, creen archivos fuera de lugar o alteren el orden sin validación.

La solución no es agregar más carpetas improvisadas, sino crear un sistema de estándares que se aplique antes, durante y después de cada operación.

### El problema
- Proyectos que se parecen pero no son lo mismo
- `.claude` o agentes leyendo el mismo contexto sin contrato explícito
- Ficheros creados sin manifest ni validación
- Confusión entre análisis, tareas, fixtures, datos y repos externos
- Ausencia de auditoría para decisiones que se toman manual o automáticamente

### La solución
Un sistema de 5 capas:

1. Estándares definidos
2. Validación pre-ejecución
3. Enforcement en tiempo real
4. Auditoría post-ejecución
5. Autocorrección

---

## 1. Estructura Base de Gobernanza

```text
/freebuff/
├── analysis/                    # RAG corpus + handoff + tracking
│   ├── 00-index.md
│   ├── handoff.md
│   ├── pendientes.md
│   ├── registro-rag.md
│   ├── extracted/
│   ├── archive/
│   └── notes/
├── standards/                   # Estándares + validadores + reglas
│   ├── projects-schema.yaml
│   ├── project-manifest-schema.json
│   ├── content-schema.yaml
│   ├── validators/
│   ├── enforcement/
│   ├── audit/
│   └── autocorrect/
├── system-state/                # Contexto compartido entre agentes
│   ├── active-context/
│   ├── agent-registry/
│   ├── tool-registry/
│   └── coordination/
├── learning-registry/           # Patrones, análisis cruzado, recomendaciones
├── app/
│   ├── presenter/
│   └── ingest/
├── vectorstore/
│   └── collections/
├── repos/
├── logs/
├── mcp-node/
├── proyecto STRCM/
├── migracion-ollama/
├── roxys-analisis/
├── ecc-main/
├── prj-rag-test/
├── examples/
├── interfaz/
├── tools/
├── legacy/
├── checkpoints.db*
└── .env*
```

### Regla importante
- No todo entra a la raíz.
- Todo proyecto debe tener un manifest.
- Todo proyecto debe declarar su tipo.
- Todo proyecto debe tener una carpeta `00-admin/` o equivalent.
- El RAG no debe indexar contenido crudo ni logs ni `.env` ni `checkpoints.db*`.

---

## 2. Tipos de Proyecto

Cada carpeta debe clasificarse en uno de estos tipos:

```yaml
project_types:
  PARENT:
    description: "Proyecto padre con subproyectos"
    required_root_files:
      - "00-admin/PROJECT-MANIFEST.yaml"
    reserved_folders:
      - "00-admin"
      - "analysis"
      - "standards"
      - "docs"

  SUBPROJECT:
    description: "Subproyecto hijo"
    required_root_files:
      - "00-admin/SUBPROJECT-MANIFEST.yaml"
    reserved_folders:
      - "00-admin"

  TASK:
    description: "Tarea temporal o experimento"
    required_root_files:
      - "TASK-MANIFEST.yaml"
    auto_cleanup: "30_days"

  FIXTURE:
    description: "Fixture o datos de prueba"
    read_only: true
    auto_cleanup: "7_days"

  REFERENCE_REPO:
    description: "Repositorio externo para consulta contractual"
    read_only: true

  LEGACY:
    description: "Proyecto viejo, no operativo"
    read_only: true
```

---

## 3. Manifest Obligatorio por Proyecto

Cada proyecto debe tener un manifest mínimo.

**Ejemplo de `PROJECT-MANIFEST.yaml`**

```yaml
project_id: proyecto-strcm
project_type: PARENT
name: proyecto STRCM
description: Proyecto principal del sistema de coordinación, agentes y estandarización
created_at: 2026-10-09T00:00:00Z
owner: xXURIELLXx
status: active
standards_version: 2.0
parent_project_id: null
folder_structure:
  reserved_folders:
    - 00-admin
    - 01-agente-madre
    - 02-agente-hijo
    - 03-datos
    - 04-visualizacion
    - 05-herramientas
    - 06-modelos
    - 07-servidor
    - analisis-ia
    - estandar
rag_indexing:
  indexed: false
  collection_name: proyecto-strcm
  indexed_files_count: 0
```

### regla
- si no existe manifest, el sistema lo considera inválido
- si falta el tipo, se bloquea
- si la carpeta no coincide con el tipo, se marca error

---

## 4. Estándares por Carpeta

### 4.1 `analysis/`

Debe contener sólamente:
- `00-index.md`
- `handoff.md`
- `pendientes.md`
- `registro-rag.md`
- `extracted/`
- `archive/`
- `notes/`

Prohibido:
- `.db`
- logs de sesión sin filtro
- datos crudos
- contenido de proyectos mezclado sin metadata

### 4.2 `vectorstore/`

Debe contener SOLO:
- `collections/`
- metadata de indexación

Prohibido:
- archivos de texto sin indexar
- archivos de datos originales
- documentos crudos

### 4.3 `repos/`

Debe contener repos clonados o repo de trabajo.
Cada repo debe tener nombre limpio, sin mezcla de tareas.

### 4.4 `project-root/`

Prohibido:
- `.env`
- `*.db`
- `*.log`
- archivos secretos
- archivos de sesión mezclados con proyecto

---

## 5. Reglas de Validación Pre-Ejecución

Antes de crear o mover algo, el sistema debe chequear:

- ¿La carpeta pertenece a un proyecto ya definido?
- ¿El nombre cumple convención?
- ¿La copia va a una carpeta reservada?
- ¿El archivo es o no permitido allí?
- ¿Se está usando el tipo correcto?
- ¿Hay una operación de bloqueo sobre la ruta?

### Ejemplo: validación de proyecto

```python
# pseudo-code

def validate_project(project_path):
    manifest = project_path / "00-admin" / "PROJECT-MANIFEST.yaml"
    if not manifest.exists():
        return fail("Missing manifest")

    data = yaml.safe_load(manifest.read_text())
    if data.get("project_type") not in ["PARENT", "SUBPROJECT", "TASK", "FIXTURE"]:
        return fail("Invalid project_type")

    if not path_is_valid(project_path.name):
        return fail("Invalid naming")

    return ok()
```

---

## 6. Enforcement Durante la Ejecución

Esto es lo importante: bloquear la operación antes de que se propague el error.

### Reglas del bloqueo
- Si una carpeta ya está asignada a un agente, no se permite otra operación sobre ella
- Si un proyecto no tiene manifest, no se permite crear contenido nuevo dentro de él
- Si el tipo del proyecto es `FIXTURE`, no se puede editar como si fuera proyecto activo
- Si el archivo es un `.env` o base de datos, no debe ser movido a `analysis/` ni `vectorstore/`
- Si el proyecto es `LEGACY`, solo lectura, no escritura

### Reglas de locking

```yaml
resource_lock_rules:
  - resource: "analysis/**"
    allowed_agents: ["researcher-agent"]
    blocked_agents: ["indexer-agent", "cleaner-agent"]

  - resource: "vectorstore/**"
    allowed_agents: ["indexer-agent"]
    blocked_agents: ["manual-editor"]

  - resource: "project-root/**"
    allowed_agents: ["project-manager"]
    blocked_agents: ["all-others"]
```

---

## 7. Auditoría Post-Ejecución

Cada operación debe dejar un log.

### Ejemplo de auditoría

```json
{
  "timestamp": "2026-10-09T21:45:00Z",
  "action": "create_folder",
  "path": "/freebuff/proyecto STRCM/00-admin",
  "actor": "agent-indexer",
  "status": "allowed",
  "reason": "Required folder for PARENT project",
  "validation": "passed",
  "contract_version": "2.0"
}
```

Esto evita la respuesta típica:
- “¿por qué se hizo así?”
- “¿quién lo movió?”
- “¿qué estándar estaba vigente?”

---

## 8. Sistema de Autocorrección

### Regla principal
- Si el error es reversible de forma segura, corregir automáticamente.
- Si el error requiere decisión humana, dejarlo bloqueado y marcarlo.

### Correcciones automáticas que sí son seguras
- Crear `00-admin/PROJECT-MANIFEST.yaml` si falta
- Renombrar folder con convención kebab-case
- Mover `*.db`, `.env`, logs fuera del proyecto a `archive/` o `legacy/`
- Mover archivos prohibidos de `vectorstore/` a `archive/`
- Crear carpeta faltante requerida

### Correcciones que NO deben hacerse auto
- Cambiar proyecto padre/hijo
- Reorganizar aleatoriamente carpetas con contenido complejo
- Mover task de producción a fixture
- Reindexar RAG sin revisión del contenido

---

## 9. Qué debes hacer ahora

### Fase 1: Estándares ejecutables
Crear estas carpetas:

```text
/freebuff/standards/
  projects-schema.yaml
  project-manifest-schema.json
  content-schema.yaml
  validators/
  enforcement/
  audit/
  autocorrect/
```

### Fase 2: Reglas de tipo
Definir estas clasificaciones:
- PARENT
- SUBPROJECT
- TASK
- FIXTURE
- REFERENCE_REPO
- LEGACY

### Fase 3: manifest obligatorio
Todo proyecto nuevo debe tener:
- `PROJECT-MANIFEST.yaml` o `TASK-MANIFEST.yaml`

### Fase 4: bloqueos
Antes de mover o crear algo:
- validar project / folder / path
- bloquear si hay conflicto

### Fase 5: auto-corrección
Si se detecta:
- manifest faltante
- carpeta faltante
- nombre inválido
- archivo prohibido
entonces se corrige y se registra.

---

## 10. Recomendación final

Tu estructura actual ya te da el contenedor. La parte que falta es esta:

- no más decisiones intuitivas
- no más proyectos sin manifest
- no más mezcla de contenido funcional con análisis temporal
- no más cambios sin auditoría

El sistema debe responder siempre a esta lógica:

```text
1. ¿Qué tipo de proyecto es esto?
2. ¿Qué estructura está permitida para ese tipo?
3. ¿Tiene manifest válido?
4. ¿Puede esta IA tocar esta ruta?
5. ¿Qué pasó exactamente?
6. ¿Se puede corregir automáticamente?
```

Si toda operación pasa ese filtro, tu sistema deja de ser “caótico con buena intención” y pasa a ser “gobernado y sostenible”.

---

## 11. Siguiente paso recomendado

Te recomiendo que ahora te pongas en esta secuencia:

1. Crear `/standards/projects-schema.yaml`
2. Crear `/standards/project-manifest-schema.json`
3. Crear `/standards/content-schema.yaml`
4. Crear un `validator.py` que recorra todos los proyectos y reporte violaciones
5. Crear un `watcher.py` que bloquee la creación de cosas fuera de estándar
6. Crear un `auto-fixer.py` que arregle casos seguros

Eso es la parte crítica para dejar tu sistema estable y con orden real.

---

**Documento Generado**: 2026-10-09  
**Versión**: 2.0 (Enforcement-first)  
**Próxima Revisión**: Después de implementar Fase 1
