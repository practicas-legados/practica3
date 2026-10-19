# Guía para trabajar con asistentes de IA

Esta guía acompaña a [`CONTRATO.md`](./CONTRATO.md). El contrato es la fuente
única de verdad para interfaces y comportamiento compartido.

## Contexto que debe conocer cada asistente

El proyecto integra una página web con una aplicación BASIC/MS-DOS existente.
Las consultas deben ejecutarse en el programa legado. No se debe modificar ni
reemplazar el programa ni leer sus ficheros de datos para responder a las
peticiones. La web, el wrapper y el lector de pantalla se desarrollan en
paralelo usando el contrato, un mock y muestras reproducibles.

## Instrucciones comunes

Antes de modificar código:

1. Leer `docs/CONTRATO.md`.
2. Revisar la estructura y convenciones ya presentes en el módulo asignado.
3. Confirmar que la tarea respeta la responsabilidad de ese módulo.

Durante el trabajo:

- No modificar archivos del programa legado.
- No responder consultas desde `DATOS.DAT`, `INFORM.DAT`, un mock en
  producción ni una caché.
- No inventar menús, teclas, formatos de pantalla, resultados OCR ni
  semánticas de búsqueda que no estén observados.
- No convertir errores de captura/parser en respuestas vacías o resultados
  aparentemente válidos.
- No implementar todas las opciones del menú legado como funciones de la web:
  solo se exponen las tres consultas indicadas en `docs/CONTRATO.md`.
- No usar la opción 3 (ordenar), ni asumir que el número de registro equivale
  al orden de grabación, sin verificarlo y acordarlo con el grupo.
- Mantener la exclusión mutua al usar una misma sesión DOSBox.
- Rechazar entradas inválidas con un mensaje; no borrar caracteres
  silenciosamente.
- Mantener la web independiente de DOSBox y del OCR.
- No cambiar interfaces o reglas comunes sin avisar al grupo y actualizar
  contrato, mock y pruebas.
- Seguir los patrones y dependencias que ya utiliza el repositorio. No añadir
  dependencias sin explicar su necesidad.
- Mantener las pruebas reproducibles; usar mocks y muestras siempre que no sea
  necesario probar el proceso DOSBox real.

Al terminar, informar brevemente:

- Archivos cambiados.
- Qué comportamiento se implementó.
- Pruebas o comprobaciones ejecutadas y su resultado.
- Decisiones pendientes o dependencias de otros integrantes.

## Prompt base para cualquier tarea

```text
Tarea: [describir una tarea concreta y acotada].

Lee primero docs/CONTRATO.md y revisa el código relacionado. Respeta el
contrato, la separación de responsabilidades y los patrones existentes. No
modifiques la aplicación BASIC/MS-DOS, no leas sus archivos de datos para
implementar respuestas y no inventes comportamiento que no haya sido
observado. Si la tarea requiere una decisión aún pendiente, deja explícito el
bloqueo y propone cómo verificarlo.

Implementa solo lo necesario para esta tarea. Añade o actualiza pruebas
apropiadas con mocks o muestras reproducibles. No cambies interfaces
compartidas sin señalarlo y actualizar el contrato. Al final resume los
archivos modificados, las comprobaciones realizadas y los pendientes.
```

## Prompt para A — control y coordinación del legado

```text
Implementa la tarea de control/coordinación del legado descrita aquí:
[tarea concreta].

Lee docs/CONTRATO.md. La aplicación real es la fuente de las respuestas y no se
puede modificar. Garantiza que una única instancia DOSBox no reciba
interacciones simultáneas, que los datos no se alteren y que cada operación
termine en el estado conocido acordado. No supongas teclas, tiempos o pantallas
que no estén caracterizados; registra lo que falte. Distingue fallo de
interacción de una consulta válida sin resultados. Prueba la coordinación con
dobles cuando sea posible.
```

## Prompt para B — captura, OCR y parser

```text
Implementa la tarea de captura/OCR/parser descrita aquí: [tarea concreta].

Lee docs/CONTRATO.md. El parser debe distinguir resultados válidos, ausencia
de resultados, paginación y salida ilegible. Usa capturas/textos guardados
para que las pruebas no necesiten DOSBox. No inventes formatos ni ajustes OCR
sin evidencia: agrega muestras observadas y documenta dudas para el grupo.
Expón errores de lectura claramente; nunca devuelvas una lista vacía por un
fallo de OCR.
```

## Prompt para C — web y mock

```text
Implementa la tarea de web descrita aquí: [tarea concreta].

Lee docs/CONTRATO.md y respeta las rutas, estructuras JSON, validaciones y
códigos de error acordados. La web consume el servicio y no debe conocer
DOSBox, OCR ni parser. Usa un mock de LegacyService/API para desarrollo y
pruebas. Presenta estados de carga, resultados, lista vacía y error. Rechaza
entradas inválidas sin modificarlas silenciosamente.
```

## Prompt para revisión de integración

```text
Revisa la integración de los componentes indicados contra
docs/CONTRATO.md: [módulos/archivos].

Busca incompatibilidades entre contrato, web, wrapper, parser y mock;
especialmente: respuesta vacía frente a error OCR, orden de resultados de
cinta, lectura de todas las páginas, concurrencia sobre DOSBox, entradas que
puedan modificar datos y uso accidental de caché o archivos DAT. Comprueba
también que la web solo exponga las tres funcionalidades de la práctica y que
el orden de cinta no se haya supuesto sin evidencia. No hagas cambios de
interfaz de forma unilateral. Informa hallazgos concretos, pruebas que los
demuestren y correcciones mínimas sugeridas.
```

## Cambios al contrato

Si una tarea parece exigir cambiar un contrato:

1. Explicar qué requisito o evidencia lo exige.
2. Señalar qué módulos y personas se verían afectados.
3. Proponer el cambio al grupo antes de implementarlo.
4. Actualizar este contrato, los mocks y las pruebas coordinadamente.
5. Avisar a los demás integrantes antes de integrar ramas o módulos.
