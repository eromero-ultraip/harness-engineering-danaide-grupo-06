# Tareas — MDA-1 Crear y consultar tickets

Plan: [plan.md](plan.md). Spec: [spec.md](spec.md). Cada tarea se hace en este orden: escribir el test, correrlo y verlo fallar, y recién ahí el código que lo pone verde. Si un test nace verde porque una tarea anterior ya cubría el caso, se conserva: traba la decisión.

Crítica: sin ella no sigue el resto, o queda abierta una operación sin el token de la constitución. Camino: T1 → T2 → T4 → T12. T5 también es crítica por esa regla.

T3 puede hacerse en paralelo con T4. T7, T8, T9, T16, T19 y T20 esperan también a T3. T11 espera a T10. T13, T14 y T15 esperan a T12.

| Tarea | Crítica | Criterios | Depende de |
| --- | --- | --- | --- |
| T1 Base y Postgres | sí | — | — |
| T2 Sesión válida | sí | R12 | T1 |
| T3 Sesión rechazada | | R13 | T2 |
| T4 Alta | sí | R1 | T2 |
| T5 Token en las tres operaciones | sí | R14 | T4 |
| T6 Rol que no es agente | | R15 | T4 |
| T7 Campos del alta | | R2 | T3, T4 |
| T8 Prioridad | | R3 | T3, T4 |
| T9 Categoría | | R4 | T3, T4 |
| T10 Consulta | | R5 | T4 |
| T11 Identificador ausente o mal formado | | R6 | T3, T10 |
| T12 Listado | sí | R7 | T4 |
| T13 Filtros juntos | | R8 | T12 |
| T14 Filtro exacto | | R9 | T12 |
| T15 Filtro por texto | | R10 | T12 |
| T16 Filtro fuera de lista | | R11 | T3, T12 |
| T17 Espacios al guardar | | R1, R2 | T4 |
| T18 Texto largo | | R1 | T4 |
| T19 Filtro de texto vacío | | — | T3, T12 |
| T20 Campo de más | | R1 | T3, T4 |
| T21 Mismo título | | R1 | T4 |

## T1 — Base y Postgres (crítica)

- Criterios: —
- Depende de: —
- Decisiones: D1, D3, D4

**Test.** En `tests/test_base.py`, `test_postgres_responde`. Llama a `app.db.comprobar()` y espera `1`.

**Ver fallar.** `pytest tests/test_base.py::test_postgres_responde -q`. Falla con `ModuleNotFoundError`. Después del código, si Compose no está levantado, falla la conexión: `docker compose up -d db` y correr de nuevo.

**Código.** `pyproject.toml` con Python `>=3.15`, `fastapi==0.163.1`, `pytest==9.2.0`, uvicorn, SQLAlchemy 2, `psycopg`, `pwdlib[argon2]`, `pyjwt` y `httpx`. `pythonpath = ["."]` para pytest. `compose.yaml` con el servicio `db`, imagen `postgres:18.7`, usuario `mesa`, contraseña `mesa`, base `mesa`, puerto `5432`, locale `en_US.UTF-8` y healthcheck `pg_isready`. `app/config.py` con los defaults de D4. `app/db.py` con el engine y `comprobar()`. `.env.example` con esos tres valores. `.gitignore` con `.env`, `.venv`, `__pycache__` y `.pytest_cache`.

## T2 — Sesión válida (crítica)

- Criterios: R12
- Depende de: T1
- Decisiones: D5, D6, D7, D13, D19

**Test.** En `tests/test_sesion.py`:

- `test_credenciales_validas_entregan_token`: `POST /sesion` con `agente` / `agente-sintetico` y con `operaciones` / `operaciones-sintetico`. Espera 200 y exactamente `token` (string no vacío) y `tipo` `bearer`.
- `test_el_token_dura_ocho_horas`: decodifica el token con el secreto de `app.config`, algoritmo HS256, y espera `sub`, `rol`, y `exp - iat` igual a 8 horas.
- `test_la_contrasena_no_se_guarda_en_claro`: el hash guardado es distinto de `agente-sintetico` y no lo contiene.
- `test_la_carga_se_puede_repetir`: llama a la carga dos veces y después entra. Espera 200.

**Ver fallar.** `pytest tests/test_sesion.py -q`. Falla: no existe `POST /sesion` (404) o no existe el módulo.

