---
layout: post
title: "Tambero - Digital transformation in the farm"
author: "Marcelo Felman"
author-link: "https://twitter.com/mfelman"
date: 2017-09-07
categories: [Bot Framework]
color: "blue"
image: "images/projectname/feat_image.png" #must be 440px wide
excerpt: Farmers around the world still use pen and paper to track everything in the farms. Learn how Tambero.com uses Microsoft Bot Framework, Cognitive Services and Skype to build a bot that guides farmers into an improved performance of up to 30%!
language: English
verticals: The industry on which this article focuses; choose from the following: ["Agriculture, Forestry & Fishing"]
geolocation: The geolocation of the article; choose one of the following: [South America]
---
# Tambero - Digital Transformation in the farm #

## Summary ##

Tambero.com is the leading cattle management software. They needed to make it extremely easy for farmers to move from pen and pencil straight into the cloud to improve production and yields in the farm.

This solution implements a Skype bot connected to natural language processing engines that can handle many different conversations that can take place in the farm. This brand new conversational experience connects to Tambero's original mobile application, offering a new channel to use the software.

## Key technologies ##

- [Microsoft Bot Framework](https://dev.botframework.com)
- [Bot Builder SDK for .NET](https://docs.microsoft.com/en-us/bot-framework/dotnet/bot-builder-dotnet-overview)
- [Language Understanding Intelligent Service](https://www.luis.ai)
- [Skype for Developers](https://dev.skype.com/bots)

## Customer profile ##

- [Tambero](https://www.tambero.com) is the most widely distributed cattle management software in the world.
- Over 200.000 users from 220 countries.
- Based in Córdoba, Argentina.
- Tambero offers web, mobile and now also conversational applications.

## Solution overview ##

On a high level, there are five basic components that interact on this solution. Each one of them has a very clear role, and has the ability to scale separately.

![Tambero architecture diagram](https://github.com/marcelofelman/case-studies/blob/master/images/1-tambero-architecture.png?raw=true)

We'll dig deeper into each one of them.

### Natural Language Processing ###

Natural Language Processing (NLP) plays a key role in this solution. It allows users express themselves in a natural or *everyday* way, and the software can understand their intentions. For NLP, we used [Language Understanding Intelligent Service](https://www.luis.ai) (LUIS) which allows for a quick set-up and development, as well as the power to scale to thousands of users.

LUIS allowed us to perform to tasks on every sentence or utterance:

- **Extract intents**: these are like verbs in sentences. *What does the user want to do?*
- **Extract entities**: these are like nouns in sentences. They represent objects relevant to this sentence.

Here's a real example from Tambero:

> *I want to know the pregnancy status for Animal 52*

The extracted intent is *GetPregnancyStatus*, an action name we previously defined; and *Animal 52* is an extacted entity as well.

Following this logic, we created many different intents and entities. A few intents:

- *GetPregnancyStatus*: obtains pregnancy status for an animal.
- *GetAverageWeight*: obtains average weight for a herd.
- *GetPortionSize*: obtains food portion size for an animal.

You could think of the above simply as the methods that your application supports.

A few entities:

- *AnimalNumber*
- *HerdNumber*

You could think of the above simply as the parameters that the methods of your application supports.

Finally, imagine that both the intent and the entity can translate the user's input into an API-friendly input, such as *GetPregnancyStatus(52)* or whatever you'd like it to be.

To train LUIS, you simply give it examples and tag the intents and entities accordingly. To learn better how to train LUIS, you can follow [this tutorial](https://docs.microsoft.com/en-us/azure/cognitive-services/LUIS/add-intents).

Currently, Tambero's bot allows for detecting 27 different intents and 5 entities.

### Handling conversations ###

Microsoft's Bot Builder has versions for C# and Javascript, and for this project we are using the C# version. As this component simply runs on a Web App, it is very simple to make an integration to 3rd party components: it's just C# code!

This way, the bot connects to Tambero's API via REST Calls. Most of the intents that LUIS can detect are directly mapped with API methods.

Here's how it works with a real example:

![Tambero architecture diagram](https://github.com/marcelofelman/case-studies/blob/master/images/3-tambero-nlp.png?raw=true)

### Connecting to Skype ###

One of the most important features of this bot, is the availability on popular communication channels such as Skype. Through the Bot Connector, this bot is connected with Skype and therefore with end-users.

Here's how this connection works:

![Tambero Skype Connection](https://github.com/marcelofelman/case-studies/blob/master/images/4-skype-connection.png?raw=true)

It's very important to realize that the connection between the Skype API and the Bot Connector happens behind the scenes. This will save you lots of time implementing different APIs.

Furthermore, you can connect to several different channels that are already available, such as Slack, Teams or even SMS. Using the Bot Connector allows for fast development on multiple channels. Truly multi-platform.

If you'd like to learn how to connect your bot to Skype, you can follow this [guide](https://dev.skype.com/bots).

### Authentication ###

Authentication was one of the main challenges we had to solve. It is very important that you make sure that the user who is chatting with the bot, is effectively who they claim to be.

This is the authentication workflow we implemented to tackle this:

**ARMAR GRAFICO**
User makes a request that needs authentication -> Bot sends a link to log-in to Tambero.com -> User enters credentials on Tambero.com -> Tambero.com provides an authentication token -> Users pastes authentication code on bot -> Bot cross-checks that pasted code and generated code match

There are other ways to do this, and you could even do the token exchange behind the scenes, as many other apps do. We thought that giving that extra layer would help users feel more secure about this credential exchange.

## Conclusion ##

Conversational bots are a different way to connect with end-users beyond web apps or mobile apps. Tambero is taking one step forward in the digital transformation that is happening in the farm.

Using a combination of Microsoft Bot Framework, Cognitive Services and Skype, we were able to build a break-through solution that allows farmers to manage their cattle straight from their phones with a personalized experience. Always keep in mind that challenges such as authentication or integration could emerge, so be ready to research and code!

Here's a video of the bot's press release held in Buenos Aires, Argentina:

**PEGAR VIDEO**

On Tambero's own words:
**CITA DE EDDIE**

## Additional resources ##

- [Microsoft Bot Framework Documentation](https://dev.botframework.com)
- [Bot Builder for C#](https://github.com/Microsoft/BotBuilder/tree/master/CSharp)
- [Microsoft Cognitive Services](https://azure.microsoft.com/en-us/services/cognitive-services/)
- [Tutorial: Train LUIS](https://docs.microsoft.com/en-us/azure/cognitive-services/LUIS/add-intents)
- [Skype for Developers](https://dev.skype.com/bots)

## Team ##

*Include Microsoft and partner team members (denote company), GitHub IDs, and photo if desired.*