---
title: Mi Experiencia con la certificación eJPTv2
date: 2024-05-17
categories: [Certifications, eJPTv2]
tags: [eJPTv2, INE Security, Junior Penetration Tester]
img_path: /assets/img/commons/ejpt/
image: ejpt_cert.png
---

## Introducción
¡Saludos a todos! Soy Juan Rivas, también conocido como r1vs3c. Recientemente, me enfrenté al desafío del examen eJPTv2 y obtuve una puntuación perfecta del 100%. Mi objetivo con este artículo es compartir mi experiencia y proporcionar orientación a aquellos que están dando sus primeros pasos en el emocionante mundo de la ciberseguridad ofensiva. Especialmente, deseo ofrecer consejos valiosos para quienes aspiran a enfrentar el eJPTv2, considerado como el primer escalón hacia certificaciones más avanzadas en pentesting y ethical hacking.

A lo largo de esta guía, exploraremos desde mi trayectoria previa hasta los recursos esenciales, estrategias de preparación efectivas, y hasta los consejos más prácticos que te llevarán al éxito en este desafío. Sumergirnos en esta experiencia no solo implica superar un examen, sino también adentrarnos en un terreno donde cada conocimiento adquirido nos acerca un paso más a comprender la complejidad y el potencial de la seguridad informática desde una perspectiva ofensiva. 

¡Acompáñenme en este recorrido y descubramos juntos los secretos para triunfar en el examen eJPTv2!

