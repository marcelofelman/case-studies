# Cómo mejorar la clasificación de imágenes con Deep Learning

Existen técnicas y algoritmos de Deep Learning para clasificación de imágenes que permiten alcanzar niveles de precisión *aceptables* sin que siquiera entremos en detalle de cómo funcionan, qué parámetros reciben o detalles más finos de implementación. En este post recorreremos distintas formas prácticas de mejorar nuestros modelos cuando deseemos llevar nuestra performance al siguiente nivel.

## Contexto

Estos aprendizajes los hemos adquirido trabajando con uno de nuestros clientes cuyo objetivo es crear una red neuronal que permita clasificar el estado fenológico de una flor a partir de una imágen de la misma.

Un estado fenológico de una planta indica el nivel de madurez de la misma según criterios que afectan su medio ambiente. Esto es particularmente interesante para agricultores o productores, ya que pueden determinar datos importantes tales como cuánto falta para cultivar, si hay enfermedades o si hay anomalías en los frutos.

![Estados fenológicos de una flor](https://github.com/marcelofelman/case-studies/blob/master/images/1-estados-fenologicos.png?raw=true)
*Imagen cortesía de Wikipedia. [Wikipedia](https://wikipedia.com)

El ejemplo anterior se trata de una clasificación de múltiples clases. Algunas resultan más fáciles de distinguir, mientras que otras a simple vista parecen similares e inclusive difíciles de distinguir para un experto del dominio.

Si bien basaremos este ejemplo sobre estados fenológicos, ten en cuenta que estas prácticas aplican a cualquier tipo de clasficicación de imágenes.

## Tecnologías empleadas

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

- Probar distintos algoritmos
- Mejorar los datasets de entrenamiento
- Re-pensar las clases
- Crear un meta-modelo

## Probar distintos algoritmos

Una de las maneras más intuitivas de tratar de mejorar es probando con nuevos algoritmos, redes o técnicas. 

Inicialmente, el modelo estaba basado en una red neuronal convolucional (CNN o ConvNet) creada con Keras. Más info [aquí](https://keras.io/layers/convolutional/). Una de las grandes ventajas de utilizar Keras, es que nos abstraemos enormemente de la implementación de los algoritmos. Además, podemos elegir entre Tensorflow, CNTK o Theano como motor de nuestras redes.

Alguno de los algoritmos, redes o técnicas que probamos:

### One-shot learning

- [One-shot learning con Keras](https://sorenbouma.github.io/blog/oneshot/)
- [One-shot learning con Theano](https://github.com/tristandeleu/ntm-one-shot)
- [One-shot learning con Tensorflow](https://github.com/AntreasAntoniou/MatchingNetworks)

// Hablar sobre la complejidad

## Mejorar los datasets de entrenamiento

Contar sobre la app..

## Re-pensar las clases

Contar sobre clustering..

## Crear un meta-modelo

Contar sobre redes binarias para estados coludidos

## Resultados

Contar cómo quedó..

## Conclusiones

Concluir..

## Equipo

George & Marce 