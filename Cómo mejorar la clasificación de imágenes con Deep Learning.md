# Cómo mejorar la clasificación de imágenes con Deep Learning

>**TL;DR** sigue algunas de estas ideas para mejorar tus modelos de clasificación de imágenes.

Existen técnicas y algoritmos de Deep Learning para clasificación de imágenes que permiten alcanzar niveles de precisión *aceptables* sin que siquiera entremos en detalle de cómo funcionan, qué parámetros reciben o detalles más finos de implementación. En este post recorreremos distintas formas prácticas de mejorar nuestros modelos cuando deseemos llevar nuestra performance al siguiente nivel.

## Contexto

Estos aprendizajes los hemos adquirido trabajando con uno de nuestros clientes cuyo objetivo es crear una red neuronal que permita clasificar el estado fenológico de una flor a partir de una imágen de la misma.

Un estado fenológico de una planta indica el nivel de madurez de la misma según criterios que afectan su medio ambiente. Esto es particularmente interesante para agricultores o productores, ya que pueden determinar datos importantes tales como cuánto falta para cultivar, si hay enfermedades o si hay anomalías en los frutos.

![Estados fenológicos de una flor](https://github.com/marcelofelman/case-studies/blob/master/images/1-estados-fenologicos.png?raw=true)

*Imagen cortesía de Wikipedia. [Wikipedia](https://wikipedia.com)*

El ejemplo anterior se trata de una clasificación de múltiples clases. Algunas resultan más fáciles de distinguir, mientras que otras a simple vista parecen similares e inclusive difíciles de distinguir para un experto del dominio.

Si bien basaremos este ejemplo sobre estados fenológicos, ten en cuenta que estas prácticas aplican a cualquier tipo de clasficicación de imágenes.

## Tecnologías

Para este proyecto contamos con:

- Un conjunto de datos de aproximadamente 1000 imágenes por cada categoría, con 7 categorías.
- Especialistas del dominio de estados fenológicos.

Las tecnologías utilizadas fueron:

- [Data Science Virtual Machine](https://docs.microsoft.com/en-us/azure/machine-learning/data-science-virtual-machine/provision-vm)
- [Jupyter Notebooks](http://jupyter.org)
- [Anaconda Python](https://anaconda.org/anaconda/python)
- [Keras](https://keras.io)
- [CNTK](https://www.microsoft.com/en-us/cognitive-toolkit/)
- [TensorFlow](http://tensorflow.org)
- [Node.js](https://nodejs.org/en/)

## Ideas sobre cómo mejorar

En un contexto inicial, se lograba entre un 70 y 80% de precisión. Estas son algunas de las ideas que tuvimos para mejorar aquella precisión:

- [Probar distintos algoritmos](#probar-distintos-algoritmos)
- [Mejorar los datasets de entrenamiento](#mejorar-los-datasets-de-entrenamiento)
- [Re-pensar las clases](#re-pensar-las-clases)
- [Crear un meta-modelo](#crear-un-meta-modelo)

## Probar distintos algoritmos

Una de las maneras más intuitivas de tratar de mejorar es probando con nuevos algoritmos, redes o técnicas. 

Inicialmente, el modelo estaba basado en una red neuronal convolucional (CNN o ConvNet) creada con Keras. Más info [aquí](https://keras.io/layers/convolutional/). Una de las grandes ventajas de utilizar Keras, es que nos abstraemos enormemente de la implementación de los algoritmos. Además, podemos elegir entre Tensorflow, CNTK o Theano como motor de nuestras redes.

Si no conoces Keras y estás empezando con Deep Learning, es altamente recomendado. La sintaxis le facilita muchas tareas al desarrollador. Sólo un ejemplo:

```python
# Compile model
model.compile(loss='binary_crossentropy', optimizer='adam', metrics=['accuracy'])
```

Alguno de los algoritmos, redes o técnicas que probamos:

### One-shot learning

- [One-shot learning con Keras](https://sorenbouma.github.io/blog/oneshot/)
- [One-shot learning con Theano](https://github.com/tristandeleu/ntm-one-shot)
- [One-shot learning con Tensorflow](https://github.com/AntreasAntoniou/MatchingNetworks)

One-shot learning es una técnica que apunta a crear modelos a partir de conjuntos de datos mucho más reducidos que los típicos. A grandes rasgos, la intención del algoritmo es emular la extracción de características como hacen los seres humanos, los cuales, por ejemplo, no necesitan ver millones de imágenes de un perro para reconocer que se trata de uno de ellos.

Notarás que los links anteriores tienen un nivel considerable de complejidad. En nuestro caso, logramos implementar varios de estos modelos sin llegar al detalle fino sobre el comportamiento de cada parte. No es lo más recomendable, pero si lo más práctico.

### AlexNet

- [AlexNet con Keras](https://github.com/duggalrahul/AlexNet-Experiments-Keras)

AlexNet es una red neuronal convolucional que ha recibido múltiples premios a nivel internacional por su performance en clasificación de imágenes. El artículo original puede encontrarse en [este link](http://vision.stanford.edu/teaching/cs231b_spring1415/slides/alexnet_tugce_kyunghee.pdf).

AlexNet es otro de esos casos en los que podemos hacer pruebas sin bajar al detalle fino detrás de la red, y tomarlo como una caja negra.

Desafortunadamente para nuestros casos, nuestros niveles de precisión no mejoraron. En un aspecto más positivo, nuestros tiempos de entrenamiento se vieron reducidos en un 50%.

>**Tip práctico** Intenta distintas redes, algoritmos o técnicas aún si tengamos que tratarlos como "cajas negras".

## Mejorar los datasets de entrenamiento

Algunas de las categorías resultaban difíciles de discernir inclusive para los expertos del dominio. Una buena idea es consultar con varios especialistas cuál es su *input* sobre una determinada imagen.

Para ello, decidimos construir una aplicación web liviana que permitiera la clasificación de imágenes. El comportamiento es muy simple: muestra una imagen y permite elegir una categoría.

![Aplicación de clasificación de imágenes](https://github.com/marcelofelman/case-studies/blob/master/images/2-app-clasificacion.png?raw=true)

En nuestro caso optamos por crearla para satisfacer una necesidad bien particular, pero ten presente que existen muchísimas herramientas para anotar o clasificar imágenes.

[Wikipedia ofrece una lista de más de una docena de herramientas.](https://en.wikipedia.org/wiki/List_of_manual_image_annotation_tools)

Recuerda que los conjuntos de entrenamientos son la parte más importante de tu aplicación, un error puede ser muy costos en la precisión del modelo.

## Re-pensar las clases

De todos los puntos que -para nuestro caso- trajeron un mayor grado de precisión, volver a pensar las clases fue el que mayor impacto tuvo.

¿Por qué 7 categorías y no menos? ¿Tiene algún sentido desde el punto de vista del negocio?

Esas dos preguntas permitieron re-pensar nuestro enfoque de manera completamente distinta. Para ello, en resumidas cuentas, empleamos un algoritmo de entrenamiento no supervisado llamado *k-means* que tiene por objetivo encontrar relaciones o grupos entre los datos.

Este algoritmo nos dio como resultado que existían idealmente 5 clases en lugar de 7. De esta forma, pudimos re-catalogar nuestra información para lograr mejor precisión. Recuerda: ante menos clases, mayor probabilidad de acertar correctamente.

Al ser este caso tan importante, creamos un documento independiente a este. Puedes ver más información [aquí](completar con el link de clustering).

## Crear un meta-modelo

Un meta-modelo es básicamente un modelo constituído por distintos modelos. Esto tiene una gran ventaja, que es que pudimos crear una red neuronal "generalista" que es relativamente buena en identificar las clases. Por otra parte, podemos crear otras redes neuronales que son muy buenas en "desempatar" o en clasificar dos estados difíciles de entender.

Un ejemplo: si observas la figura c y d del ejemplo de arriba, notarás que son imágenes muy similares. Nuestro modelo tal vez pueda ser muy bueno en detectar las diferencias entre a y d, pero no tanto para c y d. Entonces, creamos otra red que sea muy buena clasificando entre esos dos ejemplos.

A nivel pseudo-código, funciona de la siguiente forma:

-> Nueva imagen -> Consultar con la red neuronal general

    -> Si la precisión es baja para un estado "difícil"

        -> Consultar con la red experta en desempatar estados difíciles
        
    -> Sino, me quedo con el resultado

Esta última parte no tiene tanto que ver con lógica de ciencia de datos, sino más bien se trata de un *hack* de programadores 😀.

## Resultados y conclusiones

Tras estos cambios, nuestro modelo logró una mejor precisión total para la clasificación de imágenes. Dada la sensibildad de los casos de nuestro cliente, no podemos compartir los números exactos.

Puntualmente, cada sección resultó de distintas maneras:

- One-shot learning: es complejo de implementar y la performance resultó similar a una CNN.
- AlexNet: la performance resulto similar a una CNN, pero los tiempos de entrenamiento se redujeron.
- Clustering: pieza importantísima que permitió, sin cambios en el código, enfocar el problema de otra forma.
- Revisión de etiquetas: siempre ayuda refinar el conjunto de datos
- Meta-modelo: aumenta los costos y la complejidad favoreciendo la performance y precisión.

El aprendizaje más importante que obtuvimos en este escenario es que, inclusive en Deep Learning, *no todos los problemas se resuelven con redes más complejas, más parámetros, algoritmos más sofisticados o más GPU*. A veces, todo lo que necesitas es dar un paso atrás y ver tus problemas desde otra óptica.

## Equipo

Jorge Cupi - [Twitter](https://twitter.com/jorgecupi) - [GitHub](https://github.com/jorgecupi)

Marcelo Felman - [Twitter](https://twitter.com/mfelman) - [GitHub](https://github.com/marcelofelman)