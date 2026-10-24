# Contrato de trabajo — Práctica 3

Este documento fija las interfaces y reglas que permiten desarrollar la web,
el wrapper y el lector de pantalla en paralelo. Distingue los datos observados
en el ZIP de los detalles que todavía deben comprobarse ejecutando DOSBox.

## 1. Objetivo y límites

La aplicación web permite consultar los datos que administra el programa BASIC
para MS-DOS incluido en la práctica. El programa legado es quien debe ejecutar
cada consulta: no se sustituye por una reimplementación ni por una copia de
sus datos.

- No modificar los archivos de la carpeta `Database` entregada.
- No leer `DATOS.DAT` ni `INFORM.DAT` para contestar consultas. Se pueden
  inspeccionar para caracterizar la distribución, pero la respuesta de runtime
  debe proceder de una operación ejecutada en la aplicación.
- No conservar respuestas para usarlas en llamadas futuras.
- No enviar entradas que cambien o graben datos. En las pantallas de búsqueda
  que pregunten si se confirma o se quiere alterar un registro, responder
  negativamente y volver al flujo de consulta.

## 2. Funcionalidades requeridas

1. Consultar el número actual de registros.
2. Buscar un programa por nombre y mostrar los datos que presenta la aplicación.
3. Consultar los programas de una cinta, respetando su orden de grabación.

La lectura de todos los registros para una consulta de cinta debe hacerse
mediante la interfaz del legado. El wrapper filtra los registros obtenidos;
no consulta directamente los ficheros de datos.

### Alcance del menú legado

La aplicación presenta ocho opciones. La práctica pide exponer las tres
consultas anteriores en la web, no reproducir todas las funciones del menú:

| Opción | Operación del menú | Alcance de esta práctica |
| --- | --- | --- |
| 1 | Introducir datos | No exponer; podría añadir o alterar datos |
| 2 | Grabar datos | No exponer; escribe datos |
| 3 | Ordenar datos | No exponer como función web. Solo considerar como paso interno si la consulta lo necesita y el grupo verifica que es seguro |
| 4 | Información base de datos | Usar internamente para consultar el contador |
| 5 | Imprimir datos según orden | No exponer; está orientado a imprimir, no a la respuesta web |
| 6 | Listar datos según orden | Usar internamente para obtener filas; procesar todas las páginas necesarias |
| 7 | Buscar datos | Usar internamente para buscar por nombre |
| 8 | Acabar | No es consulta; solo usar al cerrar DOSBox de forma controlada |

Por tanto, las opciones 1, 2, 3, 5 y 8 no son funcionalidades que el usuario
deba poder invocar desde la web. El wrapper puede navegar por las opciones 4,
6 y 7 para cumplir los requisitos. La opción 3 queda expresamente fuera del
flujo hasta que se pruebe si es necesaria para respetar el orden pedido y qué
efectos tiene.

## 3. Lo que se ha comprobado en el ZIP

La distribución adjunta contiene `DATABASE.BAS`, `DATOS.DAT`, `INFORM.DAT`,
`database.bat` y DOSBox 0.74. El lanzador ejecuta DOSBox y, dentro de él,
`gwbasic.bat` inicia GW-BASIC con `database`.

Las cadenas visibles de `DATABASE.BAS` confirman:

- El menú principal ofrece la opción 4 para información de la base de datos,
  la 6 para listar datos según el orden y la 7 para buscar datos.
- La opción de búsqueda permite elegir entre buscar por número de registro o
  por nombre. La pantalla muestra un mensaje explícito si no hay un programa
  con ese nombre.
- La opción de listado permite indicar desde qué programa comenzar y muestra
  nombre, tipo, cinta y número de registro. El listado puede continuar con
  `SPACE`; otra tecla termina la continuación.
- El campo de cinta es texto. En los datos incluidos aparece, por ejemplo,
  `B-E`, y el guion de la práctica da `A-45` como ejemplo de varias cintas.
- El `INFORM.DAT` distribuido empieza con el número `764` y el orden
  `ANTIGUEDAD`. Es una observación de la distribución inicial, no un valor
  constante: la web debe consultar el número en runtime.

La inspección de cadenas y ficheros del ZIP no confirma por sí sola todos los
tiempos, pantallas, criterios de coincidencia o comportamiento en DOSBox. Los
detalles que dependan de interacción real se mantienen pendientes en la
sección 9.

## 4. Contrato del servicio Java

Interfaz que consume la web:

```java
public interface LegacyService {
    int contarRegistros() throws LegacyException;

    List<Programa> buscarPorNombre(String nombre) throws LegacyException;

    List<Programa> listarPorCinta(String identificador)
        throws LegacyException;
}
```