**Código.** Modelo `usuarios` (D1). `app/semilla.py` carga los dos usuarios de D13 con hash argon2, y se puede correr de nuevo sin duplicar. `POST /sesion` devuelve el JWT de D5 y el cuerpo de D19 cuando usuario y contraseña coinciden. Lifespan: crear el esquema y cargar. `tests/conftest.py` con `TestClient` y la recarga de usuarios antes de cada test. La dependencia se declara como `pyjwt` (el import es `jwt`).

## T3 — Sesión rechazada

- Criterios: R13
- Depende de: T2
- Decisiones: D8, D13

**Test.** En `tests/test_sesion.py`:

- `test_credenciales_invalidas_no_entregan_token`: contraseña distinta, usuario desconocido, `Agente`, contraseña vacía y contraseña con un espacio de más. Espera 401 y `{"detail": "credenciales_invalidas"}`.
- `test_sesion_sin_usuario_o_contrasena_se_rechaza`: falta `usuario`, falta `contrasena`, faltan los dos. Espera 422, `usuario_obligatorio` o `contrasena_obligatoria`. Si faltan los dos, el detalle es `usuario_obligatorio`.

**Ver fallar.** `pytest tests/test_sesion.py -q`. El test de R13 falla porque T2 todavía no responde 401 con ese `detail`.

**Código.** `app/errores.py`: cuerpo `{"detail": "<codigo>"}` y el manejador que convierte la validación de Pydantic en un solo código. Comparar usuario y contraseña tal cual. El desajuste responde 401 `credenciales_invalidas`. El campo ausente responde 422.

## T4 — Alta (crítica)

- Criterios: R1
- Depende de: T2
- Decisiones: D2, D7, D9, D15, D18

**Test.** En `tests/test_alta.py`, con el token del agente en `Authorization: Bearer <token>`:

- `test_un_agente_crea_un_ticket_abierto_en_nivel_1`: `POST /tickets` con el cuerpo válido del plan. Espera 201 y el ticket con esos cinco datos, `estado` `abierto`, `nivel` entero `1`, y ninguna clave de más.
- `test_acepta_cada_prioridad_y_cada_categoria`: cada valor `baja`, `media`, `alta`, `urgente` y cada valor `acceso`, `hardware`, `software`, `conectividad` vuelve igual en la respuesta.
- `test_el_identificador_es_entero_y_crece`: dos altas seguidas devuelven enteros distintos y el segundo es mayor. No afirma que el primero sea 1.

**Ver fallar.** `pytest tests/test_alta.py -q`. Falla con 404: la ruta todavía no existe.

**Código.** Modelo `tickets` con columnas `TEXT` y sin unicidad de título. `POST /tickets` exige un token vigente de rol `agente`, persiste los cinco campos, fija `abierto` y nivel `1`, y responde 201 con D18. La fixture borra los tickets antes de cada test.

## T5 — Token en las tres operaciones (crítica)

- Criterios: R14
- Depende de: T4
- Decisiones: D5, D8, D16

**Test.** En `tests/test_authz.py`:

- `test_sin_token_valido_rechaza_las_tres_operaciones`: crear, `GET /tickets/999999` y listar, sin header, con esquema distinto, con Bearer vacío y con firma de otro secreto. Espera 401 y `{"detail": "token_invalido"}`. El `GET` de un id ausente, sin token, responde 401 y no 404. Listar con `prioridad=no-existe` y sin token responde 401 y no 422.
- `test_rechaza_un_token_vencido`: un JWT firmado con el secreto, rol `agente`, `exp` en el pasado, y otro sin `exp`. En las tres operaciones espera 401 `token_invalido`.

**Ver fallar.** `pytest tests/test_authz.py::test_sin_token_valido_rechaza_las_tres_operaciones tests/test_authz.py::test_rechaza_un_token_vencido -q`. Falla mientras un token vencido, uno sin `exp` o un request sin header siga entrando, o mientras el status no sea 401.

**Código.** La dependencia acepta solo HS256, con `exp` vigente y `rol` `agente`. Cualquier otro token responde 401 `token_invalido` antes de validar el cuerpo, los filtros o el id.

## T6 — Rol que no es agente

- Criterios: R15
- Depende de: T4
- Decisiones: D8, D13

**Test.** En `tests/test_authz.py`:

