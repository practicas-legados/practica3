# practica3
Repositorio para la elaboración de la práctica 3 de sistemas legados

## Documentación del equipo

- [Contrato de componentes y reglas de integración](docs/CONTRATO.md)
- [Guía y prompts para trabajar con asistentes de IA](docs/USO_IA.md)

## Aplicación legada

La distribución de DOSBox y Database está en `legacy/original/`. En Windows,
se puede iniciar desde esa carpeta ejecutando `database.bat` (por ejemplo,
desde una consola: `cd legacy\original` y después `database.bat`). Mantén la
estructura de directorios incluida, ya que el lanzador usa rutas relativas.

Los archivos de la aplicación BASIC son material de la asignatura y no deben
modificarse. El wrapper debe consultar la aplicación en ejecución y no leer los
ficheros `.DAT` para responder.
