First steps

  Install nest.js
  Run:
    nest new project-name
    cd project-name
    npm run start
  By default, it uses express, but you can use fastify instead.

Controllers

  Handle incoming requests, return responses to the client.
  After you define a new controller or action, you have to restart nest.
  Handle paths as:
    Name of controller
    Optional additions on a per-action basis
  Generator for new controllers:
    Run:
      nest g controller cats
    Generates a new controller, and a test file, in a new directory:
      cats
        cats.controller.ts
        cats.controller.spec.ts
    E.g.:
      Code:
        import { Controller, Get } from '@nestjs/common';

        @Controller('cats')
        export class CatsController {
          @Get()
          findAll(): string {
            return 'This action returns all cats';
          }

          @Get('persians')
          findPersians(): string {
            return 'This action returns all persian cats';
          }
        }
      Paths defined:
        /cats (GET) -- findAll
        /cats/persians (GET) -- findPersians
      Method name (e.g., 'findAll', 'findPersians') can be anything.
        The @Get decorator gives the path binding to nest.
    The generator also updates the 'app.module.ts' file, which contains the list
      of controllers.  It won't be found otherwise.
  When a handler returns a JavaScript object or array, nest will serialize
    it to JSON automatically.
  When a handler returns a string, it's returned directly to the client.
  By default, the status code is 200 (201 for POSTs).
  You can optionally use library-specific (e.g., express) response object.
    To do so, change the action signature:
      findAll(@Res() response)
    Then you can (e.g.) set the repsonse code:
      response.status(200).send()
    If you use this, you can't use the standard method.
  Request info can be accessed by changing the signature:
    Code (here, express):
      @Get()
      findAll(@Req() request: Request): string {
        return 'This action returns all cats';
      }
    The request object represents the HTTP request and has properties for
      the request query string, parameters, HTTP headers, and body.  See
      express documentation for what is there, and how to access it.
    Or, you can use dedicated decorators:
      @Request()	req
      @Response()	res
      @Next()	next
      @Session()	req.session
      @Param(key?: string)	req.params / req.params[key]
      @Body(key?: string)	req.body / req.body[key]
      @Query(key?: string)	req.query / req.query[key]
      @Headers(name?: string)	req.headers / req.headers[name]
  You can add an HttpCode decorator for an action:
    Code:
      @Post()
      @HttpCode(204)
      create() {
        return 'This action adds a new cat';
      }
    For dynamic codes, use the library-specific response.
  Headers
    To specify a custom response header, you can either use a @Header()
      decorator or a library-specific response object (and call res.header()
      directly).
    Code:
      @Post()
      @Header('Cache-Control', 'none')
      create() {
        return 'This action adds a new cat';
      }
  Handling POST:
    Code:
      @Post()
      create(): string {
        return 'This action adds a new cat';
      }
  Nest provides the rest of the standard HTTP request endpoint decorators in
    the same fashion - @Put(), @Delete(), @Patch(), @Options(), @Head(),
    and @All(). Each represents its respective HTTP request method.
  Routing wildcards:
    Nest allows, e.g.:
      Code:
        @Get('ab*cd')
      Which matches 'abcd', 'ab_cd', 'abasdfcd', etc.
      The characters ?, +, *, and () may be used in a route path, and are subsets
        of their regular expression counterparts. The hyphen ( -) and the dot (.)
        are interpreted literally by string-based paths.
  Route parameters for dynamic routing:
    Code:
      @Get(':id')
      findOne(@Param() params): string {
        console.log(params.id);
        return `This action returns a #${params.id} cat`;
      }
  Route order
    NOTE: Route registration order matters!
      E.g.:
        @Get(':id')
        @Get('persians')
        @Get('all*ey')
      The 'persians' and 'all*ey' routes will never get used, since ':id'
        shadows them.
  Scopes
    Everything is shared across all incoming requests (connection pool to the
      database, singleton services with global state, etc.)  This is fine in
      'default' node, since it's single-threaded, but sometimes (e.g per-request
      caching in GraphQL) you need request-based lifetime of the controller.
      That can be done with scopes.
  Async actions
    Must return a Promise.
    Code:
      @Get()
      async findAll(): Promise<any[]> {
        return [];
      }
    RxJS streams
      Nest route handlers are even more powerful by being able to return RxJS
        observable streams. Nest will automatically subscribe to the source
        underneath and take the last emitted value (once the stream is completed).
      Code:
        @Get()
        findAll(): Observable<any[]> {
          return of([]);
        }
  Request payloads
    DTOs
      You can use the @Body decorator in POST handlers, but the recommended thing is
        to use a DTO (Data Transfer Object) to specify a schema.  The recommended
        thing is to use a class, since then nest can see the type at run-time.
      Code:
        // create-cat.dto.ts
        export class CreateCatDto {
          readonly name: string;
          readonly age: number;
          readonly breed: string;
        }
        // cats.controller.ts
        @Post()
        async create(@Body() createCatDto: CreateCatDto) {
          return `This action adds a new cat: ${createCatDto}`;
        }
      Curl:
        curl -s -X POST -d name="Fred" -d age=13 -d breed="Manx" http://localhost:3000/cats

