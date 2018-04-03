# Cómo mejorar un sistema de transcripción de audio a texto con IA

## Introducción

Convertir audio a texto no es una tarea fácil, y mucho menos en Español. En este post veremos algunas de las alternativas y formas de mejorar un sistema de transcripción basado en inteligencia artificial.

Dado que los sistemas de transcripción requieren de un conjunto de datos fonéticos de entrenamiento muy amplio, lo más práctico es hacerlo forma "manejada" por un tercero detrás de un servicio, tales como los [Servicios Cognitivos de Microsoft](https://azure.microsoft.com/es-mx/services/cognitive-services/). En algunos escenarios, nos podemos encontrar que el servicio de [Bing Speech](https://azure.microsoft.com/es-mx/services/cognitive-services/speech/) no reconoce algunas de nuestras palabras, y no tenemos manera de retroalimentar sobre la transcripción obtenida.

Por este motivo, decidimos personalizar el servicio [Custom Speech Service](https://azure.microsoft.com/es-mx/services/cognitive-services/custom-speech-service/), incorporando nuestra propia data de entrenamiento y de esta forma mejorando los resultados.

## Desafíos técnicos

El reto principal en este escenario es adaptar un modelo que pueda reconocer el idioma español con acento de Uruguay, y reconocer palabras específicas del sector financiero (en este caso) con audios tomados de llamadas de un centro telefónico; aunque encontramos más retos durante la ejecución:

* Ruido de fondo de llamadas telefónicas, tanto de los operadores del banco, como de los clientes.
* Voces encimadas.
* Uso de modismos.
* Velocidad de habla muy rápida.

## Objetivos

* Generar un sistema de transcripción acústica con resultados "aceptables" (>80%).
* Probar las funcionalidades y exactitud de [CRIS.ai](http://cris.ai) para el idioma español.

## Tecnologías utilizadas

* [Custom Speech Service (CRIS)](https://cris.ai/) adaptado con [Language Models](https://azure.microsoft.com/en-us/services/cognitive-services/custom-speech-service/)
* [Audio Converter for Custom Acoustic Model](https://github.com/vianeyja/AudioConverter)
* [Service Bus Queues](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-dotnet-get-started-with-queues)
* [Visual Studio 2017](http://www.visualstudio.com/vs)
* [Microsoft Azure](https://azure.microsoft.com/en-us/)

## Arquitectura de la solución

El siguiente diagrama muestra los principales elementos que intervienen en la arquitectura de este proyecto.

![Arquitectura general](https://github.com/dgregoraz/case-studies/blob/master/images/cs-1/architecture.png?raw=true)

### CRM

El CRM provee información básica sobre el audio, tanto del gestor como del cliente, el objeto de la llamada, la campaña relacionada cuando aplica, etc.

Habitualmente cierta información demográfica del cliente es validada en la conversación por lo que expresiones como el nombre, el documento o cédula, fecha de nacimiento, etc. podrían ser auditadas o corregidas en la transcripción.

>Desafortunadamente entrenar un Custom Language previo a cada llamada para mejorar la exactitud de la transcripción no es viable por razones pragmáticas de implementación.

### PBX

La central telefónica provee dos archivos con la conversación, uno por cada canal, *inbound* y *outbound*. Los archivos se generan en formato .mp3, Mono, a 11025 Hz, 16 bits, PCM.

### Message Broker (agente de mensajería)

Una de las soluciones para este escenario es dejar que las aplicaciones envíen y reciban mensajes a través de una cola unidireccional. En un principio, casi todas las acciones son ejecutadas sobre todos los mensajes. Si este escenario cambia, eventualmente podría implementarse un *broker* con un mecanismo de subscripción basado en la metadata provista por el CRM para diferenciar los diferentes procesamientos que se ejecutan sobre los mensajes en el *pipeline*.

Dado que el análisis de los audios es una operación asincrónica, el *Message Broker* también cumple la función de buffer para manejar los picos de carga durante los horarios de mayor cantidad de llamadas.

Todos los sistemas que interactúan con la cola son *receivers* y *senders*, excepto la central telefónica que es sólo sender. Como receivers los sistemas están aguardando un mensaje, lo leen, realizan alguna operación y, como sender, lo devuelven a la cola para que continue su flujo. A continuación se describen cada uno de los sistemas o componentes que intervienen en el proceso de transcripción.

### Pre-processing (pre-processamiento)

La primera acción a realizar en los audios es la adaptación y mejoramiento para aumentar la precisión de su transcripción.  Este pre-procesamiento implica:

* Convertir el formato de los audios de mp3 a wav y llevar su frecuencia a 16 kHz.
* Aplicar filtros de reducción de ruido.
* Normalizar el audio (se lleva al máximo el pico de amplitud de los tracks).
* Se agregan unos 100 milisegundos de silencio al principio y al final del audio.
* Otras mejoras de ecualización y aplicación de efectos.

El pre-prosesamiento se realiza con los audios separados por canales, lo cual resulta mucho más eficiente que hacerlos en *dual-channel* (stereo).

La separación de canales, de forma gráfica, puede verse en la siguiente imagen:

![Espectro de sonido](https://github.com/marcelofelman/case-studies/blob/master/images/3-sound-waves.png?raw=true)

Tal como se puede apreciar, cada uno de los oradores *mayormente* habla mientras el otro está en silencio. De esta forma, reducimos completamente el problema de solapamiento. La complejidad está luego en re-sincronizar los audios.

### Transcripción

Esta aplicación fragmenta los audios e invoca al servicio `CRIS` a través del endpoint desplegado con el *Custom Model* preparado para este propósito.

Esta operación añade a cada audio el texto relacionado de su transcripción en pequeños fragmentos que provee CRIS. Estos fragmentos o párrafos contienen metadata sobre su duración, offset y nivel de confianza.

>En esta etapa se añade también información provista por el CRM y la PBX, como ser el tipo e identificación del speker (gestor/cliente), fecha y hora de la llamada, la campaña reacionada, etc.

### Sync

El propósito de la sincronización es convertir las transcripciones de ambos audios en una única conversación. La metadata de ambos mensajes son fusionados y los textos identificados son puestos en forma de diálogo.

### Merge & Store (unir y almacenar)

Por razones de auditoría los audios deben conservarse en dual-channel por lo que es necesario volver a integrarlos y convertirlos a formato mp3, ahora en stereo.

### Analytics

Estas reglas pueden aplicarse todas juntas o concebirse como agentes independientes de la cola. Cada regla analiza el mensaje y anexa su resultado. Ejemplo de estas reglas pueden ser:

* Analisis sentimental.
* Búsqueda de expresiones agraviantes.
* Medir el ratio de conversación de cada speaker, por ejemplo 70% vendedor, 30% cliente.
* Validar ciertas expresiones que deben ser mencionadas por el gestor de acuerdo a la campaña, por ejemplo "crédito sujeto a aprovación crediticia en la sucursal".
* Etcétera

Dado que estas reglas se ejecutan, dependiendo del estado de la cola, en near-realtime, algunas de estas reglas podrían producir alertas tempranas de acuerdo a la severidad de su análisis.

Finalmente el archivo de audio dual-channel, la transcripción sincronizada y los insights generados por las reglas son persistidos en un data warehouse.

## Entrenar el modelo de Custom Speech

Partiremos del modelo base `es-ES` para adaptarlo a los modismos, tonos y acentos de Uruguay.

### Language Model (modelo de lenguaje)

Un `language model` en términos prácticos es un conjunto de enunciados. Estos enunciados, deben ser aquellos que aparecen frecuentemente en mis discursos y que son importantes de identificar.

Proveyendo decenas, cientos o tal vez miles de ejemplos de enunciados, le estoy enseñando a mi procesador de audio que esa palabra es importante para detectar, generando una suerte de *sesgo* hacia esa identificación.

Supongamos el siguiente enunciado:
`¿Cuál es tu número de cédula?`

Un ejemplo de `language model` puede ser el siguiente cuerpo de enunciados:

* cual
* cual es tu numero
* cedula
* cual es tu numero de cedula
* cedula

>Nota: los anteriores enunciados no tienen mayúsculas, acentos ni signos de exclamación ya que están [normalizados](https://docs.microsoft.com/en-us/azure/cognitive-services/custom-speech-service/customspeech-how-to-topics/cognitive-services-custom-speech-transcription-guidelines#inverse-text-normalization).

En este caso, la palabra cédula es importante de identificar. Al mismo tiempo, existen muchas palabras que suenan parecidas a cédula, tales como:

* Célula
* Crédula
* Médula

Lo que estoy haciendo al proveer de un `language model`, es asegurarme de "desempatar" esos casos confusos en las palabras que son relevantes en mi dominio, en este caso `cédula`. Además, puedo incorporar la palabra cédula muchas veces, de forma tal de darle más peso que a otras palabras.

El servicio de Custom Speech actualmente permite subir datos de entranamiento para `language model` de hasta 500 MB (!) de enunciados en texto plano.

Para aprender cómo crear un `language model`, [sigue estos pasos](https://docs.microsoft.com/en-us/azure/cognitive-services/custom-speech-service/customspeech-how-to-topics/cognitive-services-custom-speech-create-language-model) (Spoiler alert: es simple).

### Acoustic Model (modelo acústico)

Los modelos acústicos son también un conjunto de enunciados, pero no sólo provistos en forma de texto sino que también en forma de audio.

Para entrenarlo, proveemos un archivo comprimido de varios audios junto con sus correspondientes transcripciones.

Ejemplo:
El audio `001.wav` contiene una grabación de alguien diciendo `Buen día, ¿cuál es tu número de cédula?`

Su transcripción deberá decir `buen día cual es tu numero de cedula` (recordemos: está normalizado).

> Al momento de crear esta guía, los modelos acústicos no están disponibles en `CRIS` para español. Puedes ver [cómo funcionan en inglés](https://docs.microsoft.com/en-us/azure/cognitive-services/custom-speech-service/customspeech-how-to-topics/cognitive-services-custom-speech-create-acoustic-model).

### Publicar y probar el modelo

Por último, estos modelos se operacionalizan a través de una interfaz `REST`. Al terminar de entrenar los modelos, se publican y están listos para consumir por cualquier cliente que tenga las credenciales necesarias. Más información sobre operacionalización [aquí](https://docs.microsoft.com/en-us/azure/cognitive-services/custom-speech-service/customspeech-how-to-topics/cognitive-services-custom-speech-create-endpoint).

## Resultados

Los resultados obtenidos fueron significativamente superiores a los de nuestras expectativas. Mientras que `Bing Speech` nos entregaba alrededor de un 20% de precisión (es decir, 1 palabra correcta de cada 5), **`CRIS` nos entregaba alrededor de un 80% de precisión** (es decir, 4 palabras correctas de cada 5).

Más precisamente, encontramos distintos niveles de error según el escenario:

| Canal     | Femenino  | Masculino | Total     |
| --------- | --------- | --------- | --------- |
| Inbound   | 84%       | 71%       | 80%       |
| Outbound  | 76%       | 64%       | 70%       |
| Total     | 81%       | 67%       | 76%       |

Tal como se puede apreciar, los `Inbound` (llamadas entrantes) performan mejor que los `Outbound`. Esto *creemos* que tiene sentido desde la lógica: cuando una persona realiza una llamada, es probable que se prepare para la misma ubicándose en un lugar con poco ruido. En cambio, cuando uno recibe una llamada puede encontrarse en una situación con mucho ruido de fondo, como por ejemplo, la calle.

Al mismo tiemo, hemos visto que `Female` (mujeres) performa mejor que `Male` (hombres). Aquí realmente no tenemos claro si se trata de un tema del modelo, un tema del dataset en particular, o algún sesgo del algoritmo. Ampliaremos.

Finalmente, vemos que los resultados mejoran *drásticamente* respecto de un servicio estándar, que no está personalizado.

## Hallazgos

A modo resumen, estos son los puntos que mayor impacto han tenido:

* El uso de `language models` cambia significativamente los resultados.
* La separación entre canales `inbound` vs `outbound` es una forma fácil de mejorar precisión.
* Se debe estar atento a los sesgos o escenarios entre distintos participantes.

## Conclusiones

Convertir audio a texto no es una tarea fácil, y mucho menos en Español. No obstante, cada vez existen más alternativas que permiten a los desarrolladores personalizar modelos a través de la incorporación de datos de entrenamiento adicionales.

Estas prácticas, junto con algunos tips de pre-procesamiento, permiten acercarnos cada vez más al nivel de precisión de sistemas que performan a niveles cercanos de la paridad humano-máquina.

## Miembros del equipo de Microsoft

* [Marcelo Felman](https://github.com/marcelofelman/)
* [Diego Gregoraz](https://github.com/dgregoraz/)
* [Vianey Juarez](https://github.com/vianeysitaa)
