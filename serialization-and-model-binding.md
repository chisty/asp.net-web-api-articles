# Serialization and Model Binding

In this article, we will cover:

*	[Creating Media Formatter] (https://github.com/chisty/asp.net-web-api-articles/blob/master/serialization-and-model-binding.md#creating-media-formatter)
*	[Serializing the Data] (https://github.com/chisty/asp.net-web-api-articles/blob/master/serialization-and-model-binding.md#serializing-the-data)
*	[Negotiating the Content] (https://github.com/chisty/asp.net-web-api-articles/blob/master/serialization-and-model-binding.md#negotiating-the-content)
*	[Validating the Model] (https://github.com/chisty/asp.net-web-api-articles/blob/master/serialization-and-model-binding.md#validating-the-model)
*	[Binding Parameters] (https://github.com/chisty/asp.net-web-api-articles/blob/master/serialization-and-model-binding.md#binding-parameters)

Serialization and model binding are important topic in Web API as their performance impact is very significant. At times of working with custom *MIME types*, data validation, these concepts are very useful. We can implement our own custom logic in media formatter, data serialization, content negotiation and model validation. These gives us enough flexibility to write clean and simple code with complex data types. 

## Creating media formatter

Media type or MIME type allows the client and server to define the type of the data pass in http body. It refers the value of the *content-type header* within an Http request and response. It also used with *accept header* in the request to allow content negotiation.

*Media formatters* are used to define our own custom content-type / media type with which we can expose our logic/data to specific clients and represent data in our own specific format. The default formatters in Web API are – *XML*, *JSON* and *form-url encoded* data formatters.

To write a custom media formatter we have to extend from **BufferedMediaTypeFormatter** or **MediaTypeFormatter** abstract class. The **BufferedMediaTypeFormatter** is also extended from **MediaTypeFormatter** and uses synchronous read/write methods, where **MediaTypeFormatter** uses asynchronous methods.

Here we will write a custom media formatter which will give the http response in our specific format. The format is, every data will be separated by two/double equal sign (==).

### How to do it

Here we write a custom media formatter named **DoubleEqualMediaFormatter**. We only override the **WriteToStream** method with our custom logic.

```csharp
public class DoubleEqualCustomMediaFormatter : BufferedMediaTypeFormatter
{
	public DoubleEqualCustomMediaFormatter()
	{
		SupportedMediaTypes.Add(new MediaTypeHeaderValue("text/double-equal"));
	}

	public override bool CanReadType(Type type)
	{
		return false;
	}

	public override bool CanWriteType(Type type)
	{
		if (type == typeof(Book)) return true;
		return typeof(IEnumerable<Book>).IsAssignableFrom(type);
	}

	public override void WriteToStream(Type type, object value, Stream writeStream
										, HttpContent content)
	{
		using (var writer = new StreamWriter(writeStream))
		{
			var books = value as IEnumerable<Book>;
			if (books != null)
			{
				foreach (var book in books)
				{
					WriteItem(book, writer);
				}
			}
			else
			{
				var book = value as Book;
				if (book == null)
					throw new InvalidOperationException("Cannot serialize type");
				WriteItem(book, writer);
			}
		}
	}

	private static void WriteItem(Book book, StreamWriter writer)
	{
		writer.WriteLine("{0}=={1}=={2}=={3}", book.Title ?? string.Empty, book.Author 
							?? string.Empty, book.Category ?? string.Empty, book.Price);
	}
}
```

And at last, we have to register this custom media formatter in our **HttpConfiguration** in **WebApiConfig** class like this:

```csharp
public static class WebApiConfig
{
	public static void Register(HttpConfiguration config)
	{
		config.MapHttpAttributeRoutes();
		config.Formatters.Add(new DoubleEqualCustomMediaFormatter());
		config.Routes.MapHttpRoute(
			name: "DefaultApi",
			routeTemplate: "api/{controller}/{id}",
			defaults: new { id = RouteParameter.Optional }
		);
		//Other Code goes here
	}
}
```

### How it works

Here we extend the *DoubleEqualCustomMediaFormatter* from **BufferedMediaTypeFormatter** and did the overriding of *CanReadType*, *CanWriteType* and *WriteToStream* methods.

**CanReadType** decides which type the formatters can deserialize and **CanWriteType** decides which types it can serialize. Here the code returned false from **CanReadType** as we didn’t want to deserialize the type. In **CanWriteType**, we checked whether the type is our Book type or its *IEnumerable* collection. If the type match is successful, then we returned true otherwise false. We did overriding of **WriteToStream** with our custom logic to serialize our custom type by writing to stream. In every line which we wrote in stream, we separated the properties with *double equal sign (==)*.

Using Postman when we sent request to our **ApiController** attaching *custom accept header*, our *custom media formatter* executed and gave the expected result. The result screenshot are attached below:

![alt text](https://github.com/chisty/asp.net-web-api-articles/blob/master/images/smb_1.png "Custom media formatter result screenshot")

In the above screenshot out requested URI is *http://localhost:10157/api/book* and we passed *"text/double-equal"* as accept header. We can see that the result is not in default JSON format but our *custom double-equal* format.

If we investigate the Headers tab, we can see that the *Content-type* is *"text/double-equal"* like the below screenshot.

![alt text](https://github.com/chisty/asp.net-web-api-articles/blob/master/images/smb_2.png "Screenshot of Headers Tab in Postman")

## Serializing the data

Media formatters serialize objects to write into http message body and deserialize objects from Http message body into the pipeline. 

### How to do it

In order to serialize objects into Http message body, we should override the *CanWriteType* & *WriteToStream* methods and for deserialization, we should override the *CanReadType* & *ReadFromStream* methods as well.

By default, all public properties are serialized. We can ignore any property by using attributes (like in JsonFormatter uses JsonIgnore, XmlFormatter uses IgnoreDataMember).

In the above example within *DoubleEqualCustomMediaFormatter* class we implemented the **CanWriteType** and **WriteToStream** methods. The **WriteToStream** method does the data serialization task and **ReadFromStream** method does the data deserialization task. We need to implement the **CanReadType** method also for deserialization. The extended version of our previous media formatter example is given below. Here we implemented the deserialization also.

```csharp
public class DoubleEqualCustomMediaFormatter : BufferedMediaTypeFormatter
{
	public DoubleEqualCustomMediaFormatter()
	{
		SupportedMediaTypes.Add(new MediaTypeHeaderValue("text/double-equal"));
	}

	public override bool CanReadType(Type type)
	{
		if (type == typeof(Book)) return true;
		return typeof(IEnumerable<Book>).IsAssignableFrom(type);
	}

	public override bool CanWriteType(Type type)
	{
		if (type == typeof(Book)) return true;
		return typeof(IEnumerable<Book>).IsAssignableFrom(type);
	}

	public override object ReadFromStream(Type type, Stream readStream
											, HttpContent content, IFormatterLogger formatterLogger)
	{
		using (var reader = new StreamReader(readStream))
		{
			var books = new List<Book>();
			string text;
			while (string.IsNullOrEmpty(text = reader.ReadLine()) == false)
			{
				var book = CreateBookObjectFromText(text);
				if (book != null)
					books.Add(book);
			}
			return books;
		}
	}        

	public override void WriteToStream(Type type, object value,
										Stream writeStream,HttpContent content)
	{
		using (var writer = new StreamWriter(writeStream))
		{
			var books = value as IEnumerable<Book>;
			if (books != null)
			{
				foreach (var book in books)
				{
					WriteItem(book, writer);
				}
			}
			else
			{
				var book = value as Book;
				if (book == null)
					throw new InvalidOperationException("Cannot serialize type");
				WriteItem(book, writer);
			}
		}
	}

	private static void WriteItem(Book book, StreamWriter writer)
	{
		writer.WriteLine("{0}=={1}=={2}=={3}", book.Title ?? string.Empty, book.Author 
							?? string.Empty, book.Category ?? string.Empty, book.Price);
	}

	private static Book CreateBookObjectFromText(string text)
	{
		var tokens = text.Split(new[] { "==" }, StringSplitOptions.None);
		double price;
		var book = new Book
		{
			Title = tokens[0],
			Author = tokens[1],
			Category = tokens[2],
			Price = double.TryParse(tokens[3], out price) ? price : 0.0
		};
		return book;
	}
}
```

### How it works

Before executing **WriteToStream** method the **CanWriteType** method executes and checks whether the passed type is writeable to stream or not. And if the type is equal to our Book type or its *IEnumerable* we return true. And inside **WriteToStream** method we implemented our custom logic which actually loop through the value collection and write serialized data into stream.

And when we requested back to server with the serialized data via *Http Post* method, **CanReadType** executed which actually inspects whether the type is deserializable or not. If yes, then **ReadFromStream** is invoked in which we put our custom logic to deserialize stream to data.

## Negotiating the content

The *HTTP specification (RFC 2616)* defines content negotiation as *“the process of selecting the best representation for a given response when there are multiple representations available.”*

It’s not always possible for the server to return data as per the requested format. So, the client and server can negotiate and decide the best representation of data. The requested format is determined by inspecting the request header. 

### How to do it

There are several requested headers for content negotiation:
*	Content Type: It represents the requested data type to Api.
*	Accept: The acceptable media types of response.
*	Accept-Charset: The acceptable charset
*	Accept-Encoding: Acceptable content encoding.
*	Accept-Language: Preferred language.

We can easily understand this by seeing an example. Here we add a new **ApiController** named *PersonController*. This **ApiController** doesn't have any model. It is just a simple plain controller like bellow.

```csharp
public class PersonController : ApiController
{        
	public IEnumerable<string> Get()
	{
		return new string[] { "Person 1", "Person 2" };
	}
	
	public string Get(int id)
	{
		return "Person";
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

Now, we invoke this **ApiController** with Accept-Header *"text/double-equal"* like previous *BookApiController* example.

Request URI : *http://localhost:10157/api/person*

![alt text](https://github.com/chisty/asp.net-web-api-articles/blob/master/images/smb_3.png "Screenshot of Get request with custom Accept-Header")

But it will respond with normal JSON data in **HttpResponse**. We can see this in the following screenshot:

![alt text](https://github.com/chisty/asp.net-web-api-articles/blob/master/images/smb_4.png "Screenshot of Get request with custom Accept-Header")

### How it works

In the above first screenshot, we can see that we invoked to *PersonController* with accept header *"text/double-equal"*, but it responded with default *"application/json"* format. And in the second screenshot we can verify that the *Content-Type* in response header is *"application/json"*. So, there must have been some *negotiation* happened between client and server.

When we sent request from client to serve by setting the Accept-Header *"text/double-equal"*, the request is checked in our Custom Formatter's **CanWriteType** method. It returned false as it didn't satisfy the inner condition. So, custom formatter's **WriteToStream** method didn't take place and it responded with the default *"application/json"* format. This is content negotiation. When the client and server negotiation is not successful, the server sends the default *Content-Type* as response.

## Validating the model

We always need to validate the model before performing actions in **ApiController**. Model validation takes place in client side mainly, but we also need to implement server side validation. We can achieve this by using data annotation. 

###How to do it

Here is our *BookItem* class. 

```csharp
public class BookItem
{        
	[Required]
	public string Title { get; set; }
	
	[Required]
	public string Author { get; set; }
	public string Category { get; set; }

	[Range(0.5, 1000.0)]
	public double Price { get; set; }
}
```

And in the *BookController*, we receive BookItem in its Post Action.

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

	public HttpResponseMessage Post(BookItem book)
	{
		if (ModelState.IsValid)
		{
			return new HttpResponseMessage(HttpStatusCode.Created)
			{
				Content = new StringContent("Request model data is valid.")
			};

		}
		return new HttpResponseMessage(HttpStatusCode.BadRequest)
		{
			Content = new StringContent("Invalid data.")
		};
	}
	
	//Other Code goes here ...   
}
```

### How it works

When the client sends an *Http Post* request with JSON data, it is converted to *BookItem* object. And then, the object is checked against the rules defined in the model via **ModelState.IsValid**. It internally checks every properties and expression. And if there is any violation of rules, it returns false. Web Api does not return any error to client if data validation fails. We need to return appropriate **HttpResponseMessage** from **ApiController**. 

Here is a screenshot of our **BookController** when it received data in its *Post* action:

![alt text](https://github.com/chisty/asp.net-web-api-articles/blob/master/images/smb_5.png "Screenshot of Post request")

In the above screenshot we can see that when we passed a JSON object in *BookController's* Post Action, it created the *BookItem* object successfully. It successfully passed the rules defined in the model and the **ModelState.IsValid** returned true. Hence the debugger is pointing inside its nested scope. And we can also verify the *BookItem* object which is open in quick view mode. We can see the following screenshot to verify the **HttpResponse** :

![alt text](https://github.com/chisty/asp.net-web-api-articles/blob/master/images/smb_6.png)

In the above screenshot we can see that we send a JSON object to *BookController's* Post action using postman. As the request is valid, it returned a valid JSON data with **HttpStatusCode.Created**.

But when we passed *0 (zero)* in Price property, the **ModelState.IsValid** returned false as we already declared the Price range should be between *0.5 to 1000.0*. So, it returned a custom **HttpResponseMessage** saying *"Invalid Data"* with **HttpStatusCode.BadRequest**. The screenshot of postman is given below.

![alt text](https://github.com/chisty/asp.net-web-api-articles/blob/master/images/smb_7.png "Model validation scrrenshot")

## Binding parameters

When Web Api calls a method of any **ApiController**, it sets value for the parameters of that action. This is called parameter binding. The main goal of parameter binding is to convert the request into **.Net types**. It sets value to the **ApiController** action method’s parameter. By default Web Api follows the following rules:

*	For simple types, Web API gets the value from request URI. Simple types are .NET primitive types, TimeSpan, DateTime, Guid, Decimal, String or anything with a TypeConverter which converts from strings.
*	For complex types, Web API gets the value from message body.
*	We can use *[FromBody]* and *[FromUri]* attributes to specify that the parameter should be from body  and from URI accordingly.

We will first see how to read a complex type from the request body using *[FromBody]*. But, there is a restriction in that. We can read only one value from the message body. So, if we want to post multiple complex types, we need to implement our custom model binder. We will see these two examples in the next section.

###How to do it

Here is the code implementing the *[FromBody]* attribute in the **ApiController's** HttpPut action: 

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

	public HttpResponseMessage Put(int id, [FromBody]Book book)
	{
		if (book != null && id > 0)
		{
			using (var db = new BookContext())
			{
				var bookInDb = db.Books.FirstOrDefault(i => i.Id == id);
				if (bookInDb != null)
				{
					bookInDb.Title = book.Title;
					bookInDb.Author = book.Author;
					bookInDb.Category = book.Category;
					bookInDb.Price = book.Price;

					db.Entry(bookInDb).State = EntityState.Modified;
					db.SaveChanges();

					return new HttpResponseMessage(HttpStatusCode.OK)
					{
						Content = new StringContent("Resource updated successfully.")
					};
				}
			}
		}                     

		return new HttpResponseMessage(HttpStatusCode.BadRequest)
		{
			Content = new StringContent("Invalid data.")
		};
	}
			
	//Other Code goes here ...    
}
```

The *Put* method is taking a simple type id and a complex type Book object. The Book object is updating the existing book with the same id in database. 

But as we know we can't pass multiple complex type via message body. So, we need to send other complex type in some other way. Here comes the model binders; we can create our custom model binders and we can access the HttpRequest, the action description, values from route data using them. We can put our custom logic to convert a CLR type of our choice by the data retrieved from **ValueProviders**.

To create a custom model binder, we need to implement the **IModelBinder** interface which contains a single method named **BindModel**. The method signature is given below. **HttpActionContext** provides executing action information and **ModelBindingContext** provides the context in which model binder executes.

```csharp
bool BindModel(HttpActionContext actionContext, ModelBindingContext bindingContext)
```

Here we create a new model BookDetail:

```csharp
public class BookDetail
{
	public int Id { get; set; }
   public int BookId { get; set; }
   public int TotalChapter { get; set; }
   public string Publisher { get; set; }
   public string ISBN { get; set; }
}
```

We want to post *BookDetail* and Book both complex type in our Controller. We want to pass the *Book* object as a JSON string in request body and *BookDetail* object in query string.  Here we need to implement our custom model binder. We shall put our custom logic in which we create our *BookDetail* model from the retrieved data of query string and we shall bind it with the **ModelBindingContext**. 

Here is the code of our custom model binder.

```csharp
public class CustomModelBinder : IModelBinder
{
	public bool BindModel(HttpActionContext actionContext, ModelBindingContext bindingContext)
	{
		if (bindingContext.ModelType != typeof(BookDetail)) return false;

		var key = bindingContext.ModelName;
		var value = bindingContext.ValueProvider.GetValue(key);

		if (value != null && value.AttemptedValue != null)
		{
			var tokens = value.AttemptedValue.Split(';');
			if (tokens.Count() < 3) return false;

			int totalChapter;

			if (int.TryParse(tokens[0], out totalChapter) == false) return false;

			var details = new BookDetail
			{
				TotalChapter = totalChapter,
				Publisher = tokens[1],
				ISBN = tokens[2]
			};
			bindingContext.Model = details;
			return true;

		}
		return false;
	}
}
```

And in the *BookController* **ApiController**, we use it in our Post method like this.

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

	public HttpResponseMessage Post([ModelBinder(typeof (CustomModelBinder))] 
									BookDetail detail, [FromBody]Book book)
	{
		if (detail != null && book != null)
		{
			using (var db= new BookContext())
			{
				db.Books.Add(book);
				db.SaveChanges();

				detail.BookId = book.Id;                    
				db.BookDetails.Add(detail);
				db.SaveChanges();
				
				return new HttpResponseMessage(HttpStatusCode.Created)
				{
					Content = new StringContent("Successfully created.")
				};
			}
		}
		
		return new HttpResponseMessage(HttpStatusCode.BadRequest)
		{
			Content = new StringContent("Invalid data.")
		};
	}

	public HttpResponseMessage Put(int id, [FromBody]Book book)
	{
		if (book != null && id > 0)
		{
			using (var db = new BookContext())
			{
				var bookInDb = db.Books.FirstOrDefault(i => i.Id == id);
				if (bookInDb != null)
				{
					bookInDb.Title = book.Title;
					bookInDb.Author = book.Author;
					bookInDb.Category = book.Category;
					bookInDb.Price = book.Price;

					db.Entry(bookInDb).State = EntityState.Modified;
					db.SaveChanges();

					return new HttpResponseMessage(HttpStatusCode.OK)
					{
						Content = new StringContent("Resource updated successfully.")
					};
				}
			}
		}                     

		return new HttpResponseMessage(HttpStatusCode.BadRequest)
		{
			Content = new StringContent("Invalid data.")
		};
	}
	
	public void Delete(int id)
	{
	}
	//Other Code goes here...
}
```

We shall see how all these works in the next section.

### How it works

In the first example, the Put method takes a primitive type and a complex type. Primitive type can be easily collected from the query string parameter. And the complex type is collected from the request body. The request Content-Type is *"application/json"* and request body is in raw JSON string. Seeing the *[FromBody]*, Web API choose the necessary formatter based on request *Content-Type*.

At first, if we send an *HttpGet* request to our **ApiController** with *id=1* using postman, we get a JSON object like below.

![alt text](https://github.com/chisty/asp.net-web-api-articles/blob/master/images/smb_8.png)

If we now send an *HttpPut* request with a request body setting the *Content-Type* **"application/json"**, the Web API will use the JSON Media-Type formatter to deserialize the object.  The postman screenshot is given below.

![alt text](https://github.com/chisty/asp.net-web-api-articles/blob/master/images/smb_9.png)

Here we can see that we are sending an updated JSON string with *HttpPut* and the response is **HttpStatusCode.OK** with a content *"Resource updated successfully"* which we sent from our code. In that code, we retrieve the Book from database with the same id and update its properties with new Book object (Code is attached in the previous section). 

Again if we send the old HttpGet request with *id=1*, we get the updated Book object now like below.

![alt text](https://github.com/chisty/asp.net-web-api-articles/blob/master/images/smb_10.png)

In example 2, we wrote a custom model binder. If we pass the *BookDetail* object properties with the query string by separating them using semicolon (;), our custom logic inside the model binder will execute. It will create a *BookDetail* model and bind it with the **ModelBindingContext**. If we invoke the request using postman like below:

![alt text](https://github.com/chisty/asp.net-web-api-articles/blob/master/images/smb_11.png)

Here we are passing the Book object also via the request body setting the Content-Type *"application/json"*.

The below screenshot confirms that, the custom model binder is invoked.

![alt text](https://github.com/chisty/asp.net-web-api-articles/blob/master/images/smb_12.png)

Eventually, our Post method gets the parameters and does the rest of the work. Screenshot of Post method is attached below.

![alt text](https://github.com/chisty/asp.net-web-api-articles/blob/master/images/smb_13.png)

And now if we invoke the *BookDetailController* **ApiController**, we can get the stored data from database. Here is the screenshot of postman:

![alt text](https://github.com/chisty/asp.net-web-api-articles/blob/master/images/smb_14.png)

We can see that *BookDetail* **ApiController** responses with the JSON data and **HttpStatusCode.OK**.

## Summary

In this article, we have learned what is media formatter, how to negotiate the content, model validation and parameter binding. These are the very basic and important factors of Web API. We have also learned how to write custom media formatter, custom model binder etc. These are very handy when we need to develop a large architecture with different types of content and formatting.
