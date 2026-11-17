# Constitución

Acuerdos del proyecto Mesa de Ayuda. Cambian poco y con acuerdo del equipo.

## Misión y audiencia

Mesa de Ayuda es una API para gestionar tickets de soporte con SLA y escalamiento. Este documento es para el equipo del grupo.

El punto de partida está en el [README](README.md): Lucía fija el stack, Martín las funciones sobre tickets y Sofía los reportes de operaciones.

## Stack

Versiones estables y compatibles entre sí, fijadas el 17 de noviembre de 2026.

| Pieza | Versión |
| --- | --- |
| Python | 3.15.2 |
| FastAPI | 0.163.1 |
| PostgreSQL | 18.7 |
| pytest | 9.2.0 |
| Docker Compose | v2, el que trae Docker |

FastAPI 0.163.1 requiere Python 3.11 o superior y declara soporte para 3.15. pytest 9.2.0 requiere Python 3.10 o superior. PostgreSQL 19 estaba en candidata a release; la estable es la 18.7.

## Calidad

Cada fase se considera lista para el merge cuando la suite de pytest pasa.

## Seguridad y datos

Todos los datos del proyecto son sintéticos. No se usan datos reales de personas ni de sistemas.

La API exige autenticación. Hay usuarios sintéticos cargados al arrancar, con usuario y contraseña. Una credencial válida entrega un token Bearer. Toda operación exige ese token.

Hay dos roles:

- `agente`, para el trabajo sobre tickets.
- `operaciones`, para los reportes.

En la fase de crear y consultar tickets, esas operaciones exigen un agente autenticado.

## Forma de trabajo

- Una rama por fase, con la clave de la historia en el nombre (`MDA-1-tickets`).
- El spec de la fase se escribe antes del código, en `specs/NNN-<tema>/spec.md`.
- El orden de las fases está en [specs/backlog.md](specs/backlog.md). Se revisa entre fase y fase.
- El PR va al `main` del fork del grupo. Otro integrante lo revisa. El merge cierra la fase.
- Esta constitución cambia poco y con acuerdo del equipo.
