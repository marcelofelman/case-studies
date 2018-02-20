# Cómo predecir riesgo financiero con Machine Learning #

En este caso de estudio, me gustaría contarte cómo hicimos para identificar tempranamente clientes de una entidad financiera -específicamente tarjetahabientes- que ingresen en una situación de [default](https://en.wikipedia.org/wiki/Default_(finance)) utilizando técnicas de Machine Learning.

Puedes usar este caso como una guía para crear tu propio modelo, o simplemente leerlo para conocer la lógica y secuencia de pasos que seguimos para llegar a nuestro resultado.

## Resumen ##

En colaboración con una importante entidad financiera (*y cuyo nombre omitimos por motivos de sensibilidad*), definimos como objetivo utilizar inteligencia artificial para identificar aquellos clientes que, en base a su comportamiento, tengan una alta probabilidad de incumplir con sus obligaciones de pago de tarjeta de crédito.

Afortunadamente, tuvimos acceso a un amplio espectro de datos. Utilizamos un *dataset* de varios miles (entre 100-200, dependiendo de los tipos de filtros, más sobre eso en breve) de cuentas. **Estos datos no contienen información personal identificable sobre las personas, tal como reconoce [habeas data](https://es.wikipedia.org/wiki/Habeas_data)**.

A través de las herramientas [Azure Machine Learning](https://azure.microsoft.com/en-us/services/machine-learning/
) y [SQL Server 2016](https://www.microsoft.com/en-us/sql-server/sql-server-2016
), logramos crear diferentes modelos predictivos que permiten detectar hasta el 99% de los casos en riesgo de default.

## Contexto ##

Previo a comenzar el proyecto, definimos algunos puntos a considerar:

- Partir "desde cero" con el modelo, ignorando todo conocimiento previo para no sesgar el estudio con resultados anteriores.
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

*Null*, *missing*, etcétera. Asegúrate de remover o manejar todo eso.

### Valores categóricos ###

En el caso de valores categóricos, es buena idea chequear que no haya categorías adicionales o faltantes a raíz de un error de escritura. Por ejemplo, con un diagrama de frecuencias para la variable sexo puedo asegurarme que haya dos categorías, que son M o F.

![Valores categóricos](https://github.com/marcelofelman/case-studies/blob/master/images/3-valores-categoricos.png?raw=true)

## Preparación de los datos ##

El siguiente paso consiste en hacer un poco de limpieza habiendo identificado los casso anteriores: remover datos que faltan, remover valores extremos o *outliers*, conversiones de moneda local a dólar, conversión de valores categóricos, y otra técnica conocida como normalización. Veamos cada uno de estos.

### Remover valores extremos o outliers ###

En AzureML Studio existe un módulo llamado [Clip Values](https://docs.microsoft.com/en-us/azure/machine-learning/studio-module-reference/clip-values). El mismo te permite definir un umbral para un valor mínimo o máximo, y luego tomar una decisión sobre el mismo que puede ser removerlo o puede ser reemplazarlo.

Para seleccionar el umbral, hay dos opciones: constante o percentil. El primer caso corresponde a un valor fijo, y todo valor absoluto que esté por debajo o por encima (según definamos) será tratado. El segundo caso, corresponde a un valor relativo. Más información sobre [percentiles](https://en.wikipedia.org/wiki/Percentile).

Para reemplazar los valores, tengo algunas alternativas: el mismo valor de umbral, la media (promedio), la mediana, o un *null*.

Veamos algunos ejemplos:

- Si encuentro un registro de alguien con una edad mayor a 200 años, lo quiero marcar como *null* ya que asumo que está mal.
- Aquellos con un límite de crédito por encima del percentil 98 quiero reemplazarlos por el valor del umbral.

### Remover datos que faltan ###

Aquí también existe un módulo que te permite remover valores faltantes, el mismo se llama [Clean Missing Data](https://docs.microsoft.com/en-us/azure/machine-learning/studio-module-reference/clean-missing-data). El mismo puede trabajar por registro (fila) o por columna.

Tienes varias alternativas: puedes directamente remover el registro/columna, o bien tienes varias opciones de sustitución de valores. Mira la [documentación](https://docs.microsoft.com/en-us/azure/machine-learning/studio-module-reference/clean-missing-data) para más info.

### Conversiones de moneda local a dólar ###

A veces es necesario realizar conversiones más personalizadas, y para ello tengo algunas opciones:

- Python
- R
- SQL

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

El módulo *Filter Based Feature Selection* te permite medir [correlación](https://en.wikipedia.org/wiki/Correlation_and_dependence) entre las distintas variables y la variable objetivo. De esta manera, podemos identificar campos irrelevantes en nuestro estudio.

Es importante considerar los siguientes aspectos:

- Correlación no implica causalidad. Por ejemplo, el consumo de helados está correlacionado con la cantidad de personas que mueren ahogadas (al aumentar una, aumenta la otra). Esto no significa que el helado cause muertes, sino que ambas se explican por una tercera variable, que es el verano (éste aumenta el consumo de helado y uso de piscinas). Esto se conoce como espuria estadística y debemos evitarlas.

- Debemos evitar valores duplicados. Ejemplo, si incluyo tanto la edad como la fecha de nacimiento, la estoy entregando al modelo dos veces el mismo valor y por tanto dándole más peso a un aspecto en particular.

Para medir estas correlaciones, las dos herramientas fundamentales son *Correlación de Pearson* y *Mutual Information*. Te recomiendo también que veas más detalles en la [documentación oficial](https://docs.microsoft.com/en-us/azure/machine-learning/studio-module-reference/filter-based-feature-selection).

### Filtrar y elegir los features ###

Si bien el módulo *Filter Based Feature Selection* te permite elegir cuántos de esos campos quieres conservar, también puedes descartarlos manualmente con el campo [Select Columns in Dataset](https://docs.microsoft.com/en-us/azure/machine-learning/studio-module-reference/select-columns-in-dataset).

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