Modelo compartido:

```java
public record Programa(
    int numeroRegistro,
    String nombre,
    String tipo,
    List<String> cintas
) {}
```

Reglas del modelo y resultados:

- `numeroRegistro` es el número de registro mostrado por el legado. Es un
  entero positivo.
- `cintas` contiene identificadores separados, por ejemplo
  `List.of("B", "E")` para el campo de pantalla `B-E`.
- Los campos de texto conservan el contenido reconocido del legado; no se
  inventan ni completan valores.
- Los métodos que devuelven listas nunca devuelven `null`.
- Una consulta válida sin coincidencias devuelve una lista vacía.
- Un error de lectura o interacción nunca se disfraza de lista vacía.
- La lista de búsqueda conserva el orden en que el legado presenta los
  resultados.
- `listarPorCinta("45")` busca el identificador individual `45`, no una
  subcadena. Por tanto, la cinta `1` no coincide con la cinta `10`.
- `listarPorCinta` debe conservar el orden que corresponda al orden de
  grabación indicado por la práctica. Todavía no está demostrado que ese orden
  sea igual al número de registro, al orden actual del listado ni al orden
  alfabético por cinta. No ordenar por `numeroRegistro` ni invocar la opción 3
  hasta que la caracterización de la sección 9 confirme el procedimiento.
- El parámetro de cinta es un identificador individual (`A`–`Z` o uno o más
  dígitos), no el campo compuesto `A-45`.
- No se fija todavía si la búsqueda por nombre es exacta o parcial, ni si
  distingue mayúsculas. El wrapper debe usar la búsqueda por nombre del
  legado; el comportamiento observado se registrará al caracterizarlo.

`LegacyException` comunica fallos de interacción con el legado, por ejemplo
que DOSBox no se pudo iniciar, una operación excedió el tiempo límite, la
pantalla no pudo interpretarse o la sesión quedó en un estado desconocido.
Puede tener un código interno, pero nunca debe ocultar el fallo con un valor
de éxito.

## 5. Contrato HTTP para desacoplar la web

La tecnología del servidor puede elegirse después; para que la web avance con
un mock, se acuerda esta API JSON:

| Método y ruta | Resultado correcto |
| --- | --- |
| `GET /api/v1/registros/count` | `{"count": 764}` — el número es ilustrativo; se obtiene en runtime |
| `GET /api/v1/programas?nombre=<texto>` | `{"items": [<Programa>, ...]}` |
| `GET /api/v1/cintas/{identificador}/programas` | `{"items": [<Programa>, ...]}` |

Representación JSON de un programa:

```json
{
  "numeroRegistro": 18,
  "nombre": "MATCH POINT",
  "tipo": "S. DEPORTIVO",
  "cintas": ["B", "E"]
}
```

Respuesta de error uniforme:

```json
{
  "error": {
    "code": "LEGACY_UNAVAILABLE",
    "message": "No se pudo completar la consulta al sistema legado."
  }
}
```

Códigos HTTP acordados:

- `200`: consulta completada, incluso cuando `items` está vacío.
- `400`: entrada ausente o inválida.
- `502`: no se pudo interpretar la salida del legado.
- `503`: el legado no está disponible o no se pudo iniciar.
- `504`: se agotó el tiempo de espera.
- `500`: otro error interno no previsto.

La web valida y comunica errores, pero no controla DOSBox, captura pantalla ni
interpreta OCR. No elimina silenciosamente caracteres del nombre: valida la
entrada según las reglas acordadas y explica cómo corregirla.

## 6. Responsabilidades de los componentes

### A — Control y coordinación del legado

- Caracterizar la interacción real: lanzamiento, menú, entradas, esperas y
  vuelta a un estado seguro.
- Implementar el controlador de DOSBox y la coordinación que implementa
  `LegacyService`.
- Obtener el contador desde la opción de información del legado.
- Ejecutar la búsqueda por nombre sin confirmar modificaciones.
- Usar las opciones de consulta 4, 6 y 7. No exponer en la web el resto de las
  operaciones del menú.
- Leer todas las páginas del listado cuando la operación requiera todos los
  registros; convertir el campo de cinta en identificadores y filtrar la cinta
  solicitada.
- Establecer y documentar cómo se garantiza el orden de grabación solicitado.
  No asumir que coincide con el número de registro ni cambiar el orden del
  legado con la opción 3 sin verificarlo.
- Serializar el acceso a una única sesión DOSBox.
- Fallar explícitamente si no puede recuperar el menú o determinar el fin de
  una consulta.

### B — Captura, OCR y parser

