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
        gateways or hybrid applications (defined later).
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

Pipes

  Pipes fit in the request stream, between nest and your controller.
  A pipe is a class annotated with the @Injectable() decorator.
  Pipes should implement the PipeTransform interface.
  Pipes are typically used for:
    Transformation -- transform input data to the desired output.
    Validation -- evaluate input data and if valid, simply pass it
      through unchanged; otherwise, throw an exception when the data is
      incorrect.
  Pipes get the arguments that will be handed to the controller.
  Any exception thrown by a pipe will be caught by exception filters.
  Nest has two pipes available:
    ValidationPipe
    ParseIntPipe
  Most basic validation type pipe:
    Code:
      import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

      @Injectable()
      export class ValidationPipe implements PipeTransform {
        transform(value: any, metadata: ArgumentMetadata) {
          return value;
        }
      }
    This example does nothing to the value, just passes it on unchanged.
  PipeTransform<T, R> is a generic interface in which T indicates the type of the
    input value, and R indicates the return type of the transform() method.
  The value is the currently processed argument (before it is received by the
    route handling method), while metadata is its metadata.
  The metadata object has these properties:
    Code:
      export interface ArgumentMetadata {
        readonly type: 'body' | 'query' | 'param' | 'custom';
        readonly metatype?: Type<any>;
        readonly data?: string;
      }
  These properties describe the currently processed argument.
    type - Indicates whether the argument is a body @Body(), query @Query(),
      param @Param(), or a custom parameter.
    metatype - Provides the metatype of the argument, for example, String.
      Note: the value is undefined if you either omit a type declaration
      in the route handler method signature, or use vanilla JavaScript.
    data - The string passed to the decorator, for example @Body('string').
      It's undefined if you leave the decorator parenthesis empty.
  Warning - TypeScript interfaces disappear during transpilation. Thus, if
    a method parameter's type is declared as an interface instead of a class,
    the metatype value will be Object.
  Example, using Joi schema for object validation.
    Code:
      import * as Joi from 'joi';
      import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';

      @Injectable()
      export class JoiValidationPipe implements PipeTransform {
        constructor(private readonly schema: Object) {}

        transform(value: any, metadata: ArgumentMetadata) {
          const { error } = Joi.validate(value, this.schema);
          if (error) {
            throw new BadRequestException('Validation failed');
          }
          return value;
        }
      }
  Binding pipes
    Binding pipes (tying them to the appropriate controller or handler)
      is very straightforward. We use the @UsePipes() decorator and create
      a pipe instance, passing it a Joi validation schema.
        Code:
          const createCatSchema = // ... build the Joi schema
          @Post()
          @UsePipes(new JoiValidationPipe(createCatSchema))
          async create(@Body() createCatDto: CreateCatDto) {
            this.catsService.create(createCatDto);
          }
  Class validator
    Does magic meta-level validation.  Apparently what ValidationPipe is.
      See the docs.
    ValidationPipe requires both class-validator and class-transformer
      packages to be installed.
  Pipes, similar to exception filters, can be method-scoped, controller-scoped,
    or global-scoped. Additionally, a pipe can be param-scoped. In the example
    below, we'll directly tie the pipe instance to the route param @Body()
    decorator.
      Code:
        @Post()
        async create(
          @Body(new ValidationPipe()) createCatDto: CreateCatDto,
        ) {
          this.catsService.create(createCatDto);
        }
    Param-scoped pipes are useful when the validation logic concerns only one
      specified parameter.
    Alternatively, to set up a pipe at a method level, use the @UsePipes()
      decorator.
        Code:
          @Post()
          @UsePipes(new ValidationPipe())
          async create(@Body() createCatDto: CreateCatDto) {
            this.catsService.create(createCatDto);
          }
    Alternatively, pass the class (not an instance), thus leaving instantiation
      up to the framework, and enabling dependency injection.
        Code:
          @Post()
          @UsePipes(ValidationPipe)
          async create(@Body() createCatDto: CreateCatDto) {
            this.catsService.create(createCatDto);
          }
    Global scope:
      Code:
        async function bootstrap() {
          const app = await NestFactory.create(ApplicationModule);
          app.useGlobalPipes(new ValidationPipe());
          await app.listen(3000);
        }
        bootstrap();
      Notice
        In the case of hybrid apps the useGlobalPipes() method doesn't set
          up pipes for gateways and micro services. For "standard" (non-hybrid)
          microservice apps, useGlobalPipes() does mount pipes globally.
      See the docs for global pipe with dependency injection.
  Transformation pipes
    The code for ParseIntPipe:
      Code:
        import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';

        @Injectable()
        export class ParseIntPipe implements PipeTransform<string, number> {
          transform(value: string, metadata: ArgumentMetadata): number {
            const val = parseInt(value, 10);
            if (isNaN(val)) {
              throw new BadRequestException('Validation failed');
            }
            return val;
          }
        }
    How it's used:
      Code:
        @Get(':id')
        async findOne(@Param('id', new ParseIntPipe()) id) {
          return await this.catsService.findOne(id);
        }
    With this in place, ParseIntPipe will be executed before the request
      reaches the corresponding handler, ensuring that it will always
      receive an integer for the id parameter.
    Another example would be to get the user by ID:
      Code (use):
        @Get(':id')
        findOne(@Param('id', UserByIdPipe) userEntity: UserEntity) {
          return userEntity;
        }
      We leave the implementation of this pipe to the reader, but note that
        like all other transformation pipes, it receives an input value
        (an id) and returns an output value (a UserEntity object). This
        can make your code more declarative and DRY by abstracting
        boilerplate code out of your handler and into a common pipe.

