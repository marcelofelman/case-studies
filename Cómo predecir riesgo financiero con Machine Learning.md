# Cómo predecir riesgo financiero con Machine Learning #

En este caso de estudio, nos gustaría contarte cómo hicimos para identificar tempranamente clientes de una entidad financiera -específicamente tarjetahabientes- que ingresen en una situación de [default](https://en.wikipedia.org/wiki/Default_(finance)) utilizando técnicas de Machine Learning.

Puedes usar este caso como una guía para crear tu propio modelo, o simplemente leerlo para conocer la lógica y secuencia de pasos que seguimos para llegar a nuestro resultado.

**Los datos que mostramos en esta guía están obfuscados y modificados por razones de privacidad.**

## Resumen ##

En colaboración con una importante entidad financiera (*y cuyo nombre omitimos por motivos de sensibilidad*), definimos como objetivo utilizar inteligencia artificial para identificar aquellos clientes que, en base a su comportamiento, tengan una alta probabilidad de incumplir con sus obligaciones de pago de tarjeta de crédito.

Afortunadamente, tuvimos acceso a un amplio espectro de datos. Utilizamos un *dataset* de varios miles (entre 100-200, dependiendo de los tipos de filtros, más sobre eso en breve) de cuentas. **Estos datos no contienen información personal identificable sobre las personas, tal como reconoce [habeas data](https://es.wikipedia.org/wiki/Habeas_data)**.

A través de las herramientas [Azure Machine Learning](https://azure.microsoft.com/en-us/services/machine-learning/
) y [SQL Server 2016](https://www.microsoft.com/en-us/sql-server/sql-server-2016
), logramos crear diferentes modelos predictivos que permiten detectar hasta el 99% de los casos en riesgo de default.

## Contexto ##

Previo a comenzar el proyecto, definimos algunos puntos a considerar:

- Partir "desde cero", ignorando todo conocimiento previo para no sesgar el estudio con resultados anteriores.
- Utilizar datos de tarjetahabientes de un único país.
- Utilizar técnicas de Machine Learning en primera instancia, dejando Deep Learning para una siguiente fase.
- Utilizar herramientas visuales priorizando -en esta instancia- agilidad sobre flexibilidad.

La duración del proyecto fue de tres semanas, en las cuales iniciamos desde la exploración del dominio hasta la publicación de un reporte interactivo sobre los resultados.

## Software y herramientas ##

- [Azure Machine Learning](https://azure.microsoft.com/en-us/services/machine-learning/)
- [SQL Server 2016](https://www.microsoft.com/en-us/sql-server/sql-server-2016)

## Fases del proyecto ##

- [Exploración del dominio](#exploración-del-dominio)
- [Preparación de los datos](#preparación-de-los-datos)
- [Creación de modelos](#creación-de-modelos)
- [Iteraciones de mejora](#iteraciones-de-mejora)
- [Reporting](#reporting)

## Exploración del dominio ##

Antes de empezar, debemos comprender el dominio en el cual estamos trabajando. Si bien lo ideal es contar con un experto en el dominio, no siempre lo tendremos a disposición. Para esto, es importante tener en cuenta diferentes técnicas así como también ejercitar nuestro sentido común.

En este caso, dado que acordamos no sesgar el estudio, es lo mismo que decir que no tenemos un experto que nos dé *insights* tales como: *un saldo elevado se correlaciona muy bien con un riesgo de default*.

Si estás totalmente perdido y no sabes por donde empezar, te recomiendo que empieces por el buscador: busca *papers* académicos, documentos como éste, repositorios de código o similares. Nosotros tomamos como punto de partida una competencia de Kaggle, la cual puedes ver [aquí](https://www.kaggle.com/alexyung757/credit-card-ml-rpart-xgboost-tsne-attempts).

A partir de esta investigación, tomamos algunas decisiones y aproximaciones para empezar el análisis:

- Sólo miraremos las transacciones agregadas a nivel mensual. No analizaremos transacciones individuales.
- Haremos el análisis en dólares, aplicando tasas de conversación cuando sea necesario.
- Para simplificar, consideraremos moneda constante (no se trata de un contexto de alta inflación).
- Si bien buscamos identificar usuarios, el análisis granular será a nivel cuenta/tarjeta.
- Analizaremos inicialmente 12 meses de comportamiento hacia el pasado, descartaremos data de ser necesario.
- Consideraremos temporalidad o *seasonality*, es decir, entendemos que, por ejemplo, tus compras en Diciembre pueden ser distintas de las de Mayo.
- En caso de no lograr buenos resultados, cualquiera de las de arriba se podrá ajustar :).
- Ciertos datos como edad, cantidad de dependientes o país de residencia pueden ser predictores válidos.

A partir de las anteriores decisiones, iniciamos una fase de extracción de los datos (la cual, dicho sea de paso, fue muy compleja). El resultado fue un conjunto de datos de varios miles de registros, donde cada uno de ellos corresponde a una tarjeta de crédito. Los campos para cada registro fueron, resumidamente, los siguientes:

- Saldo en la tarjeta de crédito para los últimos 12 meses (es decir, 12 campos)
- Monto de pago mínimo en la tarjeta de crédito para los últimos 12 meses
- Monto de pago percibido en la tarjeta de crédito para los últimos 12 meses
- Saldo de financiamiento (en algunos países se lo conoce como cuotas) para los últimos 12 meses
- Límite de crédito de la persona
- Datos generales sobre la persona: edad, país de residencia, cantidad de dependientes

Estos campos los vamos a llamar *features*.

>**Tip:** Unir todos los datos en una única tabla o proyección en lugar de lidiar con muchos conjuntos de datos.

Por último, es muy importante identificar nuestra variable objetivo. En este caso, nuestro objetivo es determinar *default* _versus_ *no default*, por lo tanto, es importante contar con un campo que indique cuáles son los clientes que ingresaron en este estado en el último mes. Esta variable es binaria.

Resumiendo, en el conjunto de datos tenemos más de 100 *features* (campos predictores) y un *label* (variable objetivo) binaria. El formato que definimos para esto es un simple `CSV`.

>**Tip:** Para exportar de SQL Server a `CSV`, se puede hacer con [estos pasos](https://github.com/marcelofelman/case-studies/blob/master/Desercion%20escolar%20con%20Machine%20Learning.md#preparación-de-los-datos).

### Balance de la variable objetivo ###

Ya hemos exportado nuestro *dataset*, ahora es una buena idea ir explorando el contenido del mismo.

Lo primero que es importante visualizar, es el balance de nuestra variable objetivo, que hemos llamado *LABEL_DEFAULT* arbitrariamente. Podrás notar que sólo existen dos valores: uno o cero. El problema es que esto está fuertemente desbalanceado: hay cerca de un 98% de ceros mientras que 2% de unos. Veremos más adelante que hay varias opciones para solucionar esto.

![Variable objetivo](https://github.com/marcelofelman/case-studies/blob/master/images/1-variable-objetivo.png?raw=true)

### Valores extremos ###

Otro punto que puede introducir ruido en nuestro modelo son los variables extremos. Para esto, la visualización ideal son los diagramas de caja o [boxplots](https://en.wikipedia.org/wiki/Box_plot).

Con esta visualización puedes rápidamente ver qué valores extremos se encuentran. Por ejemplo, en el caso de abajo, hay una instancia que toma el valor de 274 cuando el promedio está bien cerca de 0. Algo resulta raro, veremos qué hacer con esto.

![Outliers](https://github.com/marcelofelman/case-studies/blob/master/images/2-outliers.png?raw=true)

### Valores faltantes ###

`NULL`, `Missing`, etcétera. Asegúrate de remover o manejar todo eso.

### Valores categóricos ###

En el caso de valores categóricos, es buena idea chequear que no haya categorías adicionales o faltantes a raíz de un error de escritura. Por ejemplo, con un diagrama de frecuencias para la variable sexo puedo asegurarme que haya dos categorías, que son `M` o `F`.

![Valores categóricos](https://github.com/marcelofelman/case-studies/blob/master/images/3-valores-categoricos.png?raw=true)

## Preparación de los datos ##

El siguiente paso consiste en hacer un poco de limpieza habiendo identificado los casso anteriores: remover datos que faltan, remover valores extremos o *outliers*, conversiones de moneda local a dólar, conversión de valores categóricos, y otra técnica conocida como normalización. Veamos cada uno de estos.

### Remover valores extremos o outliers ###

En AzureML Studio existe un módulo llamado [Clip Values](https://docs.microsoft.com/en-us/azure/machine-learning/studio-module-reference/clip-values). El mismo te permite definir un umbral para un valor mínimo o máximo, y luego tomar una decisión sobre el mismo que puede ser removerlo o puede ser reemplazarlo.

Para seleccionar el umbral, hay dos opciones: constante o percentil. El primer caso corresponde a un valor fijo, y todo valor absoluto que esté por debajo o por encima (según definamos) será tratado. El segundo caso, corresponde a un valor relativo. Más información sobre [percentiles](https://en.wikipedia.org/wiki/Percentile).

Para reemplazar los valores, tengo algunas alternativas: el mismo valor de umbral, la media (promedio), la mediana, o un `NULL`.

Veamos algunos ejemplos:

- Si encuentro un registro de alguien con una edad mayor a 200 años, lo quiero marcar como *null* ya que asumo que está mal.
- Aquellos con un límite de crédito por encima del percentil 98 quiero reemplazarlos por el valor del umbral.

### Remover datos que faltan ###

Aquí también existe un módulo que te permite remover valores faltantes, el mismo se llama [Clean Missing Data](https://docs.microsoft.com/en-us/azure/machine-learning/studio-module-reference/clean-missing-data). El mismo puede trabajar por registro (fila) o por columna.

Tienes varias alternativas: puedes directamente remover el registro/columna, o bien tienes varias opciones de sustitución de valores. Mira la [documentación](https://docs.microsoft.com/en-us/azure/machine-learning/studio-module-reference/clean-missing-data) para más info.

### Conversiones de moneda local a dólar ###

A veces es necesario realizar conversiones más personalizadas, y para ello tengo algunas opciones:

- `Python`
- `R`
- `SQL`

Mi preferencia personal es utilizar SQL, y para eso utilizo el módulo [Apply SQL Transformation](https://docs.microsoft.com/en-us/azure/machine-learning/studio-module-reference/apply-sql-transformation). De esta forma con una sintaxis bien simple puedo generar la multiplicación por el tipo de cambio para dolarizar.

```sql
SELECT
*
from
t1;
```

Aquí los links para conocer más sobre el módulo de [Python](https://docs.microsoft.com/en-us/azure/machine-learning/studio-module-reference/execute-python-script) y [R](https://docs.microsoft.com/en-us/azure/machine-learning/studio-module-reference/r-language-modules).

### Normalización ###

La técnica de normalización permite cambiar valores numéricos para utilizar una escala común a todos. Imagínate la siguiente situación: tienes una variable con valores entre 0 y 1 (ejemplo: "índice de riesgo"), y por otro lado tienes otra que varía entre 1000 y 15000 (ejemplo: límite de crédito). Para evitar problemas que puedan presentarse en la creación de modelos, usamos una escala común para todos.

El módulo [Normalization](https://docs.microsoft.com/en-us/azure/machine-learning/studio-module-reference/normalize-data) te permite llevar un conjunto de variables numéricas a un rango entre 0 y 1. Es muy simple y efectivo.

## Creación de modelos ##

Ahora que tenemos una mejor noción de la preparación y limpieza, empezaremos a crear nuestros modelos predictivos. Recuerda que esto es un proceso iterativo.

Para resolver este problema, existen diferentes algoritmos. Recuerda que los diferentes algoritmos son simplemente distintas formas de abordar un resultado. Algunos llegan a mejores resultados, pero esto no está garantizado. Si quieres ver el listado y ventajas de cada uno de los algoritmos, lo puedes ver [aquí](https://docs.microsoft.com/en-us/azure/machine-learning/machine-learning-algorithm-choice). Yo empezaré utilizando un *Two-class Boosted Decision Tree*, el cual es balanceado en cuanto a consumo de recursos y exactitud en sus resultados.

>**Tip:** Todavía no te preocupes tanto por el algoritmo que utilizarás. Tendrás tiempo de probar otros más adelante.

### Pruebas de correlación ###

Una buena manera de identificar qué campos pueden o no ser buenos predictores es realizando pruebas de correlación. La manera más simple de probar correlación con AzureML Studio es a través del módulo [Filter Based Feature Selection](https://docs.microsoft.com/en-us/azure/machine-learning/studio-module-reference/filter-based-feature-selection).

El módulo `Filter Based Feature Selection` te permite medir [correlación](https://en.wikipedia.org/wiki/Correlation_and_dependence) entre las distintas variables y la variable objetivo. De esta manera, podemos identificar campos irrelevantes en nuestro estudio.

Es importante considerar los siguientes aspectos:

- **Correlación no implica causalidad**. Por ejemplo, el consumo de helados está correlacionado con la cantidad de personas que mueren ahogadas (al aumentar una, aumenta la otra). Esto no significa que el helado cause muertes, sino que ambas se explican por una tercera variable, que es el verano (éste aumenta el consumo de helado y uso de piscinas). Esto se conoce como espuria estadística y debemos evitarlas.

- **Evitar variables derivadas**. Ejemplo, si incluyo tanto la edad como la fecha de nacimiento, la estoy entregando al modelo dos veces el mismo valor y por tanto dándole más peso a un aspecto en particular.

Para medir estas correlaciones, las dos herramientas fundamentales son *Correlación de Pearson* y *Mutual Information*. Te recomiendo también que veas más detalles en la [documentación oficial](https://docs.microsoft.com/en-us/azure/machine-learning/studio-module-reference/filter-based-feature-selection).

### Filtrar y elegir los features ###

Si bien el módulo `Filter Based Feature Selection` te permite elegir cuántos de esos campos quieres conservar, también puedes descartarlos manualmente con el campo [Select Columns in Dataset](https://docs.microsoft.com/en-us/azure/machine-learning/studio-module-reference/select-columns-in-dataset).

### Datos de entrenamiento y datos de prueba ###

Clasificación binaria utiliza una práctica llamada [Entrenamiento Supervisado](https://docs.microsoft.com/en-us/azure/machine-learning/machine-learning-algorithm-choice). Esto significa que dividiremos nuestro conjunto en dos partes: datos de entrenamiento y datos de prueba.

Los datos de entrenamiento serán utilizados para crear nuestro modelo y ajustar sus parámetros. En cuanto a los datos de prueba, ignoraremos la respuesta conocida (si abandonó o no la secundaria) para probar el comportamiento de nuestro modelo. Luego, compararemos la respuesta real con nuestra predicción, y sabremos qué tan bien funciona nuestro modelo.

Para realizar esto, utilizaremos el módulo `Split Data`

![Split data](https://github.com/marcelofelman/case-studies/blob/master/images/8-split-data.PNG?raw=true)

>**Tip:** Puedes agregar comentarios a los componentes, haciendo doble clic en la caja.

Puedes ajustar qué cantidad de datos irán para cada salida, en la solapa de la derecha.

>**Tip:** Utiliza más datos de entrenamiento que datos de prueba. Según la [distribución de Pareto](https://es.wikipedia.org/wiki/Distribución_de_Pareto), 80 para entrenamiento y 20 para pruebas es adecuado.

### Estratificación ###

Existen varias maneras de separar nuestros datos en dos subconjuntos. Aquí usaremos algo llamado `Stratified Split` para tratar de solucionar nuestro problema del *"dataset desbalanceado"* que vimos antes.

Lo que hace el `Stratified Split` es asegurarse de que los dos subconjuntos tienen el mismo porcentaje de una variable que definamos como objetivo, y en este caso, elegiremos nuestra etiqueta *LABEL_DEFAULT*.

Más info sobre [Split Data](https://docs.microsoft.com/en-us/azure/machine-learning/studio-module-reference/split-data-using-split-rows).

### Crear el modelo ###

Tenemos nuestros datos de entrenamiento y de prueba, ahora para crear el modelo debes simplemente conectar la salida izquierda de `Split Data` (que son los datos de entrenamiento) con la entrada derecha del módulo `Train model` o `Entrenar modelo` en Español.

La entrada izquierda de `Train model` debe conectarse con un algoritmo, en este caso utilizaré `Two-class Boosted Decision Tree`.

![Entrenar modelo](https://github.com/marcelofelman/case-studies/blob/master/images/9-train-model.PNG?raw=true)

Seguro notes una cruz roja en el módulo `Train model`. Esto se debe a que debemos enseñarle al software cuál es el campo que buscamos predecir. En mi caso, quiero predecir el campo binario *LABEL_DEFAULT*.

Usa el selector de columna a la derecha para elegir el campo.

![Seleccionar columna](https://github.com/marcelofelman/case-studies/blob/master/images/10-column-selector.PNG?raw=true)

En otros términos, estamos utilizando un algoritmo y datos de entrenamiento, para entrenar un modelo. Estamos casi listos.

>**Tip:** En cualquier paso de la experimentación, puedes darle `Run` y visualizar la salida de cada componente con `clic derecho` > `Visualize`. En esta instancia, puedes visualizar las salidas izquierda y derecha de `Split Data` como también podrías ver la salida del modelo entrenado.

### Realizar las predicciones ###

Estamos en condiciones de probar nuestro modelo. Para ello, debemos utilizar el componente `Score model`. Aquí es donde entran en juego los datos de prueba, los cuales (salida derecha de `Split data`) conectaremos a la entrada derecha de `Score model`.

![Score model](https://github.com/marcelofelman/case-studies/blob/master/images/11-score-model.PNG?raw=true)

Puedes darle `Run` y luego visualizar el resultado (su salida). Este componente genera las predicciones y agrega dos nuevas columnas:

- `Scored label`: indica la categoría que predice nuestro modelo. En este caso será '1' o '0'.
- `Scored probabilities`: indica la probabilidad que tenía de tomar la etiqueta positiva.

Por ejemplo, si un caso nos dio de `Scored label` 'SI' y `Scored probabilities` 0,70 entonces el modelo dice que con un 70% de confianza, esa persona entrará en una situación de *default*.

![Valores categorizados](https://github.com/marcelofelman/case-studies/blob/master/images/12-scored-model.PNG?raw=true)

### Evaluar el comportamiento del modelo ###

En el paso anterior hicimos predicciones caso por caso, ahora interpretaremos eso como un todo. Para ello, utilizaremos el módulo `Evaluate model` el cual toma como entrada la salida de `Score model`.

![Evaluar modelo](https://github.com/marcelofelman/case-studies/blob/master/images/13-evaluate-model.PNG?raw=true)

Si damos `Run` visualizamos la salida de `Evaluate model`, podremos ver la siguiente gráfica:

![Comportamiento](https://github.com/marcelofelman/case-studies/blob/master/images/14-auc.PNG?raw=true)

Esta gráfica demuestra los casos que fueron correctamente identificados, respecto los que no. Esencialmente, el área debajo de la curva (`Area under the curve` o también `AUC`) debe ser lo mayor posible: no queremos dejar casos afuera.

A simple vista, nuestro modelo parece comportarse de una manera muy acertada: el **????%** de las veces está realizando una predicción correcta.

No obstante, si nos movemos hacia abajo veremos más métricas que definen el comportamiento de nuestro modelo predictivo.

![Más métricas](https://github.com/marcelofelman/case-studies/blob/master/images/15-more-metrics.PNG?raw=true)

Como podemos apreciar, lo que estamos haciendo es identificar cuatro casos distintos:

- Clientes que dijimos que entraban en *default*, y entraron (verdadero positivo): A
- Clientes que dijimos que NO entraban en *default*, y entraron (falso negativo): B
- Clientes que dijimos que entraban en *default*, y NO entraron (falso positivo): C
- Clientes que dijimos que NO entraban en *default*, y NO entraron (verdadero negativo): D

Debemos ser cautelosos y evaluar estos puntos. Una buena pregunta para hacernos es cuál es el costo de cada escenario. En este caso, es ...

Como podemos ver arriba, estamos identificando a XXX clientes, pero "dejando pasar" a unos XXX. Debemos mejorar nuestro modelo.

## Mejorar el modelo ##

El proceso de mejora suele ser un proceso iterativo. En esta línea, existen diferentes caminos para tomar a la hora de mejorar un modelo.

**Para ver ideas sobre iteraciones de mejora, te recomiendo ver la sección de Mejoras en el caso de [Deserción Escolar con Machine Learning](https://github.com/marcelofelman/case-studies/blob/master/Desercion%20escolar%20con%20Machine%20Learning.md#iteraciones-de-mejora)**

Algunas de estas ideas son:

- Utilizar menos variables
- Utilizar más variables
- Utilizar otras variables
- Balancear el conjunto de datos
- Modificar los parámetros del algoritmo
- Utilizar cross-validation
- Medir el éxito de otra manera
- Utilizar otros algoritmos
- Encarar el problema de otra manera

>**Tip:** Si estás trabajando en equipo te recomiendo conocer el [Team Data Science Process](https://docs.microsoft.com/en-us/azure/machine-learning/team-data-science-process/overview).

## Visualización ##

Analizar la demografía de los datos suele aportar *insights* acerca del dominio de la información y del propio modelo. Ya sea con datos reales empleados para consumir el experimento publicado o con los datos de muestra en la etapa de experimentación, estos registros pueden ser expuestos a una herramienta de visualización.

En esta sección describiremos algunas alternativas para obtener registros con el *label* predictivo de `Azure Machine Learning Studio` y algunos tips para la visualización de datos el dominio financiero.

### Exportar datos ###

#### Desde el experimento ####

La manera más sencilla de exportar datos desde `AMLS` es convirtiendo un dataset al formato `CSV`. Existe una tarea `Convert to CSV` para ello que recibe un dataset como input y permite descargar el output en dicho formato.

El módulo `Convert to CSV` debe recibir datos del modelo entrenado (luego de haberse ejecutado `Train Model` para contener el label predictor, pero también será necesario incluir todos los *features* que querramos visualizar antes de ser normalizados ya que no sólo la distribución es importante, sino también los valores reales que fueron input del experimento. Para ello es imperioso haber conservado un identificador del registro ("número de cuenta" en nuestro caso) para poder hacer un `Join Data` con el *dataset* original. Finalmente conectar la salida de `Join Data` con `Convert to CSV`.

#### Desde el modelo (trained model) ####

Hay múltiples maneras de consumir el modelo entrenado, ya sea mediante una aplicación que invoque la API del modelo, a través de un [app template en Azure](https://docs.microsoft.com/en-us/azure/machine-learning/studio/consume-web-service-with-web-app-template) o directamente desde Microsoft Excel, empleando el add-in de Azure Machine Learning.

![Descarga de add-in para ejecución por lotes](https://github.com/dgregoraz/case-studies/blob/master/images/ml-1/batch-execution.png?raw=true)

En cualquiera de estos casos se puede analizar un lote de registros, incorporar el label predictivo y exportar los registros para ser visualizados en Power BI.

### Representación de resultados ###

Los buenos modelos predictivos suelen ser simples. La cantidad de features o predictores son acotados por lo que todos ellos pueden ser representados en un reporte o dashboard.

La representación descriptiva de los predictores y otras variables con la opción de filtrar la predicción (default/no default) ayuda a comprender la distribución de cada una de esos atributos.

El siguiente ejemplo muestra la distribución de edad de los clientes y, con la saturación de color, resalta la deuda adquirida.

![Ejemplo de distribución etaria de los clientes y deuda acumulada](https://github.com/dgregoraz/case-studies/blob/master/images/ml-1/age-distribution.png?raw=true)

Se puede ver tanto cómo el rango etario de los clientes está concentrado entre los 30 y 50 años mayormente pero que los que mayor deuda acumulan están, a su vez, sesgados hacia el rango de 30-40.

En etapa de experimentación también es útil analizar visualmente posibles nuevos predictores. El siguiente ejemplo muestra un *boxplot* con la misma distribución etaria de una muestra de datos en función de su historial de pagos.

![Distribución etaria según label predictivo](https://github.com/dgregoraz/case-studies/blob/master/images/ml-1/predictive-boxplot.png?raw=true)

Como puede observarse, casi no hay solapamiento en la distribución etaria para esta porción de la población por lo que la edad tiene una contribución considerable en el comportamiento de pago.

De la misma manera que el ejemplo anterior, para un problema de clasificación como el que se describe en este caso es particularmente útil analizar los falsos negativos y falsos positivos para procurar encontrar algún patrón de correlación que pueda indicar la existencia de un nuevo predictor que mejore el modelo.

## Conclusiones ##

Machine Learning es todo un mundo diferente, y puede resultar complejo para quienes venimos del ámbito de desarrollo de software.

A través de un proceso iterativo de prueba y error, podemos ir acercándonos a una respuesta correcta y finalmente determinar si nuestro modelo es se comporta como esperamos o no.

En nuestro caso, logramos un modelo predictivo que identifica correctamente a aproximadamente el 98% de los clientes que ingresaran en un *default*. Estos resultados son similares a los obtenidos por otros investigadores, grupos y competencias.

## Equipo ##

Diego Gregoraz - [Twitter](https://twitter.com/dgregoraz) - [GitHub](https://github.com/dgregoraz)

Jorge Cupi - [Twitter](https://twitter.com/jorgecupi) - [GitHub](https://github.com/jorgecupi)

Marcelo Felman - [Twitter](https://twitter.com/mfelman) - [GitHub](https://github.com/marcelofelman)