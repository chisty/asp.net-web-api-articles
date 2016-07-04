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

Web host gives the request to *HttpServer* which passes through a series of **HttpMessageHandlers**. This chain of responsibility ends at **HttpControllerDispatcher**. The dispatcher then sends the request to appropriate controller.  Both *HttpServer*, *HttpControllerDispatcher* are built-in message handler.

### How to do it

Let's see the following code of a message handler.

```csharp
public class CustomRequestMessageHandler : DelegatingHandler
{
	protected async override Task<HttpResponseMessage> SendAsync(HttpRequestMessage	request
	                                                             , CancellationToken cancellationToken)
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

Like, write the request type and redirect all *Http Post* method to *Http Get*. Here is the code:

```csharp
public class CustomRequestMessageHandler : DelegatingHandler
{
	protected async override Task<HttpResponseMessage> SendAsync(HttpRequestMessage	request
																 , CancellationToken cancellationToken)
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

And we have to register the message handler in our **WebApiConfig** class inside the *App_Start* directory like this:

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
	protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request
															, CancellationToken cancellationToken)
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

The first example illustrates a simple custom message handler. When any Http request arrives, *CustomRequestMessageHandler* executes as it is already registered in **WebApiConfig**. It can modify Http request and do custom operations. In this example, it prints the Http request type (debug mode) and then checks whether the request is a *POST* method or not. If yes, then it modifies the Http request method to *GET* method.Then the handler calls **base.SendAsync()** to pass the request to the inner message handler. The inner handler returns a response message asynchronously using a **Task<T>** object. The response message is not available until **base.SendAsync()** completes asynchronously.

This example uses the *await* keyword to perform work asynchronously after **SendAsync** completes. And in the third image we can see that it returns JSON value from the GET action.

We already said that if we need to work on the response, we need to do it in the continuation of the Task. A continuation is a task that is created in the **WaitingForActivation** state. It is activated automatically when its antecedent task or tasks complete. A continuation is itself a Task and does not block the thread on which it is started. In the second example, we implement the exact same thing. It adds a dummy custom header in the response. At times of going out from the pipeline, the *CustomResponseMessageHandler* adds a dummy custom header in the response. So, when we invoke the GET method using postman, we can see in the last image that the response header contains the extra dummy header.

## Registering Custom Message Handlers

If there are multiple custom message handlers, they will execute depending on their registered order, whether they are registered globally or per route. Global message handlers are executed before **HttpRoutineDispatcher**, while per route message handlers are executed after it. 

### How to do it

Let’s assume we have two custom message handler like below

```csharp
public class CustomMessageHandlerA : DelegatingHandler
{
	protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request
															, CancellationToken cancellationToken)
	{
		Debug.WriteLine("Message Handler A invoked on request= {0}", request.RequestUri);
		returnbase.SendAsync(request, cancellationToken).ContinueWith((task) =>
		{
			Debug.WriteLine("Message Handler A invoked on response={0}", task.Result.StatusCode);
			return task.Result;
		}, cancellationToken);
	}
}
```

```csharp
public class CustomMessageHandlerB : DelegatingHandler
{
	protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request
															, CancellationToken cancellationToken)
	{
		Debug.WriteLine("Message Handler B invoked on request= {0}", request.RequestUri);
		return base.SendAsync(request, cancellationToken).ContinueWith((task) =>
		{
			Debug.WriteLine("Message Handler B invoked on response={0}", task.Result.StatusCode);
			return task.Result;
		}, cancellationToken);
	}
}
```

To add a message handler on the server side, add the handler to **HttpConfiguration.MessageHandlers** collection. These handlers will be registered globally.

```csharp
public static class WebApiConfig
{
	public static void Register(HttpConfiguration config)
	{
		config.MessageHandlers.Add(newCustomMessageHandlerA());         				
		config.MessageHandlers.Add(newCustomMessageHandlerB());
		//Other codes go here....
	}
}
```

We can register specific message handler for any specific route. Here, if the request URI matches *Route2*, the request is dispatched to *CustomMessageHandlerB*. But, *CustomMessageHandlerA* invokes globally.

