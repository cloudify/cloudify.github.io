---
title:  "Using the TypeScript type system to validate Express handlers"
layout: post
tags: [typescript, express, typesafe]
---

In this post I'll describe a powerful technique for using the TypeScript type system to verify at build time that request handlers and middlewares receive the validated types they expect and generate the expected response types. This technique makes the code more robust without adding any runtime overhead.  

## Unsafe request handlers

The following Express snippet defines a middleware that validates a user name passed as a URL parameter, if a corresponding `User` record exists, the record gets stored in the `Request` object and the request handler will read it, or else the middelware stops the processing and returns a 404:

``` js
var app = module.exports = express();

var users = [ { name: 'tj' } ];

app.param('user', function(req, res, next, id){
  if (req.user = users[id]) {
    next();
  } else {
    next(createError(404, 'failed to find user'));
  }
});

app.get('/user/:user', function(req, res, next){
  res.send('user ' + req.user.name);
});
```

Now, this works, but is not _safe_ from the point of view of the request handler: 

* there's not guarantee that the `req` object will have a `user` attribute that is in fact a `User`, with a `name` (no type checking) - you could add this guarantee by adding some defensive logic in the handler that verifies that `req.user` is actually a valid `User` - this approach will work but it will add unneeded runtime overhead
* there's no guarantee that the middleware is there, suppose this code gets refactored and the middleware gets moved somewhere else - you'll need to write unit tests to make sure that the middleware is indeed called before the request handler

These two problems can be avoided through the TypeScript type system - let's see how.

## Functional composition

The issue with the code above is that the `User` object gets passed from the middleware to the handler via the `Request` object (essentially an untyped hash map).

This makes impossible to carry any type information from the middleware to the handler (even tools like Flow will have a hard time tracking the type of the `user` attribute).

To be able to propagate the type information we need to rethink the way we compose middlewares and handlers.

Essentially we want to do something like the following:

``` ts
// middleware that extracts the User from the request
function getUserFromRequest(req: Request): User { ... };

// handler that uses the User
function userHandler(u: User): Response { ... };

// compose the two to handle a Request
userHandler(getUserFromRequest(req)); // => response
```

Now we just need to make this functional approach as generic and reusable as possible. Let's see how.

## Functional middlewares

I usually find it very useful to start writing code by first defining the types - once you define the types and everything looks consistent, the logic will be much easier to write. 

Let's define first a type for our middleware functions: essentially a `Middleware` is a function that extracts an object of a certain type `T` from a `Request`. Extracting the object may require interacting with some asynchronous resource (e.g. a database) so want we actually want is a `Promise<T>`. Moreover the processing of the `Request` may fail, so the middleware may have to stop the processing of the request and return a response right away. Thus what we really want is a `Promise<Either<Error, T>>`. [^1] [^2]

Finally, we want to have a way to set HTTP status codes and arbitrary values when responding to the request, thus we need to define a return type that we can convert to a response logic that is suitable for the Express handlers. We can accomplish this by defining a new type that lets us encapsulate arbitrary response logic:

``` ts
interface IResponse {
  readonly apply: (res: express.Response) => void;
}
``` 

For example, to construct instance of `IResponse` for successful responses we could use this function:

``` ts
function ResponseSuccess(o: string): IResponse {
  return {
    apply: res => res.send(o)
  };
}
```

And for constructing error responses we could use this one:

``` ts
function ResponseError(
  status: number, 
  message: string
): IResponse {
  return {
    apply: res => res.status(status).send(message)
  };
}
```

Now we can define the type for a functional version of a middleware:

``` ts
type RequestMiddleware<T> = (
  request: Request
) => Promise<Either<IResponse, T>>;
```

The type is self-descriptive, it defines a middleware as a function that takes a `Request` and asynchronously returns either an `IResponse` (in case of error) or a value of type `T`.  

We can now translate our initial example code that extracts the `user` from the request into a more functional middleware:

