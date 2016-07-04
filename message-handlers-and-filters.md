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

Web host gives the request to HttpServer which passes through a series of HttpMessageHandlers. This chain of responsibility ends at HttpControllerDispatcher. The dispatcher then sends the request to appropriate controller.  Both HttpServer,HttpControllerDispatcherare built-in message handler.

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

DelegatingHandler mainly has to implement SendAsync method. This method receives aHttpRequestMessage and returns a Task<HttpResponseMessage>. Basically in a delegating handler, if you need to inspect the request then you would simply do it in the SendAsync. The purpose of SendAsync is not just about sending. The purpose is to take a request message and send a response message. However, if we need to work on the response, we will do it in the continuation of the Task.

