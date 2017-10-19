# Bots para soporte interno: reseteo de contraseñas

>**TL;DR** usa este código para crear un bot de reseteo de contraseñas con Microsoft Bot Framework y Active Directory.

Te presentamos un caso real: una empresa de casi 40.000 empleados recibe mensualmente 2.500 llamados a su mesa de ayuda para reestablecer y desbloquear las cuentas de sus empleados. Esto es un proceso costos y poco eficiente, por eso este documento te presenta algunas alternativas sobre cómo resolver esto con un bot.

Algunos detalles antes de empezar:
* Los usuarios, roles y permisos se administran con Microsoft Active Directory (on-premises)
* Existe una conexión con Azure Active Directory a través de AD Connect
* Los usuarios utilizan Skype for Business y Microsoft Teams
* Tecnología de desarrollo preferida es C#

Las primeras ideas que tuvimos:
* Un bot en Skype for Business
* Un bot en Microsoft Teams
* Un bot en una web

Las primeras dos opciones desafortunadamente no funcionaron: ¿cómo puedo desbloquear mi clave desde Skype o Teams si no puedo iniciar sesión en mi cuenta?

## Cómo funciona

Para este escenario, el flujo de conversación es el siguiente:

1. **Usuario**: Hola, quiero reestablecer mi contraseña.
2. Bot: Hola. Empecemos, ¿cómo es tu correo electrónico?
3. **Usuario**: marcelo@marcelo.com
*(Detrás de escenas: el bot chequea la existencia del usuario en el directorio)*
4. Bot: Excelente. Tengo registrado un correo alternativo que es ma****@out****.com. ¿Es correcto?
5. **Usuario**: Si
6. Bot: Ok, tu nueva contraseña será enviada a ese correo en este instante. ¿Algo más en que pueda ayudarte?
*(Detrás de escenas: el bot genera una nueva contraseña y la envía al correo alternativo del usuario)*
7. **Usuario**: Eso es todo. Gracias!
8. Bot: Adiós!

Aparentemente es simple. Estos son los pasos que debemos seguir para su construcción:
1. Crear un bot que soporte el diálogo de arriba
2. Integrar el bot con Active Directory
3. Publicar el bot con el Microsoft Bot Framework Connector
4. Generar la web y embeber el control Web Chat

## Crear el bot

No vamos a deternos mucho en esta parte, ya que hay mucho contenido online sobre esto. Este es el código base que puedes utilizar para un diálogo de reestablecimiento de contraseñas como el de arriba

```csharp
public static void elCodigo
{
    //Poner aquí el diálogo..
}
```

Si quieres ver el código completo, está en este otro repositorio **AGREGAR REPO!!!!**

No te olvides de probar el bot localmente con el [Bot Framework Emulator](https://github.com/Microsoft/BotFramework-Emulator) antes de seguir al siguiente paso.

## Integrar con Active Directory

El siguiente código resuelve toda la integración con Azure Active Directory. Dado que podíamos elegir entre Active Directory (on-premises) y Azure Active Directory, optamos por el segundo ya que era mucho más fácil. Siéntete libre de copiar y pegar, no le contaremos a nadie :)

```csharp
public static void elCodigo
{
    //Poner aquí el código de AD..
}
```

## Publicar el bot con el Microsoft Framework Connector

Esta parte es la más simple de todas:
1. Si no tienes, creáte una cuenta en [dev.botframework](https://dev.botframework.com/)
2. Registra tu nuevo bot, obtén un App Id y Key
3. Completa esos campos en tu aplicación
4. En Visual Studio, click derecho en el proyecto > Publish.. y súbelo a Azure
5. Habilita el conector web

Este es un resumen súper ligero, pero tienes más detalles en la [documentación oficial](https://docs.microsoft.com/en-us/bot-framework/portal-configure-channels#publish-a-bot)

## Generar la web y embeber el control web

De hecho no hay que hacer nada, ya que este control viene activado por defecto. Simplemente busca en dev.botframework el código HTML necesario para embeber el control web.

Debería verse como el siguiente:
```html
<iframe src='https://webchat.botframework.com/embed/multipleqna?s=YOUR_SECRET_HERE'></iframe>
```
Si quieres más detalle, mira [este link](https://docs.microsoft.com/en-us/bot-framework/channel-connect-webchat).

## ¡Funciona!

Este es el producto final de nuestro hermoso bot.

**PEGAR FOTO!!**


## Equipo

* Amin Espinoza de los Monteros
* Marcelo Felman