Providers

  Used for services, repositories, factories, helpers, and so on.
  A provider can inject dependencies; nest "wires up" instances at run time.
  A provider is simply a class annotated with an @Injectable() decorator.
  Controllers should handle HTTP requests and delegate more complex tasks to providers.
  For example, a CatsService provider could load and store cats to a database.
    Code:
      // cats/cats.service.ts
      import { Injectable } from '@nestjs/common';
      import { Cat } from './interfaces/cat.interface';

      @Injectable()
      export class CatsService {
        private readonly cats: Cat[] = [];

        create(cat: Cat) {
          this.cats.push(cat);
        }

        findAll(): Promise<Cat[]> {
          return new Promise((resolve, reject) => {
            resolve(this.cats);
          });
        }
      }

      // cats/interfaces/cat.interface.ts
      export interface Cat {
        name: string;
        age: number;
        breed: string;
      }

    Controller code that uses the service:
      import { Body, Controller, Get, Param, Post } from '@nestjs/common';
      import { CreateCatDto } from './create-cat.dto';
      import { CatsService } from './cats.service';
      import { Cat } from './interfaces/cat.interface';

      @Controller('cats')
      export class CatsController {
        constructor(private readonly catsService: CatsService) {}

        @Get()
        findAll(): Promise<Cat[]> {
          return this.catsService.findAll();
        }

        @Post()
        create(@Body() createCatDto: CreateCatDto) {
          this.catsService.create(createCatDto);
        }
      }
  Note that the CatsController just declares that its constructor take a
    CatsService, and that's all we need to do... nest automatically creates
    one and injects it for us.
  There is a command-line way to create services:
    Code:
      nest g service cats
    This automatically sets up the service in app.module.ts.
  If you don't use the command line to create the service, it'll have to be
    added to app.module.ts in the providers section by hand, so nest can find it:
      Code:
        // ...
        import { CatsService } from './cats/cats.service';
        // ...
        @Module({
          // ...
          providers: [AppService, CatsService],
  Scopes
    Providers have an instance created at application start up time, and
      destroyed when the application is destroyed.  At application start-up
      time, all dependencies must be resolved, therefore every provider has
      to be instantiated.
    However, there are ways to make your provider lifetime request-scoped as well.
  Optional providers
    Occasionally, you might have dependencies which do not necessarily have to
      be resolved. For instance, your class may depend on a configuration object,
      but if none is passed, the default values should be used. In such a case,
      the dependency becomes optional, because lack of the configuration provider
      wouldn't lead to errors.
    To indicate a provider is optional , use the @Optional() decorator in the
      constructor signature.
    Code:
      import { Injectable, Optional, Inject } from '@nestjs/common';

      @Injectable()
      export class HttpService<T> {
        constructor(
          @Optional() @Inject('HTTP_OPTIONS') private readonly httpClient: T
        ) {}
      }
  Property-based injection
    Allows having a property of the class be the injection point, instead of the
      constructor, generally due to subclassing.
    In fact, they recommend that you prefer constructor injection if your class
      does not extend another provider.

Modules

  A module is a class annotated with a @Module() decorator. The @Module() decorator
    provides metadata that Nest makes use of to organize the application structure.
  Each application has at least one module, a root module. The root module is the
    starting point Nest uses to build the application graph - the internal data
    structure Nest uses to resolve module and provider relationships and
    dependencies.
  The @Module() decorator takes a single object whose properties describe the module:
    providers	- The providers that will be instantiated by the Nest injector and
      that may be shared at least across this module.
    controllers - The set of controllers defined in this module which have to be
      instantiated.
    imports - The list of imported modules that export the providers which are
      required in this module.
    exports	- The subset of providers that are provided by this module and should
      be available in other modules which import this module.
  You can create one with the command line.
    Code:
      nest g module cats
  Feature Modules
    So if we wanted to add Dogs, Lizards, Fish, etc., to our existing app, we'd have
      a bunch of controllers, services, etc. in the main file.
    Instead, we can encapsulate each one neatly.  For example, the cats.module.ts
      could be:
        Code:
          import { Module } from '@nestjs/common';
          import { CatsController } from './cats.controller';
          import { CatsService } from './cats.service';

          @Module({
            controllers: [CatsController],
            providers: [CatsService],
          })
          export class CatsModule {}
      Then, the app.module.ts becomes:
        Code:
          import { Module } from '@nestjs/common';
          import { AppController } from './app.controller';
          import { AppService } from './app.service';
          import { CatsModule } from './cats/cats.module';

          @Module({
            imports: [CatsModule],
            controllers: [AppController],
            providers: [AppService],
          })
          export class AppModule {}
    So adding a new module becomes a single (es6) import and a single entry in the
      main module's imports.
  Shared modules
    In Nest, modules are singletons by default, so you can share an instance of any
      provider with any other module easily.
    Every module is automatically a shared module.
    For example, we might want to share the CatsService with other modules.
    We just add it to the list of exports for the module:
      import { Module } from '@nestjs/common';
      import { CatsController } from './cats.controller';
      import { CatsService } from './cats.service';

      @Module({
        controllers: [CatsController],
        providers: [CatsService],
        exports: [CatsService]
      })
      export class CatsModule {}
    Then any other module can import it.
  Module re-exporting
    Modules can export their internal providers. In addition, they can re-export
      modules that they import.
    For example:
      @Module({
        imports: [CommonModule],
        exports: [CommonModule],
      })
      export class CoreModule {}
    Now any module which imports CoreModule can use CommonModule.
  Dependency injection
    A module class can inject providers as well (e.g., for configuration purposes):
      Code:
        import { Module } from '@nestjs/common';
        import { CatsController } from './cats.controller';
        import { CatsService } from './cats.service';

        @Module({
          controllers: [CatsController],
          providers: [CatsService],
        })
        export class CatsModule {
          constructor(private readonly catsService: CatsService) {}
        }
    However, module classes themselves cannot be injected as providers due to
      circular dependency.
  Global modules
    Modules can be made global withthe "@Global" decorator, so they don't have
      to be imported.  This is just for reducing substantial boilerplate.
  Dynamic modules
    This feature enables you to easily create customizable modules, for example
      a database module which builds a module for a specific model.

Middleware
  Middleware is a function which is called before the route handler.
  Middleware functions have access to the request and response objects, and
    the next() middleware function in the application’s request-response cycle.
  Nest middleware are, by default, equivalent to express middleware.
  Middleware functions can perform the following tasks:
    execute any code.
    make changes to the request and the response objects.
    end the request-response cycle.
    call the next middleware function in the stack.
    if the current middleware function does not end the request-response cycle,
      it must call next() to pass control to the next middleware function.
      Otherwise, the request will be left hanging.
  You implement custom Nest middleware in either a function, or in a class with
    an @Injectable() decorator. The class should implement the NestMiddleware
      interface, while the function does not have any special requirements.
  An example using the class method:
    Code:
      import { Injectable, NestMiddleware } from '@nestjs/common';
      import { Request, Response } from 'express';

      @Injectable()
      export class LoggerMiddleware implements NestMiddleware {
        use(req: Request, res: Response, next: Function) {
          console.log('Request...');
          next();
        }
      }
  Applying middleware
    There is no place for middleware in the @Module() decorator. Instead, we set
      them up using the configure() method of the module class. Modules that
      include middleware have to implement the NestModule interface.
    For example:
      Code:
        import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
        import { LoggerMiddleware } from './common/middleware/logger.middleware';
        import { CatsModule } from './cats/cats.module';

        @Module({
          imports: [CatsModule],
        })
        export class ApplicationModule implements NestModule {
          configure(consumer: MiddlewareConsumer) {
            consumer
              .apply(LoggerMiddleware)
              .forRoutes('cats');
          }
        }
    You can also further restrict a middleware to a particular request method:
      Code:
        // ...
        .forRoutes({ path: 'cats', method: RequestMethod.GET });
      You'll have to import the 'RequestMethod' enum from '@nestjs/common'.
    Route wildcards
      Pattern-based routes are supported as well:
        Code:
          .forRoutes({ path: 'ab*cd', method: RequestMethod.ALL });
    Excluding routes:
      Code:
        consumer
          .apply(LoggerMiddleware)
          .exclude(
            { path: 'cats', method: RequestMethod.GET },
            { path: 'cats', method: RequestMethod.POST }
          )
          .forRoutes(CatsController);
      Exclusion doesn't work in function middleware.
    For a full level of control, you should put your paths-restriction logic
      directly into the middleware and, for example, access the request's URL
      to conditionally apply the middleware logic.
  Middleware consumer
    The MiddlewareConsumer is a helper class. It provides several built-in
      methods to manage middleware. All of them can be simply chained in the
      fluent style. The forRoutes() method can take a single string, multiple
      strings, a RouteInfo object, a controller class and even multiple
      controller classes. In most cases you'll probably just pass a list of
      controllers separated by commas.
  Functional middleware
    Useful when the class-based type has no members, no additional methods,
      and no dependencies.
    Code:
      export function logger(req, res, next) {
        console.log(`Request...`);
        next();
      };
      // use
      consumer
        .apply(logger)
        .forRoutes(CatsController);
  Multiple middleware example
    In order to bind multiple middleware that are executed sequentially,
      simply provide a comma separated list inside the apply() method.
    Code:
      consumer.apply(cors(), helmet(), logger).forRoutes(CatsController);
  Global middleware
    If we want to bind middleware to every registered route at once, we can
      use the use() method that is supplied by the INestApplication instance.
    Code:
      const app = await NestFactory.create(ApplicationModule);
      app.use(logger);
      await app.listen(3000);

Exceptions
  Base exceptions
    The built-in HttpException class is exposed from the @nestjs/common
      package.
    Example usage (here hard-coding an exception)
      Code:
        @Get()
        async findAll() {
          throw new HttpException('Forbidden', HttpStatus.FORBIDDEN);
        }
    HttpStatus is a helper enum imported from @nestjs/common.
    When the client calls this endpoint, the response looks like this:
      {
        "statusCode": 403,
        "message": "Forbidden"
      }
    The HttpException constructor takes two arguments which determine
      the JSON response body and the HTTP response status code respectively.
    The first argument can be a plain literal object with properties status
      (the status code to appear in the JSON response body) and error (the
      message string) in the first parameter, instead of a string, to
      completely override the response body.
    The second constructor argument should be the actual HTTP response status
      code.
    Example overrding the entire response body
      Code:
        @Get()
        async findAll() {
          throw new HttpException({
            status: HttpStatus.FORBIDDEN,
            error: 'This is a custom message',
          }, 403);
        }
      Response:
        {
          "statusCode": 403,
          "error": "This is a custom message"
        }
  Extend HttpException for your own classes, to get automatic filter handling.
  In order to reduce the need to write boilerplate code, Nest provides a set
    of usable exceptions that inherit from the core HttpException. All of
    them are exposed from the @nestjs/common package:
      BadRequestException
      UnauthorizedException
      NotFoundException
      ForbiddenException
      NotAcceptableException
      RequestTimeoutException
      ConflictException
      GoneException
      PayloadTooLargeException
      UnsupportedMediaTypeException
      UnprocessableEntityException
      InternalServerErrorException
      NotImplementedException
      BadGatewayException
      ServiceUnavailableException
      GatewayTimeoutException

Exception filters
  Nest comes with a built-in exceptions layer which is responsible for
    processing all unhandled exceptions across an application. When an
    exception is not handled by your application code, it is caught by
    this layer, which then automatically sends an appropriate user-friendly
    response.
  Out of the box, this action is performed by a built-in global exception
    filter, which handles exceptions of type HttpException (and subclasses
    of it). When an exception is unrecognized (is neither HttpException nor
    a class that inherits from HttpException), the client receives the
    following default JSON response:
      {
        "statusCode": 500,
        "message": "Internal server error"
      }
  Exception filters are useful when you want full control over the
    exceptions layer. For example, you may want to add logging or use a
    different JSON schema based on some dynamic factors.
  Example which is responsible for catching exceptions that are an instance
    of the HttpException class, and implementing custom response logic
    for them.
    Code:
      import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
      import { Request, Response } from 'express';

      @Catch(HttpException)
      export class HttpExceptionFilter implements ExceptionFilter {
        catch(exception: HttpException, host: ArgumentsHost) {
          const ctx = host.switchToHttp();
          const response = ctx.getResponse<Response>();
          const request = ctx.getRequest<Request>();
          const status = exception.getStatus();

          response
            .status(status)
            .json({
              statusCode: status,
              timestamp: new Date().toISOString(),
              path: request.url,
            });
        }
      }
  All exception filters should implement the generic ExceptionFilter<T>
    interface. This requires you to provide the
    catch(exception: T, host: ArgumentsHost) method with its indicated
    signature. T indicates the type of the exception.
  The @Catch(HttpException) decorator binds the required metadata to the
    exception filter, telling Nest that this particular filter is looking
    for exceptions of type HttpException and nothing else. The @Catch()
    decorator may take a single parameter, or a comma-separated list.
  Arguments host
    Let's look at the parameters of the catch() method. The exception
      parameter is the exception object currently being processed.
      The host parameter is an ArgumentsHost object. ArgumentsHost is a
      wrapper around the arguments that have been passed to the original
      request handler (where the exception originated). It contains a
      specific arguments array based on the type of the application
      (and platform which is being used). Here's what an ArgumentsHost
      looks like:
        export interface ArgumentsHost {
          getArgs<T extends Array<any> = any[]>(): T;
          getArgByIndex<T = any>(index: number): T;
          switchToRpc(): RpcArgumentsHost;
          switchToHttp(): HttpArgumentsHost;
          switchToWs(): WsArgumentsHost;
        }
    ArgumentsHost is nothing more than an array of arguments. For example,
      when the filter is used within the HTTP application context,
      ArgumentsHost will contain a [request, response] array. However,
      when the current context is a web sockets application, it will
      contain a [client, data] array, as appropriate to that context.
      This approach enables you to access any argument that would
      eventually be passed to the original handler in your custom
      catch() method.
  Binding filters
    An example of how to tie the HttpExceptionFilter to the
      CatsController's create() method.
        Code:
          @Post()
          @UseFilters(new HttpExceptionFilter())
          async create(@Body() createCatDto: CreateCatDto) {
            throw new ForbiddenException();
          }
    The @UseFilters() decorator is imported from @nestjs/common.
    It can take a single filter instance, or a comma-separated list of
      filter instances.
    Alternatively, you may pass the class (instead of an instance),
      leaving responsibility for instantiation to the framework, and
      enabling dependency injection.
        Code:
          @Post()
          @UseFilters(HttpExceptionFilter)
          async create(@Body() createCatDto: CreateCatDto) {
            throw new ForbiddenException();
          }
      Prefer applying filters by using classes instead of instances when
        possible. It reduces memory usage since Nest can easily reuse
        instances of the same class across your entire module.
    Exception filters can be scoped at different levels: method-scoped,
      controller-scoped, or global-scoped. For example, to set up a filter
      as controller-scoped:
        Code:
          @UseFilters(new HttpExceptionFilter())
          export class CatsController {}
      This construction sets up the HttpExceptionFilter for every route
        handler defined inside the CatsController.
    Global filters:
      Code:
        async function bootstrap() {
          const app = await NestFactory.create(ApplicationModule);
          app.useGlobalFilters(new HttpExceptionFilter());
          await app.listen(3000);
        }
        bootstrap();
      Warning: The useGlobalFilters() method does not set up filters for
        gateways or hybrid applications.
      Global-scoped filters are used across the whole application, for
        every controller and every route handler. In terms of dependency
        injection, global filters registered from outside of any module
        (with useGlobalFilters() as in the example above) cannot inject
        dependencies since this is done outside the context of any module.
        In order to solve this issue, you can register a global-scoped
        filter directly from any module using the following construction.
          Code:
            import { Module } from '@nestjs/common';
            import { APP_FILTER } from '@nestjs/core';

            @Module({
              providers: [
                {
                  provide: APP_FILTER,
                  useClass: HttpExceptionFilter,
                },
              ],
            })
            export class ApplicationModule {}
        When using this approach to perform dependency injection for the
          filter, note that regardless of the module where this construction
          is employed, the filter is, in fact, global. Where should this be
          done? Choose the module where the filter (HttpExceptionFilter in
          the example above) is defined. Also, useClass is not the only
          way of dealing with custom provider registration.
        You can add as many filters with this technique as needed; simply
          add each to the providers array.
    Catch everything
      In order to catch every unhandled exception (regardless of the
        exception type), leave the @Catch() decorator's parameter list
        empty, e.g., @Catch().
      Code:
        @Catch()
        export class AllExceptionsFilter implements ExceptionFilter {
          catch(exception: unknown, host: ArgumentsHost) {
            // ...
          }
        }
