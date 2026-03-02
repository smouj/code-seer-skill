name: Code Seer
slug: code-seer
version: 1.2.0
description: Motor de análisis predictivo de código que anticipa cómo evolucionará tu codebase, identifica oportunidades de refactorización antes de que se conviertan en cuellos de botella, y proporciona recomendaciones arquitectónicas basadas en datos
author: SMOUJBOT Team
tags:
  - refactoring
  - prediction
  - architecture
  - technical-debt
  - code-quality
  - maintenance
type: analysis
entrypoint: code-seer
depends:
  - python>=3.9
  - tree-sitter
  - networkx
  - scikit-learn
  - pandas
  - rich
  - gitpython
  - SQLite (for caching)
permissions:
  - read:files
  - analyze:code
  - suggest:refactor
environment:
  CODESEER_MODEL: default
  CODESEER_CACHE_TTL: "86400"
  CODESEER_VERBOSE: "false"
  CODESEER_MIN_CONFIDENCE: "0.7"
```

# Code Seer

Motor de análisis predictivo de código que anticipa cómo tu codebase necesitará evolucionar, identifica oportunidades de refactorización antes de que se conviertan en cuellos de botella, y proporciona recomendaciones arquitectónicas basadas en datos.

## Propósito

### Casos de Uso Reales

1. **Refactorización Preventiva** - Identifica código que se volverá problemático en 3-6 meses basado en patrones de cambio actuales y crecimiento de complejidad
2. **Detección de Deriva Arquitectónica** - Detecta cuando el código se desvía de patrones arquitectónicos previstos (ej. Clean Architecture, hexagonal) antes de que se acumule deuda técnica
3. **Predicción de Evolución de Dependencias** - Predice qué módulos se volverán enredados y sugiere modularización antes de que el acoplamiento sea inmanejable
4. **Análisis de Estabilidad de API** - Determina qué interfaces son estables vs volátiles basado en historial de cambios y patrones de uso
5. **Inteligencia para Onboarding de Equipo** - Resalta áreas de código que cambian frecuentemente (hotspots) y sugiere mejoras en documentación

## Alcance

### Comandos

```bash
# Predecir hotspots de código para los próximos 90 días
code-seer predict-hotspots --path ./src --days 90 --output json

# Analizar deriva arquitectónica contra Clean Architecture
code-seer check-architecture --pattern clean --boundaries-dir ./docs/architecture

# Generar roadmap de refactorización con estimaciones de prioridad y esfuerzo
code-seer roadmap --path . --weeks 12 --format markdown

# Predecir crecimiento de acoplamiento y sugerir límites de módulos
code-seer coupling-forecast --threshold 0.8 --suggest-split

# Identificar APIs volátiles (interfaces con >3 cambios disruptivos/mes)
code-seer api-stability --path ./packages --min-stability-score 0.7

# Detectar God Objects y Sugerir Divisiones
code-seer detect-god-objects --complexity-threshold 50 --output-diff
```

## Proceso de Trabajo Detallado

### 1. Ingesta del Codebase y Análisis Histórico

```
Paso 1.1: Minería de Historial Git (30-60 min para proyectos medianos)
  - Extraer frecuencia de commits por archivo/módulo (últimos 90 días)
  - Calcular métricas de churn: líneas añadidas/borradas por commit
  - Identificar patrones de co-cambio (archivos modificados juntos >5 veces)
  
  Comando: git log --since="90 days ago" --name-only --pretty=format: | sort | uniq -c | sort -nr

Paso 1.2: Análisis Estático de Código
  - Parsear AST usando tree-sitter para todos los lenguajes objetivo
  - Calcular complejidad ciclomática, complejidad cognitiva, líneas de código
  - Identificar capas arquitectónicas (si se proporciona patrón)
  - Extraer grafo de dependencias (imports, includes, requires)

Paso 1.3: Predicción con Machine Learning
  - Cargar modelo pre-entrenado (entrenado con 10k+ repos open source)
  - Características de entrada: churn, complejidad, acoplamiento, ownership del equipo, cobertura de tests
  - Predecir: probabilidad de modificación en próximos N días, puntuación de urgencia de refactor
```

### 2. Detección de Refactorización Específica

```
Paso 2.1: Detección de God Object
  Condiciones que activan la detección:
    - Clase con >150 LOC Y >10 métodos Y >15 campos
    - Clase con >20 dependencias directas (imports/inherits)
    - Clase con acoplamiento entre 2+ capas arquitectónicas
  
  Acciones sugeridas:
    - Extract Class (Extraer Métodos/Campos a nuevos tipos)
    - Delegar responsabilidades a objetos de servicio
    - Introducir interfaces para reducir acoplamiento

