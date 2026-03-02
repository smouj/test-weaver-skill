---
name: test-weaver
description: "Generación y tejido inteligente de suites de pruebas a partir de requisitos, código y reportes de cobertura"
version: "2.1.0"
author: "OpenClaw Engineering Team"
tags: ["test", "automation", "coverage", "quality", "integration", "weaving"]
category: "quality-assurance"
maintainer: "devops@openclaw.io"
dependencies:
  - "python>=3.9"
  - "pytest>=7.0"
  - "pytest-cov>=4.0"
  - "libcst>=1.0"  # for Python AST manipulation
  - "jq>=1.6"  # JSON processing
  - "git>=2.30"
  - "docker>=20.10"  # optional: for containerized test execution
inputs:
  - "source_code: Ruta al directorio fuente o módulo específico"
  - "requirements: Ruta al documento de requisitos (YAML/Markdown/JSON)"
  - "coverage_report: Archivo .coverage o lcov.info existente"
  - "existing_tests: Directorio de suite de pruebas actual"
  - "framework: Framework de pruebas objetivo (pytest|jest|mocha|googletest)"
  - "strategy: Estrategia de tejido (coverage-gap|risk-based|spec-compliance)"
outputs:
  - "Archivos de prueba generados en directorio de salida especificado"
  - "Reporte de análisis de brechas de cobertura (coverage-gaps.md)"
  - "Manifiesto de tejido (weaving-manifest.json)"
  - "Plan de ejecución de pruebas (test-plan.md)"
requires:
  - "docker (opcional, para ejecución aislada)"
  - "repositorio git con árbol de trabajo limpio"
  - "Permisos de escritura a directorios fuente y de pruebas"
exposes:
  - "test-weaver generate"
  - "test-weaver weave"
  - "test-weaver analyze"
  - "test-weaver validate"
  - "test-weaver optimize"
  - "test-weaver rollback"
---

# Test Weaver Skill

Transforma requisitos y código en suites de pruebas comprehensivas e inteligentemente tejidas con brechas mínimas.

## Purpose

Test Weaver automatiza la creación de cobertura exhaustiva de pruebas analizando código fuente, especificaciones de requisitos y reportes de cobertura existentes para generar pruebas faltantes y tejerlas en suites coherentes. Va más allá de la simple generación de pruebas al comprender dependencias de pruebas, orden de ejecución y complementariedad de cobertura.

**Casos de Uso Reales:**

1. **Llenado de Brechas de Cobertura**: Analiza `.coverage` o `lcov.info` y genera pruebas dirigidas para ramas, condiciones y rutas de excepción no probadas.
2. **Mapeo de Requisitos-a-Prueba**: Consume `requirements.yaml` con especificaciones de características y genera suites de pruebas de cumplimiento que garantizan que todos los criterios de aceptación sean validados.
3. **Pruebas de Código Legado**: Analiza codebases no probadas y produce pruebas de integración que ejercitan interacciones complejas sin requerir mocking completo.
4. **Tejido de Suite de Pruebas**: Fusiona módulos de prueba disjuntos en planes de ejecución optimizados que reducen redundancia mientras mantienen independencia.
5. **Priorización Basada en Riesgo**: Genera pruebas adicionales para módulos de alto riesgo (alta complejidad, cambios recientes, rutas críticas) identificados mediante análisis estático.

## Alcance

### Commands Handled

```bash
# Generate tests from source code analysis
test-weaver generate --source src/module/ --framework pytest --output tests/generated/

# Generate tests from requirements spec
test-weaver generate --requirements docs/features.yaml --framework jest --tag "login,auth"

# Weave existing test files into optimized suite
test-weaver weave --tests tests/unit/ tests/integration/ --strategy coverage-gap --output tests/woven/

# Analyze coverage and identify gaps
test-weaver analyze --coverage .coverage --source src/ --threshold 85

# Validate woven test suite integrity
test-weaver validate --suite tests/woven/ --dry-run

# Optimize suite execution order
test-weaver optimize --suite tests/ --profile execution-profile.json --output tests/optimized/

# Rollback weaving operation
test-weaver rollback --weaving-id WVR-20240115-ABCD1234 --restore-tests
```

