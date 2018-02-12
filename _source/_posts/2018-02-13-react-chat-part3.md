---
layout: post
title: "Creating a chat application using React and ASP.NET Core - Part 3"
teaser: "In this blog series I'm going to create a small chat application using React and ASP.NET Core, to learn more about React and to learn how React behaves in an ASP.NET Core project during development and deployment. This post describes how to add SignalR to the chat UI and get it working."
author: "Jürgen Gutsch"
comments: true
image: /img/cardlogo-dark.png
tags: 
- ASP.NET Core
- React
- TypeScript
---

In this blog series, I'm going to create a small chat application using React and ASP.NET Core, to learn more about React and to learn how React behaves in an ASP.NET Core project during development and deployment. This Series is divided into 5 parts, which should cover all relevant topics:

1. [React Chat Part 1: Requirements & Setup]({% post_url react-chat-part1.md %})
2. [React Chat Part 2: Creating the UI & React Components]({% post_url react-chat-part2.md %})
3. **React Chat Part 3: Adding Websockets using SignalR**
4. React Chat Part 4: Authentication & Storage
5. React Chat Part 5: Deployment to Azure

I also set-up a GitHub repository where you can follow the project: [https://github.com/JuergenGutsch/react-chat-demo](https://github.com/JuergenGutsch/react-chat-demo). Feel free to share your ideas about that topic in the comments below or in issues on GitHub. Because I'm still learning React, please tell me about significant and conceptual errors, by dropping a comment or by creating an Issue on GitHub. Thanks.

## About SignalR

SignalR for ASP.NET Core is a framework to enable Websocket communication in ASP.NET Core applications. Modern browsers already support Websocket, which is part of the HTML5 standard. For older browser SignalR provides a fallback based on standard HTTP1.1. SignalR is basically a server side implementation based on ASP.NET Core and Kestrel. It uses the same dependency injection mechanism and can be added via a NuGet package into the application. Additionally, SignalR provides various client libraries to consume Websockets in client applications. In this chat application, I use `@aspnet/signalr-client` loaded via NPM. The package also contains the TypeScript definitions, which makes it easy to use in a TypeScript application, like this.

I added the React Nuget package in the first part of this blog series. To enable SignalR we need to add it to the ServiceCollection:

~~~ csharp
services.AddSignalR();
~~~



## The server part

In C#, I created a `ChatService` that will later be used to connect to the data storage. Now it is using a dictionary to store the messages and is working with this dictionary. I don't show this service here, because the implementation is not relevant here and will change later on. But I will use this Service in in the code I show here. Also in the `ChatHub`, which is the Websocket endpoint for this chat. The service gets injected via dependency injection that is configured in the `Startup.cs`:

~~~ csharp
services.AddSingleton<IChatService, ChatService>();
~~~

The Websocket endpoints are defined in so called Hubs. One Hub is defining one single Websocket endpoint. I created a `ChatHub`, that is the endpoint for this application. The methods in the `ChatHub` are handler methods, to handle incoming messages through a specific channel. 

The `ChatHub` needs to be added to the SignalR middleware:

~~~ csharp
app.UseSignalR(routes =>
{
    routes.MapHub<ChatHub>("chat");
});
~~~



> A SignalR Methods in the Hub are the channel definitions and the handlers at the same time, while NodeJS socket.io is defining channels and binds an handler to this channel.

The currently used data are still fake data and authentication is not yet implemented. This is why the users name is hard coded yet:

~~~ csharp
using Microsoft.AspNetCore.SignalR;
using ReactChatDemo.Services;

namespace ReactChatDemo.Hubs
{
    public class ChatHub : Hub
    {
        private readonly IChatService _chatService;

        public ChatHub(IChatService chatService)
        {
            _chatService = chatService;
        }

        public void AddMessage(string message)
        {
            var chatMessage = _chatService.CreateNewMessage("Juergen", message);
            // Call the MessageAdded method to update clients.
            Clients.All.InvokeAsync("MessageAdded", chatMessage);
        }
    }
}
~~~

This Hub only contains a method `AddMessage`, that gets the actual message as a string. Later on we will replace the hard coded user name, with the name of the logged on user. Than a new message gets created and also added to the data store via the `ChatService`. The new message is an object, that contains a unique id, the name of the authenticated user, a create date and the actual message text.

Than the message gets, send to the client through the Websocket channel "MessageAdded".

## The client part

On the client side, I want to use the socket in two different components, but I want to avoid to create two different Websocket clients. The idea is to create a `WebsocketService` class, that is used in the two components. Usually I would create two instances of this `WebsocketService`, but this would create two different clients too. I need to think about dependency injection in React and a singleton instance of that service. 

While googling for dependency injection in React , I read a lot about the fact, that DI is not needed in React. I was kinda confused. DI is everywhere in Angular, but it is not necessarily needed in React? There are packages to load, to support DI, but I tried to find another way. And actually there is another way. In ES6 and in TypeScript it is possible to immediately create an instance of an object and to import this instance everywhere you need it.

~~~ typescript
import { HubConnection, TransportType, ConsoleLogger, LogLevel } from '@aspnet/signalr-client';

import { ChatMessage } from './Models/ChatMessage';

class ChatWebsocketService {
    private _connection: HubConnection;

    constructor() {
        var transport = TransportType.WebSockets;
        let logger = new ConsoleLogger(LogLevel.Information);

        // create Connection
        this._connection = new HubConnection(`http://${document.location.host}/chat`,
            { transport: transport, logging: logger });
        
        // start connection
        this._connection.start().catch(err => console.error(err, 'red'));
    }

    // more methods here ...
   
}

const WebsocketService = new ChatWebsocketService();

export default WebsocketService;
~~~

Inside this class the Websocket client, called `HubConnection` gets created and configured. The transport type needs to be WebSockets. Also a `ConsoleLogger` gets added to the Client, to send log information the the browsers console. In the last line of the constructor, I start the connection and add an error handler, that writes to the console. The instance of the connections is stored in a private variable inside the class. Right after the class I create an instance and export the instance. This way the instance can be imported in any class:

~~~ typescript
import WebsocketService from './WebsocketService'
~~~

To keep the `Chat` component and the `Users` component clean, I created additional service classes for each the components. This service classes encapsulated the calls to the Web API endpoints and the usage of the `WebsocketService`. Please [have a look into the GitHub repository](https://github.com/JuergenGutsch/react-chat-demo) to learn more about these services.

The `WebsocketService` contains three methods. One is to handle incoming messages, when a user logged on the chat:

~~~ typescript
registerUserLoggedOn(userLoggedOn: (id: number, name: string) => void) {
    // get new user from the server
    this._connection.on('UserLoggedOn', (id: number, name: string) => {
        userLoggedOn(id, name);
    });
}
~~~

This is not yet used. I need to add the authentication first.

The other two methods are to send a chat message to the server and to handle incoming chat messages:

~~~ typescript
registerMessageAdded(messageAdded: (msg: ChatMessage) => void) {
    // get nre chat message from the server
    this._connection.on('MessageAdded', (message: ChatMessage) => {
        messageAdded(message);
    });
}
sendMessage(message: string) {
    // send the chat message to the server
    this._connection.invoke('AddMessage', message);
}
~~~

Now it is time to try it out. F5 starts the application and opens the configured browser:

![]({{site.baseurl}}/img/react-chat-app/working-chat-SignalR.PNG)

This is almost the same view as in the last post about the UI. To be sure React is working, I had a look into the network tap in the browser developer tools:

![]({{site.baseurl}}/img/react-chat-app/working-chat-SignalR-network.PNG)

Here it is. Here you can see the message history of the web socket endpoint. The second line displays the message sent to the server and the third line is the answer from the server containing the `ChatMessage` object.

## Closing words

This post was less easy than the posts before. Not the technical part, but I refactored the the client part a little bit to keep the React component as simple as possible. For the functional components, I used regular TypeScript files and not the TSX files. This worked great.

I'm still impressed by React. 

In the next post I'm going to add Authorization to get the logged on user and to authorize the chat to logged-on users only. I'll also add a permanent storage for the chat message.