---
name: flutter-upgrade
description: Actualiza un proyecto Flutter a la última versión estable (idealmente con FVM) y luego sus paquetes, por fases, investigando breaking changes que afecten al proyecto, y genera docs/flutter-upgrade/README.md como tablero de seguimiento. Usar cuando el usuario pida actualizar Flutter, subir dependencias, migrar a la última stable, o planear/continuar una actualización de Flutter y paquetes.
argument-hint: [versión-flutter-objetivo]
---

# Actualización de Flutter y dependencias por fases

Actualiza Flutter y luego los paquetes, con **evidencia** de qué rompe en *este* proyecto y un tablero de seguimiento versionado. Argumento opcional: versión objetivo (`$ARGUMENTS`); si no se pasa, la última estable.

## Reglas de oro

1. **Investiga antes de tocar.** Entra en modo plan: lee el proyecto, averigua versiones reales y breaking changes, escribe el plan y pide aprobación. Nada de ediciones antes.
2. **Datos reales, no memoria.** Las versiones cambian: consulta releases oficiales, `pub outdated` y changelogs. No afirmes una versión "más reciente" de memoria.
3. **Una fase = un commit (o uno por paquete en majors de riesgo).** Si el usuario pide "un solo commit", un solo commit. **Commitea solo cuando lo pida.**
4. **Stage explícito, nunca `git add -A`.** Hay cambios del usuario sin commitear que no deben colarse. Revisa `git status` antes de cada commit.
5. **Cada paso se verifica:** `analyze` + `test` + builds nativos. No declares una fase completa sin haberlos corrido.
6. **Las pruebas visuales/manuales las hace el usuario** salvo que pida lo contrario: dile exactamente qué probar a mano al cerrar cada fase.
7. Si el proyecto usa FVM (`.fvmrc`), prefija `fvm flutter` / `fvm dart` en todo.
8. Respeta el idioma y las reglas del proyecto (CLAUDE.md: comentarios mínimos, no formatear carpetas completas, convenciones de commits).
9. Si algo huele a decisión de producto o de riesgo (migrar toolchain, perder datos), **pregunta** con una recomendación; no decidas solo.
10. **Docs con Context7**: si está instalado, úsalo para leer changelogs, guías de migración y APIs; si no, propón instalarlo (ver sección siguiente).

## Documentación con Context7

Para consultar docs de Flutter, Dart y paquetes usa **Context7** (MCP), que sirve documentación actualizada por versión en vez de depender de la memoria del modelo.

1. **Detecta si está disponible**: comprueba si tienes las herramientas `resolve-library-id` y `query-docs` del servidor `context7` (p. ej. `mcp__context7__*` o `mcp__plugin_context7_context7__*`). Si el servidor aparece pero pide autenticación, díselo al usuario (`/mcp`).
2. **Si está disponible, úsalo**: `resolve-library-id` con el nombre del paquete → elige el ID `/org/proyecto` (versionado si existe para la versión del proyecto) → `query-docs` con una pregunta concreta por tema (una API, una migración, una configuración). Cita en el documento la fuente consultada.
3. **Si no está instalado, propón instalarlo** antes de la fase de investigación, sin bloquear:
   ```bash
   claude mcp add --transport http context7 https://mcp.context7.com/mcp
   ```
   (o desde una sesión: `/plugin install context7@claude-plugins-official`). Requiere reiniciar Claude Code. Si el usuario no quiere, sigue con las fuentes oficiales (pub.dev changelog, `docs.flutter.dev`, `dart.dev`, repos de GitHub) y anótalo en el documento.
4. Context7 complementa, no sustituye: **las versiones reales salen de `pub outdated`/releases oficiales** y el impacto se confirma siempre con `grep` en el código del proyecto.

## Fase 0 — Contexto y línea base

1. Rama: trabaja en la actual salvo que el usuario pida una nueva. Anota la rama base.
2. Versiones actuales: `flutter --version`, `pubspec.yaml` (`environment`, dependencias, **pins y comentarios** sobre por qué existen), `ios/Podfile`, `android/settings.gradle*`, `gradle-wrapper.properties`, `gradle.properties`, Xcode (`xcodebuild -version`), JDK.
3. Última estable real:
   ```bash
   curl -s https://storage.googleapis.com/flutter_infra_release/releases/releases_macos.json | python3 -c "
   import json,sys
   d=json.load(sys.stdin); cur=d['current_release']['stable']
   [print(r['version'],r['release_date'][:10],r['dart_sdk_version']) for r in d['releases'] if r['hash']==cur]"
   ```