## Proceso de Trabajo

### 1. Generation Phase

**Input Collection**:
- Analiza código fuente usando Tree-sitter o LibCST para construir AST y grafos de flujo de control.
- Carga reportes de cobertura (pytest-cov, lcov, JaCoCo) para mapear líneas ejecutadas vs. no ejecutadas.
- Lee documentos de requisitos (formato YAML con claves `feature:`, `scenarios:`, `acceptance:`).
- Escanea archivos de prueba existentes para evitar duplicación y entender patrones.

**Analysis**:
- Identifica funciones/métodos/módulos no probados.
- Detecta ramas condicionales sin cobertura true/false.
- Mapea requisitos a ubicaciones de código mediante matriz de trazabilidad.
- Clasifica complejidad (ciclomática) para priorizar generación de pruebas.

**Generation**:
- Para Python: genera funciones pytest con parametrize, fixtures pytest y mocks usando `unittest.mock`.
- Para JavaScript/TypeScript: genera suites de prueba Jest con bloques describe/it e imports apropiados.
- Incluye casos edge: entradas vacías, valores límite, lanzamiento de excepciones, manejo asíncrono.
- Genera fábricas de datos de prueba usandoFactory Boy o patrones test-data-bot.

**Output**:
- Escribe pruebas a `tests/generated/` con nombres de archivo timestamped: `test_module_generated_20240115_1423.py`.
- Crea manifiesto de tejido (`weaving-manifest.json`) vinculando pruebas generadas a líneas fuente y requisitos.
- Genera reporte de brechas de cobertura (`coverage-gaps.md`) con números de línea faltantes y justificación.

### 2. Weaving Phase

**Dependency Analysis**:
- Analiza todos los archivos de prueba para extraer fixtures, setup/teardown y estado compartido.
- Construye grafo de dependencias de pruebas (nodos=pruebas, aristas=dependencias de fixture, DB compartida, E/S de archivos).
- Identifica pruebas que pueden ejecutarse en paralelo (sin estado mutable compartido).

**Strategy Execution**:
- `coverage-gap`: Prioriza pruebas que cubren más líneas faltantes; minimiza cobertura redundante.
- `risk-based`: Ordena por complejidad de módulo × frecuencia de cambios.
- `spec-compliance`: Agrupa pruebas por ID de requisito para trazabilidad.

**Suite Construction**:
- Ordena pruebas para minimizar overhead de setup/teardown (agrupa por alcance de fixture).
- Divide en shards para ejecución CI paralela (si se proporciona `--shards N`).
- Genera script runner principal que ejecuta en orden óptimo.

**Output**:
- Suite de pruebas optimizada en `tests/woven/`.
- Plan de ejecución (`test-plan.md`) con tiempo estimado y desglose de shards.
- Grafo de dependencias (`dependencies.dot` o `.json`).

### 3. Validation Phase

- Ejecuta pruebas generadas con `--collect-only` para garantizar descubrimiento por runner de pruebas.
- Verifica que todas las imports se resuelvan y los fixtures estén definidos.
- Ejecuta subconjunto de pruebas críticas generadas para confirmar validez básica.
- Revisa que ninguna prueba generada exceda umbrales de timeout (ej. 5s para unit, 30s para integration).
- Asegura que reporte de cobertura muestre mejora (opcional `--verify-coverage`).

## Reglas de Oro

