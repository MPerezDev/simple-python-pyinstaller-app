# simple-python-pyinstaller-app

Este repositorio es para el tutorial [Construir una aplicación Python con PyInstaller](https://jenkins.io/doc/tutorials/build-a-python-app-with-pyinstaller/) en la [Documentación de Usuario de Jenkins](https://jenkins.io/doc/).

El repositorio contiene una aplicación simple de Python que es una herramienta de línea de comandos llamada "add2vals" que muestra la suma de dos valores. Si al menos uno de los valores es una cadena de texto, "add2vals" trata ambos valores como cadenas y en su lugar los concatena. La función "add2" en la librería "calc" (que importa "add2vals") está acompañada por un conjunto de pruebas unitarias. Estas pruebas son evaluadas con pytest para verificar que esta función funcione según lo esperado y los resultados se guardan en un reporte XML de JUnit.

La entrega de la herramienta "add2vals" a través de PyInstaller convierte esta herramienta en un archivo ejecutable independiente para Linux, el cual puedes descargar a través de Jenkins y ejecutar en la línea de comandos en máquinas Linux sin Python.

El directorio `jenkins` contiene un ejemplo del archivo `Jenkinsfile` (es decir, Pipeline) que crearás tú mismo durante el tutorial.
