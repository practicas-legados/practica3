# Caracterización de la aplicación legada

Este documento recoge las observaciones realizadas manualmente sobre la
aplicación BASIC ejecutada mediante DOSBox. La distribución original está en
`legacy/original/` y no debe modificarse para realizar consultas.

## 1. Alcance confirmado

La web solo expondrá estas operaciones:

1. Consultar el número de registros.
2. Buscar programas por nombre.
3. Listar los programas de una cinta.

El orden original de la aplicación es `ANTIGUEDAD`. El equipo ha confirmado
que este es el orden de grabación que debe conservarse al listar los
programas de una cinta.

## 2. Menú principal observado

```text
MENU

1 - INTRODUCIR DATOS
2 - GRABAR DATOS
3 - ORDENAR DATOS
4 - INFORMACION BASE DE DATOS
5 - IMPRIMIR DATOS SEGUN ORDEN
6 - LISTAR DATOS SEGUN ORDEN
7 - BUSCAR DATOS
8 - ACABAR
```

Las únicas opciones de consulta que puede usar el wrapper son `4`, `6` y `7`.

## 3. Opción 4: información de la base de datos

### Secuencia observada

```text
Menú
  4
Pantalla de información
  ENTER
Menú
```

### Pantalla observada

```text
INFORMACION BASE DE DATOS

CONTIENE 764 ARCHIVOS

LA BASE DE DATOS ESTA DESORDENADA.
LA ULTIMA ORDENACION FUE SEGUN EL CAMPO 'ANTIGUEDAD'.
MEMORIA DISPONIBLE = 22503 BYTES

PULSA ENTER
```

En otra ejecución, después de recorrer el listado, el contador continuó
siendo `764`, aunque la memoria disponible cambió. La memoria es un dato de
estado de ejecución y no forma parte de la respuesta web.

El wrapper debe extraer el número de `CONTIENE <n> ARCHIVOS` en cada consulta;
`764` no es una constante.

## 4. Opción 6: listado según el orden

### Inicio

```text
Menú
  6
Pantalla de inicio del listado
  ENTER
Primera página
```

La pantalla inicial solicita:

```text
INTRODUCE EL DATO DEL PRIMER PROGRAMA A VISUALIZAR
O ENTER PARA VER DESDE EL PRINCIPIO
```

Para recorrer todos los registros se pulsa `ENTER` sin introducir un dato.

### Columnas observadas

Las páginas muestran las columnas:

```text
Nº
NOMBRE
TIPO
CINTA
REGISTRO
```

Una primera página mostró 18 filas, desde `MUGSY` hasta `HIGH NOON`, con
registros `1` a `18`. La segunda página mostró registros `19` a `36`.

El campo `Nº` no debe confundirse con `REGISTRO`. Después de ordenar una copia
aislada por `TIPO`, se observó, por ejemplo, una fila con `Nº = 1` y
`REGISTRO = 617`. El campo `REGISTRO` es el candidato para
`Programa.numeroRegistro`, pendiente de confirmación con el parser de la
Persona B.

### Paginación y final

Cada pantalla muestra:

```text
PULSA SPACE PARA CONTINUAR U OTRA TECLA PARA ACABAR
```

Al pulsar `SPACE` se obtiene la página siguiente. La última página observada
contenía los registros `757` a `764`, solo ocho filas, y seguía mostrando el
mismo mensaje. Al pulsar `SPACE` en esa página, la aplicación volvió
directamente al menú; no mostró un mensaje explícito de fin.

Una tecla distinta de `SPACE`, por ejemplo `ENTER`, termina el listado y
vuelve al menú. Para una consulta de cinta no se debe usar esa salida hasta
haber procesado la última página.

La estrategia acordada es:

1. Consultar el contador mediante la opción `4`.
2. Entrar en `6` y comenzar con `ENTER`.
3. Entregar cada página al parser.
4. Pulsar `SPACE` mientras queden registros por procesar.
5. Procesar también la última página parcial.
6. Confirmar el retorno al menú.
7. Filtrar la cinta solicitada sin reordenar los resultados.

El wrapper no debe usar la opción `3` para obtener este orden.

## 5. Opción 7: búsqueda por nombre

### Caso con resultado

Con el nombre observado `MUGSY`, la secuencia fue:

