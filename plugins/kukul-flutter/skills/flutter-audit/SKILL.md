---
name: flutter-audit
description: Ejecuta una auditoría técnica completa de un proyecto Flutter (seguridad, performance, refactor, arquitectura, dependencias, tests, CI) con evidencia medida y genera docs/AUDITORIA_<fecha>.md priorizado P0–P3 con plan de trabajo. Usar cuando el usuario pida auditar, revisar la salud técnica o evaluar el nivel de producción de una app Flutter.
---

# Auditoría técnica de proyecto Flutter

Produce un documento `docs/AUDITORIA_<YYYY-MM>.md` con hallazgos **verificados con evidencia**, priorizados y con plan de ataque. Estructura: auditoría base + anexo profundo + plan de seguimiento.

## Reglas de oro

1. **Evidencia o no existe.** Cada hallazgo cita `ruta:línea` y/o una métrica obtenida con un comando. Nada de "parece que...". Si no pudiste verificarlo, va en "No verificable desde el repo".
2. **Nunca incluyas valores de secretos**, solo rutas/nombres de variables. El documento debe ser seguro de versionar.
3. **Verifica antes de afirmar.** Las heurísticas dan falsos positivos (ej. `Timer` sin `dispose` que sí se cancela vía `ref.onDispose`). Lee el código antes de reportar; documenta los falsos positivos descartados para cerrar la duda.
4. **Distingue basura de bug latente.** Un campo "sin usar" puede ser una máquina de estados que nunca se pinta; léelo antes de proponer borrarlo.
5. **Respeta decisiones de producto.** Si algo parece bug pero podría ser intencional (ej. `||` que vuelve una validación tautológica), repórtalo como pregunta, no lo "arregles" sin confirmación.
6. **Primero lee, luego cambia.** La auditoría es read-only salvo los *quick-wins* de la Fase 3, y esos solo si el usuario lo autoriza.
7. Comprueba si el proyecto usa FVM (`.fvmrc`/`.fvm/`): en ese caso prefija `fvm flutter` / `fvm dart`.
8. Respeta el idioma del proyecto (docs, CLAUDE.md). Si ya existe un `docs/AUDITORIA_*.md`, **verifica su estado actual** en vez de rehacerlo (ver Fase 6).
9. **Docs con Context7**: antes de afirmar que una API está deprecada, que un patrón no es el recomendado o que un paquete tiene una configuración insegura, compruébalo en la documentación de la versión que usa el proyecto. Si Context7 está instalado, úsalo; si no, propón instalarlo (ver sección siguiente).

## Documentación con Context7

Para consultar docs de Flutter, Dart y paquetes usa **Context7** (MCP), que sirve documentación actualizada por versión en vez de depender de la memoria del modelo.

1. **Detecta si está disponible**: comprueba si tienes las herramientas `resolve-library-id` y `query-docs` del servidor `context7` (p. ej. `mcp__context7__*` o `mcp__plugin_context7_context7__*`). Si el servidor aparece pero pide autenticación, díselo al usuario (`/mcp`).
2. **Si está disponible, úsalo**: `resolve-library-id` con el nombre del paquete → elige el ID `/org/proyecto` (versionado si existe para la versión del proyecto) → `query-docs` con una pregunta concreta por tema (una API, una migración, una configuración). Cita en el documento la fuente consultada.
3. **Si no está instalado, propón instalarlo** al empezar la auditoría, sin bloquear:
   ```bash
   claude mcp add --transport http context7 https://mcp.context7.com/mcp
   ```
   (o desde una sesión: `/plugin install context7@claude-plugins-official`). Requiere reiniciar Claude Code. Si el usuario no quiere, sigue con las fuentes oficiales (pub.dev changelog, `docs.flutter.dev`, `dart.dev`, repos de GitHub) y anótalo en el documento.
4. Context7 complementa, no sustituye: **las versiones reales salen de `pub outdated`/releases oficiales** y el impacto se confirma siempre con `grep` en el código del proyecto.

## Escala de prioridad

- **P0** bloquea release / riesgo real (firma release, secretos abusables, fail-open a dev).
- **P1** alto impacto (fuga de PII a logs, CI ausente, suite de tests rota, analyzer ahogado).
- **P2** calidad (rebuilds, cobertura, codegen, archivos-Dios, números mágicos).
- **P3** pulido (imports muertos, catch vacíos, TODOs, versiones a medias).