Guards

  Guards have a single responsibility. They determine whether a given request
    will be handled by the route handler or not, depending on certain
    conditions (like permissions, roles, ACLs, etc.) present at run-time.
    This is often referred to as authorization. Authorization (and its
    cousin, authentication, with which it usually collaborates) has typically
    been handled by middleware in traditional Express applications.
    Middleware is a fine choice for authentication, since things like token
    validation and attaching properties to the request object are not
    strongly connected with a particular route context (and its metadata).
  But middleware, by its nature, is dumb. It doesn't know which handler will
    be executed after calling the next() function. On the other hand,
    Guards have access to the ExecutionContext instance, and thus know
    exactly what's going to be executed next. They're designed, much like
    exception filters, pipes, and interceptors, to let you interpose
    processing logic at exactly the right point in the request/response
    cycle, and to do so declaratively. This helps keep your code DRY and
    declarative.
  Guards are executed after each middleware, but before any interceptor or pipe.
  A guard is a class annotated with the @Injectable() decorator.
  Guards should implement the CanActivate interface.
  Authorization example.
    Check caller has sufficient permissions.
    The AuthGuard that we'll build now assumes an authenticated user (and
      that, therefore, a token is attached to the request headers). It will
      extract and validate the token, and use the extracted information to
      determine whether the request can proceed or not.
    Code:
      import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
      import { Observable } from 'rxjs';

      @Injectable()
      export class AuthGuard implements CanActivate {
        canActivate(
          context: ExecutionContext,
        ): boolean | Promise<boolean> | Observable<boolean> {
          const request = context.switchToHttp().getRequest();
          return validateRequest(request);
        }
      }
  Guards return a boolean, either synchronously or async, via a Promise or
    an rxjs Observable.
  Nest uses the return value to control the next action:
    If it returns true, the request will be processed.
    If it returns false, Nest will deny the request.
  The canActivate() function takes a single argument, the ExecutionContext
    instance. The ExecutionContext inherits from ArgumentsHost. We saw
    ArgumentsHost before in the exception filters chapter. There, we saw
    that it's a wrapper around arguments that have been passed to the
    original handler, and contains different arguments arrays based on
    the type of the application.
  Execution context
    By extending ArgumentsHost, ExecutionContext provides additional
      details about the current execution process. Here's what it looks like:
    Code:
      export interface ExecutionContext extends ArgumentsHost {
        getClass<T = any>(): Type<T>;
        getHandler(): Function;
      }
    The getHandler() method returns a reference to the handler about to be
      invoked. The getClass() method returns the type of the Controller
      class which this particular handler belongs to. For example, if the
      currently processed request is a POST request, destined for the
      create() method on the CatsController, getHandler() will return a
      reference to the create() method and getClass() will return a
      CatsControllertype (not instance).
  Binding guards
    Like pipes and exception filters, guards can be controller-scoped,
      method-scoped, or global-scoped. Below, we set up a controller-scoped
      guard using the @UseGuards() decorator. This decorator may take a
      single argument, or a comma-separated list of arguments. This lets
      you easily apply the appropriate set of guards with one declaration.
    Code:
      @Controller('cats')
      @UseGuards(RolesGuard)
      export class CatsController {}
    In order to set up a global guard, use the useGlobalGuards() method of
      the Nest application instance.
        Code:
          const app = await NestFactory.create(ApplicationModule);
          app.useGlobalGuards(new RolesGuard());
    Notice
      In the case of hybrid apps the useGlobalGuards() method doesn't set
        up guards for gateways and micro services. For "standard"
        (non-hybrid) microservice apps, useGlobalGuards() does mount the
        guards globally.
    As with everything else, you do a special declaration to get dependency
      injection for your global guard.  See the docs.
  Role-base authentication example.
    Initial code:
      import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
      import { Observable } from 'rxjs';

      @Injectable()
      export class RolesGuard implements CanActivate {
        canActivate(
          context: ExecutionContext,
        ): boolean | Promise<boolean> | Observable<boolean> {
          return true;
        }
      }
    Here, it always returns 'true', so everything is allowed.
  Reflection
    Our RolesGuard is working, but it's not very smart yet. We're not yet
      taking advantage of the most important guard feature - the execution
      context. It doesn't yet know about roles, or which roles are allowed
      for each handler. The CatsController, for example, could have
      different permission schemes for different routes. Some might be
      available only for an admin user, and others could be open for
      everyone. How can we match roles to routes in a flexible and
      reusable way?
    This is where custom metadata comes into play. Nest provides the ability
      to attach custom metadata to route handlers through the @SetMetadata()
      decorator. This metadata supplies our missing role data, which a smart
      guard needs to make decisions. Let's take a look at using @SetMetadata().
        Code:
          @Post()
          @SetMetadata('roles', ['admin'])
          async create(@Body() createCatDto: CreateCatDto) {
            this.catsService.create(createCatDto);
          }
    With the construction above, we attached the roles metadata (roles is a
      key, while ['admin'] is a particular value) to the create() method.
      While this works, it's not good practice to use @SetMetadata()
      directly in your routes. Instead, create your own decorators, as
      shown below.
        Code:
          // roles.decorator.ts
          import { SetMetadata } from '@nestjs/common';
          export const Roles = (...roles: string[]) => SetMetadata('roles', roles);
    This approach is much cleaner and more readable, and is strongly typed.
      Now that we have a custom @Roles() decorator, we can use it to decorate
      the create() method.
        Code:
          @Post()
          @Roles('admin')
          async create(@Body() createCatDto: CreateCatDto) {
            this.catsService.create(createCatDto);
          }
    We want to make the return value conditional based on the comparing the
      roles assigned to the current user to the actual roles required by the
      current route being processed. In order to access the route's role(s)
      (custom metadata), we'll use the Reflector helper class, which is
      provided out of the box by the framework and exposed from the
      @nestjs/core package.
        Code:
          import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
          import { Observable } from 'rxjs';
          import { Reflector } from '@nestjs/core';

          @Injectable()
          export class RolesGuard implements CanActivate {
            constructor(private readonly reflector: Reflector) {}

            canActivate(context: ExecutionContext): boolean {
              const roles = this.reflector.get<string[]>('roles', context.getHandler());
              if (!roles) {
                return true;
              }
              const request = context.switchToHttp().getRequest();
              const user = request.user;
              const hasRole = () => user.roles.some((role) => roles.includes(role));
              return user && user.roles && hasRole();
            }
          }
      In the node.js world, it's common practice to attach the authorized
        user to the request object. Thus, in our sample code above, we are
        assuming that request.user contains the user instance and allowed
        roles. In your app, you will probably make that association in your
        custom authentication guard (or middleware).
  When a user with insufficient privileges requests an endpoint, Nest
    automatically returns the following response:
      {
        "statusCode": 403,
        "message": "Forbidden resource"
      }
    Note that behind the scenes, when a guard returns false, the framework
      throws a ForbiddenException. If you want to return a different error
      response, you should throw your own specific exception.
        Code:
          throw new UnauthorizedException();
    Any exception thrown by a guard will be handled by the exceptions layer
      (global exceptions filter and any exceptions filters that are applied
      to the current context).