- Implementar la captura/OCR elegida por el equipo.
- Crear muestras de las pantallas observadas y pruebas reproducibles que no
  necesiten iniciar DOSBox en cada ejecución.
- Analizar contador, filas del listado, resultados de búsqueda y estados de
  continuación/fin.
- Distinguir “no hay resultados” de “la pantalla no se pudo leer”.
- No codificar formatos de pantalla no observados; comunicar dudas a A y
  registrarlas en la sección 9.

### C — Web y mock

- Implementar las tres operaciones descritas por la API HTTP.
- Crear un mock de `LegacyService` con datos de ejemplo para desarrollo y
  pruebas.
- Mostrar carga, resultados vacíos, resultados y errores.
- Validar entradas sin alterarlas silenciosamente.
- No acceder a la aplicación, archivos de datos, OCR ni parser directamente.

### Integración conjunta

- A y B deben acordar la interfaz de pantallas antes de implementarla en
  paralelo. La interfaz recomendada se define en la sección 7.
- Los tres integrantes usan este documento como contrato común.
- Si se propone cambiar una interfaz o regla, avisar al grupo primero y
  actualizar contrato, mock y pruebas en el mismo cambio coordinado.
- El mock sirve para desarrollar la web; no reemplaza la prueba final contra
  la aplicación real.

## 7. Reglas para la sesión DOSBox

- Solo una operación puede enviar teclas o capturar pantalla de una misma
  instancia DOSBox a la vez. La exclusión mutua pertenece al wrapper, no a la
  web.
- Cada operación comienza desde el estado conocido acordado (normalmente el
  menú) y, si tiene éxito, termina de vuelta en ese estado.
- La captura debe esperar una condición observable de pantalla o un límite de
  tiempo razonable; no asumir que una pausa fija siempre basta.
- Para un listado paginado, comenzar desde el primer registro, procesar las
  filas de cada pantalla y continuar con `SPACE` hasta que la pantalla indique
  el final. No truncar silenciosamente el listado.
- Las teclas de confirmación de búsqueda y las de paginación deben distinguirse:
  no confirmar cambios de datos y no abandonar páginas sin haberlas procesado.
- Si el estado final es desconocido, marcar la sesión como fallida y
  reiniciarla o recuperarla antes de aceptar otra petición.

### 7.1. Interfaz entre Persona A y Persona B

La coordinación de DOSBox y la interpretación de pantallas son
responsabilidades separadas:

- Persona A controla la sesión, el estado, las teclas, los tiempos de espera,
  la exclusión mutua y la recuperación del menú.
- Persona B captura/interpreta las pantallas y devuelve resultados tipados o
  un error explícito de lectura.

La interfaz recomendada combina lectores especializados con resultados
tipados:

```java
interface LegacyScreenReader {
    MenuState readMenu(ScreenCapture capture)
        throws ScreenReadException;

    CountScreen readCount(ScreenCapture capture)
        throws ScreenReadException;

    ListPage readListPage(ScreenCapture capture)
        throws ScreenReadException;

    SearchScreen readSearch(ScreenCapture capture)
        throws ScreenReadException;
}

record CountScreen(int count) {}

record ListPage(List<LegacyRow> rows, boolean continuePrompt) {}

record LegacyRow(
    int numeroVisible,
    String nombre,
    String tipo,
    List<String> cintas,
    int numeroRegistro
) {}

sealed interface SearchScreen
        permits SearchFound, SearchNotFound {}

record SearchFound(List<LegacyRow> rows) implements SearchScreen {}

record SearchNotFound() implements SearchScreen {}
```

`ScreenCapture` representa una captura o texto de pantalla producido por la
sesión; su implementación concreta se acuerda con B. `ScreenReadException`
indica que la pantalla no se pudo interpretar con fiabilidad.

Reglas obligatorias de la interfaz:

1. Una pantalla ilegible produce `ScreenReadException`, nunca una lista vacía.
2. `SearchNotFound` solo se devuelve cuando se reconoce el mensaje de ausencia
   del legado.
3. `ListPage.rows` contiene únicamente filas completas y reconocidas.
4. Una fila cortada, ambigua o incompleta produce un error de lectura.
5. `continuePrompt` indica que se reconoce la instrucción de continuar; no
   demuestra por sí solo que exista otra página.
6. A decide cuándo enviar `SPACE` y comprueba después el estado de la sesión.
7. B conserva el orden de las filas presentado por el legado.
8. La conversión de `CINTA` (`B-E`) a `List.of("B", "E")` no puede reordenar
   identificadores.
9. `numeroRegistro` corresponde al campo `REGISTRO`, sujeto a confirmación
   mediante muestras parserizadas.
10. A puede usar un lector falso con estos mismos resultados para probar la
    coordinación antes de integrar el lector real.