```csharp
public static class WebApiConfig
{
	public static void Register(HttpConfiguration config)
	{
		config.Routes.MapHttpRoute(
			name: "DefaultApi",
			routeTemplate: "api/{controller}/{id}",
			defaults: new {id = RouteParameter.Optional}
		);

		config.Routes.MapHttpRoute(
			name: "Route2",
			routeTemplate: "api2/{controller}/{id}",
			defaults: new {id = RouteParameter.Optional},
			constraints: null,
			handler: new CustomMessageHandlerB// per-route message handler
			{
				InnerHandler =new HttpControllerDispatcher(config)
			}
		);
		config.MessageHandlers.Add(new CustomMessageHandlerA());  // global message handler
    }
}
```

### How it works

Message handlers are called in the same order that they registered in **HttpConfiguration**. And the response message flows in other direction. The last handler performs first in response message. When we register the custom handlers globally and make any request to the server, we can see the following output in Visual Studio. The debug output screenshot is given below.

# Insert Image

In the above screenshot it is clearly visible that, the request message and response message goes *bidirectional*.  We registered & sequentially. But when we invoked a request to *http://localhost:10157/api/book* , *CustomMessageHandlerA* invoked first on request and *CustomMessageHandlerB* invoked second. But, on response *CustomMessageHandlerB* invoked first and *CustomMessageHandlerA* invoked later.

In the second example we implemented per route message handler. Here we registered *CustomMessageHandlerB* for Route2. If we invoke with **DefaultApi** then only globally registered messagehandler *CustomMessageHandlerA* invokes. The debug output is attached below.

*Request Url: http://localhost:10157/api/book*

# Insert Image

But if we invoke *Route2*, both the global and per route registered message handler works. In that case, we will see that both *CustomMessageHandlerA* and *CustomMessageHandlerB* is working sequentially.

*Request Url: http://localhost:10157/api2/book*


# Insert Image

## Implementing Filters

Filters add some extra customization in our web services. We can implement our custom logic using filter which we want to run in some specific region. Most of the times filters are used as attributes as it is very handy to use in controllers, actions. 

There are four types of filter.
*	Authentication (determines the identity of the client)
*	Authorization (determines the client has access to requested resources)
*	Action(should be registered with controller/action)
*	Exception (invokes if there is any exception)

When a request comes first in their pipeline, the **AuthenticationFilter** runs first. It implements the **IAuthenticationFilter** interface which has 2 methods: **AuthenticateAsync**, **ChallengeAsync**. **AuthenticateAsync** contains the core authentication logic like authenticate the request by validating credentials. If the authentication is successful, this filter creates an **IPrincipal** and attaches to the request by setting **context.Principal**. Otherwise, **context.ErrorResult** is set which basically gets translated to **“401-Unauthorized”** Http status code.

**AuthorizationFilter** runs after **AuthenticationFilter**. It extends the **AuthorizationFilterAttribute** and we have to override the **OnAuthorization** or **OnAuthorizationAsync** method which will contain the main logic of authorization of any resources. If this filter fails then it may return a **“401-Unauthorized”** Http status code also.

**ActionFilters** are invoked before entering to the **ApiController** action. We need to extend the filter from **ActionFilterAttribute** and override the **OnActionExecuting** or **OnActionExecuted** method. **OnActionExecuting** runs before the controller action is invoked and **OnActionExecuted** runs after the controller is done performing its task.

We need to extend the **ExceptionFilter** from **ExceptionFilterAttribute** and override the **OnException** or **OnExceptionAsync** method. When there is any exception situation, the **ExceptionFilter** fires immediately if any filter is registered with the **ApiController**.

All the filters run sequentially without violating the order. But controller level registered filter runs before the action level registered filter. And if there is multiple filter of same category registered then invoking order is determined by *FilterScope* given below. Globally defined filters run first, then controller specific filters and then action specific filters. The lowest value gets the privilege to perform first.

```csharp
public enum FilterScope
{
	Action= 20,
	Controller= 10,
	Global= 0
}
```

### How to do it

Here is an example of custom authorization filter:

```csharp
public class CustomFilter : AuthorizationFilterAttribute
{
	public override void OnAuthorization(HttpActionContext actionContext)
	{
		const string apiControllerActionName = "book";
		var absolutePath = actionContext.Request.RequestUri.AbsolutePath.ToLower();
		if (absolutePath.Contains(apiControllerActionName))
		{
			var response = new HttpResponseMessage(HttpStatusCode.Unauthorized)
			{
				Content = newStringContent("You are not authorized.")
			};
			actionContext.Response = response;
		}
	}
}
```

And we attached the above custom filter with our *BookController* **ApiController** like this -

