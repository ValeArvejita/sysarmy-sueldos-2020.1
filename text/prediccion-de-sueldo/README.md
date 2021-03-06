# Predicción de sueldo

Vamos a ver cómo crear un modelo de Machine Learning que pueda determinar el
sueldo de una persona según información personal y laboral. El código está
disponible en
[github](https://github.com/seppo0010/sysarmy-sueldos-2020.1/blob/master/notebook/Predicci%C3%B3n%20de%20sueldo.ipynb),
en este texto vamos a exponer el razonamiento y explicar las técnicas
utilizadas.

## Origen de datos

La información que vamos a usar como base es la [encuesta de sueldos de sysarmy
2020.1](https://sysarmy.com/blog/posts/resultados-de-la-encuesta-de-sueldos-2020-1/).
Esta encuesta se realiza cada seis meses anónimamente y luego los microdatos
(cada respuesta recibida) son publicados. Las respuestas corresponden al
período 9/12/2019 - 3/2/2020, y corresponden a 5.982 personas.

Para obtener información sysarmy crea un formulario y publica el link en
redes sociales y grupos de usuarios de tecnologías relacionadas. Por ello hay
que considerar que las respuestas son de una población _autoseleccionada_, y
por lo tanto no representan a toda la población de IT de Argentina. Cada vez
que lleguemos a una conclusión tenemos que tener en cuenta que no se puede
generalizar sino que como mucho se puede apreciar una tendencia.

También tenemos que tener cuidado con la calidad de la información. Como el
formulario es abierto para que cualquier persona ingrese los datos que quiera,
no se puede estar seguro de la calidad de los mismos. Por ejemplo en esta
edición alguien respondió que tiene 1.555.555.555.555.555.555.555.555 empleados
a cargo, un número claramente exagerado. Sin embargo puede haber otros casos
donde no se detecten tan fácilmente los problemas, o incluso que la información
sea verosímil y no sea posible de diferenciar.

Los datos corresponden a los meses de diciembre de 2019 y de enero de 2020
en Argentina, meses en los cuales la inflación fue de
[3,7% y 2,3%](https://www.indec.gob.ar/uploads/informesdeprensa/ipc_06_201F5D8F36A1.pdf)
respectivamente. Esto puede traer variabilidad en los números porque no tenemos
la fecha de cada registro para normalizar los valores si hubo ajustes de
sueldo.

## Detección de anomalías

El primer paso, considerando el origen de los datos, va a ser "limpiar" la
información. Queremos eliminar valores extremos que nunca serían posibles,
números que un humano encuestando no hubiese aceptado. Para eso podemos
graficar cada columna y buscar números que no tengan sentido.

Un par de ejemplos:

![Gráficos de salario; (a) sin excluir anomalías todos
los valores se concentran en torno a 0, y la escala llega hasta 7.000.000.000
(b) hasta $1.000.000 la mayoría está entre $0 y $200.000, achicandose
rápidamente hasta casi no haber datos cerca de $1.000.000 (c) hasta $10.000
tiene la gran mayoría en torno a 0 (d) entre $10.000 y $1.000.000 la mayoría
está entre $0 y $200.000, achicandose rápidamente hasta casi no haber datos
cerca de $1.000.000](anomalies-salary.png)

En el gráfico (a) vemos que alguien respondió que ganaba alrededor de
$7.000.000.000.000 de pesos por mes. Si eliminamos ese dato, que claramente no
es válido, obtenemos el gráfico (b). Pero, además, vemos que muchas otras
personas pusieron que ganaban $1, quizás personas desempleadas que igual
querían participar de la encuesta (gráfico (c)). Si también eliminamos esos
valores, entonces obtenemos el gráfico (d).

![Gráfico de años en la empresa actual (A) sin
excluir anomalías todos los resultados están en torno a 0 y la escala sobrepasa
los 2.000 (B) hasta 40 concentra los valores en menos de 4 y baja rápidamente,
hasta 40](anomalies-years-in-co.png)

Por otra parte, en el gráfico (e) una persona puso que tenía alrededor de 2000
años en la misma empresa, probablemente alguien que malinterpretó la pregunta
como desde qué año estaba en la empresa actual. Si eliminamos ese dato,
obtenemos el gráfico (f).

Una vez hecha esta "limpieza", tenemos los datos con los cuales vamos a
trabajar para construir nuestro modelo.

## Evaluación de modelo

El modelo va a recibir información de una persona y con eso va a determinar un
sueldo. Lo primero que debemos determinar es cómo vamos a evaluar cuán bueno
un modelo es. Lo que vamos a usar para esto es el [coeficiente de determinación
(o r2)](https://es.wikipedia.org/wiki/Coeficiente_de_determinaci%C3%B3n). Esto
nos va a dar un número no mayor a 1 que nos dice, básicamente, qué proporción
del sueldo se puede explicar con el modelo. Un modelo que siempre devuelva el
mismo valor va a tener un r2 de 0 porque no explica nada, uno que sea muy malo
(aumente su predicción cuando debería disminuir) va a dar negativo, y mientras
más cercano a 1 estemos mejor será nuestro modelo.

Hay que recordar que la inflación total del tiempo en el que se realizó la
encuesta fue del 6,1%, lo cual aumenta la dificultad del problema.

Para evaluar los resultados vamos a usar
[validación cruzada](https://es.wikipedia.org/wiki/Validaci%C3%B3n_cruzada).
Esto consiste en dividir los datos en cinco (podría ser otro número) grupos, e
ir tomando de a cuatro para entrenar y el restante para evaluar. De esta
forma vamos a poder medir la calidad del modelo con información que nunca antes
haya visto. Por supuesto esto nos va a dar cinco valores de r2, pero podemos
promediarlos para tener una noción de la calidad. Además como cada registro va
a ser evaluado sólo una vez (en los otros cuatro casos se usa para entrenar)
podemos guardar el resultado y calcular el r2 total.

Cuando vayamos a usar el modelo para información nueva, no disponible en la
planilla, vamos a poder entrenar con todo el dataset.

### Modelo base

Sabemos que si el modelo siempre estima un valor constante (por ejemplo el
promedio de los sueldos) el r2 va a ser 0. Como queremos mejorar eso, [y basado
en trabajo previo](https://github.com/seppo0010/sysarmy-sueldos-2019.1/blob/master/notebook/Sysarmy%20-%20Predicci%C3%B3n%20de%20sueldos.ipynb),
sabemos que hay tres características que son buenas predictoras del sueldo:
el género, la provincia en que trabaja y cuántos años de experiencia tiene una
persona. Entonces un modelo sencillo que podemos construir, consiste en tomar
en cuenta sólo estas características, y tratar de ubicar la línea recta que más
cerca pase de los puntos, o sea, armar un modelo de regresión lineal.

Pero para eso tenemos que llevar cada características a un espacio lineal. Que
el espacio sea lineal significa que se puede medir la distancia entre dos puntos
y que esa misma distancia tendrá el mismo valor en otro lugar de la escala.

Tenemos datos de dos tipos: numéricos (la experiencia y el sueldo), es decir que
podríamos directamente operar con ellos, y no numéricos (el género y la
provincia), a los que vamos a tener que transformar en números de alguna manera.

Miremos primero los datos numéricos.

En cuanto a la experiencia, si bien ya tenemos números con los que podemos hacer
cuentas, podríamos pensar que no es lo mismo para el sueldo pasar de no tener
experiencia a tener 1 año, que de tener 10 a 11. La diferencia parece ser
decreciente, es decir que mientras más años de experiencia tenga una persona,
menos le va a significar, en el sueldo, la adición de uno nuevo.

![Gráfico de logáritmo de x más uno; se destacan los puntos (0, 0);
(1, 0.3); (6, 0.85); (7, 0.9)](log10.png).

Una buena manera de ajustar este comportamiento es usando una función
logarítmica porque nos permite transformar un número en otro manteniendo el
orden pero achicando la distancia con el siguiente a medida de que el número se
hace mayor. En el gráfico (g) vemos como el valor correspondiente a 1 está más
lejos del de 0 que el correspondiente al 7 respecto del de 6.

También podrímos aplicar logaritmo al salario porque no es lo mismo ganar $1.000
más para alguien que gana $10.000 que una persona que cobra $200.000.

Pasemos ahora a los datos no numéricos.

En el caso de la provincia, podríamos asignarle a cada persona el sueldo
promedio de su provincia. Tenemos que tener en cuenta que cuando recibamos
datos para estimar no vamos a saber su sueldo, pero podemos usar el promedio
que calculamos de la provincia como un indicador de los salarios de ésta.
Es decir que si el salario promedio de la Provincia de Buenos Aires y el de
Santa Fe es parecido la provincia va a aportarle un valor cercano, mientras que
si Jujuy tiene un promedio más bajo, la diferencia va a ser mayor.

Lo mismo se puede aplicar para género usando como valor el promedio de sueldo
para hombres, mujeres y otros.

Con estas transformaciones podemos aplicar validación cruzada para entrenar
cinco modelos de regresión lineal y calcular el r2 promedio resultante.
Obtenemos 0.2398. Es decir que este modelo sencillo ya puede explicar casi el
24% del sueldo de las personas.

## Agrupar regiones

La encuesta recolecta información geográfica a nivel provincia. El costo de
vida en cada provincia es bastante desparejo y los sueldos también suelen
serlo, por eso, como dijimos antes, sirve como predictor. Pero hay provincias
que tienen muy pocas respuestas y eso dificulta la generalización. Recordemos
que los datos en pequeña escala no son confiables por la forma de la encuesta
así que tratemos de agruparlos en bloques más grandes.

[
![Respuestas por provincia con una escala de 0 a poco más de 4.000. La Ciudad
Autónoma de Buenos Aires tiene 4.000, la Provincia de Buenos Aires,
Santa Fe y Córdoba rondan los 1.000, el resto está cerca del 0](mapcount.png)
Ver detalles
](mapcount.md)

No se puede apreciar mucha diferencia entre las provincias dado que la
concentración en la Ciudad de Buenos Aires distorsiona la escala. Veámoslo en
escala logaritmica.

[
![Logaritmo de cantidad de respuestas por provincia](maplogcount.png)
Ver detalles
](mapcount.md)

Vemos que hay muy pocos datos fuera de la Ciudad de Buenos Aires, la Provincia
de Buenos Aires, Santa Fe y Córdoba.

Veamos ahora los sueldos promedios en cada provincia. Esto nos puede dar una
idea de qué provincias que tengan pocos datos podrían ser unidas porque
consideramos que tienen similitudes culturales y esperamos que el
comportamiento sea semejante.

[
![Salario bruto promedio por provincia](mapsalary.png)
Ver detalles
](mapsalary.md)

Con estos dos gráficos podemos pensar en dividir a las provincias de la
siguiente forma:
* Noroeste: Catamarca, Jujuy, La Rioja, Salta, Santiago del Estero, Tucumán.
* Noreste: Chaco, Corrientes, Entre Ríos, Formosa, Misiones.
* Cuyo: Mendoza, San Juan, San Luis.
* Llanura Pampeana: La Pampa, Santa Fe, Córdoba, Provincia de Buenos Aires.
* Patagonia: Chubut, Neuquén, Río Negro, Santa Cruz, Tierra del Fuego.
* AMBA: Ciudad de Buenos Aires y Gran Buenos Aires.

[
![Salario bruto promedio y logaritmo de cantidad de respuestas por región](regions.png)
Ver detalles
](regions.md)

La decisión de cómo agrupar las provincias es en parte arbitraria y podría
hacerse distinto. Por ejemplo San Luis y La Pampa, con pocos registros, no
se parecen tanto a ninguna provincia de la zona.

## Características categóricas

Ahora que tenemos las regiones tenemos que ver cómo pasarselas al modelo.
Podemos usar el promedio como usamos antes en el modelo base, pero existen
otras alternativas que pueden servir mejor. El promedio sirve en el caso de
una regresión lineal porque mueve cualquier categoría a un punto pero con
otro tipo las opciones crecen.

En este caso vamos a usar una columna por cada región, y un valor binario.
Entonces alguien de AMBA va a tener un verdadero en esa columna y un falso en
Patagonia, Llanura Pampeana, etc.

Lo mismo podemos hacer para características que no son mutuamente excluyente,
como los lenguajes de programación que sabe una persona. Pueden seleccionarse
múltiples y esa información puede verse reflejada en la matriz.

Las características que tengan pocas ocurrencias pueden ser eliminadas dado que
no proveen información para que el modelo pueda aprender.

Entonces una persona de la Ciudad de Buenos Aires, Hombre, que usa Java y
JavaScript en su trabajo será representado de la siguiente manera:

|Me identifico=Hombre|Me identifico=Mujer|region=AMBA|region=Pampa|region=Patagonia|...|Lenguajes de programación=javascript|Lenguajes de programación=java|Lenguajes de programación=rust|
|--|--|--|--|--|--|--|--|--|
|1|0|1|0|0|...|1|1|0|

## El modelo final

Usando XGBoost, un modelo basado en árboles de decisión que suele tener buenos
resultados, y con las técnicas mencionadas anteriormente, obtenemos un r2
promedio de 0.5149. Todavía se puede mejorar este modelo, por supuesto.

## Pendiente

Queda pendiente analizar cuán justo es el modelo y si se puede hacer una
variante mejor en ese aspecto. Por ejemplo una de las características que
usamos fue género y el modelo puede usar esta información para perjudicar a un
grupo. Sacarla no necesariamente es suficiente para solucionar este problema.

El modelo puede mejorarse de incontables formas. Para enumerar algunas de ellas:

* Describir mejor las tecnologías. Hoy en día cada lenguaje, framework, _stack_
está sólo descripto por su nombre. Sin embargo en la práctica algunas se
parecen entre ellas más que otras. Por ejemplo Microsoft Azure es más parecido
a Amazon Web Services que a Solaris. Este tipo de relaciones que conocemos
puede reflejarse en los datos pero como la muestra es relativamente pequeña no
ser suficiente, así que usar una fuente externa que informe al modelo sobre
las tecnologías puede mejorar los resultados.

* Selección de características. Casi toda la información brindada fue dada al
modelo para que establezca las relaciones. Con un poco de trabajo se pueden
quitar las menos relevantes y esto puede ayudar a encontrar mejores inferencias.

* Búsqueda más profunda de anomalías. Se podría usar el modelo actual para
encontrar casos sospechosos de ser anómalos y candidatos a exclusión.
