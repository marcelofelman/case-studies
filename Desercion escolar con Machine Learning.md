# Predecir deserción escolar con Machine Learning #

En este post, me gustaría contarte cómo hicimos para detectar jóvenes en riesgo de abandonar la secundaria utilizando técnicas de Machine Learning.

## Resumen ##

En colaboración con el Ministerio de Primera Infancia del Gobierno Provincial de Salta, definimos como objetivo utilizar inteligencia artificial para identificar aquellos jóvenes con mayor riesgo de abandonar sus estudios secundarios, de manera tal de poder darles mayor apoyo.

Afortunadamente, tuvimos acceso a un amplio espectro de datos. Utilizamos un *dataset* de más de 200.000 residentes de la ciudad de Salta, Argentina. **Estos datos no contienen información personal identificable sobre las personas**.

A través de las herramientas [Azure Machine Learning](https://azure.microsoft.com/en-us/services/machine-learning/
) y [SQL Server 2016](https://www.microsoft.com/en-us/sql-server/sql-server-2016
), logramos crear diferentes modelos predictivos que permiten detectar hasta el 90% de los casos en riesgo de deserción.

## Contexto ##

La ciudad de Salta es una de las ciudades más pobladas de Argentina y capital de la provincia con igual nombre.

Con una población que supera los 500.000 habitantes, el Ministerio de Primera Infancia tiene por misión erradicar la pobreza en la provincia. Con este objetivo en mente, la educación constituye un pilar fundamental para el desarrollo. El Ministerio de Primera Infancia propone mejorar los niveles educativos identificando con Inteligencia Artificial aquellos jóvenes en riesgo de abandonar sus estudios.

## Software y herramientas ##

