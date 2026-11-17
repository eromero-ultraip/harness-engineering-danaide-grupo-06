# MDA-1 — Crear y consultar tickets

Historia: [MDA-1](../../README.md#historias-cargadas-en-jira).

Los acuerdos de proyecto están en [constitution.md](../../constitution.md). Este spec no elige stack.

## Contexto

Un agente de soporte registra un ticket y después lo busca. Al crearlo indica título, cliente, descripción, prioridad y categoría. El ticket queda abierto y en nivel 1.

La prioridad es una lista cerrada: `baja`, `media`, `alta`, `urgente`. La categoría es una lista cerrada: `acceso`, `hardware`, `software`, `conectividad`. El cliente es texto libre sintético. Título y descripción son obligatorios.

Crear, consultar por identificador y listar exigen un agente autenticado, con el token Bearer de la constitución. El rol `operaciones` no realiza estas operaciones.

El listado acepta filtros por prioridad, categoría, cliente y título. Prioridad y categoría coinciden de forma exacta. Cliente y título coinciden si el texto aparece en el campo, sin distinguir mayúsculas de minúsculas. Con varios filtros, el ticket tiene que cumplir todos.

## Historias

- [MDA-1](../../README.md#historias-cargadas-en-jira) — Crear y consultar tickets.

## Criterios

R1. Cuando un agente autenticado crea un ticket con título, cliente y descripción con texto, y con una prioridad y una categoría de las listas cerradas, el sistema debe guardar el ticket, asignarle un identificador único, dejarlo abierto y en nivel 1, y devolverlo con esos datos.

R2. Si al crear falta título, cliente, descripción, prioridad o categoría, o el título, el cliente o la descripción están vacíos o contienen solo espacios, entonces el sistema debe rechazar la creación.

R3. Si la prioridad no es `baja`, `media`, `alta` ni `urgente`, entonces el sistema debe rechazar la creación.

R4. Si la categoría no es `acceso`, `hardware`, `software` ni `conectividad`, entonces el sistema debe rechazar la creación.

R5. Cuando un agente autenticado consulta un identificador que existe, el sistema debe devolver ese ticket.

R6. Si se consulta un identificador que no existe, entonces el sistema debe rechazar la consulta.

R7. Cuando un agente autenticado lista sin filtros, el sistema debe devolver todos los tickets.

R8. Cuando un agente autenticado lista con uno o más filtros, el sistema debe devolver solo los tickets que cumplen todos los filtros enviados.

R9. Cuando el filtro es prioridad o categoría, el sistema debe conservar el ticket solo si el campo es igual al valor indicado.

R10. Cuando el filtro es cliente o título, el sistema debe conservar el ticket si el texto indicado aparece en ese campo, sin distinguir mayúsculas de minúsculas.

R11. Si el filtro de prioridad o de categoría no pertenece a su lista cerrada, entonces el sistema debe rechazar el listado.

R12. Cuando se presentan usuario y contraseña válidos de un usuario sintético, el sistema debe entregar un token Bearer.

R13. Si el usuario o la contraseña no corresponden a un usuario sintético cargado, entonces el sistema debe rechazar la entrega del token.

R14. Si la llamada no incluye un token Bearer válido, entonces el sistema debe rechazar crear, consultar por identificador y listar.

R15. Si el token Bearer válido corresponde al rol `operaciones`, entonces el sistema debe rechazar crear, consultar por identificador y listar.

## Fuera de alcance

- Asignar y responder tickets ([MDA-2](../../README.md#historias-cargadas-en-jira)).
- Cerrar tickets ([MDA-3](../../README.md#historias-cargadas-en-jira)).
- Plazos de SLA por prioridad ([MDA-4](../../README.md#historias-cargadas-en-jira)).
- Cambiar el nivel del ticket ([MDA-5](../../README.md#historias-cargadas-en-jira)). El alta lo deja en nivel 1.
- Marcar un SLA vencido y escalar por ese motivo ([MDA-6](../../README.md#historias-cargadas-en-jira)).
- Reportes de operaciones ([MDA-7](../../README.md#historias-cargadas-en-jira)).
- Alta, baja y cambio de rol de usuarios. Los usuarios sintéticos se cargan al arrancar.
- Paginación del listado. La respuesta incluye cada ticket que cumple.
- Decisiones de stack. Están en la constitución.

## Preguntas abiertas

- Formato del identificador del ticket.
- Duración del token Bearer.
- Usuario y contraseña concretos del agente y de operaciones.
- Longitud máxima de título, cliente y descripción.
- Orden de los tickets en el listado.
- Código con el que se comunica cada rechazo.