Interceptors

  Overview
    An interceptor is a class annotated with the @Injectable() decorator.
    Interceptors should implement the NestInterceptor interface.
    Interceptors have a set of useful capabilities which are inspired by
      the Aspect Oriented Programming (AOP) technique.
    They make it possible to:
      bind extra logic before / after method execution
      transform the result returned from a function
      transform the exception thrown from a function
      extend the basic function behavior
      completely override a function depending on specific conditions
        (e.g., for caching purposes)
    Each interceptor implements the intercept() method, which takes two
      arguments.
  Execution context
    The first one is the ExecutionContext instance (exactly
      the same object as for guards).
  Call handler
    The second argument is a CallHandler. The CallHandler interface
      implements the handle() method, which you can use to invoke the
      route handler method at some point in your interceptor. If you
      don't call the handle() method in your implementation of the
      intercept() method, the route handler method won't be executed
      at all.
    The handle() method returns an Observable, we can use powerful RxJS
      operators to further manipulate the response.
    Consider, for example, an incoming POST /cats request. This request
      is destined for the create() handler defined inside the CatsController.
      If an interceptor which does not call the handle() method is called
      anywhere along the way, the create() method won't be executed. Once
      handle() is called (and its Observable has been returned), the
      create() handler will be triggered. And once the response stream
      is received via the Observable, additional operations can be
      performed on the stream, and a final result returned to the caller.
  Example - logging, both before and after, and timing of the 'handle' call.
    Code:
      import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
      import { Observable } from 'rxjs';
      import { tap } from 'rxjs/operators';

      @Injectable()
      export class LoggingInterceptor implements NestInterceptor {
        intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
          console.log('Before...');

          const now = Date.now();
          return next
            .handle()
            .pipe(
              tap(() => console.log(`After... ${Date.now() - now}ms`)),
            );
        }
      }
  The NestInterceptor<T, R> is a generic interface in which T indicates
    the type of an Observable<T> (supporting the response stream), and
    R is the type of the value wrapped by Observable<R>.
  Notice
    Interceptors, like controllers, providers, guards, and so on, can
      inject dependencies through their constructor.
  Since handle() returns an RxJS Observable, we have a wide choice of
    operators we can use to manipulate the stream. In the example above,
    we used the tap() operator, which invokes our anonymous logging function
    upon graceful or exceptional termination of the observable stream, but
    doesn't otherwise interfere with the response cycle.
  Binding interceptors
    In order to set up the interceptor, we use the @UseInterceptors()
      decorator imported from the @nestjs/common package. Like pipes and
      guards, interceptors can be controller-scoped, method-scoped, or
      global-scoped.
    Code:
      @UseInterceptors(LoggingInterceptor)
      export class CatsController {}
    Note that we passed the LoggingInterceptor type (instead of an instance),
      leaving responsibility for instantiation to the framework and enabling
      dependency injection. As with pipes, guards, and exception filters, we
      can also pass an in-place instance.
    Usual global dependency injection magic applies.
  Response mapping
    We already know that handle() returns an Observable. The stream contains
      the value returned from the route handler, and thus we can easily
      mutate it using RxJS's map() operator.
    Warning
      The response mapping feature doesn't work with the library-specific
        response strategy (using the @Res() object directly is forbidden).
    Let's create the TransformInterceptor, which will modify each response
      in a trivial way to demonstrate the process. It will use RxJS's map()
      operator to assign the response object to the data property of a
      newly created object, returning the new object to the client.
        Code:
          // transform.interceptor.ts
          import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
          import { Observable } from 'rxjs';
          import { map } from 'rxjs/operators';

          export interface Response<T> {
            data: T;
          }

          @Injectable()
          export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
            intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
              return next.handle().pipe(map(data => ({ data })));
            }
          }
      With the above construction, when someone calls the GET /cats
        endpoint, the response would look like the following (assuming
        that route handler returns an empty array []):
          {
            "data": []
          }
    Nest interceptors work with both synchronous and asynchronous
      intercept() methods. You can simply switch the method to async
      if necessary.
    Interceptors have great value in creating re-usable solutions to
      requirements that occur across an entire application. For example,
      imagine we need to transform each occurrence of a null value to an
      empty string ''. We can do it using one line of code and bind the
      interceptor globally so that it will automatically be used by each
      registered handler.
        Code:
          import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
          import { Observable } from 'rxjs';
          import { map } from 'rxjs/operators';

          @Injectable()
          export class ExcludeNullInterceptor implements NestInterceptor {
            intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
              return next
                .handle()
                .pipe(map(value => value === null ? '' : value ));
            }
          }
  Exception mapping
    Another interesting use-case is to take advantage of RxJS's
      catchError() operator to override thrown exceptions:
        Code:
          // errors.interceptor.ts
          import {
            Injectable,
            NestInterceptor,
            ExecutionContext,
            BadGatewayException,
            CallHandler,
          } from '@nestjs/common';
          import { Observable, throwError } from 'rxjs';
          import { catchError } from 'rxjs/operators';

          @Injectable()
          export class ErrorsInterceptor implements NestInterceptor {
            intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
              return next
                .handle()
                .pipe(
                  catchError(err => throwError(new BadGatewayException())),
                );
            }
          }
  Stream overriding
    The docs have a caching example, mainly to say [...] the response
      (a hardcoded, empty array) will be returned immediately. In order to
      create a generic solution, you can take advantage of Reflector and
      create a custom decorator.
  More operators
    The possibility of manipulating the stream using RxJS operators gives
      us many capabilities. Let's consider another common use case. Imagine
      you would like to handle timeouts on route requests. When your
      endpoint doesn't return anything after a period of time, you want
      to terminate with an error response. The following construction
      enables this:
        Code:
          // timeout.interceptor.ts
          import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
          import { Observable } from 'rxjs';
          import { timeout } from 'rxjs/operators';

          @Injectable()
          export class TimeoutInterceptor implements NestInterceptor {
            intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
              return next.handle().pipe(timeout(5000))
            }
          }
      After 5 seconds, request processing will be canceled.