4. Paquetes: `flutter pub outdated` (columnas Current / Upgradable / Resolvable / Latest). Separa directas, dev y transitivas.
5. **Línea base:** `flutter analyze` y `flutter test`. Si ya hay tests rojos o issues, repórtalos y **resuélvelos antes** (para no confundirlos con regresiones). Causas típicas de tests rojos heredados: plantilla `widget_test.dart` del contador; notifiers que crean datasources reales (red/secure storage sin mock) → inyectar el datasource por constructor opcional y usar un fake (sigue el patrón ya existente en el repo).
6. Limpia el analyze antes de empezar: `dart fix --apply <archivo>` (**un archivo o carpeta por ejecución**, no acepta varios), guards `if (!mounted) return;` antes de usar `context` tras un `await`, imports sin usar, APIs deprecadas.

## Fase 1 — Investigación de breaking changes (modo plan)

Para cada paquete con salto **major** (y para el framework):
- **Context7 primero** (si está disponible): guía de migración y APIs eliminadas/renombradas de la versión destino de cada paquete y de Flutter/Dart. Contrasta con el changelog en pub.dev (`https://pub.dev/packages/<pkg>/changelog`) y `docs.flutter.dev/release/breaking-changes`.
- Anota: mínimos nuevos (Flutter/Dart/iOS/Android/AGP/Gradle/Kotlin), APIs eliminadas/renombradas, migraciones de datos (p. ej. secure storage), cambios de plataforma.
- **Cruza con el código:** `grep` de cada API afectada en `lib/` y `test/` para saber si te toca. Lista archivos y líneas.
- **Agrupa por dependencias compartidas.** Paquetes que comparten una dependencia (ej. `win32` 5→6 en `package_info_plus`, `device_info_plus`, `file_picker`, `share_plus`, `geolocator`) **no pueden subir por separado**; planéalos como un bloque.
- Toolchain nativo: lee las constantes del propio Flutter (`~/fvm/versions/<v>/packages/flutter_tools/lib/src/android/gradle_utils.dart`: `templateDefaultGradleVersion`, `templateAndroidGradlePluginVersion`, `templateKotlinGradlePluginVersion`, `maxKnownAndSupported*`) y las plantillas (`packages/flutter_tools/templates/app/android*.tmpl`). **Alinéate con la plantilla oficial** en vez de inventar versiones.
- Pines existentes en `pubspec.yaml` con comentario: verifica si su causa sigue vigente (p. ej. versión de Xcode) antes de conservarlos.

Escribe el plan con fases, riesgos y verificación; presenta con ExitPlanMode.

## Fase 2 — Flutter

1. Con FVM: `fvm install <v>` y `fvm use <v>` (puede pedir un prompt interactivo y colgarse en un entorno no interactivo: si pasa, ciérralo; `.fvmrc` suele haberse escrito ya). Pregunta si `.fvm/` va o no en `.gitignore`.
2. Sube `environment.sdk` al mínimo que trae la nueva versión de Dart; actualiza docs (CLAUDE.md/README) con las versiones.
3. `pub get`, `analyze`, `test`. Nota: `pub get` puede modificar `analysis_options.yaml` (excluye `android/**`, `ios/**`); es normal, incluirlo en el commit.
4. Builds de humo Android e iOS (ver Verificación).

## Fase 3 — Minor/patch de dependencias directas

Sube constraints a las últimas versiones compatibles y `pub get`. Si el solver falla, **lee el mensaje**: casi siempre indica la dependencia compartida que obliga a mover otro paquete. Mueve el paquete conflictivo a la fase de majors. Si un minor añade valores a un `enum` (ej. `DioExceptionType.transformTimeout`) los `switch` exhaustivos dejan de compilar: ajústalos.

## Fase 4 — Majors

Uno por uno (o por bloque si comparten dependencia), de menor a mayor riesgo. Tras cada uno: `pub get && analyze && test` y, si toca código nativo, builds. Registra en el tablero.
- Adapta el código a las APIs eliminadas (grep primero).
- Un tercero puede bloquear un major (ej. `chewie` exige `cupertino_icons ^1`): **no fuerces con `dependency_overrides`**; déjalo, anótalo y sigue.
- Una transitiva rota tras desbloquear versiones (ej. un plugin con import erróneo en una versión puntual): sube solo esa transitiva con `flutter pub add <pkg>:<v>` y luego **quita la dependencia directa** del `pubspec.yaml` para que quede solo en el lockfile. Revisa por qué la versión siguiente está bloqueada.
- Paquetes de almacenamiento seguro: revisa migración de datos. Si la app **aún no está en producción**, no hay riesgo de perder datos y puedes saltar directo; si ya está publicada, sube primero al paso intermedio y publica.

