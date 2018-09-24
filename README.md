# Log-Requests-and-Responses-in-ASP.NET-Core
Using Middleware in ASP.NET Core to Log Requests and Responses

The presentation of ASP.NET Core to the wider world has given me the opportunity to dive deeper into some of its features while building an app that will be used in the real world, a rare opportunity I'm loathe to waste.

This new app will be in ASP.NET Core 2.1 Web API, the most recent production version available at time of writing.  One of our requirements is that, because this API will be used by a great many different apps with different loads, we want to be able to record requests to and responses from this API, similarly to how a court stenographer (aka court reporter) records all questions from lawyers and responses from defendants.

Though, hopefully, with a bit more modern technology.

In any case, I need a way to record requests and responses, and in ASP.NET Core the simplest way to do this is using middleware.  Come along with me as we build a new piece of middleware that will automatically record requests and responses!

### The Setup
In order to make this demo, we'll need to create a few dummy classes.  First off, let's create a new Employee class with a few properties:
````
public class Employee
{
    public int ID { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public DateTime DateOfBirth { get; set; }
}
````

Let's also create a new API controller called EmployeeController which returns employees by their ID:
````
[Route("api/[controller]")]
[ApiController]
public class EmployeeController : ControllerBase
{
    [HttpGet("{id}")]
    public ActionResult<Employee> GetByID(int id)
    {
        var employee = new Employee()
        {
            ID = id,
            FirstName = "firstName",
            LastName = "lastName",
            DateOfBirth = DateTime.Now.AddYears(-30)
        };

        return Ok(employee);
    }
}
````

That's all the setup we need!  Now we can get started building the meat of this application: the request/response recorder middleware.

### The Middleware
You might be familiar with ASP.NET Core's concept of middleware, but if not, here's a quick rundown.

Imagine a "pipeline" connecting the client to the server.  Middleware sits in that pipe, between client and server, and has the ability to inspect all incoming requests and outgoing responses, and if necessary, return a custom response.  For example, you could have a piece of middleware that inspects all requests for a particular query string value, and if that value isn't included, returns an HTTP error code.

We're using middleware in this demo because of that inherent ability to look at both requests and responses.  Since we want to record those things, using middleware for this task is a natural choice.

Below is the complete, functional middleware class we'll be using.  Almost every line is commented for your reading enjoyment, or at least understanding. :)

````
public class RequestResponseLoggingMiddleware
{
    private readonly RequestDelegate _next;

    public RequestResponseLoggingMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task Invoke(HttpContext context)
    {
        //First, get the incoming request
        var request = await FormatRequest(context.Request);

        //Copy a pointer to the original response body stream
        var originalBodyStream = context.Response.Body;

        //Create a new memory stream...
        using (var responseBody = new MemoryStream())
        {
            //...and use that for the temporary response body
            context.Response.Body = responseBody;

            //Continue down the Middleware pipeline, eventually returning to this class
            await _next(context);

            //Format the response from the server
            var response = await FormatResponse(context.Response);

            //TODO: Save log to chosen datastore

            //Copy the contents of the new memory stream (which contains the response) to the original stream, which is then returned to the client.
            await responseBody.CopyToAsync(originalBodyStream);
        }
    }

    private async Task<string> FormatRequest(HttpRequest request)
    {
        var body = request.Body;

        //This line allows us to set the reader for the request back at the beginning of its stream.
        request.EnableRewind();

        //We now need to read the request stream.  First, we create a new byte[] with the same length as the request stream...
        var buffer = new byte[Convert.ToInt32(request.ContentLength)];

        //...Then we copy the entire request stream into the new buffer.
        await request.Body.ReadAsync(buffer, 0, buffer.Length);

        //We convert the byte[] into a string using UTF8 encoding...
        var bodyAsText = Encoding.UTF8.GetString(buffer);

        //..and finally, assign the read body back to the request body, which is allowed because of EnableRewind()
        request.Body = body;

        return $"{request.Scheme} {request.Host}{request.Path} {request.QueryString} {bodyAsText}";
    }

    private async Task<string> FormatResponse(HttpResponse response)
    {
        //We need to read the response stream from the beginning...
        response.Body.Seek(0, SeekOrigin.Begin);

        //...and copy it into a string
        string text = await new StreamReader(response.Body).ReadToEndAsync();

        //We need to reset the reader for the response so that the client can read it.
        response.Body.Seek(0, SeekOrigin.Begin);

        //Return the string for the response, including the status code (e.g. 200, 404, 401, etc.)
        return $"{response.StatusCode}: {text}";
    }
}
````

I'll summarize the steps this piece of middleware accomplishes like this:

First, read the request and format it into a string.
Next, create a dummy MemoryStream to load the new response into.
Then, wait for the server to return a response.
Finally, copy the dummy MemoryStream (containing the actual response) into the original stream, which gets returned to the client.
The reason for the creation of a dummy MemoryStream is that the response stream can only be read once.  During that read, we log what the response was as well as create a new stream (which hasn't been read yet) and return that new stream to the client.

### One Last Bit
We also need to include our new middleware in the ASP.NET Core pipeline.  In the Startup.cs file, we can do this in the ConfigureServices method:

````
public class Startup
{
    //...
    
    public void Configure(IApplicationBuilder app, IHostingEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        //Add our new middleware to the pipeline
        app.UseMiddleware<RequestResponseLoggingMiddleware>();

        app.UseMvc();
    }
}
````

### Summary
n this tutorial, we created a piece of middleware in an ASP.NET Core application which can read and record both the requests into and the responses from a Web API application.  We had to do some stupid tricks regarding memory streams, but we did it, and in truth a piece of middleware very similar to this one is currently running in prod in one of my applications.

Did I miss something?  Did you find a way to record the requests and responses in an ASP.NET Core app more efficiently?  I'd like to know about it!  Share in the comments!

Happy Coding!
