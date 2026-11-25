# Plan — MDA-1 Crear y consultar tickets

Spec: [spec.md](spec.md). Acuerdos: [constitution.md](../../constitution.md). Este plan no modifica el spec.

## Enfoque

La fase es una API HTTP síncrona en FastAPI que persiste en PostgreSQL 18.7. Un agente con token Bearer crea un ticket, lo consulta por identificador y lista, con los filtros del spec. El alta guarda título, cliente, descripción, prioridad y categoría, y el sistema deja el ticket abierto y en nivel 1. Al arrancar se cargan dos usuarios sintéticos, agente y operaciones; operaciones obtiene token y no opera tickets. La suite de pytest corre contra el PostgreSQL de Compose y es la condición de merge. Las preguntas abiertas y los huecos se cierran acá con una decisión por defecto y un test por caso.

Los criterios R1–R15 se implementan como están escritos. La tabla de abajo solo cierra huecos y elecciones técnicas.

## Decisiones por defecto

| Id | Decisión | Por qué | Alternativa descartada | Criterios |
| --- | --- | --- | --- | --- |
| D1 | SQLAlchemy 2 síncrono, driver psycopg 3, `create_all` al arrancar. La validación vive en la API. | En esta fase el esquema no cambia y el 422 tiene que salir con el `detail` de D8. | Alembic desde MDA-1. Checks de PostgreSQL para las listas. | R1, R5 |
| D2 | Dos tablas, solo con los campos de esta fase. Sin fecha, asignación, respuesta, cierre ni SLA. | El backlog se revisa entre fases. El orden del listado sale del `id`. | Columnas nulables de MDA-2 en adelante. | R1 |
| D3 | pytest 9.2.0 y TestClient contra Postgres 18.7 de Compose. Cada test borra los tickets y vuelve a cargar los usuarios. La base se crea con locale `en_US.UTF-8`. | La constitución ata el merge a pytest y a ese Postgres. El locale deja estable la prueba de acentos. | SQLite en la suite. Locale C. | R1–R15 |
| D4 | `DATABASE_URL` con default `postgresql+psycopg://mesa:mesa@localhost:5432/mesa`. `TOKEN_SECRETO` default `mesa-ayuda-secreto-sintetico`. `TOKEN_HORAS` default 8. En Compose, la API apunta el host `db`. | La suite arranca sola, con datos sintéticos, y el secreto se puede cambiar por entorno. | Secreto y URL solo escritos en el código, sin entorno. | R12, R14 |
| D5 | JWT HS256 con `sub`, `rol`, `iat` y `exp = iat + 8 horas`. Se acepta solo HS256. | El rol viaja en el token y la duración queda medida en el test. | Token opaco en tabla, sin vencimiento. | R12, R14, R15 |
| D6 | Contraseña con argon2 (pwdlib). La columna guarda el hash. | La contraseña sintética no queda en claro en la base. | Texto plano. | R12, R13 |
| D7 | `POST /sesion`, `POST /tickets`, `GET /tickets/{id}`, `GET /tickets`. Query: `prioridad`, `categoria`, `cliente`, `titulo`. Claves JSON sin acento. | Coinciden con los nombres del spec y con un solo recurso. | Formulario OAuth2 y claves en inglés. | R1, R5, R7, R8, R12 |
| D8 | Códigos HTTP y `detail` de la tabla de abajo. `detail` es un string, un solo código por respuesta. | El spec dice rechazar y deja abierto el código. Un solo formato se puede afirmar en el test. | Un 400 para todo rechazo. El `detail` lista de Pydantic. | R2, R3, R4, R6, R11, R13, R14, R15 |
| D9 | `id` entero, secuencia de PostgreSQL desde 1. | Es único y da el orden de D10. | UUID. | R1, R5, R6, R7 |
| D10 | El listado ordena por `id` ascendente. | El orden queda fijo para quien lee y para el test. | El orden sin `ORDER BY`. | R7 |
| D11 | Título, cliente y descripción son `TEXT`. No hay longitud máxima de aplicación. | R2 cubre vacío y espacios. Un tope sería un criterio nuevo. | `varchar` corto y 422. | R1, R2 |
| D12 | Al validar y al guardar se recortan los extremos de título, cliente y descripción, incluidos los saltos de línea. Los espacios internos se conservan. Prioridad y categoría se comparan tal cual. | R2 define el blanco en esos tres textos. El valor devuelto queda alineado con eso. | Guardar los espacios de los extremos. Recortar también la prioridad. | R1, R2, R3 |
| D13 | Usuarios cargados al arrancar, de forma repetible: `agente` / `agente-sintetico` (rol `agente`) y `operaciones` / `operaciones-sintetico` (rol `operaciones`). Usuario y contraseña se comparan tal cual. | R15 necesita un token de operaciones y R12 uno válido, con datos sintéticos publicados. | Un solo usuario. Recortar la contraseña. | R12, R13, R15 |
| D14 | En cliente y título, los extremos del filtro se recortan. Si queda vacío, 422. Si no, la búsqueda es literal: `ILIKE` escapando `\`, `%` y `_`. Los acentos distinguen. Varios filtros se aplican todos. | R10 pide que el texto aparezca, sin mirar mayúsculas. El vacío y el acento no están dichos. | Tratar el filtro vacío como filtro ausente. `LIKE` sin escape. Unaccent. | R8, R10 |
| D15 | El alta acepta exactamente título, cliente, descripción, prioridad y categoría. Otro campo se rechaza. Estado y nivel los pone el sistema. | MDA-3 y MDA-5 están fuera de alcance. Un campo de más no puede cerrar ni escalar. | Ignorar campos de más. | R1 |
| D16 | El `id` de la ruta es un entero ≥ 1. Otro valor es 422. Un entero ≥ 1 que no está en la base es 404. La autenticación se resuelve antes de buscar el id. | R6 queda para el que no existe. Sin token, un id ausente responde 401. Con otro rol, responde 403. | 404 para `abc` y para el id ausente. | R6, R14, R15 |
| D17 | El título puede repetirse. El único valor único del ticket es el `id`. | El spec pide identificador único, y en una mesa dos tickets pueden llamarse igual. | Restricción única de título. | R1 |
| D18 | El ticket devuelto tiene exactamente `id`, `titulo`, `cliente`, `descripcion`, `prioridad`, `categoria`, `estado`, `nivel`. `estado` es `abierto`. `nivel` es el entero `1`. El listado es un arreglo JSON de esos objetos, sin envoltura. | R1 manda devolver el ticket abierto y en nivel 1, y la paginación está fuera. | `abierto` booleano y nivel en texto. Envoltura `{"items": ...}`. | R1, R5, R7 |
| D19 | La sesión válida responde exactamente `token` (el JWT) y `tipo` con valor `bearer`. El header de las operaciones es `Authorization: Bearer <token>`. | R12 pide entregar un token Bearer, en el mismo JSON en español del resto. | Cuerpo `access_token` / `token_type`. | R12, R14 |

### Esquema (D1, D2)

`usuarios`: `id` entero clave, `usuario` texto único, `hash_contrasena` texto, `rol` texto.

`tickets`: `id` entero clave autoincremental, `titulo` texto, `cliente` texto, `descripcion` texto, `prioridad` texto, `categoria` texto, `estado` texto, `nivel` entero.

### Códigos (D8)

Si el cuerpo trae más de un error, se responde uno solo: el primero entre `titulo`, `cliente`, `descripcion`, `prioridad`, `categoria`. En la sesión, `usuario` antes que `contrasena`. En el listado, `prioridad` antes que `categoria`, y cualquiera de esas dos antes que un filtro de texto.

| Situación | HTTP | `detail` |
| --- | --- | --- |
| Alta correcta | 201 | el ticket |
| Consulta correcta | 200 | el ticket |
| Listado correcto, también vacío o sin coincidencias | 200 | arreglo |
| Título ausente, vacío o solo espacios | 422 | `titulo_obligatorio` |
| Título con un tipo distinto de texto | 422 | `titulo_invalido` |
| Cliente ausente, vacío o solo espacios | 422 | `cliente_obligatorio` |
| Cliente con un tipo distinto de texto | 422 | `cliente_invalido` |
| Descripción ausente, vacía o solo espacios | 422 | `descripcion_obligatoria` |
| Descripción con un tipo distinto de texto | 422 | `descripcion_invalida` |
| Prioridad ausente | 422 | `prioridad_obligatoria` |
| Prioridad distinta de `baja`, `media`, `alta`, `urgente` | 422 | `prioridad_invalida` |
| Categoría ausente | 422 | `categoria_obligatoria` |
| Categoría distinta de `acceso`, `hardware`, `software`, `conectividad` | 422 | `categoria_invalida` |
| Filtro de prioridad o de categoría fuera de su lista | 422 | `prioridad_invalida` o `categoria_invalida` |
| Filtro de cliente o de título presente y vacío o en blanco | 422 | `filtro_vacio` |
| Campo de más en el alta | 422 | `campo_no_permitido` |
| Id de ruta que no es un entero ≥ 1 (`abc`, `1.5`, `0`, negativo) | 422 | `identificador_invalido` |
| Entero ≥ 1 que no existe | 404 | `identificador_inexistente` |
| Falta `usuario` en la sesión | 422 | `usuario_obligatorio` |
| Falta `contrasena` en la sesión | 422 | `contrasena_obligatoria` |
| Usuario o contraseña que no coinciden con la carga | 401 | `credenciales_invalidas` |
| Sin header, esquema distinto, token vacío, firma inválida, vencido o sin `exp` | 401 | `token_invalido` |
| Token vigente cuyo rol no es `agente` | 403 | `rol_no_autorizado` |

El cuerpo de error es exactamente `{"detail": "<codigo>"}`.

### Contrato JSON (D7, D18, D19)

Cuerpo válido de alta:

```json
{
  "titulo": "Corte de red",
  "cliente": "Cliente Sintetico",
  "descripcion": "Sin enlace",
  "prioridad": "alta",
  "categoria": "software"
}
```

Respuesta de ese alta: los cinco campos, más `id` entero, `estado` `abierto` y `nivel` `1`.

Sesión:

```json
{ "usuario": "agente", "contrasena": "agente-sintetico" }
```

Respuesta: `{"token": "<jwt>", "tipo": "bearer"}`.

Las listas cerradas son las del spec, en minúsculas y tal cual.

## Lo que el spec no nombra

Cada fila es comportamiento que el spec no fija. El test lo traba. La columna Aviso dice si conviene pasarlo a `spec.md`. Hasta entonces manda este plan.

| Caso | Qué fija el plan | Test | Aviso |
| --- | --- | --- | --- |
| Formato del identificador | Entero de secuencia, creciente | `test_el_identificador_es_entero_y_crece` | Pregunta abierta. Conviene cerrarla en el spec. |
| Duración del token | 8 horas (`exp - iat`) | `test_el_token_dura_ocho_horas` | Pregunta abierta. Conviene cerrarla en el spec. |
| Token vencido o sin `exp` | 401 `token_invalido` | `test_rechaza_un_token_vencido` | Con la duración escrita en el spec, esto queda cubierto por R14. |
| Usuarios y contraseñas | Los de D13, comparados tal cual | `test_credenciales_validas_entregan_token` | Pregunta abierta. Conviene cerrarla en el spec. |
| Código de cada rechazo | La tabla de D8 | Los tests de rechazo de [tasks.md](tasks.md) | Pregunta abierta. Conviene pasar la tabla de HTTP y `detail` al spec. |
| Varios errores de cuerpo a la vez | Un solo `detail`, en el orden de D8 | `test_informa_un_solo_error_de_alta` | Conviene una frase en el spec, junto con los códigos. |
| Orden del listado | `id` ascendente | `test_el_listado_sale_por_id_ascendente` | Pregunta abierta. Conviene cerrarla en el spec. |
| Longitud máxima | Un texto de 4000 caracteres se acepta y se devuelve igual | `test_acepta_un_texto_largo` | Un tope sería un criterio nuevo: se escribe en el spec antes de cambiar este test. |
| Espacios de los extremos al guardar | Se recortan en título, cliente y descripción; los internos quedan | `test_guarda_el_texto_sin_espacios_de_los_extremos` | Cambia el dato que devuelve R1. Conviene una frase en el spec. |
| Espacios en prioridad y categoría | No se recortan: `alta ` y `software ` se rechazan | `test_rechaza_prioridad_fuera_de_lista`, `test_rechaza_categoria_fuera_de_lista` | Lectura de R3 y R4. Si se quieren aceptar recortadas, va una frase al spec. |
| Filtro de texto vacío o en blanco | 422 `filtro_vacio` | `test_un_filtro_de_texto_vacio_se_rechaza` | Conviene una frase en el spec. |
| Acentos en el filtro de texto | Distinguen; las mayúsculas no | `test_el_filtro_de_texto_distingue_acentos` | R10 habla de mayúsculas. Conviene una frase en el spec. |
| `%`, `_` y `\` en el filtro | Se buscan como texto literal | `test_el_filtro_de_texto_es_literal` | Lo cubre “el texto aparece”. El test traba la implementación. Se queda en el plan. |
| Espacios internos en el filtro | No se colapsan | `test_el_filtro_no_colapsa_espacios_internos` | Conviene una frase en el spec si se quiere como regla. |
| Campos de más en el alta | 422 `campo_no_permitido` | `test_rechaza_un_campo_extra_en_el_alta` | Conviene una frase en el spec. |
| Identificador mal formado | 422, distinto del 404 de R6 | `test_rechaza_un_identificador_mal_formado` | R6 es el que no existe. Conviene una frase en el spec. |
| Mismo título en dos tickets | Los dos se crean, con distinto `id` | `test_permite_el_mismo_titulo` | Conviene una frase en el spec. |
| Tipo distinto de texto en título, cliente o descripción | 422 con `*_invalido` | `test_rechaza_alta_sin_campo_o_en_blanco` | R2 habla de falta y de blanco. Conviene una frase en el spec. |
| Forma JSON del ticket y listado como arreglo | D18 | `test_un_agente_crea_un_ticket_abierto_en_nivel_1`, `test_lista_todos_los_tickets` | Conviene pasar el contrato al spec. |
| Forma JSON de la sesión | D19 | `test_credenciales_validas_entregan_token` | Conviene pasar el contrato al spec. |
| Sesión sin usuario o sin contraseña | 422, distinto del 401 de R13 | `test_sesion_sin_usuario_o_contrasena_se_rechaza` | Conviene una frase en el spec. |
| Rol firme distinto de `agente` | 403, también si el rol no es `operaciones` | `test_un_rol_que_no_es_agente_se_rechaza` | R15 nombra operaciones. Conviene una frase para el resto de los roles. |
| Hash de la contraseña | argon2, nunca el texto de D13 | `test_la_contrasena_no_se_guarda_en_claro` | Técnico. Se queda en el plan. |

## Validación

La fase está lista para el merge cuando `pytest` pasa, con datos sintéticos, sobre PostgreSQL 18.7. La rama de la fase es `MDA-1-tickets`.

La API de esta fase expone solo `POST /sesion`, `POST /tickets`, `GET /tickets/{id}` y `GET /tickets`. La suite no cubre asignar, responder, cerrar, SLA, cambio de nivel, reportes ni alta de usuarios.

Lectura que los tests usan del spec, para no correrla en el código:

- R2 rechaza falta o blanco en título, cliente y descripción, y la falta de prioridad o categoría. Una prioridad presente y distinta de la lista es R3. Una categoría así es R4.
- R8 exige que el ticket cumpla todos los filtros enviados.
- R9 es igualdad exacta. R10 es subcadena sin mirar mayúsculas.
- R11 rechaza el listado entero si prioridad o categoría no están en su lista, aunque el otro filtro sea válido.
- R12 entrega el token con usuario y contraseña, sin token previo. R14 nombra solo crear, consultar y listar.
- Sin token, un id que no existe responde 401. Con token de operaciones, ese mismo id responde 403.
- R15 es un token válido de operaciones. El rechazo es 403, distinto del 401 de R14.
- La contraseña de R13 se compara completa: un espacio de más no coincide.

Comandos, desde la raíz del repo:

```powershell
docker compose up -d db
pytest
```

```bash
docker compose up -d db
pytest
```

| Criterio | Test |
| --- | --- |
| R1 | `test_un_agente_crea_un_ticket_abierto_en_nivel_1`, `test_acepta_cada_prioridad_y_cada_categoria` |
| R2 | `test_rechaza_alta_sin_campo_o_en_blanco` |
| R3 | `test_rechaza_prioridad_fuera_de_lista` |
| R4 | `test_rechaza_categoria_fuera_de_lista` |
| R5 | `test_consulta_un_ticket_existente` |
| R6 | `test_rechaza_un_identificador_inexistente` |
| R7 | `test_lista_todos_los_tickets` |
| R8 | `test_los_filtros_se_cumplen_todos`, `test_sin_coincidencias_devuelve_lista_vacia` |
| R9 | `test_prioridad_y_categoria_coinciden_exacto` |
| R10 | `test_cliente_y_titulo_coinciden_sin_mayusculas`, `test_el_filtro_de_texto_es_literal` |
| R11 | `test_rechaza_filtro_fuera_de_lista` |
| R12 | `test_credenciales_validas_entregan_token` |
| R13 | `test_credenciales_invalidas_no_entregan_token` |
| R14 | `test_sin_token_valido_rechaza_las_tres_operaciones`, `test_rechaza_un_token_vencido` |
| R15 | `test_operaciones_no_crea_ni_consulta_ni_lista` |

Los huecos de la sección anterior tienen el test nombrado ahí. Esos tests están en [tasks.md](tasks.md).

Si el equipo escribe en el spec alguno de los puntos marcados para subirlo, se actualiza primero el spec y después la fila de este plan y su test.