```csharp
[CustomFilter]
public class BookController : ApiController
{            
	public IEnumerable<Book> Get()
	{
		using (var db= new BookContext())
		{
			return db.Books.Take(100).ToList();
		}
	}

	public IHttpActionResult Get(int id)
    {
		using (var db= new BookContext())
		{
			var book = db.Books.FirstOrDefault(b => b.Id == id);
			if (book == null)
				return NotFound();
			return Ok(book);
		}
	}

	public void Post([FromBody]string value)
	{
	}

	// Other code goes here ...
}
```

### How it works

We add this *CustomFilter* attribute on top of any **ApiController** for registration. Whenever any request invokes **ApiController**, the *CustomFilter* executes if it is already added with that **ApiController**. Inside the *CustomFilter*, we override the **OnAuthorization** method and inspected the **HttpRequestMessage** and its absolute path. We only pass those requests which do not have a fixed controller or action name *“book”*. If the absolute path contains that specific name, we purposefully attach a custom **HttpResponseMessage** and *Unauthorized HttpStatusCode* with the **HttpActionContext**. Then If we invoke the *BookApiController* using Postman, it returns us HttpStatusCode *“401-Unauthorized”*. The screenshot is attached below.

# Insert Image

## Registering Filters

Filter attributes can be easily attached in class level and method level which actually register them in controller level and action level accordingly.

### How to do it

If we want to use any filter on a full **ApiController**, we can use it like attribute. We just need to attach the filter above the **ApiController** class declaration. No registration in **HttpConfiguration** is required.  Every time we call any action of that **ApiController** the filter fires on time depending its type. 

```csharp
[CustomFilter]
public class BookController : ApiController
{            
	public IEnumerable<Book> Get()
	{
		using (var db= new BookContext())
		{
			return db.Books.Take(100).ToList();
		}
	}

	public IHttpActionResult Get(int id)
	{
		using (var db= new BookContext())
		{
			var book = db.Books.FirstOrDefault(b => b.Id == id);
			if (book == null)
				return NotFound();
			return Ok(book);
		}
	}

	public void Post([FromBody]string value)
	{
	}

	public void Put(int id, [FromBody]string value)
	{
	}

	public void Delete(int id)
	{
	}
}
```

Sometimes we want to attach some filter with some specific action of any **ApiController**. In that scenario, we declare the filter above the **Action** name rather than **ApiController** class name. So, when the appropriate *Action/Method* is called on this **ApiController**, the filter runs.

```csharp
public class BookController : ApiController
{            
	public IEnumerable<Book> Get()
	{
		using (var db= new BookContext())
		{
			return db.Books.Take(100).ToList();
		}
	}

	[CustomFilter]
	public IHttpActionResult Get(int id)
	{
		using (var db= new BookContext())
		{
			var book = db.Books.FirstOrDefault(b => b.Id == id);
			if (book == null)
				return NotFound();
			return Ok(book);
		}
	}

	//Other Code goes here ...
}
```

If we want to use our custom filter with every request or controller action, we need to register it globally. We don’t need to attach this with every **ApiController**. Rather we have to register that filter with **HttpConfiguration**.

```csharp
public static class WebApiConfig
{
	public static void Register(HttpConfiguration config)
	{
		config.Filters.Add(newCustomFilter());
		//………………………… other codes here  ……………………………… //
	}
}
```

### How it works

In the first scenario, if we request to any action of *BookController* **ApiController**, every request goes through CustomFilter's **OnAuthorization** method. Where we purposefully blocked every request and returned *HttpStatusCode.Unauthorized* in response. So, when we invoke any action of *BookController*, the response of postman also shown by a screenshot.

Request URIexample:	

http://localhost:10157/api/book

http://localhost:10157/api/book/2

# Insert Image

And in the second scenario, we added the *CustomFilter* with only one action. So, for that action the *CustomFilter* executes. But all other action gives as expected output. Like, if request with the following url:

http://localhost:10157/api/book/2

The output is the same like above screenshot which shows *"401 Unauthorized"*.

But if we request with following url:

http://localhost:10157/api/book

The output screenshot of postman is like below:

# Insert Image

And for the third scenario, we registered out *CustomFilter* with **HttpConfiguration** in **WebApiConfig**. So, every time we send a request to the server, it fires the global custom filter automatically for every **ApiController**, **Action** name. It is very useful in cases like authorizing every request, running exception filters on time etc.

## Differentiating between Message handler and Filters

Though message handler and filter seems very likely, the major difference between them is their scope and focus. We can arrange the differences in the following table.
s