## Fase 0 — Contexto (no saltar)

```bash
cat pubspec.yaml; cat analysis_options.yaml; ls .github .gitlab-ci.yml .fvmrc 2>&1
cat CLAUDE.md README.md 2>/dev/null | head -100; ls docs
git log --oneline | head -20
find lib -name '*.dart' | wc -l; find lib -name '*.dart' | xargs cat | wc -l
ls lib lib/features lib/core 2>/dev/null
```

Anota: tamaño, nº de features, state management (Riverpod/Bloc/Provider/GetX), router, cliente HTTP, auth, backend, plataformas (android/ios/web), si usa codegen, flavors, y si la documentación del repo describe la realidad.

## Fase 1 — Baseline objetivo

```bash
flutter analyze 2>&1 | tail -40        # ¿cuántos issues? ¿vienen de build/?
dart format --output=none --set-exit-if-changed lib test | tail -5
flutter test 2>&1 | tail -30           # ¿compila? ¿pasa?
flutter pub outdated 2>&1 | head -40
```

Si `analyze` reporta cientos de issues, mira si vienen de `build/**` (SourcePackages de SPM) y no de `lib/`: es un hallazgo P1 (falta `analyzer.exclude`). Un solo test que no compila tumba toda la suite y **oculta** otros fallos: repórtalo y anota que al arreglarlo pueden aparecer más.

## Fase 2 — Barridos por categoría

Ejecuta los greps, registra los conteos en la tabla de métricas y **abre los archivos** de los resultados relevantes.

### 1. Seguridad
- **Android release**: `android/app/build.gradle*` → ¿`signingConfig` release usa keystore real (vía `key.properties` no versionado) o `debug`? ¿`isMinifyEnabled`/`isShrinkResources`/`proguardFiles`? ¿existe `proguard-rules.pro`? ¿Crashlytics sube mapping? Si `signingConfig` release usa `debug`, **no lo clasifiques P0 de inmediato**: pregunta al usuario cómo se firma realmente hoy (keystore local, Play App Signing, CI, paso manual) y custodia de credenciales. Con el proceso real: firma fuera del repo / credenciales compartidas a mano = P1 (no reproducible, factor de bus); firma debug llegando a distribución = P0.
- **Secretos**: `grep -rnE "(api[_-]?key|secret|token|AIza|sk_live|pk_)" lib android ios --include=* -i` y `defaultValue:` en `String.fromEnvironment`. Matiza: Firebase apiKey y Stripe *publishable* son públicos por diseño; el riesgo real es Google Maps key sin restringir y el **fail-open a dev** (build prod que olvida un `--dart-define` y cae a endpoints de dev). Busca un guard tipo `assertProductionReady`. ¿`.env` o keystores versionados? `git ls-files | grep -E "\.env|key\.properties|\.jks|google-services|GoogleService-Info"` (para estos últimos, valora si es política del equipo).
- **Fugas a logs**: `grep -rn "debugPrint\|print(" lib` → ¿imprimen objetos `User`, tokens, `response.data`, errores crudos? `debugPrint` **no** se elimina en release. Revisa qué contexto se manda a Crashlytics/Sentry (`grep -rn "recordError\|setCustomKey\|context:"`): no debe llevar payloads con PII.
- **Red**: certificate pinning (si hay pagos), `usesCleartextTraffic`/`networkSecurityConfig` en `AndroidManifest.xml`, `NSAppTransportSecurity`/`NSAllowsArbitraryLoads` en `Info.plist`. Reporta explícitamente si "depende del default de plataforma".
- **Auth/interceptores**: ¿qué hace ante 401/403? ¿logout agresivo? ¿almacenamiento del token (secure storage vs SharedPreferences)? ¿bucles de refresh?
- **Permisos** del manifest/Info.plist: ¿hay permisos que no se usan?