- `test_operaciones_no_crea_ni_consulta_ni_lista`: el token de `operaciones` en crear, en `GET /tickets/999999` y en listar. Espera 403 y `{"detail": "rol_no_autorizado"}`. El `GET` de un id ausente responde 403 y no 404.
- `test_un_rol_que_no_es_agente_se_rechaza`: un JWT vigente, bien firmado, con `rol` `otro` y otro sin `rol`. En `POST /tickets` espera 403 `rol_no_autorizado`.

**Ver fallar.** `pytest tests/test_authz.py::test_operaciones_no_crea_ni_consulta_ni_lista tests/test_authz.py::test_un_rol_que_no_es_agente_se_rechaza -q`. Falla si ese token llega a crear (201) o si el status no es 403.

**Código.** Con firma válida y rol distinto de `agente`, responder 403 `rol_no_autorizado` antes de buscar el ticket.

## T7 — Campos del alta

- Criterios: R2
- Depende de: T3, T4
- Decisiones: D8, D12

**Test.** En `tests/test_alta.py`:

- `test_rechaza_alta_sin_campo_o_en_blanco`: para `titulo`, `cliente` y `descripcion`, el campo ausente, `""` y solo espacios. Para `prioridad` y `categoria`, el campo ausente. Espera 422 y el `*_obligatorio` de D8. Un número en `titulo`, `cliente` o `descripcion` espera el `*_invalido` de D8.
- `test_informa_un_solo_error_de_alta`: faltan `titulo` y `cliente` a la vez. Espera 422 y `titulo_obligatorio`.

El blanco se rechaza acá. Guardar un texto con contenido recortado es T17.

**Ver fallar.** `pytest tests/test_alta.py::test_rechaza_alta_sin_campo_o_en_blanco tests/test_alta.py::test_informa_un_solo_error_de_alta -q`. Falla si el alta responde 201 o si el `detail` no es el código.

**Código.** Validar presencia, tipo texto y blanco según D8 y D12. Un solo error, en el orden de D8. Rechazar el blanco puede hacerse sin recortar todavía el valor que sí tiene contenido.

## T8 — Prioridad

- Criterios: R3
- Depende de: T3, T4
- Decisiones: D8, D12

**Test.** `test_rechaza_prioridad_fuera_de_lista` en `tests/test_alta.py`. Valores `critica`, `Alta`, `alta `, `""`, `null` y un número. Espera 422 y `{"detail": "prioridad_invalida"}`.

**Ver fallar.** `pytest tests/test_alta.py::test_rechaza_prioridad_fuera_de_lista -q`. Falla si alguno de esos valores se guarda.

**Código.** Aceptar solo `baja`, `media`, `alta`, `urgente`, sin recortar. El resto: 422 `prioridad_invalida`.

## T9 — Categoría

- Criterios: R4
- Depende de: T3, T4
- Decisiones: D8

**Test.** `test_rechaza_categoria_fuera_de_lista` en `tests/test_alta.py`. Valores `correo`, `Software`, `software `, `""` y `null`. Espera 422 y `{"detail": "categoria_invalida"}`.

**Ver fallar.** `pytest tests/test_alta.py::test_rechaza_categoria_fuera_de_lista -q`. Falla si alguno se guarda.

**Código.** Aceptar solo `acceso`, `hardware`, `software`, `conectividad`, sin recortar. El resto: 422 `categoria_invalida`.

## T10 — Consulta

- Criterios: R5
- Depende de: T4
- Decisiones: D7, D18

**Test.** `test_consulta_un_ticket_existente` en `tests/test_consulta.py`. Crea con el agente y hace `GET /tickets/{id}` con el mismo token. Espera 200 y el mismo ticket que devolvió el alta.

**Ver fallar.** `pytest tests/test_consulta.py::test_consulta_un_ticket_existente -q`. Falla con 404: la ruta de consulta todavía no existe.

**Código.** `GET /tickets/{id}` para un agente, leyendo de la base el ticket persistido. Misma forma que el alta.

## T11 — Identificador ausente o mal formado

- Criterios: R6
- Depende de: T3, T10
- Decisiones: D8, D16

**Test.** En `tests/test_consulta.py`, con token de agente:

- `test_rechaza_un_identificador_inexistente`: `GET /tickets/999999`. Espera 404 y `{"detail": "identificador_inexistente"}`.
- `test_rechaza_un_identificador_mal_formado`: `abc`, `1.5`, `0` y `-1`. Espera 422 y `{"detail": "identificador_invalido"}`.