``` ts
/**
 * A middleware that extracts a valid User from the
 * Request and returns an error in case the user
 * does not exist.
 */
function userMiddleware(
  req: Request
): Promise<Either<IResponse, User>> {
  const id = req.params.user;
  const user = users[id];
  if (user) {
    // if user exist
    // resolve promise with the user
    return Promise.resolve(right(user));
  } else {
    // if the user does not exist
    // resolve the promise with a 404 response
    return Promise.resolve(
      left(ResponseError(404, 'user not found'))
    );
  }
}
```

## Functional request handlers

Now that we have `RequestMiddleware` type for extracting arbitrary values from a `Request` in a type safe way, how do we define a handler? Well, now handlers don't have to extract information from a `Request` anymore, so they just become pure functions that take some values and return an `IResponse`.  

We can now translate our initial example into a more functional handler:

``` ts
/**
 * A request handler that returns a User's name
 */
function getUserHandler(user: User): Promise<IResponse> {
  return Promise.resolve(ResponseSuccess(user.name));
});
```

Note that in the case of a request handler, we don't need an `Either` anymore since the handler is always the last function to process the request, so it must produce an `IResponse` (that can be a success or an error response).

## Composing middlewares and request handler

Now we know how to extract values from a `Request` and how to generate an `IResponse` from those values. But how to we glue everything together into a function that can be used as an Express handler?

First of all we need a way to compose middlewares and handlers. In the previous example we used one single middleware to extract a value from the `Request` but in practice we want to be able to chain multiple reusable middlewares that can extract several values from a single `Request`.[^3] 

Let's reason about how we could create a function that can compose an arbitrary number of middlewares with a request handler. 

Starting with a single middleware we have:

``` ts
function withMiddleware<T>(
  middleware: RequestMiddleware<T>
)(
  handler: (value: T) => Promise<IResponse>
): (req: Request) => Promise<IResponse> {
  // given a handler
  return handler => {
    // and a request
    return request => new Promise((resolve, reject) => {
      // call the middleware to extract the required
      // value from the request
      middleware(req).then(v => {
        if(isLeft(v)) {
          // if the result is an error response,
          // resolve the promise with the error
          // response (processing of the request
          // stops here) 
          resolve(v.value);
        } else {
          // if the result is a valid value
          // pass it to the request handler...
          handler(v.value)
            // ...and resolve this promise with the 
            // handler response
            .then(resolve, reject);
        }
      }, reject);
    });
  };
}
```

The `withMiddleware` function looks a little bit complicated but is in fact very simple. It takes a `RequestMiddleware` that extracts a value of type `T` from a `Request` and a function (`handler`) that takes that value of type `T` and returns an `IResponse`. 

The `withMiddleware` function returns a new function that takes an Express `Request` and returns an `IResponse`.

We can now compose the middleware and the handler to process a `Request` and have back an `IResponse`:

``` ts
const h = withMiddleware(userMiddleware)(getUserHandler);
// h: (Request) => IResponse
``` 

Now we need a way to transform the function returned by `withMiddleware` into an Express handler:

``` ts
function wrapRequestHandler(
  handler: (req: express.Request) => IResponse
): express.RequestHandler {
  return (request, response, _) => {
    // pass the Request to the handler
    handler(request).then(
      // if the Promise resolves to an IResponse
      // call it's apply method passing the Express
      // response object
      r => r.apply(response),
      // if the Promise gets rejected,
      // return a 500 error
      e => response.status(500).send()
    );
  };
}
```

So our final code will look like this:

``` ts
// define a GET endpoint
app.get('/user/:user',
  // convert the type safe handler
  // into an Express handler
  wrapRequestHandler(
    // apply the middlewares to the type-safe handler
    withMiddleware(userMiddleware)(getUserHandler)
  )
);
```

## Multiple middlewares

Now that we have a way to compose a single middleware with a type safe handler to generate an Express request handler, let's try to extend this concept to support multiple middlewares that each can extract a different value from the request.

To do that, we need to extend `withMiddleware` to accept multiple middleware functions that can produce values of different types. For instance, here's an example of `withMiddleware` that can accept up to three middlewares:

``` ts
function withMiddleware<T1, T2>(
  m1: RequestMiddleware<T1>,
  m2: RequestMiddleware<T2>
)(
  handler: (v1: T1, v2: T2) => Promise<IResponse>
): (req: Request) => Promise<IResponse> {
  return handler => {
    return request => new Promise((resolve, reject) => {
      m1(req).then(v1 => {
        if(isLeft(v1)) {
          resolve(v1.value);
        } else {
          m2(req).then(v2 => {
            if(isLeft(v2)) {
              resolve(v2.value);
            } else {
              handler(v1.value, v2.value)
                .then(resolve, reject);
            }
          }, reject)
        }
      }, reject);
    });
  };
}
``` 

As you can see, we extend he logic by calling the middlewares in sequence, every time a middleware returns a response, we return that response - once we called all the middlewares we have all the values required by the handler. Once we reach that point we just call the handler and return its response. [Here](https://github.com/teamdigitale/digital-citizenship-functions/blob/master/lib/utils/request_middleware.ts#L156) you can see an implementation that can accept up to six middlewares.

## Type safe responses

Now that we have a way to make sure that request handlers actually get the parameter they need in a type safe way, let's see if we can do the same with the responses. 

Until now we only had one single type of response (`IResponse`), this makes it impossible for the type system to know what kind of responses middleware and handlers generate (does the handler generate only `200`s or also `404`s?).

To have this capability we need the ability to define different response types. Let's extend the `IResponse` interface to have an associated `kind` that describes the "kind" of response returned:

``` ts
interface IResponse<T> {
  readonly kind: T;
  readonly apply: (response: Response) => void;
}
```

The `kind` attribute doesn't have to have an actual value, it's just a [literal type](https://basarat.gitbooks.io/typescript/docs/types/literal-types.html), a way for the type system to discriminate different `IResponse` types. 

Now we can define specific response types and their constructors:

``` ts
// a successful response (HTTP 200)
interface IResponseOk 
  extends IResponse<"IResponseOk"> {};

function ResponseOk(value: any): IResponseOk {
  return {
    kind: "IResponseOk",
    apply: res => res.send(value) 
  };
}

// a Not Found response (HTTP 404)
interface IResponseNotFound 
  extends IResponse<"IResponseNotFound"> {};

function ResponseNotFound(message: string): IResponseNotFound = {
  kind: "IResponseNotFound",
  apply: res => res.status(404).send(message)
};
```

We can further extend our capability of defining type safe responses by adding the type of the returned value:

``` ts
interface IResponseOkJson<T>
  extends IResponse<"IResponseOkJson"> {
  value: T;
};

function ResponseOkJson<T>(o: T): IResponseOkJson<T> {
  return {
    kind: "IResponseOkJson",
    apply: res => res.status(200).json(o),
    value: o
  };
}
```

Let's see now how we can change all the functions we developed so far to make response types explicit:

``` ts
// type of the response payload
interface UserJson {
  name: string;
}

/**
 * A request handler that returns a User's name
 */
function getUserHandler(user: User): 
  Promise<IResponseOkJson<UserJson>> {
  const userJson = {
    name: user.name
  };
  Promise.resolve(ResponseOkJson(userJson));
});
```

Now the type system knows that the request handler can only respond with an HTTP 200 response containing a `UserJson` object represented as JSON.

Let's see how the middleware changes as well:

``` ts
function userMiddleware(
  req: Request
): Promise<Either<IResponseNotFound, User>> {
  const id = req.params.user;
  const user = users[id];
  if (user) {
    return Promise.resolve(right(user));
  } else {
    return Promise.resolve(
      left(ResponseNotFound("failed to find user"))
    );
  }
}
```

Now we're sure that the `userMiddleware` can only respond with a 404 in case the user is not found.

Composing middlewares with the handler should also be parametrized with response types:

``` ts
// a middleware that can either return a response
// of type R or produce a value of type T
type RequestMiddleware<R, T> = (
  request: Request
) => Promise<Either<IResponse<R>, T>>;

// the composition of the middlewares and the handler
// produces a function that can return responses of
// type given by the union of the middlewares response
// types (R1, R2) and the handler response type (RH)
function withMiddleware<RH, R1, R2, T1, T2>(
  m1: RequestMiddleware<R1, T1>,
  m2: RequestMiddleware<R2, T2>
)(
  handler: (v1: T1, v2: T2) => Promise<IResponse<RH>>
): (req: Request) => Promise<IResponse<RH | R1 | R2>> {
  // ... body stays the same as before
}
```

And so should be the `wrapRequestHandler` helper:

``` ts
function wrapRequestHandler<R>(
  handler: (req: Request) => IResponse<R>
): express.RequestHandler {
  // ... body stays the same as before
}
```

## Conclusion

Let's now rewrite our initial example with this new functional approach (the code looks a little verbose because I've made explicit all types):

``` ts
const app = module.exports = express();

// the internal representation of a User
interface IUser {
  name: string;
};

// the "User" database
const users: ReadonlyArray<IUser> = 
  [ { name: 'tj' } ];

// the middleware that looks up the User from
// the Request, note how expressive is the type
// of the middleware: we can immediately understand
// all the possible outcomes of this middleware
// from its type (and the type system knows that too) 
const userMiddleware: RequestMiddleware<IResponseNotFound, IUser> = 
(req) => {
  const id = req.params.user;
  const user = users[id];
  const res = user ? 
    right(user) : 
    left(ResponseNotFound("failed to find user"));
  return Promise.resolve(res);
}

// describes our API response
interface IResponseJson {
  user_name: string;
};

// the request handler doesn't deal with a raw
// Request anymore, it gets all params it needs,
// already parsed and validated by the middlewares
// - look also how expressive is the type of this
// handler, we immediately know what goes in (IUser)
// and comes out (IResponseOkJson<IResponseJson>)
const getUserHandler: 
  (user: IUser) => Promise<IResponseOkJson<IResponseJson>> = 
  (user) => {
    const responseJson: IResponseJson = {
      user_name: user.name
    };
    Promise.resolve(ResponseOkJson(responseJson));
  });

// now compose the middleware and the handler -
// note how the result type includes all possible
// responses (from the middleware and from the handler)
const h: (express.Request) => 
  Promise<IResponseOkJson<IResponseJson> | IResponseNotFound> = 
  withMiddlewares(userMiddleware)(getUserHandler);

// finally, we generate the handler for Express
app.get('/user/:user', wrapRequestHandler(h));
```

Let's recap what we have accomplished:

* a request middleware that takes a `Request` and produces defined response (`IResponseNotFound`) and output (`User`) types
* a request handler with well defined input (`User`) and response (`IResponseOkJson<UserJson>`) types
* a function that composes the middleware and the handler into a new function that takes a `Request` and produces well defined response types (`IResponseNotFound | IResponseOkJson<UserJson>`)
* a wrapper that transforms the composed handler into an Express compatible request handler

In essence, we now have instructed the TypeScript compiler to verify that:

* all required parameters of the request handler get correctly extracted and validated from a `Request` by the middleware functions
* all types of responses gets explicitly defined both by the middleware functions and the request handler

__We know know exactly what goes into a request handler and what comes out, and it is all verified by the TypeScript compiler without having to create unit tests.__

This is the real power of type systems in action!

What's next? The next step would be to __automatically generate request handlers and response types from Swagger API definitions__, I'll write about this in a new article.

You can see this technique implemented in a [real world project](https://github.com/teamdigitale/digital-citizenship-functions) (look under `lib/utils` and `lib/controllers`.

[^1]: Why can't we just use `Promise<T>` since promises can carry an `Error`? The problem is that you can't specify the type of your `Error`, instead we want to be able to define the type of error response too. 

[^2]: An `Either<T,V>` type represents either an error of type `T` or a successful value of type `V` - you can see an implementation in the [fp-ts](https://github.com/gcanti/fp-ts/blob/master/src/Either.ts) library.

[^3]: We could have just one single middleware that extracts all the values we need for a handler, but that would become impractical as we would have to create a middleware specific for the data required by each handler - it is better instead to have a way of composing multiple simple middlewares.