1. **Nunca modificar archivos de prueba existentes**: Las pruebas generadas siempre van a directorio separado (`tests/generated/` o `tests/woven/`). El tejido crea nuevas suites orquestadas pero preserva originales.
2. **Mantener siempre independencia de pruebas**: Las pruebas generadas no deben depender de orden de ejecución o efectos secundarios de otras pruebas. Usar fixtures o patrones de fábrica.
3. **Incluir teardown**: Cada prueba que cree recursos (registros DB, archivos, listeners de red) debe tener cleanup correspondiente.
4. **Ser determinista**: Sin seeds aleatorios sin control explícito; evitar aserciones dependientes de tiempo.
5. **Objetivo 80% cobertura mínima** para módulos nuevos, pero no sacrificar calidad de pruebas por cantidad.
6. **Preservar pruebas existentes que pasan**: El tejido nunca elimina ni altera pruebas originales; solo añade capas de orquestación.
7. **Seguir convenciones del proyecto**: Adherir a nomenclatura existente (`test_*.py` vs `*_test.py`), estilos de aserción (pytest vs unittest) y patrones de ubicación.
8. **Documentar supuestos**: Cada archivo de prueba generado incluye comentario de encabezado vinculando a ID de requisito y ubicación de brecha de cobertura.

## Ejemplos

### Ejemplo 1: Generar pruebas unitarias para función no cubierta

```bash
# Comando
test-weaver generate \
  --source src/auth/token_manager.py \
  --framework pytest \
  --output tests/generated/auth/ \
  --include-integration false

# Salida
INFO: Analizado src/auth/token_manager.py (342 líneas, 18 funciones)
INFO: Reporte de cobertura: función `refresh_token` tiene 0% cobertura de ramas en líneas 156-178
INFO: Archivo de prueba generado: tests/generated/auth/test_token_manager_generated_20240115_1442.py
INFO: Creados 14 casos de prueba cubriendo 23 ramas no cubiertas
INFO: Manifiesto de tejido: weaving-manifest.json actualizado con 14 entradas

# Fragmento de prueba generado (tests/generated/auth/test_token_manager_generated_20240115_1442.py)
import pytest
from unittest.mock import patch, MagicMock
from auth.token_manager import TokenManager

class TestTokenManagerRefreshToken:
    """Generado para brecha de cobertura: cobertura de ramas refresh_token (líneas 156-178)"""
    
    @pytest.fixture
    def token_manager(self):
        return TokenManager(secret="test-secret", ttl=3600)
    
    @patch("auth.token_manager.requests.post")
    def test_refresh_token_success(self, mock_post):
        # Probar ruta exitosa con refresh token válido
        ...
    
    @patch("auth.token_manager.requests.post")
    def test_refresh_token_expired_grant(self, mock_post):
        # Probar excepción cuando grant está expirado
        ...
```

### Ejemplo 2: Tejer pruebas en suite optimizada

```bash
# Comando
test-weaver weave \
  --tests tests/unit/ tests/integration/ \
  --strategy risk-based \
  --shards 4 \
  --output tests/woven/

# Salida:
INFO: Escaneados 247 archivos de prueba (1,843 casos de prueba)
INFO: Grafo de dependencias: 1,102 aristas, 247 nodos
INFO: Identificados 4 shards óptimos para ejecución paralela
INFO: Shard 1 (alto riesgo): tests/woven/shard-01.yml (512 pruebas, ~45m)
INFO: Shard 2: tests/woven/shard-02.yml (483 pruebas, ~38m)
INFO: Shard 3: tests/woven/shard-03.yml (429 pruebas, ~42m)
INFO: Shard 4 (lentas integración): tests/woven/shard-04.yml (419 pruebas, ~55m)
INFO: Suite tejida escrita en tests/woven/ (4 manifiestos YAML de shards)

# Manifiesto de shard generado (tests/woven/shard-01.yml)
shard_id: 1
strategy: risk-based
tests:
  - path: tests/unit/auth/test_token_manager.py::TestTokenManager::test_refresh_token_success
    priority: 10
    estimated_duration: 0.8
    dependencies: ["fixture:db_session"]
  - path: tests/unit/payment/validator_test.py::TestValidator::test_invalid_card_number
    priority: 9
    estimated_duration: 0.3
    dependencies: []
...
```