### 2. Performance
- Riverpod: `grep -rc "\.select(" lib | …` vs `ref.watch(` vs `ref.read(`. `watch` de estado completo en pantallas grandes con `Timer.periodic`/streams = rebuilds de más. Si es Bloc: `BlocBuilder` sin `buildWhen`, `context.watch` amplio. Provider: `Consumer` sin `Selector`.
- `MediaQuery.of(context).size` → `MediaQuery.sizeOf` (reconstruye al abrir el teclado).
- `ListView(children:` con listas largas → `.builder`/`.separated`.
- `Timer`/`StreamSubscription`/`AnimationController`/`TextEditingController`: contrasta creación vs `cancel()`/`dispose()`. **Verifica cada candidato** (puede cancelarse en `ref.onDispose`).
- `const`: conteo de uso; `setState` masivo; `StatefulWidget` vs total.
- Imágenes: `Image.network` sin caché (¿`cached_network_image`?), assets sin optimizar (`find assets -size +500k`).
- Trabajo pesado en el hilo UI (parseo JSON grande sin `compute`/`Isolate`).

### 3. Refactor / calidad de código
- **Archivos-Dios**: `find lib -name '*.dart' | xargs wc -l | sort -rn | head -15` (umbral >900 líneas) y archivos barril con muchas clases (`grep -c "^class" archivo`).
- **Errores silenciados**: `grep -rnE "catch \(_\) *\{ *\}"` y `grep -rc "catch (_)"`. Clasifica: aceptables (reportar error no debe crashear) vs fallos invisibles en flujos de negocio/dinero.
- **Parsing frágil**: `grep -rc "as Map<"`, `as List`, `as String`. ¿Hay `freezed`/`json_serializable`? Sin codegen, un cambio de shape del backend revienta en runtime. Detecta **duplicación del unwrap** del backend (ej. `res[0]`, `data['data']`) repetida por repositorio.
- **Números/strings mágicos**: IDs de dominio como enteros crudos (`== 1`, `typeUser == 2`, `countryId`), roles, estados como strings. Propón `enum` con valor. Cuidado con enums existentes que colisionen de nombre.
- **Duplicación de constantes** (URLs, teléfonos, mails en 2+ sitios).
- **i18n**: ratio de `.tr()`/`AppLocalizations` vs `Text('literal')`; lista los strings hardcodeados *reales* (descarta dartdoc y comentarios); verifica que existan en **todos** los idiomas.
- **TODO/FIXME/HACK**: `grep -rnE "TODO|FIXME|HACK|XXX" lib | wc -l`; separa notas de deuda funcional real, y detecta TODOs obsoletos.
- Código muerto, imports sin usar (`dart fix --dry-run`), dev-dependencies redundantes.

### 4. Arquitectura y buenas prácticas
- ¿Estructura feature-first / capas coherente? ¿hay `data/domain/presentation` o se filtra lógica de negocio a widgets? ¿widgets que llaman directamente a Dio/Firebase?
- Inyección de dependencias y testabilidad (¿hay costuras/fakes posibles?).
- Router: guards, deep links, rutas placeholder pendientes.
- **State**: `ref.read` dentro de `build()` o getters reactivos (UI desactualizada: es caza de bugs, no cosmético). Estados mutables compartidos, notifiers "gordos".
- **Observabilidad**: manejo global de errores (`runZonedGuarded`, `FlutterError.onError`, `PlatformDispatcher.onError`), logger con redacción, analytics real vs solo breadcrumbs.
- **Config por entorno**: flavors / `--dart-define`; ¿se lee `.env` en runtime?
- **CI/CD**: existencia de `.github/workflows`, `.gitlab-ci.yml`, Codemagic, fastlane. Si no hay: P1 (mayor multiplicador de calidad por esfuerzo).
- **Documentación**: ¿README/CLAUDE.md/ARQUITECTURA describen la realidad actual?

### 5. Tests
- `find test -name '*_test.dart' | wc -l` vs nº de features/líneas. Mapa de qué features tienen cobertura y cuáles no.
- **Cobertura en flujos de dinero/críticos** (pagos, suscripciones, permisos, tarifas, auth): suele ser cero → P2 con prioridad concreta.
- Tests plantilla (`widget_test.dart` del contador), fakes desactualizados que no compilan, tests que prueban tautologías.
- Cuando destrabes la compilación, **corre la suite completa**: pueden aparecer fallos ocultos; investígalos (¿bug de código o test desactualizado? ¿decisión de producto?).

