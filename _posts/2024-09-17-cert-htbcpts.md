---
layout: post
toc: true
title: "HTB - Certified Penetration Testing Specialist (CPTS) Review 2024"
categories: Certs
tags: [Certification, Active Directory, Windows, Pivoting, Linux, Privilege Escalation]
author:
  - H4RRIZN
---

Hola a todos! luego de meses largos y agitados intentando complementar el trabajo, los estudios y los hobbies puedo decir que soy Certified Penetration Testing Specialists (CPTS) de Hack The Box. 🫡

![certpwned](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CERTIMG/cpts%20aproved.webp)

A continuación, comparto mi experiencia con esta certificación, abordando desde la preparación hasta la valoración final.
## Preparación

Para poder rendir el examen, es necesario completar el PATH de [Penetration Tester](https://academy.hackthebox.com/paths/jobrole).
![pentestpath](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CERTIMG/Penetration%20Tester%20Path.png)

Sobre el material proporcionado, para mí es un 10 de 10. El contenido es directo y no se da vueltas innecesarias en aspectos que podemos profundizar por nuestra cuenta. Para mi esto fue crucial ya que me permitió concentrarme en los conceptos clave. Aún si tienes experiencia estoy seguro que aprenderás cosas nuevas incluso si creías que ya dominabas algunos conceptos de los módulos del path.

Además, el path es amigable para quienes están comenzando, ya que enseña desde conceptos básicos hasta avanzados, y los módulos se actualizan constantemente. Completar el PATH es prácticamente una obligación si deseas obtener la certificación; sin embargo, también lo recomiendo a aquellos que simplemente quieran aprender o afianzar conocimientos.

En cuanto a mi preparación, Tengo la fortuna de trabajar en el área de seguridad ofensiva y no he dejado de estudiar TTPs, nuevos ataques, herramientas, etc, desde hace un buen par de años lo que significa que he estado constantemente capacitandome. Esto me ayudó a sentirme fresco con la terminal. No quiero decir que se necesiten años de experiencia para estar listo y mucho menos estar trabajando en el campo de infosec, pero es fundamental familiarizarse con todo el contenido presentado en los módulos del PATH. 

En mi caso, ya tenía experiencia en la mayoría de los módulos, excepto en Active Directory, por lo que reforcé esta área realizando máquinas de Active Directory de Hack The Box y tomando notas del módulo ["Active Directory Enumeration & Attacks"](https://academy.hackthebox.com/module/details/143) de HTB Academy. Para quienes estén comenzando o tengan más "debilidades", es útil realizar máquinas relacionadas con estos módulos. HTB Academy tiene una [herramienta poderosa](https://academy.hackthebox.com/academy-lab-relations) para facilitar este trabajo; por ejemplo, si necesitamos reforzar Pivoting, seleccionamos el módulo y se despliegan las máquinas relevantes:
<video src="https://github.com/H4RRIZN/H4RRIZN.github.io/raw/refs/heads/master/_includes/CERTIMG/academyxlabs.mp4" width="500" height="240" controls></video>

El consejo que daría a quienes se están preparando para el examen es tener un enfoque estructurado y disciplinado. Es importante desarrollar la habilidad de administrar tu tiempo para trabajar de forma eficiente. Estudia de manera balanceada, duerme lo suficiente y toma descansos para mantenerte en óptimas condiciones durante la preparación y durante el transcurso del examen.

Asegúrate de conocer los temas y habilidades según el temario del examen. Aunque la práctica es crucial, no subestimes la importancia de la teoría. Un sólido entendimiento de los conceptos es esencial, así que toma buenas notas y crea tu propia CheatSheet.

## Mi experiencia general
La verdad fue bastante desafiante para mí, ya que Active Directory no es mi día a día, pero logré certificarme con 100 puntos. En este último tiempo, he estado fortaleciendo mi conocimiento en infraestructura y redes, así como en Active Directory. Diría que no fue sencillo; sentí que el tiempo me presionaba constantemente. Casi sentí que estaba en una carrera contra el reloj. A pesar de la presión, la experiencia fue buena, ya que el laboratorio se mantuvo estable en todo momento.
![100points](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CERTIMG/exam%20feed.png)

Mi expectativa antes de comenzar era encontrarme con una cadena de ataque de las que solo Hack The Box sabe hacer. Si has tocado máquinas CTF en la plataforma de Hack The Box, sabrás a lo que me refiero. Me refería a una cadena de ataque en la que muchas veces tienes que pensar fuera de la caja. Sabía que sería un examen desafiante, el cual superó mis expectativas.

En cuanto a la dificultad del examen, creo que realmente dependerá de cuán afianzados tengas los conceptos presentados. Mi talón de Aquiles era Active Directory, por lo que enfoqué mi preparación en este aspecto (reflejado en mis últimos writeups). Sin ánimo de hacer **spoiler**, algunas explotaciones no eran complejas, más bien un poco rebuscadas; no a nivel descabellado, pero sí requerían pensar un poco fuera de la caja. 

Sin duda, no es un examen sencillo. Utilicé los 10 días disponibles para el desarrollo del examen y del informe. Tomando mis descansos obviamente. A pesar de tener experiencia en CTFs, certificaciones y trabajo en el sector, no subestimaría la dificultad de esta certificación. Es desafiante y requiere preparación. El examen fue una cadena de ataque compleja y emocionante basado en un proyecto real. Por lo que a pesar de sentir que podia encontrar una **missconfiguration** en cualquier parte no senti la sensación de una dificultad artificial forzada en el transcurso del examen.

Finalmente mi reporte contuvo un total de 170 paginas, en mi caso utilice el template proporcionado por Hack The Box. Y luego de enviar el informe y esperar un par de semanas obtuve el certificado. 

![cert](https://raw.githubusercontent.com/H4RRIZN/H4RRIZN.github.io/refs/heads/master/_includes/CERTIMG/CPTS.png)

## Conclusión

Para concluir, me gustaría realizar un pequeño PROS vs CONTRAS del examen.
Pros:
- El examen es asequible y el contenido es excelente para su costo.
- Soporte prácticamente instantáneo; en mi caso, siempre estaban ahí cuando necesitaba ayuda con algun problema durante el desarollo de los laboratorios de los modulos. La eficiencia de las respuestas me sorprendió gratamente.
- El laboratorio se mantuvo estable en todos los ataques, y no experimenté pérdidas de conexión en ningún momento.

Contras:
- Puede sonar contradictorio dado el último PRO mencionado, pero hubo un punto en el examen en el que tuve que reiniciar el laboratorio más de tres veces. Esto me frustró un poco, ya que no sabía si estaba ante un rabbit hole o si mis payloads no funcionaban. Sin embargo, luego de un reinicio y probando el mismo ataque dio frutos. Por lo que el único contra del examen seria ese.

La certificación CPTS de Hack The Box es una excelente oportunidad para quienes buscan desafiarse y validar sus habilidades en pentesting. Sin duda el valor del contenido y la calidad de la enseñanza son indiscutibles.

Gracias por llegar hasta acá, Happy Hacking! 🏴‍☠️