**Ver fallar.** `pytest tests/test_consulta.py -q`. El id ausente falla si el status no es 404 o el `detail` no es ese. El mal formado falla si responde 404 o si el `detail` es la lista cruda de FastAPI.

**Código.** Entero ≥ 1 que no está en la base: 404 `identificador_inexistente`. Cualquier otro segmento: 422 `identificador_invalido`, con el mismo sobre `{"detail": ...}` de T3.

## T12 — Listado (crítica)

- Criterios: R7
- Depende de: T4
- Decisiones: D7, D10, D18

**Test.** En `tests/test_listado.py`, con token de agente:

- `test_lista_todos_los_tickets`: sin tickets, espera 200 y `[]`. Después de crear dos, espera 200, un arreglo y exactamente esos dos, con la forma de D18.
- `test_el_listado_sale_por_id_ascendente`: crea dos y espera el de `id` menor primero. No afirma que el primer id sea 1.

**Ver fallar.** `pytest tests/test_listado.py -q`. Falla con 404 hasta que exista `GET /tickets`. El de orden falla si la respuesta no sale por `id` ascendente.

**Código.** `GET /tickets` devuelve todos los tickets en un arreglo, ordenados por `id` ascendente.

## T13 — Filtros juntos

- Criterios: R8
- Depende de: T12
- Decisiones: D14

**Test.** En `tests/test_listado.py`:

- `test_los_filtros_se_cumplen_todos`: tres tickets que se cruzan en prioridad, categoría y cliente. Con `prioridad`, `categoria` y `cliente` a la vez, vuelve solo el que cumple los tres.
- `test_sin_coincidencias_devuelve_lista_vacia`: un filtro válido que no coincide con nadie. Espera 200 y `[]`.

**Ver fallar.** `pytest tests/test_listado.py::test_los_filtros_se_cumplen_todos tests/test_listado.py::test_sin_coincidencias_devuelve_lista_vacia -q`. Falla si vuelven tickets de más o si el listado vacío no es 200 con `[]`.

**Código.** Aplicar todos los filtros presentes. Cero coincidencias es 200 y arreglo vacío.

## T14 — Filtro exacto

- Criterios: R9
- Depende de: T12
- Decisiones: D14

**Test.** `test_prioridad_y_categoria_coinciden_exacto` en `tests/test_listado.py`. Un ticket `alta` / `software` aparece con `prioridad=alta` y con `categoria=software`. No aparece con `prioridad=media` ni con `categoria=hardware`.

**Ver fallar.** `pytest tests/test_listado.py::test_prioridad_y_categoria_coinciden_exacto -q`. Falla si el filtro no separa los valores de la lista.

**Código.** Prioridad y categoría del filtro se comparan por igualdad con el valor guardado.

## T15 — Filtro por texto

- Criterios: R10
- Depende de: T12
- Decisiones: D14

**Test.** En `tests/test_listado.py`:

- `test_cliente_y_titulo_coinciden_sin_mayusculas`: título `Corte de RED` coincide con `corte` y con `  RED  `. Cliente `Cliente Norte` coincide con `cliente` y con `nor`, y no coincide con `sur`.
- `test_el_filtro_de_texto_es_literal`: entre `100% listo` y `1000 listo`, el filtro `100%` devuelve solo el primero. Entre `a_b` y `axb`, el filtro `a_b` devuelve solo el primero. El filtro `a\b` encuentra el título `a\b`.
- `test_el_filtro_de_texto_distingue_acentos`: cliente `Ácme` coincide con `ácme` y no con `acme`. Cliente `Acme` coincide con `acme`.
- `test_el_filtro_no_colapsa_espacios_internos`: el título `corte de red` no coincide con `corte  de`, y sí coincide con `corte de`.

**Ver fallar.** `pytest tests/test_listado.py::test_cliente_y_titulo_coinciden_sin_mayusculas tests/test_listado.py::test_el_filtro_de_texto_es_literal tests/test_listado.py::test_el_filtro_de_texto_distingue_acentos tests/test_listado.py::test_el_filtro_no_colapsa_espacios_internos -q`. Fallan mientras el filtro distinga mayúsculas, trate `%` o `_` como comodín, iguale acentos o colapse espacios.