Custom route decorators

  Nest is built around a language feature called decorators.
  An ES2016 decorator is an expression which returns a function
    and can take a target, name and property descriptor as arguments.
    You apply it by prefixing the decorator with an @ character and
    placing this at the very top of what you are trying to decorate.
    Decorators can be defined for either a class or a property.
  Param decorators
    Nest provides a set of useful param decorators that you can use
      together with the HTTP route handlers. Below is a list of the
      provided decorators and the plain Express (or Fastify) objects
      they represent:
        @Request()	req
        @Response()	res
        @Next()	next
        @Session()	req.session
        @Param(param?: string)	req.params / req.params[param]
        @Body(param?: string)	req.body / req.body[param]
        @Query(param?: string)	req.query / req.query[param]
        @Headers(param?: string)	req.headers / req.headers[param]
  Custom decorators
    In the node.js world, it's common practice to attach properties
      to the request object. Then you manually extract them in each
      route handler, e.g.:
        Code:
          const user = req.user;
    In order to make your code more readable and transparent, you can
      create a @User() decorator and reuse it across all of your
      controllers.
        Code:
          // user.decorator.ts
          import { createParamDecorator } from '@nestjs/common';

          export const User = createParamDecorator((data, req) => {
            return req.user;
          });
      Then, you can simply use it wherever it fits your requirements.
        Code:
          @Get()
          async findOne(@User() user: UserEntity) {
            console.log(user);
          }
    Passing data
      When the behavior of your decorator depends on some conditions,
        you can use the data parameter to pass an argument to the
        decorator's factory function. One use case for this is a custom
        decorator that extracts properties from the request object by key.
        Let's assume, for example, that our authentication layer validates
        requests and attaches a user entity to the request object. The
        user entity for an authenticated request might look like:
          {
            "id": 101,
            "firstName": "Alan",
            "lastName": "Turing",
            "email": "alan@email.com",
            "roles": ["admin"]
          }
      Let's define a decorator that takes a property name as key, and
        returns the associated value if it exists (or undefined if it
        doesn't exist, or if the user object has not been created).
          Code:
            // user.decorator.ts
            import { createParamDecorator } from '@nestjs/common';

            export const User = createParamDecorator((data: string, req) => {
              return data ? req.user && req.user[data] : req.user;
            });
      Here's how you could then access a particular property via the
        @User() decorator in the controller:
          Code:
            @Get()
            async findOne(@User('firstName') firstName: string) {
              console.log(`Hello ${firstName}`);
            }
      You can use this same decorator with different keys to access
        different properties. If the user object is deep or complex,
        this can make for easier and more readable request handler
        implementations.
    Working with pipes
      Nest treats custom param decorators in the same fashion as the
        built-in ones (@Body(), @Param() and @Query()). This means
        that pipes are executed for the custom annotated parameters
        as well (in our examples, the user argument). Moreover, you
        can apply the pipe directly to the custom decorator:
          Code:
            @Get()
            async findOne(@User(new ValidationPipe()) user: UserEntity) {
              console.log(user);
            }

Custom providers

  Overview
    There are a lot of scenarios when you might want to bind something
      directly to the Nest inversion of control container. For example,
      any constant values, configuration objects created based on the
      current environment, external libraries, or pre-calculated values
      that depends on few other defined providers. Moreover, you are able
      to override default implementations, e.g. use different classes or
      make use of various test doubles (for testing purposes) when needed.
    One essential thing that you should always keep in mind is that Nest
      uses tokens to identify dependencies. Usually, the auto-generated
      tokens are equal to classes. If you want to create a custom provider,
      you'd need to choose a token. Mostly, the custom tokens are
      represented by either plain strings or symbols. Following best
      practices, you should hold those tokens in the separated file, for
      example, inside constants.ts.
  Use value
    The useValue syntax is useful when it comes to either define a
      constant value, put external library into Nest container, or
      replace a real implementation with the mock object.
    Code:
      // definition
      import { connection } from './connection';

      const connectionProvider = {
        provide: 'CONNECTION',
        useValue: connection,
      };

      @Module({
        providers: [connectionProvider],
      })
      export class ApplicationModule {}

      // use
      @Injectable()
      export class CatsRepository {
        constructor(@Inject('CONNECTION') connection: Connection) {}
      }
    Useful for setting up mocks as well.
  Use class
    The useClass syntax allows you using different class per chosen factors.
    E.g.:
      Code:
        const configServiceProvider = {
          provide: ConfigService,
          useClass:
            process.env.NODE_ENV === 'development'
              ? DevelopmentConfigService
              : ProductionConfigService,
        };

        @Module({
          providers: [configServiceProvider],
        })
        export class ApplicationModule {}
      In this case, even if any class depends on ConfigService, Nest will
        inject an instance of the provided class (DevelopmentConfigService
        or ProductionConfigService) instead.
  Use factory
    The useFactory is a way of creating providers dynamically. The
      actual provider will be equal to a returned value of the factory
      function. The factory function can either depend on several different
      providers or stay completely independent. It means that factory may
      accept arguments, that Nest will resolve and pass during the
      instantiation process. Additionally, this function can return
      value asynchronously.
    E.g.:
      Code:
        const connectionFactory = {
          provide: 'CONNECTION',
          useFactory: (optionsProvider: OptionsProvider) => {
            const options = optionsProvider.get();
            return new DatabaseConnection(options);
          },
          inject: [OptionsProvider],
        };

        @Module({
          providers: [connectionFactory],
        })
        export class ApplicationModule {}
  Export custom provider
    In order to export a custom provider, we can either use a token or
      a whole object.  See the docs.

Asynchronous providers

  Overview
    When the application start has to be delayed until some asynchronous
      tasks will be finished, for example, until the connection with the
      database will be established, you should consider using asynchronous
      providers. In order to create an async provider, we use the
      useFactory. The factory has to return a Promise (thus async
      functions fit as well).
    E.g.:
      Code:
        {
          provide: 'ASYNC_CONNECTION',
          useFactory: async () => {
            const connection = await createConnection(options);
            return connection;
          },
        }
  Injection
    The asynchronous providers can be simply injected to other components
      by their tokens (in the above case, by the ASYNC_CONNECTION token).
      Each class that depends on the asynchronous provider will be
      instantiated once the async provider is already resolved.

Circular dependencies

  Nest permits creating circular dependencies between both providers and
    modules, but we advise you to avoid whenever it's possible. See the
    docs.

Injection scopes

  Overview
    For the people coming from different languages, it might be awkward
      that in Nest almost everything is shared across the incoming
      requests. We have a connection pool to the database, singleton
      services with a global state etc. Generally, Node.js doesn't
      follow request/response Multi-Threaded Stateless Model in which
      every request is being processed by the separate thread. Hence,
      using singleton instances is fully safe for our applications.
    However, there are edge-cases when request-based lifetime of the
      controller may be an intentional behavior, for instance per-request
      cache in GraphQL applications, request tracking or multi-tenancy.
  Scopes
    Basically, every provider can act as a singleton, be request-scoped,
      and be switched to the transient mode. See the following table to
      get familiar with the differences between them.
    SINGLETON	Each provider can be shared across multiple classes. The
      provider lifetime is strictly tied to the application lifecycle.
      Once the application has bootstrapped, all providers are already
      instantiated. The singleton scope is being used by default.
    REQUEST	A new instance of the provider is going to be exclusively
      created for every incoming request and garbage collected after
      the request processing is completed.
    TRANSIENT	Transient providers cannot be shared between providers.
      Every time when another provider asks the Nest container for
      particular transient provider, the container will create a new,
      dedicated instance.
  See the docs.

Lifecycle Events

  Overview
    Every application element has a lifecycle managed by Nest. Nest
      offers lifecycle hooks that provide visibility into key life
      moments and the ability to act when they occur.
  Lifecycle sequence
    After creating a injectable/controller by calling its constructor,
      Nest calls the lifecycle hook methods in the following sequence
      at specific moments:
    OnModuleInit	Called once the host module has been initialized
    OnApplicationBootstrap	Called once the application has fully
      started and is bootstrapped
    OnModuleDestroy	Cleanup just before Nest destroys the host
      module (app.close() method has been evaluated)
    OnApplicationShutdown	Responds to the system signals (when
      application gets shutdown by e.g. SIGTERM)
  Usage
    Each lifecycle hook is represented by interface. Interfaces are
      technically optional because they do not exist anyway after
      TypeScript compilation. Nonetheless, it's a good practice to
      use them in order to benefit from strong typing and editor
      tooling.
    E.g.:
      Code:
        import { Injectable, OnModuleInit } from '@nestjs/common';

        @Injectable()
        export class UsersService implements OnModuleInit {
          onModuleInit() {
            console.log(`The module has been initialized.`);
          }
        }
  Additionally, both OnModuleInit and OnApplicationBootstrap hooks
    allow you to defer the application initialization process
    (return a Promise or mark the method as async).
  OnApplicationShutdown
    The OnApplicationShutdown responds to the system signals
      (when application gets shutdown by e.g. SIGTERM). Use this
      hook to gracefully shutdown a Nest application.