### 6. Dependencias e integraciones
- Para los paquetes clave (state management, router, HTTP, auth, pagos, almacenamiento seguro) consulta en Context7 las APIs deprecadas y buenas prácticas de **la versión instalada** y contrasta con el uso real en `lib/`.
- `pubspec.yaml`: paquetes pre-1.0 con caret (`^0.x`) → fijar versión exacta; comentarios de versiones alternas (`#^12.2.1`) = upgrades a medias; deps abandonadas; `dependency_overrides`; dev-deps duplicadas.
- SDKs nativos con licencia/tracking en background, integraciones **comentadas o con feature flag** (verifica si siguen comentadas: el estado cambia).
- Versiones de AGP/Gradle/Kotlin/Xcode/min SDK, targetSdk vs requisitos actuales de las tiendas.
- Plataformas generadas que no se usan (web/windows/linux).

## Fase 3 — Quick-wins (solo con autorización del usuario)

Cambios mecánicos y sin riesgo, aplicables en la misma pasada. Reporta antes/después con números:
- `analyzer.exclude` con `build/**`, `**/*.g.dart`, `**/*.freezed.dart`.
- `dart fix --apply` (inspecciona `--dry-run` antes).
- Eliminar `debugPrint` con PII; `kDebugMode` en los demás; recortar `response.data` de reportes de error.
- Dev-dependency redundante; comentarios de versiones muertas.
Deja **fuera** cualquier cambio de lógica, formato masivo (`dart format` va en commit aislado) y lints estrictos (suben el conteo de golpe: MR aparte).

## Fase 4 — Redactar el documento

Estructura de `docs/AUDITORIA_<YYYY-MM>.md`:

1. **Cabecera**: fecha, nº archivos/líneas/features, nota "cita rutas, nunca valores".
2. **Resumen ejecutivo**: 1 párrafo: qué está sólido y que las brechas son de endurecimiento o estructurales. Leyenda P0–P3.
3. **Secciones numeradas** por categoría (Seguridad, Refactor, Arquitectura, Performance, Limpieza, Dependencias). Cada hallazgo:
   `### P<n> — <título>` → archivo:línea, snippet corto si ayuda, **por qué importa** (riesgo concreto, no genérico), matiz (qué es falso positivo/público por diseño) y **Acción** concreta (sin estimar viabilidad ni tiempo).
4. **Verificado y limpio**: falsos positivos descartados.
5. **Métricas base** (tabla con conteo + comando fuente): archivos·líneas, issues de analyze, `ref.read/watch/select`, casts `as Map<`, `catch (_) {}`, timers/cancels, `const`, archivos >900, tests·features, archivos sin formatear.
6. **Plan de ataque** ordenado: P0 ahora → P1 sprint 1 → P2 sprint 2 → P3 continuo.
7. **Checklist de seguimiento** en tandas por esfuerzo/impacto, con IDs estables (`B1…Bn`) y casillas `- [ ]`:
   - Tanda 1: trivial, mecánico, sin riesgo.
   - Tanda 2: mecánico con criterio (≈0.5–1 día).
   - Tanda 3: transversal, requiere revisión (≈1–3 días).
   - Tanda 4: especializado, alto impacto (por sprint).
   Anota **dependencias entre ítems** (ej. "CI después de arreglar tests y formatear", "dividir archivos grandes después de tener tests") y los que no son cerrables desde el repo (ej. restringir API key en consola cloud).
8. **Tabla de seguimiento** con columnas de fecha (antes/después/objetivo).
9. **Pendiente fuera del código**: lo que requiere confirmar en consolas externas.

## Fase 5 — Entrega

Resume al usuario: top 3 riesgos, métricas clave, qué quick-wins aplicaste (con números) y qué decisiones necesitas de él (producto/backend). No pegues el documento entero en el chat.

## Fase 6 — Re-auditoría (si ya existe un doc previo)

Añade sección "Plan de trabajo y seguimiento (fecha)": verifica **cada** hallazgo anterior contra el repo hoy (lee los archivos, reejecuta los greps), clasifica en **A. Resuelto ✅ (con evidencia)** y **B. Pendiente**, anota cambios no documentados (ej. una integración que estaba comentada y ya está activa), actualiza la tabla de métricas y registra hallazgos nuevos destapados al arreglar bloqueantes. No reescribas el historial: añade.

## Anti-patrones

- Listar problemas genéricos de Flutter sin evidencia del repo.
- Reportar conteos sin abrir los archivos (falsos positivos).
- Mezclar decisiones de producto con bugs.
- Proponer refactors grandes (dividir archivos de 2k líneas) sin red de tests previa.
- Incluir valores de secretos en el documento.
