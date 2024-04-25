---
title: Tratamiento de la TTY
date: 2023-03-27
categories: [Tutoriales]
tags: [TTY, Reverse Shell]
image:
  path: /assets/img/commons/tratamiento-tty/bash.png
---

Cuando obtenemos una shell inversa en un sistema Linux, nos damos cuenta de que la shell que conseguimos no es muy cómoda ni funcional. No podemos usar las flechas, el tabulador, los atajos de teclado (`Ctrl+C`, `Ctrl+L`, etc.), ni siquiera ejecutar algunos programas como Python o Vim. Además, si pulsamos `Ctrl+C`, la conexión se cierra y debemos volver a empezar.

La razón de esto es que la shell que obtenemos no es una terminal real, sino una pseudo-terminal (`PTY`) con limitaciones. Para solucionar este problema, es necesario realizar un **tratamiento de la TTY**, que consiste en seguir una serie de pasos para mejorar la interactividad y funcionalidad de la shell remota.

A continuación, vamos a describir los pasos necesarios para solucionar este problema y conseguir una shell interactiva totalmente funcional, con capacidad de tabulación, atajos de teclado, sesiones interactivas, etc. Una vez completados estos pasos, podremos trabajar con la shell inversa como si estuviéramos conectado al sistema mediante SSH.

- Crear una nueva sesión de bash con un **pseudo-terminal**.

Un **pseudo-terminal** simula la funcionalidad de una terminal física y se utiliza a menudo para proporcionar acceso a sistemas remotos a través de SSH u otros protocolos de red. Para crear una nueva sesión de bash con un **pseudo-terminal**, podemos usar uno de estos dos comandos:

```bash
script /dev/null -c bash
```

O a través de python, para ello se recomienda primero verificar las versiones disponibles en el sistema mediante el comando `whereis python` a nivel del sistema. Luego, podemos ejecutar el siguiente comando junto con la versión correspondiente.

```bash
python -c 'import pty;pty.spawn ("/bin/bash")'
```

- Suspender la shell actual.

Para hacer esto, podemos usar la combinación de teclas `[ctrl+z]`, que envía una señal al proceso actual para que se detenga temporalmente.

- Establecer la terminal local en modo `raw` y deshabilitar el `echo`.

El modo `raw` es un modo en el que la terminal no procesa ni interpreta los caracteres especiales como las teclas de control o el backspace, sino que los envía directamente al proceso. Esto nos permite usar las teclas de control en la shell inversa. El `echo` es una función que hace que la terminal muestre lo que escribimos en la pantalla. Al deshabilitarlo, evitamos que se dupliquen los caracteres en la shell inversa. Para establecer la terminal local en modo `raw` y deshabilitar el `echo`, podemos usar este comando:

```bash
stty raw -echo; fg
```

El comando `fg` (foreground) sirve para recuperar la shell suspendida y traerla al primer plano.

- Reiniciar la configuración de la terminal.

Esto nos permite limpiar la pantalla y restablecer los valores por defecto de la terminal. Para reiniciar la configuración de la terminal, podemos usar este comando:

```bash
reset
```

- Especificar el tipo de terminal.

El tipo de terminal define las características y capacidades de la terminal, como los colores, las fuentes o las secuencias de escape. Para especificar el tipo de terminal, podemos usar este comando:

```bash
xterm
```

El comando `xterm` nos asigna el tipo de terminal `xterm`, que es uno de los más comunes y compatibles.

- Asignar `xterm` a la variable `TERM`.

La variable `TERM` es una variable de entorno que indica al sistema operativo el tipo de terminal que estamos usando. Para asignar `xterm` a la variable `TERM`, podemos usar este comando:

```bash
export TERM=xterm
```

- Asignar `bash` a la variable `SHELL`.

La variable `SHELL` es una variable de entorno que indica al sistema operativo el tipo de shell que estamos usando. Para asignar `bash` a la variable `SHELL`, podemos usar este comando:

```bash
export SHELL=bash
```

- Establecer el valor de filas y columnas.

El valor de filas y columnas define el tamaño de la ventana de la terminal en términos de caracteres. Para establecer el valor de filas y columnas, podemos usar este comando:

```bash
stty rows <row_value> columns <column_value>
```

Donde `<row_value>` y `<column_value>` son los valores numéricos que queremos asignar.    

> Para consultar el valor actual de filas y columnas de nuestra terminal, podemos usar uno de estos comandos: `stty size` o `stty -a`
{: .prompt-tip }

Siguiendo estos pasos, podremos disfrutar de una shell totalmente interactiva y con todas las ventajas de una terminal local. Esto nos facilitará mucho el trabajo y nos permitirá realizar tareas más complejas y avanzadas en el sistema objetivo.