```text
7
N + ENTER
MUGSY + ENTER
S + ENTER
N + ENTER
N + ENTER
Menú
```

Las pantallas observadas fueron:

```text
SABES EL NUMERO DE REGISTRO QUE QUIERES BUSCAR? (S/N)
```

Después de responder `N`:

```text
DIME EL NOMBRE DEL PROGRAMA QUE QUIERES VER
```

El resultado mostró:

```text
CONVERSACIONAL

CINTA:A

1 - MUGSY

ES ESTE ? (S/N)
```

Al confirmar el resultado con `S`, apareció:

```text
QUIERES ALTERAR SUS DATOS? S/N
```

El wrapper debe responder siempre `N` a esta pregunta. Después aparece:

```text
QUIERES BUSCAR ALGUNO MAS? S/N
```

Se responde `N` y se comprueba el retorno al menú.

### Caso sin resultados

Para un nombre inexistente se observó:

```text
NO HAY NINGUN PROGRAMA CON ESE NOMBRE; PULSA ENTER
```

Después de `ENTER` aparece:

```text
QUIERES BUSCAR ALGUNO MAS? S/N
```

Se responde `N + ENTER` y se vuelve al menú. Este caso es una consulta válida
sin coincidencias y debe producir una lista vacía, no una excepción.

La coincidencia exacta/parcial y el tratamiento de mayúsculas todavía deben
confirmarse. El wrapper debe delegar la búsqueda al legado y no inventar una
semántica propia.

## 6. Opción 3: ordenación y riesgo de modificación

La opción 3 muestra:

```text
1 - NOMBRE
2 - TIPO
3 - CINTA
4 - ANTIGUEDAD
¿ SEGUN QUE CAMPO QUIERES ORDENAR LOS DATOS ?
```

En una copia aislada se eligió `TIPO`. La operación tardó un tiempo apreciable
y solo terminó cuando apareció el indicador de entrada; después se pulsó
`ENTER` para volver al menú.

La opción 4 pasó a indicar que la base estaba ordenada por `TIPO`. La opción 6
también cambió su orden, mostrando registros no consecutivos y agrupados por
tipo. La cantidad de registros siguió siendo `764`.

La opción 2 puede guardar ese criterio de ordenación antes de salir. Para
cerrar la aplicación se observó la secuencia:

```text
8
S + ENTER
```

Conclusión: las opciones `2` y `3` quedan fuera del wrapper. No deben
ejecutarse en `legacy/original/`, ni siquiera como parte de una consulta.

## 7. Reglas de seguridad para Persona A

- Mantener una única sesión DOSBox controlada.
- Serializar todas las operaciones que envían teclas o capturan pantalla.
- Comenzar cada operación desde el menú conocido.
- Terminar cada operación correctamente en el menú.
- Responder `N` ante cualquier pregunta de alterar datos.
- No usar `1`, `2`, `3` ni `5` desde el wrapper.
- Usar `8` solo para cerrar la sesión de forma controlada.
- No leer `DATOS.DAT` ni `INFORM.DAT` para responder consultas.
- No convertir una pantalla ilegible o un estado desconocido en una respuesta
  vacía.
- Si no se puede recuperar el menú, marcar la sesión como fallida antes de
  aceptar otra petición.

## 8. Propuestas de interfaz entre Persona A y Persona B

La interfaz debe mantener separadas la coordinación de DOSBox y la
interpretación de pantallas. Persona A controla estados y teclas; Persona B
interpreta capturas o textos y comunica errores explícitos.

### Propuesta 1: resultados tipados por pantalla

Persona A entrega una captura a un lector y recibe un resultado discriminado:

```java
interface ScreenReader {
    ScreenResult read(ScreenCapture capture) throws ScreenReadException;
}

sealed interface ScreenResult
        permits MenuScreen, CountScreen, ListPageScreen,
                SearchResultScreen, NoSearchResultsScreen {
}
```

Ventajas:

- Los estados posibles quedan explícitos.
- Es difícil confundir “sin resultados” con una pantalla ilegible.
- Facilita probar cada pantalla con muestras guardadas.

Inconvenientes:

- Requiere definir tipos para las capturas y cada estado.
- Puede ser más trabajo inicial para B.