## ¿Qué es el eJPT?
El **eJPT**, acrónimo de **eLearn Security Junior Penetration Tester**, es una certificación de nivel inicial en el ámbito del pentesting, ofrecida por [**INE Security**](https://security.ine.com/). Esta certificación valida que el candidato posee los conocimientos, habilidades y capacidades mínimas requeridas para desempeñar el rol de “Pentester Junior”. 

Según INE Security, el examen abarca:

- Metodologías de Evaluación
- Auditoría de Host y Red
- Pruebas de Penetración de Host y Red
- Pruebas de Penetración de Aplicaciones Web

El costo del voucher es de **$249.00**, aunque suele haber ofertas durante el año, especialmente en Black Friday y Navidad. El voucher incluye el plan “Fundamentals” de INE, que proporciona acceso gratuito durante tres meses al curso [**Penetration Testing Student (PTS)**](https://my.ine.com/CyberSecurity/learning-paths/61f88d91-79ff-4d8f-af68-873883dbbd8c/penetration-testing-student), con más de 150 horas de video y alrededor de 121 laboratorios prácticos que te preparan para adquirir las habilidades y la práctica necesaria para el examen eJPT.

Cuando adquieras el voucher, dispondrás de aproximadamente seis meses para programar y presentar el examen. Es reconfortante saber que, en caso de no superarlo en el primer intento, tendrás la oportunidad de volver a intentarlo sin coste alguno, ya que el segundo intento está incluido de forma gratuita.

Para obtener información actualizada sobre el examen, te recomiendo visitar la página oficial de INE: [**eJPT Certification**](https://security.ine.com/certifications/ejpt-certification/).

## ¿Cómo es el examen?
El examen se realiza en un entorno de laboratorio dentro del navegador, utilizando un sistema Kali Linux preconfigurado con todas las herramientas, scripts y diccionarios necesarios para completar exitosamente las preguntas y desafíos asociados. Este sistema Kali Linux no tiene acceso a Internet, aunque se permite usar el navegador del sistema anfitrión para investigar si es necesario. Inicialmente, el sistema Kali Linux tendrá acceso a una red DMZ, donde deberás aplicar la metodología de un **Pentest Real** en modalidad **Black Box** en los distintos hosts y redes internas identificados. A medida que avances en esta metodología, podrás responder las preguntas de opción múltiple del examen. El examen consta de 35 preguntas de opción múltiple, que deberás completar en un plazo máximo de 48 horas. Para aprobar, es necesario obtener al menos un 70% de respuestas correctas en el conjunto total de preguntas del examen.

## Mi Experiencia y Preparación
### Experiencia Previa
En cuanto a mi experiencia previa, ya contaba con más de un año de práctica resolviendo numerosas máquinas en plataformas como [**HackTheBox**](https://www.hackthebox.com/), [**TryHackMe**](https://tryhackme.com/), [**VulnHub**](https://www.vulnhub.com/), entre otras, además de haber realizado varios cursos centrados en seguridad ofensiva. Por ello, opté por no realizar el curso completo del PTS ofrecido por INE Security, ya que la mayoría de los temas ya los tenía claros debido a mi experiencia previa. 

En su lugar, me concentré en algunos videos y laboratorios específicos del curso PTS para repasar conocimientos que no había explorado mucho. Sin embargo, si eres principiante y no tienes experiencia en este tipo de CTFs, te recomiendo encarecidamente aprovechar todo el material del PTS ofrecido por INE Security. Con el contenido del PTS, estarás más que preparado para aprobar el examen.

### Preparación
Mi preparación se basó principalmente en el curso de [**Introducción al Hacking**](https://hack4u.io/cursos/introduccion-al-hacking/) de la academia [**hack4u**](https://hack4u.io/) de s4vitar. Opté por este curso para estar más que preparado para enfrentar el eJPT, ya que aborda temas mucho más avanzados que los que se encuentran en la propia certificación. Además, aproveché el temario del curso como preparación para certificaciones más avanzadas, como el eCPPT y eWPT. 

Adicionalmente, practiqué con máquinas de plataformas ya mencionadas como HackTheBox, TryHackMe, y VulnHub, entre otras. También monté mi propio laboratorio local para ejercitar el pivoting con Metasploit, una herramienta esencial para este examen. Registré todos los conocimientos adquiridos meticulosamente en mis notas de Notion, una metodología que recomiendo ampliamente para administrar tu "segundo cerebro" en este campo.

Con toda esta preparación, sumada a mi experiencia previa, me sentía más que preparado y con mucha confianza para afrontar el examen.

## Opinión Personal
Personalmente, encontré el examen bastante interesante, aunque lo consideré relativamente fácil, logrando concluirlo en unas 4 horas. Creo que es ideal para aquellos que están dando sus primeros pasos en el pentesting, sobre todo debido a su asequible precio en comparación con otras certificaciones del mercado. Esta certificación te sumerge en un escenario empresarial real, en contraposición a los típicos CTFs a los que estamos acostumbrados. Te ayuda a familiarizarte con las diferentes fases de un pentest real y a dominar herramientas esenciales como **Nmap**, **Metasploit**, **Nikto**, **Dirb**, **CrackMapExec**, entre otras.

Sin embargo, considero que el contenido y la certificación como tal se quedan un poco cortos para denominarla “Junior Penetration Tester”, ya que los requerimientos actuales que las empresas exigen están mucho más por encima de lo que contempla la certificación. Incluso hoy por hoy, la propia OSCP se considera una certificación para juniors. Esta certificación, que hace algunos años se consideraba inalcanzable para muchos profesionales, ahora es accesible gracias a la abundancia de recursos disponibles en Internet, como plataformas de aprendizaje, cursos y creadores de contenido.

A pesar de todo, como primera certificación para comprobar que posees los conocimientos mínimos de un Pentester Junior, está bastante bien. Por algo es la primera certificación que los profesionales recomiendan tomar como primer paso en este apasionante mundo. Además, cada vez se está empezando a valorar más esta certificación.

### ¿Vale la Pena?
En cuanto a si vale la pena, diría que depende de tu nivel de experiencia y tus objetivos. Si eres un principiante absoluto y deseas introducirte en el mundo de la ciberseguridad, te recomiendo esta certificación sin dudarlo, ya que cubre aspectos esenciales. Sin embargo, si ya tienes experiencia con plataformas como TryHackMe o HackTheBox, es posible que encuentres el examen relativamente sencillo. En ese caso, podrías considerar certificaciones más avanzadas como eCPPT, eWPT, eWPTX, BSCP, OSCP, etc. Igualmente, si ya tienes experiencia previa pero deseas afrontarla como el primer paso para perder el miedo a las certificaciones prácticas, también es válido. Eso fue justamente lo que hice. Tomé la eJPT como el primer acercamiento a este tipo de certificaciones de INE para estar más preparado para futuras certificaciones como la eCPPT, eWPT o eWPTX.

## Mis Consejos para Aprobar a la Primera
1. **Escaneo Global**: Realiza un escaneo global de todos los hosts identificados en la red DMZ y no vayas de uno en uno. Esto te permitirá obtener una información global de todos los servicios y tecnologías que corren en estos hosts. Además, con los resultados obtenidos, prácticamente podrás responder todas las preguntas asociadas a la red DMZ.

2. **Mapeo Visual**: Una vez realizado el escaneo global, te recomiendo que utilices herramientas como draw.io o excalidraw.com para representar de manera gráfica la configuración de red a la que te estás enfrentando. En este mapa, grafica los diferentes hosts encontrados, referenciándolos con su respectivo hostname o algún servicio representativo.

3. **Lectura Detallada**: Lee todas las preguntas detalladamente ya que están desordenadas. De esta manera, tendrás un mapa mental de todas las preguntas asociadas a determinado host, red interna a la que tendrás que pivotar, red DMZ, etc. Además, leer las preguntas también te ayudará a identificar los hosts que simplemente están de relleno y no necesitas comprometer para aprobar el examen.

4. **Responde por Host**: Te recomiendo que intentes responder todas las preguntas asociadas a determinado host antes de pasar al siguiente. Como mencioné, las preguntas están desordenadas, así que no debes contestar en el orden que te las presentan. Primero, responde las de la red DMZ y luego escoge un host con el que te sientas más cómodo, ya sea por su sistema operativo, los servicios que corre, etc.

5. **Documentación**: Mantén un orden y documenta todo lo que vayas encontrando y los resultados que obtengas de las diferentes herramientas que ejecutes. Puede que necesites esa información más adelante y de esta manera la encontrarás más rápido. Puedes tomar notas en aplicaciones como Obsidian, Notion, CherryTree, entre otras. También, mantén un orden en tu espacio de trabajo, creando una carpeta individual y subcarpetas por cada host que estés auditando para almacenar información relevante.

6. **Descansos**: No es una carrera. Tienes 48 horas para completar el examen, tómatelo con calma. Si te sientes atascado en alguna pregunta, toma descansos para comer, salir a caminar, dormir, ver algún video, etc., para que la mente se refresque y no se ofusque. Esto te vendrá genial para que las ideas fluyan mejor cuando regreses a tu espacio de trabajo.

7. **Aprovecha las Respuestas**: Hay muchas preguntas que te darán pistas para responder otras, por lo que leer todas las preguntas te ayudará a identificar esas pistas.

8. **Cheat Sheets**: Recomiendo crear tus propias hojas de referencia (Cheat Sheets), que incluyan comandos comunes, módulos útiles de Metasploit y técnicas de explotación. Recuerda que el examen permite el uso de materiales de consulta. Además, puedes buscar en Internet desde tu máquina anfitriona, ya que en el laboratorio del examen no tendrás acceso a la red.

9. **Fuerza Bruta**: No dudes en realizar fuerza bruta si es necesario, sobre todo en servicios o formularios de acceso a CMS. Utiliza los diccionarios que se encuentran en el sistema Kali Linux para intentar obtener acceso con credenciales por defecto.

10. **Simplicidad**: No te compliques con scripts adicionales. Utiliza las herramientas y scripts que ya vienen en el laboratorio. Todo lo que necesitas está ahí.

## Herramientas Imprescindibles
- `dirb`: Para realizar fuerza bruta de directorios en sitios web.
- `arp-scan`: Para descubrir dispositivos en la red local.
- `nmap`: Para el escaneo de puertos y servicios.
- `wpscan`: Para la enumeración y explotación de vulnerabilidades en sitios web WordPress.
- `crackmapexec`: Para la explotación de servicios SMB.
- `msfconsole` **(Metasploit)**: Para la explotación de vulnerabilidades y post-explotación.
- `searchsploit`: Para buscar exploits locales.
- `hydra`: Para la fuerza bruta de servicios de autenticación.
- `xfreerdp`: Para acceder a escritorios remotos RDP.

## Recursos Recomendados
- [**Laboratorio de preparación eJPTv2 \| Simulación de examen**](https://youtu.be/v20IsEd5nUU?si=A9vjJDlGuak8wFfE): Un laboratorio que simula el entorno del examen eJPTv2 para practicar.
- [**Simulación de Examen CEH o EJPTv2 - Laboratorio de Máquinas Virtuales**](https://youtu.be/l6tHH2qQmQ8?si=EVoe1p8TUZbQB17b): Otro recurso para practicar en un entorno simulado.
- [**Cómo Hacer PIVOTING con METASPLOIT en Entornos WINDOWS - Preparación eJPTv2**](https://youtu.be/WM8lHCHblDU?si=j7wn8LcwRRUVgcUv): Tutorial para aprender a hacer pivoting con Metasploit en entornos Windows.
- [**Path Junior Penetration Tester TryHackMe**](https://tryhackme.com/path/outline/jrpenetrationtester): Ruta de aprendizaje en TryHackMe diseñada para preparar para el eJPT.
- [**Curso de Introducción al Hacking de S4vitar**](https://hack4u.io/cursos/introduccion-al-hacking/): Curso avanzado que cubre temas esenciales y más allá del eJPT.
- [**Curso de Preparación para la Certificación del eJPTv2 de El Pingüino De Mario**](https://elrincondelhacker.es/courses/preparacion-certificacion-ejptv2/): Curso específico para preparar el eJPTv2.

## Máquinas para Practicar
### TryHackMe
- [**Blue**](https://tryhackme.com/r/room/blue)
- [**Ignite**](https://tryhackme.com/r/room/ignite)
- [**Blog**](https://tryhackme.com/r/room/blog)
- [**Startup**](https://tryhackme.com/r/room/startup)
- [**Chill Hack**](https://tryhackme.com/r/room/chillhack)

### HackTheBox
- [**Devel**](https://app.hackthebox.com/machines/3)
- [**Armageddon**](https://app.hackthebox.com/machines/323)
- [**Blue**](https://app.hackthebox.com/machines/51)
- [**Jerry**](https://app.hackthebox.com/machines/144)
- [**Lame**](https://app.hackthebox.com/machines/1)

### DockerLabs
- [**Todas las máquinas fáciles**](https://dockerlabs.es/#/): Son ideales para principiantes y cubren una amplia gama de técnicas y vulnerabilidades comunes.

## Despedida
Espero que este artículo haya sido útil y que les sirva al enfrentarse al examen eJPTv2. Si tienen alguna pregunta, no duden en contactarme a través de [**LinkedIn**](https://www.linkedin.com/in/juan-rivas-sec/) o [**X**](https://x.com/r1vs3c).

¡Buena suerte en su viaje en el apasionante mundo del pentesting!

![eJPT](eJPT.jpg)