La interfaz no obliga a que las clases pertenezcan a un módulo concreto. El
grupo debe acordar si se ubican en `contracts/` o en `wrapper/`, sin duplicar
los tipos.

## 8. Estructura de trabajo prevista

```text
contracts/       Tipos compartidos del servicio y del modelo
dosbox-control/  Entrada/control de DOSBox y coordinación de Persona A
screen-reader/   Captura, OCR, parser y pruebas con muestras (B)
wrapper/         Fachada de servicio e integración con el legado
web/             Interfaz web, cliente HTTP y mock (C)
samples/         Capturas/textos anonimizados o distribuibles para pruebas
legacy/original/ Distribución original; no modificar
docs/            Contrato, decisiones y memoria
scripts/         Preparación y ejecución
```

El formato exacto de módulos y la herramienta de construcción se decidirán
según la tecnología que adopte el equipo. La dependencia funcional debe seguir
esta dirección:

```text
web -> contrato HTTP -> wrapper -> LegacyService
                                  |-> dosbox-control
                                  `-> screen-reader/parser
```

La web puede trabajar desde el primer día contra el mock. El parser puede
trabajar desde el primer día con muestras guardadas. A puede desarrollar la
coordinación usando un lector/control falso hasta que B entregue los
componentes reales.

### 8.1. Decisión de nomenclatura para el código de Persona A

El código fuente de Persona A se ubicará en `dosbox-control/`. Este nombre
identifica que el componente controla la sesión DOSBox, envía teclas, coordina
las capturas y serializa las consultas.

La separación de módulos será:

- `dosbox-control/`: control de DOSBox, estados de sesión, exclusión mutua y
  coordinación de las operaciones del legado.
- `wrapper/`: fachada `LegacyService` e integración con el resto del sistema.
- `screen-reader/`: captura, OCR, parser y muestras de Persona B.

No se utilizará `legacy/` como nombre del código nuevo, porque ese nombre se
reserva conceptualmente para la aplicación original. La distribución
entregada seguirá permaneciendo en `legacy/original/` y no se modificará.

El paquete Java recomendado para el componente de A es:

```java
package dosbox.control;
```

La interfaz `LegacyScreenReader` seguirá siendo la frontera compartida con B;
su ubicación física se acordará al crear los módulos, sin duplicar la
interfaz.

## 9. Pendientes de caracterización en DOSBox

Registrar observación, captura de muestra, decisión y responsable para cada
punto:

| Pregunta | Estado |
| --- | --- |
| Secuencia exacta de lanzamiento y menú inicial | Observada y documentada |
| Teclas y pantallas de las opciones 4, 6 y 7 | Observadas y documentadas |
| Forma exacta de volver al menú desde cada consulta | Observada y documentada |
| Coincidencia por nombre: exacta/parcial y mayúsculas/minúsculas | Pendiente de ejecución |
| Número de resultados que puede devolver la búsqueda por nombre | Pendiente de ejecución |
| Formato de filas y posición del número de registro en pantalla | Pendiente de ejecución |
| Texto fiable de la última página del listado | Observado: página parcial 757–764 |
| Qué significa concretamente “orden en que se encuentran grabados” y cómo lo produce la opción 6 | Acordado: orden original `ANTIGUEDAD` |
| Si hace falta la opción 3 para obtener ese orden, y si usarla altera estado persistente | Resuelto: no usar; cambia el criterio de ordenación |
| Texto y secuencia de teclas del caso sin resultados | Observado y documentado |
| Formato permitido de cinta y espacios/guiones en la salida | Ejemplos en ZIP; confirmar pantallas |
| Tiempo máximo razonable por operación y estrategia de recuperación | Pendiente de prueba |

No se bloquea el trabajo con mocks mientras se investigan estos puntos. Sí se
deben resolver antes de cerrar la integración real.

## 10. Criterios de aceptación

- Las tres consultas se ejecutan contra la aplicación legada.
- El contador procede de la pantalla del legado, no de un valor fijo.
- La búsqueda sin coincidencias devuelve `items: []`; una pantalla ilegible
  produce error.
- La consulta de cinta compara identificadores completos y conserva el orden
  de grabación una vez verificado su mecanismo; no presupone que sea el número
  de registro.
- Se leen todas las páginas necesarias del listado.
- Solicitudes simultáneas no mezclan entradas ni capturas.
- Ninguna consulta acepta o graba cambios en los datos.
- La web funciona con el mock; el parser tiene pruebas con muestras guardadas.
- La memoria incluye experimentos, dificultades, resultados OCR, capturas o
  vídeo, reparto de tareas, tiempos e instrucciones de ejecución en Windows.