### Ejemplo 3: Analizar cobertura y generar reporte de brechas

```bash
# Comando
test-weaver analyze \
  --coverage coverage-reports/coverage.xml \
  --source src/ \
  --requirements docs/features.yaml \
  --threshold 85 \
  --output reports/

# Salida
INFO: Parseado coverage.xml (líneas totales: 12,453, cubiertas: 9,671, %: 77.6)
INFO: Umbral 85% no alcanzado. Faltan 1,782 líneas en 234 funciones.
INFO: Mapeo de requisitos: 18 criterios de aceptación carecen de validación de pruebas.
INFO: Generado coverage-gaps.md con lista priorizada:

# reports/coverage-gaps.md
## Brechas de Cobertura -Orden de Prioridad

1. **src/payment/processor.py:process_refund (líneas 201-245)**
   - Ramas faltantes: manejo de excepciones (201-218), caso edge (219-230)
   - Requisitos: REQ-PAY-004 (Procesamiento de reembolso)
   - Complejidad: ciclomática 12 (alta)
   - Acción: generar 4 pruebas cubriendo todas las ramas

2. **src/user/profile.py:update_privacy (líneas 87-103)**
   - Líneas faltantes: 95-98 (validación de campos GDPR)
   - Requisitos: REQ-USER-011 (Cumplimiento de privacidad)
   - Complejidad: ciclomática 3
   - Acción: generar 1 prueba con casos edge GDPR

...
```

## Dependencias y Requisitos

**Paquetes del Sistema**:
- Python 3.9+ (para motor de generación de pruebas)
- Node.js 18+ (si se generan pruebas Jest)
- jq (procesamiento JSON)
- git (para detectar archivos cambiados)

**Paquetes Python** (`requirements.txt` para test-weaver):
```
pytest>=7.0
pytest-cov>=4.0
libcst>=1.0
PyYAML>=6.0
jinja2>=3.0  # plantillas para generación de pruebas
networkx>=3.0  # grafo de dependencias
click>=8.0  # CLI
```

**Variables de Entorno**:
```bash
# Opcional: sobrescribir directorios de prueba por defecto
export TEST_WEAVER_SOURCE_DIR="src/"
export TEST_WEAVER_TEST_DIR="tests/"
export TEST_WEAVER_GENERATED_DIR="tests/generated/"
export TEST_WEAVER_WOVEN_DIR="tests/woven/"

# Integración CI
export TEST_WEAVER_CI_MODE="true"  # omite prompts interactivos
export TEST_WEAVER_MAX_TESTS_PER_MODULE="20"  # límite de generación
export TEST_WEAVER_TIMEOUT_PER_TEST="5"  # segundos

# Umbrales de cobertura
export TEST_WEAVER_COVERAGE_THRESHOLD="85"
```

## Pasos de Verificación

Después de ejecutar cualquier comando test-weaver, verificar:

1. **Pruebas generadas son sintácticamente válidas**:
   ```bash
   python -m py_compile tests/generated/*.py
   # o para JS:
   npx jshint tests/generated/*.js
   ```

2. **Pruebas son descubribles por runner**:
   ```bash
   pytest --collect-only tests/generated/ | grep "test_"
   # debe listar pruebas generadas sin errores
   ```

3. **Cobertura mejora** (si `--verify-coverage`):
   ```bash
   pytest tests/generated/ --cov=src/module --cov-report=json
   # inspeccionar coverage.json para confirmar líneas dirigidas ahora cubiertas
   ```

4. **Suite tejida ejecuta sin errores de dependencia**:
   ```bash
   pytest tests/woven/shard-01.yml -v  # o usar script runner generado
   # debe ejecutar todas las pruebas en shard sin errores de import/fixture
   ```

