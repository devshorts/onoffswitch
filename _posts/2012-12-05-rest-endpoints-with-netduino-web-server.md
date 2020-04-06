---
layout: post
title: RESTful web endpoints on Netduino Plus
date: 2012-12-05 16:04:59.000000000 -08:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Code
- Imported
tags:
- c#
- netduino
meta:
  _syntaxhighlighter_encoded: '1'
  _edit_last: '1'
  dsq_thread_id: '960058829'
  _oembed_6644b5b4a8d30fa08744bc9c40b01817: "{{unknown}}"
  _su_title: ''
  _jetpack_related_posts_cache: a:1:{s:32:"8f6677c9d6b0f903e98ad32ec61f8deb";a:2:{s:7:"expires";i:1561690990;s:7:"payload";a:3:{i:0;a:1:{s:2:"id";i:4939;}i:1;a:1:{s:2:"id";i:3392;}i:2;a:1:{s:2:"id";i:4919;}}}}
  _oembed_05817a7a6a9145d7919112c8afdc35ce: "{{unknown}}"
  _oembed_44de245a5cb4857f2c9dd76d6025a72f: "{{unknown}}"
author:
  login: akropp
  email: akropp@gmail.com
  display_name: akropp
  first_name: ''
  last_name: ''
permalink: "/2012/12/05/rest-endpoints-with-netduino-web-server/"
---
_This article was originally published at [tech.blinemedical.com](http://tech.blinemedical.com/rest-endpoints-with-netduino-web-server/)_

I have a [Netduino plus](http://www.Netduino.com/Netduinoplus/specs.htm) at home and I love it. Not only can you use C# to write for it, but you get full visual studio integration including live breakpoints! I got the Netduino plus over the Netduino because the Netduino plus has a built in ethernet jack and ethernet stack support. This way I could access my microcontroller over the web if I wanted to (and who wouldn't?).

But to expose your Netduino to the web you need to write a simple web server. Basically open a socket at port 80 and read/write requests out to it. The Netduino community is great at sharing code and I quickly found a nice web server by [Jasper Schuurmans](http://www.schuurmans.cc/multi-threaded-web-server-for-Netduino-plus). His server code let you define [RESTful](http://en.wikipedia.org/wiki/Representational_state_transfer) routes like this

[csharp]  
http://NetduinoIPAddress/targetFunction/arg1/arg2/...  
[/csharp]

Which was super cool. It even filtered out non-registered commands, allowing you to control what requests would trigger a "_command found_" event. Here is the basic main of his demo.

[csharp]  
public static void Main()  
{  
 // Instantiate a new web server on port 80.  
 WebServer server = new WebServer(80);

// Add a handler for commands that are received by the server.  
 server.CommandReceived += new WebServer.CommandReceivedHandler(server\_CommandReceived);

// Add a command that the server will parse.  
 // Any command name is allowed; you will decide what the command does  
 // in the CommandReceived handler. The server will only fire CommandReceived  
 // for commands that are defined here and that are called with the proper  
 // number of arguments.  
 // In this example, I define a command 'SetLed', which needs one argument (on/off).  
 // With this statement, I defined that we can call our server on (for example)  
 // http://[server-ip]/SetLed/on  
 // http://[server-ip]/SetLed/off  
 server.AllowedCommands.Add(new WebCommand("SetLed", 1));

// Start the server.  
 server.Start();

// Make sure Netduino keeps running.  
 while (true)  
 {  
 Debug.Print("Netduino still running...");  
 Thread.Sleep(10000);  
 }  
}

/// \<summary\>  
/// Handles the CommandReceived event.  
/// \</summary\>  
private static void server\_CommandReceived(object source, WebCommandEventArgs e)  
{

Debug.Print("Command received:" + e.Command.CommandString);

switch (e.Command.CommandString)  
 {  
 case "SetLed":  
 {  
 // Do you stuff with the command here. Set a led state, return a  
 // sampled value of an analog input, whatever.  
 // Use the ReturnString property to (optionally) return something  
 // to the web user.

// Read led state from command and set led state.  
 bool state = ( e.Command.Arguments[0].Equals("on") ? true : false);  
 onBoardLed.Write(state);

// Return feedback to web user.  
 e.ReturnString = "\<html\>\<body\>You called SetLed with argument: " + e.Command.Arguments[0].ToString() + "\</body\>\</hmtl\>";  
 break;  
 }  
 }  
}  
[/csharp]

While this certainly works, there were a few things I didn't like about this setup:

- You have to route the logic from a single switch statement. If you were building more than one restful endpoint in your Netduino, this centralized switch statement would get messy.
- You have to declare the target argument length when registering the command. This means that if you update the target function's argument parameters, you also have to update the registration code.
- The server was single-threaded. It uses events to alter program flow. But since events execute in the dispatchers thread, if your execution code took a while, you basically stalled the entire server.
- REST endpoints were actually case sensitive

# The reworked final copy

Before we dig into what I changed, lets look at my final reworked main and you can compare it to the original main I posted above:

[csharp]  
public static void Main()  
{  
 LcdWriter.Instance.Write("Web Demo Ready!" + DateTime.Now.TimeOfDay);

WebServerWrapper.InitializeWebEndPoints(new ArrayList  
 {  
 new BasicPage()  
 });

WebServerWrapper.StartWebServer();

RunUtil.KeepRunning();  
}  
[/csharp]

Here, `BasicPage` is an object that encapsulates its route definitions as well as what to invoke when a target route is found. Next, I'm registering the object with a web service wrapper and then starting the web server. This way, I've removed the command handling from our main loop and encapsulated logic into individual components.

# Injecting endpoints

In order to get rid of the central switch statement, I wanted to encapsulate all the logic of endpoint name, endpoint arguments, and target function to invoke in a single object. This would let me build a single class whose sole job was to be executed when the web server routed it the command. On top of that, you now can cleanly maintain endpoint state and other information all within a single object. So, if you were building an endpoint whose job is to show you the temperature of your refrigerator over the last 3 hours, you can store that information in your endpoint object and when the endpoint is invoked, print out some nice html that shows the current and historical data.

As an example, let's create a class that prints whatever arguments were received from the server onto a connected LCD. First we'll have it implement a target interface called `IEndPointProvider` which looks like this:

[csharp]  
public interface IEndPointProvider  
{  
 void Initialize();  
 ArrayList AvailableEndPoints();  
}  
[/csharp]

- `Initialize` would be class specific initialization logic. If we don't need to use resources until we are about to fire up the server then we can put that logic into here.
- `AvailableEndPoints` is a list of `EndPoints` that we can use to register with the server. In case you're wondering about the `ArrayList`, .NET Micro [doesn't support generics](http://informatix.miloush.net/Microframework/Articles/CisFeatures.aspx), so we're not using something like `List<T>`

And here is my implementation of `IEndPointProvider` which echos the arguments to a connected LCD:

[csharp]  
public class BasicPage : IEndPointProvider  
{  
 #region Endpoint initialization

public void Initialize() { }

public ArrayList AvailableEndPoints()  
 {  
 var list = new ArrayList  
 {  
 new EndPoint  
 {  
 Action = Echo,  
 Name = "echoArgs",  
 Description = "Writes the URL arguments to a serial LCD hooked up to COM1"  
 }  
 };  
 return list;  
 }

#endregion

#region Endpoint Execution

private string Echo(EndPointActionArguments misc, string[] items)  
 {  
 String text = "";  
 if (items != null && items.Length \> 0)  
 {  
 foreach (var item in items)  
 {  
 text += item + " ";  
 }  
 }  
 else  
 {  
 text = "No arguments!";  
 }

LcdWriter.Instance.Write(text);

return "OK. Wrote out: " + (text.Length == 0 ? "n/a" : text);  
 }

#endregion  
}  
[/csharp]

You can see that we're exposing an array list of `EndPoint` objects that define the action to execute, what the target action's name is (i.e. the REST endpoint), and a short description about what the endpoint does (for an API listing we can create later).

The target function `Echo` takes an `EndPointActionArguments` object that contains some state about the current connection, and a list of objects representing the variable arguments to the REST endpoint.

# End point

Let's take a look at what an endpoint is.

[csharp]  
public delegate string EndPointAction(EndPointActionArguments arguments, params string[] items);

public class EndPointActionArguments  
{  
 public Socket Connection { get; set; }  
}

public class EndPoint  
{  
 private string[] \_arguments;

public bool UsesManualSocket { get; set; }

public string Description { get; set; }

/// \<summary\>  
 /// The function to be called when the endpoint is hit  
 /// \</summary\>  
 public EndPointAction Action  
 {  
 private get; set;  
 }

/// \<summary\>  
 /// The name of the endpoint, this is basically the servers route  
 /// \</summary\>  
 public String Name { get; set; }

public string[] Arguments { set { \_arguments = value; } }

/// \<summary\>  
 /// Execute this endpoint. We'll call the action with the supplied arguments and  
 /// return whatever string the action returns.  
 /// \</summary\>  
 /// \<returns\>\</returns\>  
 public String Execute(EndPointActionArguments misc)  
 {  
 if (Action != null)  
 {  
 return Action(misc, \_arguments);  
 }  
 return "Unknown action";  
 }  
}  
[/csharp]

An `EndPoint` has a delegate named `Action` for a function with a signature

[csharp]  
string Foo(EndPointActionArguments arguments, params string[] items)  
[/csharp]

The `Action` would return a string that the web server will then write back out onto the target socket. We also pass an `EndPointActionArguments` to the delegate which contains a reference to the original socket request (outgoing to the client) and serves as encapsulation if we want to add more parameters to send through to the endpoint later. The last argument is a variable list of strings that relates to the REST url argument list.

An endpoint `Description` defines what the endpoint does; we'll use this to describe the endpoint in a default API listing if the server gets a request it doesn't know about.

`UseManualSocket` is a boolean that will indicate to the server that the endpoint handled the socket request manually (i.e. it held onto the request) and that the server shouldn't close the socket; the endpoint will deal with socket cleanup.

# Getting the endpoint to the server

Now that I've encapsulated action/state information into a single class, I wrapped Jasper's original web server with a new facade. The facade will hide some of the internals of the server such as starting the web server, registering endpoints (from `IEndPointProvider` instances), and provides a single entry point for found commands. When we start the server we'll pass along our registered endpoints with the actual server. If we wanted to do more endpoint manipulation later, we now have a centralized point of access before the endpoints get to the server.

Keeping with the original event dispatching mechanism, I moved the handling of the `EndPointReceived` event into the wrapper and out of the main program.

[csharp]  
/// \<summary\>  
/// Wrapper class on top of a multi threaded web server  
/// Allows classes to register REST style endpoints  
/// \</summary\>  
public static class WebServerWrapper  
{  
 private static WebServer \_server;  
 private static ArrayList \_endPoints;

/// \<summary\>  
 /// Register REST endpoint for callback invocation with the web server  
 /// \</summary\>  
 /// \<param name="endPoints"\>\</param\>  
 private static void RegisterEndPoints(ArrayList endPoints)  
 {  
 if(\_endPoints == null)  
 {  
 \_endPoints = new ArrayList();  
 }

foreach(var endPoint in endPoints)  
 {  
 \_endPoints.Add(endPoint);  
 }  
 }

public static void InitializeWebEndPoints(ArrayList items)  
 {  
 foreach (IEndPointProvider endpoint in items)  
 {  
 endpoint.Initialize();  
 RegisterEndPoints(endpoint.AvailableEndPoints());  
 }  
 }

/// \<summary\>  
 /// Start listening on the port and enable any registered callbacks  
 /// \</summary\>  
 /// \<param name="port"\>\</param\>  
 /// \<param name="enabledLedStatus"\>\</param\>  
 public static void StartWebServer(int port = 80, bool enabledLedStatus = true)  
 {  
 \_server = new WebServer(port, enabledLedStatus);

\_server.EndPointReceived += EndPointHandler;

foreach (EndPoint endpoint in \_endPoints)  
 {  
 \_server.RegisterEndPoint(endpoint);  
 }

// Initialize the server.  
 \_server.Start();  
 }

/// \<summary\>  
 /// We'll get an endpoint invocation from the web server  
 /// so we can execute the endpoint action and response based on its supplied arguments  
 /// in a separate thread, hence the event. we'll set the event return string  
 /// so the web server can know how to respond back to the ui in a seperate thread  
 /// \</summary\>  
 /// \<param name="source"\>\</param\>  
 /// \<param name="e"\>\</param\>  
 private static void EndPointHandler(object source, EndPoinEventArgs e)  
 {  
 var misc = new EndPointActionArguments  
 {  
 Connection = e.Connection  
 };

e.ReturnString = e.Command.Execute(misc);

// we can override the manual use of the socket if we returned a value other than null  
 if (e.ReturnString != null && e.Command.UsesManualSocket)  
 {  
 e.ManualSent = false;  
 }  
 else  
 {  
 e.ManualSent = e.Command.UsesManualSocket;  
 }  
 }  
}  
[/csharp]

# A few web server changes

Jaspers web server is simple and ingenious. I like it's simplicity and it was easy to extend. When the web server receives a request, it parses the first line of a raw http GET from the header to figure out it's "route". As an example, here is a request I generated for _http://localhost/function/arg1/arg2_. Everything after the first line is discarded since we just care about the _/function/arg1/arg2_ part

[csharp]  
GET /function/arg1/arg2 HTTP/1.1  
Host: localhost  
User-Agent: Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.11 (KHTML, like Gecko) Chrome/23.0.1271.64 Safari/537.11  
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,\*/\*;q=0.8  
Accept-Encoding: gzip,deflate,sdch  
Accept-Language: en-US,en;q=0.8  
Accept-Charset: ISO-8859-1,utf-8;q=0.7,\*;q=0.3  
Cookie: ASP.NET\_SessionId=ue1s3blzxxwbrrohasgwpbbv  
[/csharp]

Once it has the right request url from the header, the server will see if any registered endpoint `Name` property matches the request name. If it does it'll parse the remaining arguments. This all happens in `InterpretRequest`. I didn't change any of this logic. What I changed was what `InterpretRequest` returns and how the final command is dispatched. Here is the main server listening loop:

[csharp]  
using (var server = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp))  
{  
 server.Bind(new IPEndPoint(IPAddress.Any, Port));

server.Listen(1);

while (!\_cancel)  
 {  
 var connection = server.Accept();

if (connection.Poll(-1, SelectMode.SelectRead))  
 {  
 // Create buffer and receive raw bytes.  
 var bytes = new byte[connection.Available];

connection.Receive(bytes);

// Convert to string, will include HTTP headers.  
 var rawData = new string(Encoding.UTF8.GetChars(bytes));

//====================================  
 // My changes begin here  
 //====================================  
 EndPoint endPoint = InterpretRequest(rawData);

if (endPoint != null)  
 {  
 if (\_enableLedStatus)  
 {  
 PingLed();  
 }

// dispatch the endpoint  
 var e = new EndPoinEventArgs(endPoint, connection);

if (EndPointReceived != null)  
 {  
 ThreadUtil.SafeQueueWorkItem(() =\>  
 {  
 EndPointReceived(null, e);

if (e.ManualSent)  
 {  
 // the client should close the socket  
 }  
 else  
 {  
 var response = e.ReturnString;

SendResponse(response, connection);  
 }  
 });  
 }  
 }  
 else  
 {  
 // if we didn't match a response return with the generic API listing  
 SendResponse(GetApiList(), connection);  
 }  
 }

}  
}  
[/csharp]

What I modified from the original server code was

- InterpretRequest now returns an `EndPoint` with a string array of arguments. Previously, it looked for only the number of arguments that were registered to it. Now it parses as much as is there giving you a clean variable argument list.
- Events are now dispatched in a [custom threadpool](https://github.com/blinemedical/NWebREST/blob/master/NetDuinoUtils/Utils/ThreadUtil.cs), since [.NET Micro doesn't have any](http://netmf.codeplex.com/workitem/78) built in threadpooling. The threadpool is a collection of 3 threads that pull off an event queue and execute. This way the web server is asynchronous and won't ever block for other requests. You could easily just have it fire off independent threads if you wanted to, but I found a threadpool to be more effective since you don't need to spin up new threads (and allocate extra thread stack space) each time.
- If a request comes in that doesn't match any endpoint, we'll print out all the available endpoints with their description. This is a nice API listing for you.
- Event arguments contain a reference to the original socket if you need it.
- If an endpoint is going to to manually write to the socket and close the socket later, it can set the `UsesManualSocket` flag on registration. The wrapper then tells the server that the executed endpoint manually sent data to the socket and is expected to close it. This can be useful if you want to maintain a persistent connection in your endpoint (maybe you are streaming something per client). By default the server will write out the string response from the endpoint and close the socket.
- Optionally pulse the onboard LED whenever a request comes in. This is useful for debugging and viewing activity.
- I updated the code that searched for endpoint name and compared it to url request to be case insensitive. Even though the [w3c spec](http://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html#sec3.2.3) says comparing urls should be case sensitive (with some exceptions), [by convention](http://stackoverflow.com/questions/778203/are-there-any-naming-convention-guidelines-for-rest-apis) REST endpoints are case insensitive

# Review

Lets take a look again at our main program block.

[csharp]  
public static void Main()  
{  
 LcdWriter.Instance.Write("Web Demo Ready!" + DateTime.Now.TimeOfDay);

WebServerWrapper.InitializeWebEndPoints(new ArrayList  
 {  
 new BasicPage()  
 });

WebServerWrapper.StartWebServer();

RunUtil.KeepRunning();  
}  
[/csharp]

You can see that we've now decoupled public interaction with the web server, as well as allow each class to define whatever routes it wants. If we wanted to rename the route `EchoArgs` and have it point to another function, it'd be trivial to change that within `BasicPage`. If we wanted `BasicPage` to implement two functions such as `EchoArgs` and `BlinkLEDABunch` we could do that, all without having to update our main entrypoint.

Just to recap, the basic pattern here is:

[![Program Flow Diagram](http://tech.blinemedical.com/wp-content/uploads/2012/11/flow-300x179.png)](http://tech.blinemedical.com/wp-content/uploads/2012/11/flow.png)

- First register all `IEndPointProvider`s with the web server wrapper.
- Then start web server.
- When a request comes in, the server will find a matching endpoint by name and dispatch the `EndPointReceived` event which is caught by the wrapper.
- The wrapper executes the target endpoint in a separate thread and returns the endpoints result.

From a users perspective, you just create your class, expose your endpoint, and everything works.

# Demo

Firing up the app

[![Netduion output: starting app](http://tech.blinemedical.com/wp-content/uploads/2012/11/netduinoOutput1-e1353965295230-300x225.jpg)](http://tech.blinemedical.com/wp-content/uploads/2012/11/netduinoOutput1-e1353965312698.jpg)

Using curl to send some arguments

[csharp]  
\>curl http://192.168.2.11/echoargs/heyguys!/whatsup!  
OK. Wrote out: heyguys! whatsup!  
[/csharp]

Results on the Netduino

[![Netduion output: endpoint executed](http://tech.blinemedical.com/wp-content/uploads/2012/11/netduinoOutput2-e1353965375217-300x225.jpg)](http://tech.blinemedical.com/wp-content/uploads/2012/11/netduinoOutput2-e1353965375217.jpg)

The API listing (this prints when no known route was found)

[![The API listing](http://tech.blinemedical.com/wp-content/uploads/2012/11/apiList.-300x255.png)](http://tech.blinemedical.com/wp-content/uploads/2012/11/apiList..png)

# The source

Full source and demo code available at our [github](https://github.com/blinemedical/NWebREST). Note, the project is built against .net micro 4.2. I've run the code on .net micro 4.1 and 4.2 and everything worked fine. For reference, currently my Netduino is on firmware 4.2.0.0. RC3, though I'm not relying on any major framework specific choices here so it should continue to work fine for later revisions.

