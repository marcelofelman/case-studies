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

### Qué hay en nuestro conjunto de datos ###

Ya hemos exportado nuestro *dataset*, ahora es una buena idea ir explorando el contenido del mismo.

Lo primero que es importante visualizar, es el balance de nuestra variable objetivo, que hemos llamado *LABEL_DEFAULT* arbitrariamente. Podrás notar que sólo existen dos valores: uno o cero. El problema es que esto está fuertemente desbalanceado: hay cerca de un 98% de ceros mientras que 2% de unos. Veremos más adelante que hay varias opciones para solucionar esto.

## Preparación de los datos ##

El siguiente paso consiste en hacer un poco de limpieza inicial: remover datos que faltan, remover valores extremos o *outliers*, conversiones de moneda local a dólar, conversión de valores categóricos, y otra técnica conocida como normalización. Veamos cada uno de estos.

### Remover datos que faltan ###

aaa

### Remover valores extremos o outliers ###

aaa

### Conversiones de moneda local a dólar ###

aaa

### Conversión de valores categóricos ###

aaa

### Normalización ###

aaa

## Creación de modelos ##

Ahora empezaremos a crear nuestros modelos predictivos.

Para resolver este problema, existen diferentes algoritmos. Recuerda que los diferentes algoritmos son simplemente distintas formas de abordar un resultado. Algunos llegan a mejores resultados, pero esto no está garantizado. Si quieres ver el listado y ventajas de cada uno de los algoritmos, lo puedes ver [aquí](https://docs.microsoft.com/en-us/azure/machine-learning/machine-learning-algorithm-choice). Yo empezaré utilizando un *Two-class Boosted Decision Tree*, el cual es balanceado en cuanto a consumo de recursos y exactitud en sus resultados.

>**Tip:** Todavía no te preocupes tanto por el algoritmo que utilizarás. Tendrás tiempo de probar otros más adelante.

### Pruebas de correlación ###

Una buena manera de identificar qué campos pueden o no ser buenos predictores es realizando pruebas de correlación.