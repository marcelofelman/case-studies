# Bots con Bot Builder v3 y múltiples QnA Makers

>**TL;DR** usa este código para consumir múltiples QnA Maker apps desde Microsoft Bot Framework.

## Qué estamos haciendo
Microsoft Bot Framework permite la integración con QnA Maker. Esto es muy simple de hacer, cuado uno tiene una única QnA Maker app. Con este código estamos creando un "reconocedor" que toma múltiples fuentes.

Algunos links relevantes para que tengas presente:
* [QnA Maker](https://qnamaker.ai/)
* [Bot Builder para C#](https://github.com/Microsoft/BotBuilder/tree/master/CSharp)
* [Integrar Bot Builder con QnA Maker](https://docs.microsoft.com/en-us/bot-framework/azure-bot-service-template-question-answer)

## Vamos al código

Ante todo, el código completo está o más abajo o en este repositorio: https://github.com/marcelofelman/cognitive-tools/tree/master/csharp  

Primero vamos a definir una clase QnAMakerResult
```csharp
public class QnAMakerResult
{
    [JsonProperty(PropertyName = "answer")]
    public string Answer { get; set; }

    [JsonProperty(PropertyName = "score")]
    public double Score { get; set; }
}
```

No te olvides de utilizar Newtonsoft

```csharp
using Newtonsoft.Json;
```

También necesataremos una clase QnAMakerApp. Esta representa las credenciales necesarias para conectarnos a un QnA Maker.

```csharp
private class QnAMakerApp
{
    public string KnowledgeBaseId { get; set; }

    public string SubscriptionKey { get; set; }
}
```

Luego, tendremos un único método invocable externamente que nos entregará una respuesta. Yo lo llamé *Recognize*.

```csharp
QnAMakerResult qnaResult = QnAMakerRecognizer.Recognize(message, userCulture);
```

Este método *Recognize* itera nuestro enunciado (ó *utterance*, ó pregunta en este caso) por cada QnA Maker hasta encontrar una respuesta que supere un umbral de confianza que nosotros definamos.

Para ello, debemos primero crear nuestras aplicaciones en QnA Maker. Voy a asumir que eso ya lo hiciste, si no es así, visita [QnAMaker.ai](https://qnamaker.ai) y aprende más.

Cuando tengas tus apps creadas, dirígete al archivo *Web.config* de tu aplicación y busca la sección *AppSettings*:

```xml
<appSettings>
<!-- update these with your BotId, Microsoft App Id and your Microsoft App Password-->
    <add key="BotId" value="MultipleQnA" />
    <add key="MicrosoftAppId" value="6870..." />
    <add key="MicrosoftAppPassword" value="Ah3G..." />
    <!-- custom fields for QnA Maker Apps-->
    <add key ="QnaMakerConfidenceThreshold" value="50"/>
    <!-- English-->
    <add key ="QnAMakerAppId_1_en-US" value="41d1..."/>
    <add key ="QnAMakerAppKey_1_en-US" value="bdfa..."/>
    <add key ="QnAMakerAppId_2_en-US" value="ac9e..."/>
    <add key ="QnAMakerAppKey_2_en-US" value="bdfa..."/>
    <!-- Spanish -->
    <add key ="QnAMakerAppId_1_es-ES" value="9b09..."/>
    <add key ="QnAMakerAppKey_1_es-ES" value="bdfa..."/>
    <add key ="QnAMakerAppId_2_es-ES" value="0558..."/>
    <add key ="QnAMakerAppKey_2_es-ES" value="bdfa..."/>
</appSettings>
```

Si te fijas, lo hice con el siguiente patrón: *QnAMakerAppId_* + Número de app + *_* + Cultura

De esta forma, el Id de la primer app en Español sería *QnAMakerAppId_1_es-ES* y la llave de la segunda app en inglés sería *QnAMakerAppKey_2_en-US*.

También notarás un campo *QnaMakerConfidenceThreshold* que estoy utilizando para definir un umbral de confianza. En este caso es 50%.

Ahora, el siguiente paso es instanciar cada una de esas apps e iterar hasta encontrar un buen resultado.

```csharp
public static QnAMakerResult Recognize(string utterance, string userCulture)
{
    //Crear una lista de "apps"
    IList<QnAMakerApp> _apps = new List<QnAMakerApp>();

    //¿Cuántas apps hay para el idioma del usuario?
    int count = System.Configuration.ConfigurationManager.AppSettings.AllKeys
        .Where(x => x.StartsWith("QnAMakerAppId") && x.EndsWith(userCulture))
        .Count();

    //Instanciar una QnAMaker app obteniendo el par Id-Llave para cada una
    for (int i = 1; i <= count; i++)
    {
        string id = System.Configuration.ConfigurationManager.AppSettings.GetValues($"QnAMakerAppId_{i}_{userCulture}").First();
        string key = System.Configuration.ConfigurationManager.AppSettings.GetValues($"QnAMakerAppKey_{i}_{userCulture}").First();

        _apps.Add(new QnAMakerApp() { KnowledgeBaseId = id, SubscriptionKey = key });
    }

    //Iterar por cada una hasta encontrar una buena respuesta
    foreach (QnAMakerApp app in _apps)
    {
        QnAMakerResult result = Recognize(app, utterance);

        //Obtener el umbral pre-definido
        float.TryParse(System.Configuration.ConfigurationManager.AppSettings.GetValues("QnaMakerConfidenceThreshold").First(), out float confidenceThreshold);

        if (result.Score >= confidenceThreshold)
        {
            return result;
        }
    }

    //Ningún resultado superó el umbral, ignoremos esta app
    return null;
}
```

Ahora, debemos ver cómo hacer nuestro pedido REST a la API de QnAMaker. Lo hacemos con el siguiente código:

```csharp
private static QnAMakerResult Recognize(QnAMakerApp app, string utterance)
{
    var responseString = String.Empty;

    QnAMakerResult QnAresponse = null;

    if (utterance.Length > 0)
    {
        var knowledgebaseId = app.KnowledgeBaseId; 
        var qnamakerSubscriptionKey = app.SubscriptionKey; 

        Uri qnamakerUriBase = new Uri("https://westus.api.cognitive.microsoft.com/qnamaker/v1.0");
        var builder = new UriBuilder($"{qnamakerUriBase}/knowledgebases/{knowledgebaseId}/generateAnswer");

        var postBody = $"{{\"question\": \"{utterance}\"}}";

        using (WebClient client = new WebClient())
        {
            client.Encoding = System.Text.Encoding.UTF8;
            client.Headers.Add("Ocp-Apim-Subscription-Key", qnamakerSubscriptionKey);
            client.Headers.Add("Content-Type", "application/json");
            responseString = client.UploadString(builder.Uri, postBody);
        }

        try
        {
            QnAresponse = JsonConvert.DeserializeObject<QnAMakerResult>(responseString);
        }
        catch
        {
            throw new Exception("Unable to deserialize QnA Maker response string.");
        }
    }

    return QnAresponse;
}
```

Un ejemplo de cómo invocarlo desde un diálogo:

```csharp
//"es-ES" para todas las variantes de Español, sino será "en-US"
var userCulture = Thread.CurrentThread.CurrentCulture.Name.StartsWith("es") ? "es-ES" : "en-US";

// Trigger QnA Makers
QnAMakerResult qnaResult = QnAMakerRecognizer.Recognize(message, userCulture);

if (qnaResult != null && !string.IsNullOrEmpty(qnaResult.Answer))
{
    await context.SayAsync($"[**QnA MAKER RESPONSE**]: {qnaResult.Answer}");
}
else
{
    await context.SayAsync(Properties.Resources.QuestionNotFound);
}
```
¡Eso es todo!

Aquí el código completo de *QnAMakerRecognizer*

```csharp
using Microsoft.Bot.Connector;
using Newtonsoft.Json;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Net;
using System.Threading.Tasks;

public static class QnAMakerRecognizer
{
    private class QnAMakerApp
    {
        public string KnowledgeBaseId { get; set; }

        public string SubscriptionKey { get; set; }
    }

    public static QnAMakerResult Recognize(string utterance, string userCulture)
    {
        IList<QnAMakerApp> _apps = new List<QnAMakerApp>();

        int count = System.Configuration.ConfigurationManager.AppSettings.AllKeys
            .Where(x => x.StartsWith("QnAMakerAppId") && x.EndsWith(userCulture))
            .Count();

        for (int i = 1; i <= count; i++)
        {
            string id = System.Configuration.ConfigurationManager.AppSettings.GetValues($"QnAMakerAppId_{i}_{userCulture}").First();
            string key = System.Configuration.ConfigurationManager.AppSettings.GetValues($"QnAMakerAppKey_{i}_{userCulture}").First();

            _apps.Add(new QnAMakerApp() { KnowledgeBaseId = id, SubscriptionKey = key });
        }

        foreach (QnAMakerApp app in _apps)
        {
            QnAMakerResult result = Recognize(app, utterance);

            float.TryParse(System.Configuration.ConfigurationManager.AppSettings.GetValues("QnaMakerConfidenceThreshold").First(), out float confidenceThreshold);

            if (result.Score >= confidenceThreshold)
            {
                return result;
            }
        }

        return null;
    }

    private static QnAMakerResult Recognize(QnAMakerApp app, string utterance)
    {
        var responseString = String.Empty;

        QnAMakerResult QnAresponse = null;

        if (utterance.Length > 0)
        {
            var knowledgebaseId = app.KnowledgeBaseId; 
            var qnamakerSubscriptionKey = app.SubscriptionKey; 

            Uri qnamakerUriBase = new Uri("https://westus.api.cognitive.microsoft.com/qnamaker/v1.0");
            var builder = new UriBuilder($"{qnamakerUriBase}/knowledgebases/{knowledgebaseId}/generateAnswer");

            var postBody = $"{{\"question\": \"{utterance}\"}}";

            using (WebClient client = new WebClient())
            {
                client.Encoding = System.Text.Encoding.UTF8;
                client.Headers.Add("Ocp-Apim-Subscription-Key", qnamakerSubscriptionKey);
                client.Headers.Add("Content-Type", "application/json");
                responseString = client.UploadString(builder.Uri, postBody);
            }

            try
            {
                QnAresponse = JsonConvert.DeserializeObject<QnAMakerResult>(responseString);
            }
            catch
            {
                throw new Exception("Unable to deserialize QnA Maker response string.");
            }
        }

        return QnAresponse;
    }
}
``` 

Cómo mejorar esto: 
+ Definir estrategia: primer match mayor al umbral vs. mayor match 
+ Agregar más fuentes 
+ Portarlo a más lenguajes de programación 

Si decides hacerlo, por favor escribime vía Twitter en @mfelman o por mail a mfelman@microsoft.com 

## Equipo

* [Amin Espinoza de los Monteros](https://github.com/aminespinoza/)
* [Marcelo Felman](https://github.com/marcelofelman/)