### Propuesta 2: lectores especializados por operación

Cada operación tiene un lector específico:

```java
interface CountScreenReader {
    int readCount(ScreenCapture capture) throws ScreenReadException;
}

interface ListPageReader {
    ListPage read(ScreenCapture capture) throws ScreenReadException;
}

interface SearchScreenReader {
    SearchResult read(ScreenCapture capture) throws ScreenReadException;
}
```

`ListPage` tendría que indicar explícitamente las filas reconocidas y si la
pantalla es una página válida. El lector de búsqueda distinguiría `FOUND` y
`NOT_FOUND`.

Ventajas:

- Contratos pequeños y fáciles de probar.
- Cada lector conoce solo los formatos que necesita.
- Encaja bien si B implementa parsers separados.

Inconvenientes:

- Puede duplicar la detección de elementos comunes, como el menú.
- A necesita coordinar varios lectores.

### Propuesta 3: protocolo de eventos de interacción

Persona A controla el flujo y B devuelve eventos:

```java
interface ScreenInterpreter {
    ScreenEvent interpret(ScreenCapture capture)
        throws ScreenReadException;
}

sealed interface ScreenEvent
        permits MenuDetected, CountRead, PageRead,
                SearchFound, SearchNotFound {
}
```

Ejemplos de eventos:

```text
MenuDetected
CountRead(764)
PageRead(rows, hasSpacePrompt)
SearchFound(programData)
SearchNotFound
```

Ventajas:

- Representa bien una sesión interactiva.
- A decide las teclas según eventos observados.
- Permite registrar trazas de diagnóstico sin mezclar OCR y control.

Inconvenientes:

- Hay que especificar cuidadosamente qué evento es válido en cada estado.
- Es más amplio que una interfaz de lectores simples.

## 9. Propuesta recomendada

Se recomienda combinar la claridad de la propuesta 1 con lectores
especializados de la propuesta 2:

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
```

Tipos mínimos:

```java
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

Reglas del acuerdo A/B:

1. Una pantalla ilegible produce `ScreenReadException`, nunca una lista vacía.
2. `SearchNotFound` solo se usa cuando se reconoce el mensaje de ausencia.
3. `ListPage.rows` contiene únicamente filas reconocidas completamente.
4. Si una fila está cortada o es ambigua, se produce un error de lectura.
5. `continuePrompt` indica que la pantalla contiene la instrucción de
   continuar; no significa por sí sola que exista otra página.
6. El controlador A decide cuándo pulsar `SPACE` y comprueba después si la
   sesión volvió al menú.
7. El parser B conserva el orden de las filas recibido desde el legado.
8. La conversión de `CINTA` (`B-E`) a `List.of("B", "E")` debe estar en el
   parser o en un componente acordado, pero no puede reordenar identificadores.
9. La correspondencia de `numeroRegistro` se hará con el campo `REGISTRO`,
   una vez confirmada por B mediante muestras.

Esta interfaz permite que A implemente y pruebe la coordinación con un lector
falso antes de integrar el OCR real.

## 10. Pendientes de integración

- Confirmar con Persona B el mecanismo de captura (`ScreenCapture`).
- Confirmar el tipo de `numeroRegistro` con una muestra parserizada.
- Confirmar coincidencia exacta/parcial y mayúsculas de la búsqueda.
- Definir cómo se detecta el menú después de cada tecla.
- Definir el límite de tiempo y el error de pantalla sin cambios.
- Acordar si el contrato de lector se implementa en un módulo `contracts/` o
  dentro de `wrapper/`, sin duplicarlo.

## 11. Decisión de nomenclatura del código de Persona A

El código nuevo de Persona A se desarrollará en:

```text
dosbox-control/
```

Se elige este nombre porque describe su responsabilidad real: controlar
DOSBox, enviar teclas, capturar pantallas, serializar el acceso y coordinar las
consultas seguras.

La distribución original no cambia de ubicación:

```text
legacy/original/
```

El nombre `legacy` no se utilizará para el código nuevo, para distinguir la
aplicación BASIC/MS-DOS entregada del controlador que la utiliza. La carpeta
`wrapper/` se reservará para la fachada `LegacyService` y la integración final.

El paquete Java recomendado para Persona A es:

```java
package dosbox.control;
```