**Código.** Recortar extremos del filtro. Buscar por contención literal sin mirar mayúsculas, escapando `\`, `%` y `_`. No unificar acentos ni colapsar espacios internos.

## T16 — Filtro fuera de lista

- Criterios: R11
- Depende de: T3, T12
- Decisiones: D8

**Test.** `test_rechaza_filtro_fuera_de_lista` en `tests/test_listado.py`. `prioridad=Alta`, `prioridad=` , `categoria=correo`, y `prioridad=alta` junto con `categoria=correo`. Espera 422. Si la prioridad es la inválida, el `detail` es `prioridad_invalida`. Si solo la categoría es la inválida, `categoria_invalida`. No espera un arreglo vacío.

**Ver fallar.** `pytest tests/test_listado.py::test_rechaza_filtro_fuera_de_lista -q`. Falla si responde 200.

**Código.** Validar los filtros de lista antes de consultar. Un valor fuera de la lista rechaza el listado con el código de D8.

## T17 — Espacios al guardar

- Criterios: R1, R2
- Depende de: T4
- Decisiones: D12

**Test.** `test_guarda_el_texto_sin_espacios_de_los_extremos` en `tests/test_alta.py`. Título `  Corte de red  `, cliente `  Cliente Sintetico  ` y una descripción con espacios y un salto de línea en los extremos se guardan como `Corte de red`, `Cliente Sintetico` y `Sin enlace`. El título `corte  de red` se guarda con los dos espacios internos.

**Ver fallar.** `pytest tests/test_alta.py::test_guarda_el_texto_sin_espacios_de_los_extremos -q`. Falla si la respuesta conserva los espacios de los extremos o si aplasta los internos.

**Código.** Recortar extremos de los tres textos antes de guardarlos. No recortar prioridad ni categoría. No colapsar espacios internos.

## T18 — Texto largo

- Criterios: R1
- Depende de: T4
- Decisiones: D11

**Test.** `test_acepta_un_texto_largo` en `tests/test_alta.py`. Título, cliente y descripción de 4000 caracteres. Espera 201 y los tres valores iguales a los enviados.

**Ver fallar.** `pytest tests/test_alta.py::test_acepta_un_texto_largo -q`. Con columnas `TEXT` puede nacer verde: se conserva. Si falla por longitud, ampliar la columna a `TEXT`.

**Código.** Dejar las tres columnas en `TEXT` y sin tope en la validación.

## T19 — Filtro de texto vacío

- Criterios: —
- Depende de: T3, T12
- Decisiones: D8, D14

**Test.** `test_un_filtro_de_texto_vacio_se_rechaza` en `tests/test_listado.py`. `cliente` y `titulo`, cada uno enviado como `""` y como solo espacios, con token de agente. Espera 422 y `{"detail": "filtro_vacio"}`. Un listado sin esos parámetros sigue devolviendo los tickets.

**Ver fallar.** `pytest tests/test_listado.py::test_un_filtro_de_texto_vacio_se_rechaza -q`. Falla si el filtro vacío se ignora y responde 200.

**Código.** Si el filtro de texto, después de recortar, queda vacío, responder 422 `filtro_vacio`.

## T20 — Campo de más

- Criterios: R1
- Depende de: T3, T4
- Decisiones: D8, D15

**Test.** `test_rechaza_un_campo_extra_en_el_alta` en `tests/test_alta.py`. El cuerpo válido más `estado` `cerrado`, y otro más `nivel` `3`. Espera 422 y `{"detail": "campo_no_permitido"}`.

**Ver fallar.** `pytest tests/test_alta.py::test_rechaza_un_campo_extra_en_el_alta -q`. Falla si responde 201 o si el ticket queda cerrado o en nivel 3.

**Código.** Rechazar cualquier campo fuera de los cinco. El sistema sigue fijando `abierto` y nivel `1` en el alta que sí pasa.

## T21 — Mismo título

- Criterios: R1
- Depende de: T4
- Decisiones: D17

**Test.** `test_permite_el_mismo_titulo` en `tests/test_alta.py`. Dos altas con el mismo título, cliente y descripción. Espera 201 las dos, con `id` distintos.

**Ver fallar.** `pytest tests/test_alta.py::test_permite_el_mismo_titulo -q`. Si las dos responden 201, el test nace verde y se conserva. Si la segunda falla por unicidad, se saca esa restricción.

**Código.** Sin restricción única sobre título, cliente ni descripción.
