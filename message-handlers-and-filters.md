# Message Handlers and Filters

In this chapter, we will cover:

*	Exploring the Message Handler Mechanism
*	Creating Custom Message handlers
*	Registering Custom Message Handlers
*	Implementing Filters
*	Registering Filters
*	Differentiating between Message handler and Filters


## Exploring the Message Handler Mechanism

Message handlers are a set of classes which intercept Http request and response through a pipeline. The pipeline for Web API message handlers is bidirectional. They process Http request message on the way in, and Http response message on the way out. 

Typically, a series of message handlers are chained together. Every handler receives the Http request sequentially, does some processing if needed, gives the request to next handler until any handler generates a response, at which point the response message flows back out of the pipeline.

# IMAGE INSERT

Web host gives the request to **HttpServer** which passes through a series of **HttpMessageHandlers**. This chain of responsibility ends at **HttpControllerDispatcher**. The dispatcher then sends the request to appropriate controller.  Both **HttpServer**, **HttpControllerDispatcher** are built-in message handler.

### How to do it

Let's see the following code of a message handler.

```csharp
public class CustomRequestMessageHandler : DelegatingHandler
{
	protected async override Task<HttpResponseMessage> SendAsync(HttpRequestMessage	request, CancellationToken cancellationToken)
	{
		Debug.WriteLine(request.Method);
	  if (request.Method == HttpMethod.Post)
	  {
			Debug.WriteLine("Request method is changed to Http Get.");
			request.Method = new HttpMethod("Get");
	  }
	  var response = await base.SendAsync(request, cancellationToken);
	  return response;
	}
}
```

**DelegatingHandler** mainly has to implement **SendAsync** method. This method receives a **HttpRequestMessage** and returns a **Task<HttpResponseMessage>**. Basically in a delegating handler, if you need to inspect the request then you would simply do it in the **SendAsync**. The purpose of **SendAsync** is not just about sending. The purpose is to take a request message and send a response message. However, if we need to work on the response, we will do it in the continuation of the Task.

### How it works

In Web API, **DelegatingHandler** is used to process **HttpRequestMessage** & **HttpResponseMessage** and invoke another **DelegatingHandler**. It is an abstraction which already knows how to work in a pipeline. **DelegatingHandler** is a derived class from **HttpMessageHandler** and has a property of **InnerHandler**. All these custom handlers are chained together using the **InnerHandler** property. Every **InnerHandler** is assigned to another outside message handler and the last message handler’s **InnerHandler** has to be assigned with the **HttpControllerDispatcher**, to create a valid **ApiController**. If we don’t assign the last handler to **HttpControllerDispatcher**, the pipeline will not work and exception will be created at runtime saying that **InnerHandler** property of the last message handler is null or is not set.

In the next section, we will discuss about custom message handlers and its implementation.

## Creating Custom Message handlers

Sometimes customization is necessary. And custom message handler gives the opportunity to perform our custom logic at the level of Http messages rather than controller actions. Custom message handler might be useful in some cases, like –
*	Log each and every request.
*	Monitor each request, protocol version, type etc.
*	Modify request headers, add response headers.
*	Perform custom verification and validation logic in every request etc.

### How to do it

To write a custom message handler, derive from **System.Net.Http.DelegatingHandler** and override the **SendAsync** method. This method has the following signature:

```csharp
Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
```

Here we will see both custom request and response message handler separately and also how they work.

In the first example, we will write a custom message handler which will log the requested Http method and do some tweak based on their types. 

Like, write the request type and redirect all Http Post method to Http Get. Here is the code:

```csharp
public class CustomRequestMessageHandler : DelegatingHandler
{
	protected async override Task<HttpResponseMessage> SendAsync(HttpRequestMessage	request, CancellationToken cancellationToken)
	{
		Debug.WriteLine(request.Method);
		if (request.Method == HttpMethod.Post)
		{
			Debug.WriteLine("Request method is changed to Http Get.");
			request.Method = new HttpMethod("Get");
		}
		var response = await base.SendAsync(request, cancellationToken);
		return response;
	}
}
```

And we have to register the message handler in our **WebApiConfig** class inside the **App_Start** directory like this:

```csharp
public static class WebApiConfig
{
	public static void Register(HttpConfiguration config)
	{            
		config.MessageHandlers.Add(new CustomRequestMessageHandler());
		//Other code goes here
    }
}
```

Now using Postman if we invoke to its Post method, we will see that the debugger hits the Get action in the controller and results a JSON value in response.

# Image Insert

# Image Insert

In the next example we will see another message handler which will add a custom header with the response header. The code is shown below.

```csharp
public class CustomResponseMessageHandler:DelegatingHandler
{
	protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
	{
		return base.SendAsync(request, cancellationToken).ContinueWith(t =>
		{
			var res = t.Result;
			res.Headers.Add("Dummy-CustomTest-Header", "This is a dummy header for test!!");
			return res;
		}, cancellationToken);
	}
}
```

And again, we have to add this message handler to our **WebApiConfig** class like the previous example. 

After running the web service, if we invoke the URL using postman, the response header brings an extra custom header with it.

# Image Insert

### How it works

The first example illustrates a simple custom message handler. When any Http request arrives, **CustomRequestMessageHandler** executes as it is already registered in **WebApiConfig**. It can modify Http request and do custom operations. In this example, it prints the Http request type (debug mode) and then checks whether the request is a *POST* method or not. If yes, then it modifies the Http request method to *GET* method.Then the handler calls **base.SendAsync()** to pass the request to the inner message handler. The inner handler returns a response message asynchronously using a **Task<T>** object. The response message is not available until **base.SendAsync()** completes asynchronously.

This example uses the *await* keyword to perform work asynchronously after **SendAsync** completes. And in the third image we can see that it returns JSON value from the GET action.

We already said that if we need to work on the response, we need to do it in the continuation of the Task. A continuation is a task that is created in the **WaitingForActivation** state. It is activated automatically when its antecedent task or tasks complete. A continuation is itself a Task and does not block the thread on which it is started. In the second example, we implement the exact same thing. It adds a dummy custom header in the response. At times of going out from the pipeline, the **CustomResponseMessageHandler** adds a dummy custom header in the response. So, when we invoke the GET method using postman, we can see in the last image that the response header contains the extra dummy header.