Paso 2.2: Patrón Shotgun Surgery
  Detección: Múltiples módulos no relacionados modifican mismo archivo (>3 PRs/semana modificando mismo archivo)
  Sugerencia: Extraer Strategy Pattern, usar inyección de dependencias

Paso 2.3: Detección de Feature Envy
  Métricas: Método usa datos de otra clase >70% de sus operaciones
  Sugerencia: Mover método a la clase de la que es "codicioso"
```

### 3. Generación de Salida

- Primaria: JSON estructurado con puntuaciones y números de línea específicos
- Secundaria: Markdown legible por humanos con snippets de código mostrando before/after
- Opcional: Plantilla de descripción de PR lista para copiar-pegar

## Reglas de Oro Específicas de Code Seer

1. **Nunca** sugerir refactorización sin confianza >= `MIN_CONFIDENCE` (por defecto 0.7)
2. **Siempre** proporcionar números de línea concretos y diffs de 3-5 líneas
3. **Priorizar** sugerencias por: (impact_score * probability) / estimated_effort_hours
4. **Respetar** tests existentes - nunca sugerir refactorización que rompería >5% de cobertura de tests
5. **Validar** todas las predicciones contra historial git (backtest de precisión a 30 días)
6. **Cachear** resultados por 24h para evitar re-analizar archivos sin cambios
7. **Nunca** sugerir cambios disruptivos a APIs públicas marcadas como v1.0+ sin guía de migración
8. **Siempre** buscar patrones duplicados antes de sugerir nuevas abstracciones

## Ejemplos

### Ejemplo 1: Predecir Hotspots

**Entrada del Usuario:**
```bash
code-seer predict-hotspots --path ./src --days 90 --format json
```

**Salida:**
```json
{
  "hotspots": [
    {
      "file": "src/auth/middleware.ts",
      "prediction": {
        "modification_probability": 0.94,
        "estimated_churn_next_90d": 320,
        "refactor_urgency": "critical",
        "reason": "8 commits in last 30d, cyclomatic complexity 28, 12 dependencies"
      },
      "suggested_refactor": {
        "type": "extract_strategy",
        "description": "Split authentication strategies (OAuth, JWT, Session) into separate classes",
        "estimated_effort_hours": 8,
        "risk": "low"
      }
    }
  ]
}
```

### Ejemplo 2: Detectar God Objects con Diff

**Entrada del Usuario:**
```bash
code-seer detect-god-objects --complexity-threshold 40 --output-diff
```

**Salida:**
```diff
--- a/src/core/UserService.ts
+++ b/src/core/UserService.ts (suggested split)
@@ -1,5 +1,10 @@
+// SUGGESTION: Extract UserValidator and UserNotifier
+// Rationale: Class has 214 LOC, 18 methods, and handles 3 concerns
+
 class UserService {
-  // 214 lines of mixed validation, notification, persistence logic
+  private validator: UserValidator;
+  private notifier: UserNotifier;
   private repository: UserRepository;
 }
```

**Archivo adicional generado:** `refactor-plan/user-service-split-20240115.md` con pasos completos de migración y estrategia de preservación de tests.

### Ejemplo 3: Deriva Arquitectónica

**Entrada del Usuario:**
```bash
code-seer check-architecture --pattern hexagonal --domain src/domain --ports src/ports --adapters src/adapters --report html
```

**Salida:**
```
Architecture Drift Report (Hexagonal)
=====================================

VIOLATIONS FOUND: 3

1. Domain layer depends on Infrastructure (HIGH)
   File: src/domain/models/Order.ts:45
   Imports: import {DatabaseConnection} from '../../infrastructure/db'
   Fix: Move DatabaseConnection to Ports, inject via constructor

2. Adapters directly access Domain entities (MEDIUM)
   Files: src/adapters/http/OrderController.ts:12,23
   Issue: Direct property access instead of using domain methods
   Fix: Use Order.confirm() instead of order.status='confirmed'

3. Missing dependency rule enforcement (INFO)
   Ports layer imports concrete adapter
   Action: Add dependency-check CI step
```

## Dependencias y Requisitos

**Requeridos:**
- `tree-sitter` (parsing para 15+ lenguajes)
- `gitpython` (análisis de historial git)
- `networkx` + `scikit-learn` (modelos de predicción)
- `sqlite3` (caché de resultados)

**Opcionales (predicciones mejoradas):**
- `pandas` (análisis estadístico)
- `radon` (métricas de complejidad para Python)
- `eslint`/`typescript` (para codebases JS/TS)

**Instalación:**
```bash
pip install -r requirements-code-seer.txt
code-seer init --cache-dir ~/.cache/code-seer --model-dir ~/.local/share/code-seer/models
```

## Pasos de Verificación

```bash
# 1. Verificar instalación y dependencias
code-seer --version && code-seer health