## Fase 5 — Toolchain Android/iOS

Android:
- Alinea AGP/Gradle/Kotlin con las plantillas de la nueva versión de Flutter. Gradle ya descargado se ve en `~/.gradle/wrapper/dists`.
- Kotlin ≥ 2.3: `kotlinOptions { jvmTarget = "17" }` con String pasa a ser error → migrar a `kotlin { compilerOptions { jvmTarget = org.jetbrains.kotlin.gradle.dsl.JvmTarget.JVM_17 } }`.
- Con AGP 9 sigue la plantilla: `android.newDsl=false` y `android.builtInKotlin=false`, plugin de Kotlin declarado en `settings.gradle.kts`, **sin** `kotlin-android` en `app/build.gradle.kts`. `builtInKotlin=true` suele fallar si algún plugin (p. ej. `firebase_core`) aún aplica KGP o si el Kotlin integrado de AGP es menor que el mínimo de Flutter: déjalo documentado como pendiente.
- Tras subir plugins nativos, si el build falla con clases "no encontradas" en `GeneratedPluginRegistrant`, haz `flutter clean` y reconstruye (estado de Gradle viciado).

iOS:
- Comprueba si SPM ya está activo (`FlutterGeneratedPluginSwiftPackage` en el proyecto; `flutter config --list`). Un plugin sin soporte SPM provoca un aviso en cada build.
- Plugins con paquete específico por SDK (ej. `google_maps_flutter_ios_sdk9`/`sdk10`) permiten migrarlos a SPM: añade el paquete al `pubspec`. **Antes** revisa qué hace el `Podfile` (hooks `pre_install` que generan `Secrets.xcconfig`, claves de API): verifica empíricamente que el hook sigue corriendo (borra el archivo generado, construye y comprueba que se regenera y que el valor llega al `Info.plist` del `.app`, sin imprimir secretos).
- Un build de Xcode puede fallar una vez con un crash de resolución de SPM: reintenta tras `flutter clean` antes de investigar.
- El build de iOS reescribe `ios/Runner.xcodeproj/project.pbxproj` (churn) y `Package.resolved`. Pregunta al usuario cómo tratarlo y respétalo (¿revertir o dejar pasar?). Si la preferencia es dejarlo, no lo reviertas.

## Fase 6 — Transitivas y limpieza

- `flutter pub upgrade` normal (**no** `--major-versions` en bloque) para subir transitivas dentro de rangos; luego verificación completa.
- `flutter pub outdated` final: lo que quede debe estar explicado (bloqueado por SDK o por un tercero).
- Documenta pendientes y de quién dependen (tabla "Pendientes y dependencias de terceros").

## Verificación (cada fase)

```bash
fvm flutter analyze            # 0 issues
fvm flutter test               # todo en verde
fvm flutter build apk --debug   --dart-define-from-file=.env   # si hay .env
fvm flutter build apk --release --dart-define-from-file=.env   # tras tocar Gradle/AGP/R8
fvm flutter build ios --debug --no-codesign --dart-define-from-file=.env
```

Usa `--dart-define-from-file` solo si el proyecto lo requiere (CLAUDE.md). Los builds pueden pasar de 5 min: ejecútalos con timeout alto o en background. Al cerrar cada fase entrega al usuario una **lista concreta de pruebas manuales** según los paquetes tocados (login/biometría ↔ secure_storage/local_auth; cámara/archivos ↔ image_picker/file_picker; compartir ↔ share_plus; mapas/ubicación ↔ google_maps/geolocator; push ↔ firebase_messaging; pagos/WebView ↔ webview_flutter/url_launcher; router ↔ go_router).

## Tablero de seguimiento

Crea y mantén `docs/flutter-upgrade/README.md` (en el idioma del proyecto) con:
- Cabecera: versión inicial → objetivo, rama, FVM, entorno (Xcode, JDK, Gradle, AGP, Kotlin, iOS target).
- Tabla de **estado por fase** (✅ / 🟡 / ⬜).
- Por fase: checklist de pasos, tabla de paquetes (de → a, impacto en el código, hecho) y **Notas** con hallazgos, decisiones e incidencias (la causa y la solución, para que el siguiente que actualice no repita el problema).
- Sección **Pendientes y dependencias de terceros**: qué, de quién depende, impacto, acción.
- Checklist de verificación al cerrar cada fase y tabla de registro de incidencias.
Actualízalo en cada paso y mantenlo dentro del commit de la fase que documenta.

## Cierre

Resume al usuario: versiones antes/después, qué rompió y cómo se resolvió, qué quedó pendiente y de quién depende, y qué debe probar a mano. No hagas commits ni push sin que lo pida.
