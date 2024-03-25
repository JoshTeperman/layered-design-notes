## From web requests to abstraction layers
<img width="620" alt="image" src="https://github.com/JoshTeperman/layered-design-notes/assets/36095443/795675b0-9689-4398-8427-225fd284f0f6">

### What are the properties of a good abstraction layer?
- An abstraction layer should have a _single responsibility_. However the responsibilities themselves can be broad but should not overlap (thus following the separation of concerns principle)
- Layers should be _loosely coupled_ and have no circular or reverse dependencies. If we draw the request processing flow from top to bottom, the inter-layer connections should never go up, and we should try to minimize the number of connections between layers.
- Abstractions _should not leak their internals_. The main idea of extracting an abstraction is to separate an interface from the implementation.
- It should be possible to _test abstractions in isolation_.

From a developer's perspective a good abstraction provides a clear interface to solve a common problem and is easy to debug, refactor and test. 

### Rack
The component responsible for HTTP <-> Ruby translation. An interface describing two fundamental abstractions, _request_ and _response_, a Ruby implementation of a CGI (Common Gateway Interface), a protocol provides a standardized way for web servers to run programs. The contract between the web server (eg Puma) and a Ruby application. The contract can be described using the following source:

```ruby
request_env = {"HTTP_HOST" => "www.example.com", ...}
response = application.call(request_env)
status, headers, body_iterator = *response
```

- The first line defines an HTTP request defined as a hash. This hash defines the request environment and contains HTTP headers and specific Rack fields (such as `rack.input` to access the request body). The API and naming conventions came from the old days of CGI web servers, which passed data via environment variables
- The second line calls a Rack-compatible application, who's only requirement is that it responds to `#call`.
- The final line describes the structure of the return value, and array consisting of three elements - the response status code (integer), HTTP response headers (hash), and an enumerable body (anything that responds to `#each` and yields string values). Using enumerables allows us to implement streaming responses, which can help reduce memory allocation.

The simplest Rack application is just a lambda returning a predefined response tuple:

```
$ rackup -s webrick --builder 'run -> (env) { [200, {}, "hello world"] }'
```

### Rails on Rack

#### 0) Entrypoint
Entrypoint into a Rails application is in `config.ru`. You can start a rack-compatible application with `rackup` command, which will search for a `config.ru` file in the current directory. 

```
require './app`
run App
```

Rack's `run` commmand here means for requests to the web server, make the `App` singleton the context for which commands are executed. All methods on the `main` object are delegated to this class. 

Here the `App` class, which defines a `call` class method to be rack-compatible, is passed to Rack which uses `rackup` internally to interface between incoming HTTP requests and the framework. Rack will now pass an `env` object representing incoming HTTP requests to `App` so that the application can process and respond to requests.

#### 2) Middleware
Middleware is a copmonent that wraps a core unit (function) execution and can inspect and modify input and output data without changing its interface. Middleware is usually chained, so eachone invokes the next one, and only the last one in the chain executes the core logic. 

![image](https://github.com/JoshTeperman/layered-design-notes/assets/36095443/dab622ae-d0a2-4eaf-a9dd-5b780afd395f)

Rack allows you to extend basic request-handling funcionality by injecting middleware. Middleware intercepts HTTP requests to perform some additional, usually utilitarian logic - enhancing a Rack env object, adding additional response headers (eg X-Runtim or CORS-related), logging the request execution, performing security checks, etc. The middleware stack can be called the HTTP pre-/post-processing layer. It should treat the application as a black box and know nothing about its business logic. A Rack middleware should not enhance the application web interface, but act as a mediator between the outer world and the Rails application. 

View the default middleware stack with `bin/rails middleware` command: 

```
$bin/rails middleware
use ActionDispatch::HostAuthorization
use Rack::Sendfile
use ActionDispatch::Static
use ActionDispatch::Executor
use ActionDispatch::ServerTiming
use Rack::Runtime 
use Rack::Head
use Rack::ConditionalGet
use Rack::ETag
use Rack::TempfileReaper
run MyProject::Application.routes
```

#### 3) Routes

The final middleware called above runs rails routes, which is itself a Rack app `run` call with `routes` passed (an instance of `ActionDispatch::Routing::RouteSet`.
The Rails routing application uses the `routes.rb` file to match the request with a particular resolver - a controller / action pair, or another Rack application. 

#### 4) Controllers
Abstraction layer that standardises the way the application processes inbound requests. Theoretically can process any kind of request because it's just an abstraction, but in practice tightly coupled to Rack/HTTP. Translates requests into business actions or operations and trigger UI updates (if application has a view layer)

<img width="571" alt="image" src="https://github.com/JoshTeperman/layered-design-notes/assets/36095443/c11ae59e-c432-4886-b7c3-9fbccefae6c0">