5. **Integridad de manifiesto**:
   ```bash
   jq . weaving-manifest.json  # debe ser JSON válido
   jq -e '.[] | select(.source_line == null)' weaving-manifest.json && echo "FAIL" || echo "PASS"
   # todas las entradas deben tener source_line y requirement_id
   ```

## Solución de Problemas

**Síntoma**: Pruebas generadas fallan con errores de import.
- **Causa**: Framework de pruebas no está en dependencias del proyecto.
- **Solución**: Instalar framework requerido o especificar `--framework none` y ajustar plantillas.

**Síntoma**: Tejido produce advertencias de dependencia circular.
- **Causa**: Dos pruebas comparten fixtures que dependen entre sí.
- **Solución**: Refactorizar fixtures para usar scope session-scoped o function-scoped apropiadamente; usar `--strategy coverage-gap` para minimizar compartir fixtures.

**Síntoma**: Cobertura no mejora después de añadir pruebas generadas.
- **Causa**: Pruebas generadas apuntan a rutas fuente incorrectas o patrones exclude en configuración de cobertura.
- **Solución**: Asegurar que `--source` coincide con rutas de import; verificar `omit` patterns en `.coveragerc`.

**Síntoma**: Ejecución de prueba extremadamente lenta después de tejido.
- **Causa**: Suite tejida incluye muchas pruebas de integración que deberían estar separadas.
- **Solución**: Usar `--shards` y ejecutar separadamente; generar solo pruebas unitarias con `--include-integration false`.

**Síntoma**: Rollback falla con "weaving-id no encontrado".
- **Causa**: ID proporcionado no coincide con ninguna entrada en log de weaver.
- **Solución**: Revisar `weaving-manifest.json` para IDs válidos; usar `test-weaver list-backups` para ver puntos de rollback disponibles.

## Comandos de Rollback

Test Weaver mantiene puntos de rollback antes de cada operación `generate` y `weave`.

```bash
# Listar puntos de rollback disponibles
test-weaver list-backups
# Salida:
# WVR-20240115-ABCD1234: weave 2024-01-15 14:23 UTC (targeto: tests/woven/)
# WVR-20240115-EFGH5678: generate 2024-01-15 10:11 UTC (targeto: tests/generated/auth/)

# Rollback de operación de tejido (remueve suite tejida, restaura selección de pruebas original)
test-weaver rollback --weaving-id WVR-20240115-ABCD1234

# Rollback de última operación (más reciente)
test-weaver rollback --latest

# Rollback pero mantener pruebas generadas (solo remover manifiestos de tejido)
test-weaver rollback --weaving-id WVR-20240115-ABCD1234 --keep-generated

# Restauración completa: remueve tanto pruebas generadas como tejidas, revierte árbol git de trabajo
# ADVERTENCIA: destructivo
test-weaver rollback --weaving-id WVR-20240115-ABCD1234 --hard --git-rollback

# Verificar estado de rollback (dry-run)
test-weaver rollback --weaving-id WVR-20240115-ABCD1234 --dry-run
```

Implementación de rollback:
1. Copias de directorios de prueba originales almacenadas en `.test-weaver-backups/WVR-<id>/`.
2. Integración git: commit cambios pre-operación a rama temporal `test-weaver/WVR-<id>`; `--git-rollback` resetea a ese commit.
3. Manifiesto de tejido mantiene campo `backup_path` apuntando a ubicación de backup.

## Rendimiento y Escalabilidad

- **Generación**: ~50-200 pruebas por minuto dependiendo de complejidad de fuente.
- **Tejido**: O(V+E) en grafo de dependencias de pruebas; 1,000 pruebas completa en ~5s.
- **Memoria**: ~200MB para grafo de dependencias de 10,000 pruebas.

Usar `--shards N` para paralelismo CI; usar `--dry-run` para validar sin computación pesada.
```