# Salida esperada:
# Code Seer v1.2.0
# Cache: healthy (/home/user/.cache/code-seer)
# Model: loaded (architecture-predictor-v3.pkl)
# Tree-sitter: 15 languages available

# 2. Ejecutar dry-run para ver qué se analizaría
code-seer predict-hotspots --path ./test-projects/sample --dry-run

# Esperado: "Would analyze 42 files, 2,847 LOC, 6 months of git history"

# 3. Ejecutar con umbral de confianza
code-seer predict-hotspots --path . --min-confidence 0.8

# 4. Validar predicciones manualmente (revisar 2-3 sugerencias)
code-seer validate --prediction-file predictions.json --check-accuracy

# Esperado: "Backtest accuracy: 87% (predictions matched actual changes)"
```

## Variables de Entorno

```bash
export CODESEER_MODEL=architecture-predictor-v3  # Por defecto: latest
export CODESEER_CACHE_TTL=86400                  # TTL de caché en segundos (por defecto: 1 día)
export CODESEER_VERBOSE=true                     # Habilitar logging debug
export CODESEER_MIN_CONFIDENCE=0.65              # Confianza mínima de predicción para mostrar
export CODESEER_GIT_BIN=/usr/bin/git             # Sobrescribir ruta de git binary
export CODESEER_MAX_FILES=10000                  # Límite de seguridad para análisis
export CODESEER_EXCLUDE="node_modules,venv,.git,build"
```

## Solución de Problemas Comunes

### Problema 1: "No git history found"
**Causa:** Ejecutando en repo con <30 días de historia o shallow clone
**Fix:**
```bash
git fetch --unshallow 2>/dev/null || git fetch --depth=10000
code-seer reset-cache && code-seer predict-hotspots --path .
```

### Problema 2: "MemoryError on large codebase"
**Causa:** Memoria insuficiente para parsing completo de AST
**Fix:**
```bash
code-seer predict-hotspots --path . --max-files 5000 --batch-size 100
# O dividir por subdirectorio
find src -name "*.ts" -type f | head -n 2000 > filelist.txt
code-seer predict-hotspots --file-list filelist.txt
```

### Problema 3: "Model loading failed"
**Causa:** Archivo de modelo faltante o corrupto
**Fix:**
```bash
code-seer init --download-model architecture-predictor-v3
# O resetear a por defecto
rm -rf ~/.local/share/code-seer/models
code-seer init
```

### Problema 4: "False positive: Suggesting refactor for generated code"
**Causa:** Code Seer analizando archivos auto-generados
**Fix:** Añadir patrones para ignorar:
```bash
code-seer config set ignore_patterns '["*.pb.go", "*_generated.*", "migrations/*"]'
# O usar archivo .codeseerignore (similar a .gitignore)
echo "dist/" >> .codeseerignore
echo "*.min.js" >> .codeseerignore
```

## Comandos de Rollback

### Deshacer Refactorización Sugerida (si se aplicó manualmente)
```bash
# Restaurar desde git antes de que se mergeara la rama de refactor
git checkout main
git log --oneline --grep="refactor:" | head -1 | cut -d' ' -f1 | xargs git revert --no-edit

# O usar Code Seer para generar diff de rollback
code-seer rollback --refactor-id "extract-strategy-20240115" --format patch > rollback.patch
patch -p1 < rollback.patch
```

### Deshabilitar Predicciones para Archivo/Directorio Específico
```bash
# Añadir exclusión permanente
code-seer config add exclude_path ./legacy/unsupported-module

# O skip temporal con flag
code-seer predict-hotspots --path . --skip-pattern "legacy/*"
```

### Restaurar Estado Original de Git (si análisis causó cambios no deseados)
```bash
# Seguridad: Code Seer nunca modifica archivos, solo sugiere
# Pero si aplicaste sugerencias y quieres revertir:
git reset --hard HEAD
git clean -fd

# Si ya hiciste commit:
git revert HEAD
```

### Limpiar Caché y Re-analizar con Nueva Configuración
```bash
code-seer cache clear --all
code-seer config set min_confidence 0.6
code-seer predict-hotspots --path .
```

## Notas de Rendimiento

- Análisis inicial: 1-5 minutos para codebase de 10k LOC
- Cache hit en ejecuciones subsecuentes: <1 segundo
- Precisión de predicción: ~85% cuando se backtestea contra cambios de 6 meses
- Mejores resultados: Ejecutar semanalmente, integrar con CI/CD para跟踪 deriva a lo largo del tiempo
```