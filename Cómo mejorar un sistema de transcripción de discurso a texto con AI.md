# Cómo mejorar un sisema de transcripción de discurso a texto con AI
## Introducción
Una opción para convertir audio a texto, es a través de los [Servicios Cognitivos de Microsoft](https://azure.microsoft.com/es-mx/services/cognitive-services/). 

## Technical Challenge
> ¿Por qué buscaron el apoyo de CSE?
## Miembros del equipo
## Alcance/Objetivos
## Tecnologías utilizadas

- [Custom Speech Service (CRIS)](https://cris.ai/) adaptado con [Custom Models](https://azure.microsoft.com/en-us/services/cognitive-services/custom-speech-service/)
- [Audio Converter for Custom Acoustic Model](https://github.com/vianeyja/AudioConverter)
- [Service Bus Queues](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-dotnet-get-started-with-queues)
- [Visual Studio 2017](http://www.visualstudio.com/vs)
- Una subscripción de Azure

## Arquitectura de la solución

El siguiente diagrama muestra los principales elementos que intervienen en la arquitectura propuesta para este proyecto.


![Arquitectura general](https://github.com/dgregoraz/case-studies/blob/master/images/cs-1/architecture.png?raw=true)


### CRM
El CRM provee información básica sobre el audio, tanto del gestor como del cliente, el objeto de la llamada, la campaña relacionada cuando aplica, etc.

Habitualmente cierta información demográfica del cliente es validada en la conversación por lo que expresiones como el nombre, el documento o cédula, fecha de nacimiento, etc. podrían ser auditadas o corregidas en la transcripción.

>Desafortunadamente entrenar un Custom Language previo a cada llamada para mejorar el accuracy de la transcripción no es viable por razones pragmáticas de implementación.

### PBX
La central telefónica provee dos archivos con la conversación, uno por cada canal, inbound y outbound. Los archivos se generan en formato .mp3, Mono, a 11025 Hz, 16 bits, PCM.

### Message Broker
La mejor solución para este escenario es dejar que las aplicaciones envíen y reciban mensajes a través de una cola unidireccional. En un principio casi todas las acciones son ejecutadas sobre todos los mensajes. Si este escenario cambia, eventualmente podría implementarse un broker con un mecanismo de subscripción basado en la metadata provista por el CRM para diferenciar los diferentes procesamientos que se ejecutan sobre los mensajes en el pipeline.

Dado que el análisis de los audios es una operación asincrónica, el Message Broker también cumple la función de buffer para manejar los picos de carga durante los horarios de mayor cantidad de llamadas.

Todos los sistemas que interactúan con la cola son receivers and senders, excepto la central telefónica que es sólo sender. Como receivers los sistemas están aguardando un mensaje, lo leen, realizan alguna operación y, como sender, lo devuelven a la cola para que continue su flujo. A continuación se describen cada uno de los sistemas o componentes que intervienen en el proceso de transcripción.

### Pre-processing
La primera acción a realizar en los audios es la adaptación y mejoramiento para aumentar la precisión de su transcripción.  Este pre-procesamiento implica:
- Convertir el formato de los audios de mp3 a wav y llevar su frecuencia a 16 kHz.
- Aplicar filtros de reducción de ruido.
- Normalizar el audio (se lleva al máximo el pico de amplitud de los tracks).
- Se agregan unos 100 milisegundos de silencio al principio y al final del audio.
- Otras mejoras de ecualización y aplicación de efectos.

El preprosesamiento se realiza con los audios separados por canales, lo cual resulta mucho más eficiente que hacerlos en dual-channel (stereo).

### Transcription
Esta aplicación fragmenta los audios e invoca al servicio CRIS a través del endpoint desplegado con el Custom Model preparado para este propósito.

Esta operación añade a cada audio el texto relacionado de su transcripción en pequeños fragmentos que provee CRIS. Estos fragmentos o párrafos contienen metadata sobre su duración, offset y nivel de confianza.

>En esta etapa se añade también información provista por el CRM y la PBX, como ser el tipo e identificación del speker (gestor/cliente), fecha y hora de la llamada, la campaña reacionada, etc.

### Sync
El propósito de la sincronización es convertir las transcripciones de ambos audios en una única conversación. La metadata de ambos mensajes son fusionados y los textos identificados son puestos en forma de diálogo.

### Merge & Store
Por razones de auditoría los audios deben conservarse en dual-channel por lo que es necesario volver a integrarlos y convertirlos a formato mp3, ahora en stereo.

### Analytics
Estas reglas pueden aplicarse todas juntas o concebirse como agentes independientes de la cola. Cada regla analiza el mensaje y anexa su resultado. Ejemplo de estas reglas pueden ser:
- Analisis sentimental.
- Búsqueda de expresiones agraviantes.
- Medir el ratio de conversación de cada speaker, por ejemplo 70% vendedor, 30% cliente.
- Validar ciertas expresiones que deben ser mencionadas por el gestor de acuerdo a la campaña, por ejemplo "crédito sujeto a aprovación crediticia en la sucursal".
- Etc.

Dado que estas reglas se ejecutan, dependiendo del estado de la cola, en near-realtime, algunas de estas reglas podrían producir alertas tempranas de acuerdo a la severidad de su análisis.

Finalmente el archivo de audio dual-channel, la transcripción sincronizada y los insights generados por las reglas son persistidos en un data warehouse.
## Hallazgos
> Pueden ser hallazgos como el caso de globo studio en el que se simplificó el modelo, etc.
## Conceptos clave
> Un listado de pasos o configuraciones que sean importantes para el cliente/parter recordar
## Siguientes pasos 
## Referencias o Recursos
