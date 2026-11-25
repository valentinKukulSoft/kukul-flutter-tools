# kukul-flutter-tools

Marketplace de plugins de Claude Code del equipo Kukul para proyectos Flutter.

## Plugins

| Plugin | Qué incluye |
|---|---|
| `kukul-flutter` | Skills `flutter-audit` y `flutter-upgrade` |

## Instalación

Desde la terminal:

```bash
claude plugin marketplace add valentinKukulSoft/kukul-flutter-tools
claude plugin install kukul-flutter@kukul-flutter-tools
```

O desde una sesión de Claude Code:

```
/plugin marketplace add valentinKukulSoft/kukul-flutter-tools
/plugin install kukul-flutter@kukul-flutter-tools
```

Para recibir cambios nuevos (reinicia Claude Code para aplicarlos):

```bash
claude plugin marketplace update kukul-flutter-tools
claude plugin update kukul-flutter@kukul-flutter-tools
```

En sesión: `/plugin marketplace update kukul-flutter-tools`.

## Skills

- **`flutter-audit`**: auditoría técnica completa de un proyecto Flutter (seguridad, performance, refactor, arquitectura, dependencias, tests, CI). Cada hallazgo lleva evidencia y se genera `docs/AUDITORIA_<fecha>.md` priorizado de P0 a P3, con plan de trabajo. Se activa al pedir auditar la salud técnica de una app Flutter.
- **`flutter-upgrade [versión]`**: actualiza Flutter a la última estable (idealmente con FVM) y luego sus paquetes, por fases, investigando los breaking changes que afectan al proyecto. Mantiene `docs/flutter-upgrade/README.md` como tablero de seguimiento.

Ambas usan [Context7](https://context7.com) (MCP) para leer documentación actualizada de Flutter, Dart y paquetes. Si no lo tienes, la skill te propondrá instalarlo:

```bash
claude mcp add --transport http context7 https://mcp.context7.com/mcp
```

Claude las invoca solo cuando la petición coincide con su descripción, o puedes llamarlas por nombre (por ejemplo `/kukul-flutter:flutter-upgrade 3.47`).

## Estructura

```
.claude-plugin/marketplace.json
plugins/kukul-flutter/
├── .claude-plugin/plugin.json
└── skills/
    ├── flutter-audit/SKILL.md
    └── flutter-upgrade/SKILL.md
```

Este repositorio es público: no subas secretos, rutas locales ni datos internos de proyectos.