- [Azure Machine Learning](https://azure.microsoft.com/en-us/services/machine-learning/)
- [SQL Server 2016](https://www.microsoft.com/en-us/sql-server/sql-server-2016)

## Fases del proyecto ##

- Exploración del dominio
- Preparación de los datos
- Creación de modelos
- Iteraciones de mejora
- Integración

## Exploración del dominio ##

Antes de empezar, debemos comprender el dominio en el cual estamos trabajando. Si bien lo ideal es contar con un experto en el dominio, no siempre lo tendremos a disposición. Para esto, es importante tener en cuenta diferentes técnicas así como también ejercitar nuestro sentido común.

Para comenzar a visualizar los datos, soy partidario de utilizar un motor de bases de datos, como puede ser SQL Server. De esta forma puedo escribir consultas SQL fácilmente y además exportar como CSV o inclusive conectar con Azure Machine Learning.

Para este proyecto, contamos con una tabla principal la cual contiene información de las personas: edad, altura, máximo nivel educativo, barrio/zona en donde residen. A su vez, contiene información referencial (*foreign keys*) hacia sus padres o jefes de hogar.

Una práctica que encuentro conveniente es unir todos los datos en una única tabla o proyección a través de la cláusula *JOIN*. De esta forma, podrás portarlo de manera más simple a Azure Machine Learning.

>**Tip:** Unir todos los datos en una única tabla o proyección.

### Ejemplo #1 ###

Supongamos que sólo quiero obtener la edad del padre y del hijo, haríamos algo así:

```sql
SELECT joven.edad, padre.edad
FROM Personas joven
INNER JOIN Personas padre
ON joven.idPadre = padre.id
```

Esto es importante saberlo, ya que en la fase inicial lo haremos muchas veces.

### Pruebas simples ###

Para ir familiarizándote con el conjunto de datos, es bueno que ejecutes algunas consultas. Estas corrí yo como ejemplo:

- Cantidad total de personas
- Cantidad de jóvenes (menores de 21 años)
- Cantidad de jóvenes que terminan la secundaria vs los que no
- Cantidad de personas que terminan la secundaria vs los que no
- Tasa de deserción por zona en donde viven

>**Tip:** Invierte todo el tiempo que creas necesario para entender las relaciones entre los datos. 2 o 3 días enfocado en esto puede ser un tiempo razonable (aunque parezca mucho), dependiendo del dominio.

### Ejemplo #2 ###

Este es un ejemplo para calcular la cantidad total de jóvenes:

```sql
SELECT COUNT(*) FROM Personas WHERE Edad <= 21
```

Este es un ejemplo para calcular la deserción por zona:

```sql
SELECT Barrio, COUNT(*) AS Desertor,
    (SELECT COUNT(*) FROM Personas p2 WHERE p2.Barrio = p.Barrio) AS TotalBarrio,
    CAST(COUNT(*) AS FLOAT) / CAST((SELECT COUNT(*) FROM Personas p2 WHERE p2.Barrio = p.Barrio) AS FLOAT) * 100 AS Porcentaje
FROM Personas p
WHERE AbandonoEstudios = 'SI'
GROUP BY Barrio
ORDER BY Porcentaje DESC
```

Aquí, obtuvimos resultados similares a los siguientes:

| Barrio        | Desertor           | TotalBarrio  | Porcentaje |
| ------------- |:-------------:| :-----:| :------:|
| Algún barrio      | 102 | 184 | 55,4347|
| Otro barrio      | 108      |   225 |48|
| Zona diferente | 289      |    874 |33,0663|

Como puedes ver, distintas zonas tienen distintas tasas de deserción. Esto pueda llevarnos a pensar que la zona donde un chico resida, tenga que ver con su probabilidad de abandonar la secundaria. Por ahora, es sólo una hipótesis que luego probaremos.

>**Tip:**Seguro encuentres mejores maneras de escribir estas consultas. No te preocupes, la performance aquí no es importante. 

### Iteración ###

A estas alturas, empezarás a darte cuenta de ciertas variables que pueden llegar a ser o no relevantes en nuestro modelo. Proyécatalas todas y luego en Azure Machine Learning validaremos cuáles son las que sirven.

Para nuestro modelo, utilizaré inicialmente las siguientes variables:

- Edad del jóven
- Zona donde reside
- Etnia
- Sexo
- Tipo de discapacidad, si es que tiene
- Máximo nivel educativo del jefe de hogar
- Cantidad de personas con quien vive

Luego iteraremos para entender si estas variables nos sirven o no.

## Preparación de los datos ##

Ahora que sabemos qué datos queremos utilizar, debemos prepararlos para ir hacia Azure Machine Learning. Estando en SQL Server, una manera simple de hacerlo es guardando los resultados de nuestra consulta como CSV. Para ello, debemos seguir estos pasos:

1. Abrir SQL Server Management Studio
2. Ir a *Tools > Options > Query Results > SQL Server > Results to text
3. A la derecha, busca el menú que dice *Output Format*
4. Elegí *Comma Delimited* y click en OK

![CSV](/images/1-sql-options.png)

Luego, haciendo click en el siguiente botón, podrás correr tus consultas y guardar los resultadso en un archivo.

![Guardar resultado como archivo](/images/2-sql-results.png)

Ahora podrás guardar tus consultas como CSV.

Esto nos generará un archivo de tipo .rpt, al cual simplemente cambiar su extensión a .csv y abrirlo como tal.

>**Tip:**Los archivos CSV pueden abrirse con Excel. Si te hace sentir más cómodo, chequealo antes de seguir con el próximo paso.

Ahora, procedemos a ingresar al [Azure Machine Learning Studio](https://studio.azureml.net). Si no tienes una cuenta, puedes crearla de forma gratuita allí mismo.

Al ingresar, hacemos click en *New* y luego en *Dataset*

>>>SCREENSHOT AZURE ML

Subimos nuestro conjunto de datos, y al cabo de unos segundos estará en Azure. El mismo aparecerá en la solapa *My datasets* y lo podrás utilizar inmediatamente. Para esto, simplemente lo arrastras a la ventana principal.

>>>SCREENSHOT DATASET

Si quieres visualizarlo, puedes hacer click en la salida del módulo y otro click en *visualize*.

>>>SCREENSHOT VISUALIZE

## Creación de modelos ##

Ahora empezaremos a crear nuestros modelos predictivos. Como mencionamos anteriormente, lo que estamos buscando es predecir *si un jóven terminó la secundaria o no*. En otras palabras, es una clasificación binaria (de dos clases). Para más información sobre clasificación binaria y otros tipos de problemas, puedes ver [aquí](https://docs.microsoft.com/en-us/azure/machine-learning/machine-learning-algorithm-choice).

Para resolver este problema, existen diferentes algoritmos. Recuerda que los diferentes algoritmos son simplemente distintas formas de abordar a un resultado. Algunos llegan a mejores resultados, pero esto no está garantizado. Si quieres ver el listado y ventajas de cada uno de los algoritmos, lo puedes ver [aquí](https://docs.microsoft.com/en-us/azure/machine-learning/machine-learning-algorithm-choice). Yo empezaré utilizando un *Two-class Boosted Decision Tree*, el cual es balanceado en cuanto a consumo de recursos y exactitud en sus resultados.

*Tip #5*
Todavía no te preocupes tanto por el algoritmo que utilizarás. Tendrás tiempo de probar otros más adelante.

### Entrenamiento supervisado ###

Clasificación binaria utiliza una práctica llamada [Entrenamiento Supervisado](https://docs.microsoft.com/en-us/azure/machine-learning/machine-learning-algorithm-choice). Esto significa que diviremos nuestro conjunto en dos partes: datos de entrenamiento y datos de prueba.

Los datos de entrenamiento serán utilizados para crear nuestro modelo y ajustar sus parámetros. En cuanto a los datos de prueba, ignoraremos la respuesta conocida (si abandonó o no la secundaria) para probar el comportamiento de nuestro modelo. Luego, compararemos la respuesta real con nuestra predicción, y sabremos qué tan bien funciona nuestro modelo.

Para realizar esto, utilizaremos el módulo *Split Data*

>>>Screenshot SPLIT DATA

Puedes ajustar qué cantidad de datos irán para cada salida, en la solapa de la derecha.

*Tip #6*
Utiliza más datos de entrenamiento que datos de prueba. Según la [distribución de Pareto](https://es.wikipedia.org/wiki/Distribución_de_Pareto), 80 para entrenamiento y 20 para pruebas es adecuado.

### Crear el modelo ###

Para crear el modelo, debes simplemente conectar la salida izquierda de *Split Data* (que son los datos de entrenamiento) con la entrada derecha del módulo *Train model* ó Entrenar modelo en Español.

La entrada izquierda de *Train model* debe conectarse con un algoritmo, en este caso utilizaré *Two-class Boosted Decision Tree*.

>>>SCREENSHOT MODELITO BASICO

En otros términos, estamos utilizando un algoritmo y datos de entrenamiento, para entrenar un modelo.

### Realizar las predicciones ###

Estamos en condiciones de probar nuestro modelo. Para ello, debemos utilizar el componente *Score model*. Este componente genera las predicciones y agrega dos nuevas columnas:

- *Scored label*: indica la categoría que predice nuestro modelo. En este caso será 'SI' o 'NO'.
- *Scored probabilities*: indica la probabilidad que tenía de tomar la etiqueta positiva.

Por ejemplo, si un caso nos dio de *Scored label* 'SI' y *Scored probabilities* 0,70 entonces el modelo dice que con un 70% de confianza, ese jóven abandonará la secundaria.

>>>SCREENSHOT DE SCORE MODEL

### Evaluar el comportamiento del modelo ###

En el paso anterior hicimos predicciones caso por caso, ahora interpretaremos eso como un todo. Para ello, utilizaremos el módulo *Evaluate model* el cual toma como entrada la salida de *Score model*.

Si visualizamos la salida de *Evaluate model*, podremos ver la siguiente gráfica:

>>>GRAFICA DE EVALUATE MODEL**

Esta gráfica demuestra los casos que fueron correctamente identificados, respecto los que no. Esencialmente, el área debajo de la curva (*Area under the curve* o también *AUC*) debe ser lo mayor posible: no queremos dejar casos afuera.

A simple vista, nuestro modelo parece comportarse de una manera muy acertada: el **XXX %** de las veces está realizando una predicción correcta.

No obstante, si nos movemos hacia abajo veremos más métricas que definen el comportamiento de nuestro modelo predictivo.

>>>ELABORAR SOBRE LAS METRICAS

Como podemos apreciar, lo que estamos haciendo es identificar cuatro casos distintos:

>>>COMPLETAR METRICAS

- Estudiantes que dijimos que iban a abandonar, y abandonaron (verdadero positivo):
- Estudiantes que dijimos que NO iban a abandonar, y abandonaron (falso negativo):
- Estudiantes que dijimos que iban a abandonar, y NO abandonaron (falso positivo):
- Estudiantes que dijimos que NO iban a abandonar, y NO abandonaron (verdadero negativo):

Debemos ser cautelosos y evaluar estos puntos. Una buena pregunta para hacernos es cuál es el costo de cada escenario. En este caso, es mucho peor NO ayudar a un estudiante en riesgo de abandonar, que ayudar por demás a alguno que en realidad no iba abandonar. El costo de los falsos positivos supera el de los falsos negativos.

## Iteraciones de mejora ##
 
Descubrimos que nuestro modelo no es tan preciso, o que tal vez puede mejorar. ¿Cómo podemos mejorarlo? A continuación, algunas ideas:

- Utilizar menos variables
- Utilizar más variables
- Utilizar *otras* variables
- Balancear el conjunto de datos
- Modificar los parámetros del algoritmo
- Utilizar *cross-validation*
- Medir el éxito de otra manera
- Utilizar otros algoritmos
- Encarar el problema de otra manera

Todos los anteriores puntos fueron parte de nuestro análisis para este proyecto. A continuación una breve explicación de cada uno.

### Utilizar menos variables ###

A veces menos es más, especialmente cuando pueda existir correlación entre dos o más variables que deberían ser independientes.

Un ejemplo en nuestro caso, podría ser utilizar tanto la variable *edad* como la variable *año de nacimiento*. Claramente, habrá una correlación lineal y perfecta entre estas dos variables. El modelo se verá desbalanceado, favoreciendo la edad (o fecha de nacimiento) como un campo más poderoso.

*Tip #7*
Azure Machine Learning hace sencilla la búsqueda de correlación entre variables. Lo veremos más adelante.

### Utilizar más variables ###

A veces también más es más, especialmente cuando estamos utilizando pocos campos para describir nuestros casos. A veces es necesario analizar el agregado de más campos, puede ser de forma directa o pueden ser como campos calculados.

Para este proyecto, pensamos en agregar los siguientes campos calculados:

- Promedio de notas del último semestre
- Distancia del alumno a la escuela
- Si sus padres completaron la secundaria
- Si trabaja actualmente

Debes evaluar en cada caso si pueden existir más campos que tengan sentido agregar. Recuerda que es un proceso iterativo y en raras ocasiones armarás un buen modelo en tu primer iteración.

### Utilizar otras variables ###

Tal vez pueda sorprendernos que aquel campo del cual creíamos que nos servía, en realidad no lo hacía. Una manera simple de saber si estamos utilizando las variables correctas, es buscando correlación.

Esto puede hacerse a través del módulo *Feature Selection*

>>>COMPLETAR CON SCREENSHOTS DE CORRELACION

### Balancear el conjunto de datos ###

Un problema típico son los conjuntos de datos desbalanceados. En este caso, la probabilidad de ocurrencia de un menor de 21 años de abandonar la secundaria está alrededor del 7%. En otras palabras, el 7% de los registros dice AbandonoEstudios 'SI' mientras que el 93% dice 'NO'.

Si lo piensas en frío, si creáramos un modelo que siempre diga 'NO' entonces estaríamos acertando el 93% de las veces (lo cual es puntaje muy alto). Esto no es bueno.

Esta situación puede generar un balanceo que favorezca el escenario mayoritario, es decir el 'NO'. Para ello, podemos aplicar dos técnicas distintas:

- *Undersampling*: tomar menos casos del escenario mayoritario a fin de reducir las ocurrencias.
- *Oversampling*: aumentar o simular más ocurrencias del caso minoritario.

De esta manera, logramos un conjunto de datos más balanceado. Una manera simple y casi automática de lograr esto, es utilizando el módulo *SMOTE*.

>>>SCREENSHOTS SOBRE SMOTE

>>>screenshots sobre MODULO OUTPUT SMOTE

### Modificar los parámetros del algoritmo ###

Recuerda que el algoritmo es la secuencia lógica que utilizaremos para llegar a la mejor respuesta posible. Una buena práctica es realizar un barrido de parámetros para encontrar la mejor combinación posible.

El módulo *Tune Model Hyperparameters* hace exactamente esto: barre los parámetros hasta llegar al mejor modelo encontrado.

Puedes utilizarlo de la siguiente manera:

>>>SCREESHOTS TUNE HYPERMODEL

### Utilizar cross-validation ###

>>>BLABLA EXPLICAR CROSS VALIDATION

### Medir el éxito de otra manera ###

Existen distintas maneras de medir si un modelo es exitoso o no. Estas son algunas de ellas:

- *Accuracy* o exactitud
- *Precision* o precisión
- Falsos negativos
- Falsos positivos
- Verdaderos negativos
- Verdaderos positivos
- *Recall*
- Puntaje F1

>>>PROFUDIZAR CADA UNO DE ESTOS**

### Utilizar otros algoritmos ###

En determinados casos, distintos algoritmos podrían llegar a mejores resultados. No dejes de probar distintas alternativas. Para ello, simplemente conecta un algoritmo diferente a la entrada de *Train model*. Eso es todo.

>>>SCREENSHOT DE OTRO ALGORITMO**

*Tip #8*
El módulo *Evaluate model* permite dos entradas. De esta forma, puedes comparar dos algoritmos fácilmente.

### Encarar el problema de otra manera ###

Cuando nuestras opciones se agotan y no llegamos a buen puerto, es probable que tal vez simplemente estemos encarando el problema de una manera inapropiada.

En nuestro caso, hemos evaluado la posibilidad de enfrentar este escenario como un problema de detección de anomalías. Estos algoritmos están especialmente pensados para conjuntos de datos muy desbalanceados. Dos ejemplos son detección de fraudes o detección de cáncer.

Si quieres conocer más sobre detección de anomalías, puede ir a [este artículo](https://msdn.microsoft.com/en-us/library/azure/dn913096.aspx?f=255&MSPPError=-2147217396).

## Integración ##

>>>CONTAR COMO PUBLICAR COMO WEB SERVICE Y COMO USAR..**

## Conclusiones ##

## Agradecimientos ##