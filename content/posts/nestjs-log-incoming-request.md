+++
title = "Nest.js log incoming request using interceptor"
date = 2020-12-08T16:41:00+08:00
description = "Demo on how to log all incoming http requests in Nest.js framework using interceptor"
categories = ['Node.js']
tags = ['Nest.js', 'Swagger', 'OAS 3.0']
+++

It is common to do audit logging for a json-over-http RESTful API server. What should be logged depends on your business logic and requirements while very often we need to access both the request object and the actual response. And we want to handle them in one place.

In Express, we attach a series of middleware to handle our route, i.e., by calling the `next()` function to pass the handle to our next middleware. In the simplest case, we might use middleware before and after our handle function for such purpose. Laravel lets you declare a middleware to perform as [Before or After Middleware](https://laravel.com/docs/8.x/middleware#before-after-middleware).

In Nest.js framework, the idea is similar by using middleware. Yet Nest.js has a more aspect-oriented architecture where incoming requests go through sequential handle layers.

#### Which layer to use?
Nest.js `Request Lifecycle` as mentioned in its [doc](https://docs.nestjs.com/faq/request-lifecycle#interceptors):
> In general, a request flows through middleware to guards, then to interceptors, then to pipes and finally back to interceptors on the return path (as the response is generated).

It is straight-forward to access the request object in each of these layers. As `guards` are used to implement our role-based access control. I want the audit logging to happen behind the guards. Plus interceptor seems to be the right place to get the actual response after route handler method executed. We may also access the decoded JWT injected by the auth guards.

There is also an official [example](https://docs.nestjs.com/interceptors#aspect-interception) on a `LoggingInterceptor`. The [response data](https://docs.nestjs.com/interceptors#response-mapping) is available through the RxJS observable return by invoking handler method with `next.handle()`.

#### Retrieve request info
In this demo, I will only log info from the request object, by extending the official template interceptor.
```ts
/* log.interception.ts */
@Injectable()
export class LogInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log('Before...');
    
    /* Our logging handle logic goes here */
    
    const now = Date.now();
    return next
      .handle()
      .pipe(
        tap(() => console.log(`After... ${Date.now() - now}ms`)),
      );
  }
}
```

Nest.js server can have different modes: HTTP, RPC or WebSocket. For our RESTful API http requests, the `http` context hosts our `request` object. To get the request:
```ts
const request = context.switchToHttp().getRequest();
```

From the request object, we can get info, for example:
```ts
const { method, params, query, body } = request;
const route = `${method} ${request.route.path}`; //e.g. "/users/:userId"
const url = `${method} ${request.url}`; //e.g. "/users/123"

const ip = request.headers['x-forwarded-for'] || request.connection.remoteAddress;
const hostName = request.headers['host'];
const userAgent = request.headers["user-agent"];
```

#### Retrieve other `Metadata`
In my own use case, I also want to retrieve some info from the Open API spec 3.0 properties set through Swagger. To retrieve metadata, we need to use `Reflector`. We may pass in the reflector instance at app bootstrap:
```ts
/* main.ts */

const app = await NestFactory.create(AppModule);
const reflector = app.get(Reflector);
const configService = app.get<ConfigService>(ConfigService);
const restService = app.get<RestService>(RestService);

app.useGlobalInterceptors(
  // we may pass in other singleton dependency instances get from app
  new ApiLogInterceptor(reflector, configService, restService)
);
```

Note that dependencies passed this way need to be be of the default singleton [injection scope](https://docs.nestjs.com/fundamentals/injection-scopes), which means a single instance is instantiated and shared after application bootstrapped.

Nest.js swagger module uses metadata to attach properties to controller/method. After finding our what key it uses to the the metadata, we can also retrieve them for logging:
```ts
import { ApiOperationOptions } from '@nestjs/swagger'; // "@nestjs/swagger": "4.6.0"
import { DECORATORS as SWAGGER_DECORATORS_META_KEY } from '@nestjs/swagger/dist/constants';

intercept(/* ... */) {
  //...
  const apiOperation = this.reflector.get<ApiOperationOptions|undefined>(
    SWAGGER_DECORATORS_META_KEY.API_OPERATION,
    context.getHandler()
  ) ?? {};
  const { operationId, tags, summary, description } = apiOperation;
  //...
  // log action here, catch errors and decide if you await
  logRequestAsync(log).toPromise().catch(exception => {
    this.logger.error(exception);
  })
}
```

Other metadata can also be retrieved for logging in a similar manner. Yet metadata used by other libraries might not be meant to use in such case. They are subject to unexpected values or changes on new releases.

#### Extra control with `SetMetadata`
Like setting OAS3.0 property using Swagger module's decorator, we can set our logging properties using Nest.js metadata. I used the metadata to opt-in/opt-out specific controller or route for logging, and also to set custom description for routes:
```ts
/* log-property.metadata.ts */
import { SetMetadata } from  "@nestjs/common";

export type LogProps = {
  enable?: boolean;
  // custom metadata
  description?: string;
}

const parseProps = (props?: LogProps): LogProps => {
  const  _props = props ?? {};
  _props.enable = _props.enable ?? true;
  return  _props; 
}

/**
 * @return props `enable` will default to true
 */
export const LogProperty = (props?: LogProps) =>  SetMetadata("LogProperty", parseProps(props));
```

The metadata can be retrieved by reflector with the key set:
```ts
const logPropertyCtrl = this.reflector.get<LogProps|undefined>('LogProperty', context.getClass()) ?? {};

const logProperty = this.reflector.get<LogProps|undefined>('LogProperty', context.getHandler()) ?? {};

const shouldLog = logProperty.enable ?? (logPropertyCtrl.enable ?? false);
```

The `shouldLog` opt-out strategy is hereby controlled by the usage of the metadata decorator `@LogProperty()`:
* If I use `@LogProperty()` or `@LogProperty({})` on Controllers, will mark all routes in the controller as "should log", which is equal to using `@LogProperty({ enable: true })`.
* I can use `@LogProperty({ enable: false })` further on a specific route handler method to opt-out for logging.

Although in many cases we may use logging and analytics that come with the cloud service, this use of interceptors gives us a flexible way to do a custom one.

For logging outgoing request in our service, we may also take advantage of the RxJS stream. Here's a [gist](https://gist.github.com/Roytangrb/5236366fb5f1f939a7f243296eeae88f) of simple snippet I am using to create a wrapper of the Nest.js `HttpService`, allowing us to intercept all outgoing http calls to handle errors/logging.
