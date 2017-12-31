---
title:  "Typesafe request handlers with Express and Typescript"
layout: post
tags: []
---

Recently I've been working on a software project that required building an API server in Typescript, targeting the Node runtime. Coming from Scala I've been looking for ways to apply a type safe approach to HTTP servers. I also wanted to rely on mature components, so we had to use Express for handing HTTP requests - Express was designed for pure Javascript, so there's no concept of type safety: methods that accept payloads don't know for sure what they get.

## Unsafe request handlers

For instance, the following Express snippet will validate a user name passed as URL parameter, if a corresponding `User` record exists, the HTTP handler will get a `user` param in the request:

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

Now, this works, but is not _safe_ from the point of view of the `get` handler: there's not guarantee that the `req` object will have a `user` attribute that is in fact a `User`, with a `name`. You could add this guarantee by having some defensive logic in the handler that verifies that `req.user` is actually a valid `User`. This approach will work but it will add runtime overhead when instead we could see from the code that `req.user` is indeed a `User` object.

## Functional Middleware

The issue with the code above is that the `User` object gets passed from the middleware to the handler via the `Request` object that is essentially an untyped hash map.

This makes impossible carrying any type information from the middleware to the handler.

Thus, to being able to propagate the type information we need to rethink the way we compose middlewares and handlers.

Essentially we want something like:

``` ts
// middleware that extracts the User from the request
function getUserMiddleware(req: Request): User { ... };

// handler that uses the User
function myHandler(u: User): Response { ... };

// compose the two to handle a Request
myHandler(getUser(req));
```

Now we just need to make this functional approach as generic and reusable as possible. Let's see how.

I usually find it very useful to start writing code by defining the types of things, once you define the types and everything looks consistent, the logic will be much easier to write. 

Let's define first a type for our middleware functions. Essentially a `Middleware` is a function that extracts an object of a certain type `T` from a `Request`. Extracting the object may require interacting with some asynchronous resource so want we actually want is a `Promise<T>`. Moreover the processing of the `Request` may fail, so what we really want is a `Promise<Either<Error, T>>`. [^1]

Finally, we want to have a way to set HTTP status codes and arbitrary values when responding to the request, thus we need to define a return type that we can convert to a response logic that is suitable for the Express handlers. We can accomplish this by defining a new type that lets us encapsulate arbitrary responses:

``` ts
interface IResponse {
  readonly apply: (response: express.Response) => void;
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

So the final type for a functional middleware is:

``` ts
type RequestMiddleware<T> = (
  request: Request
) => Promise<Either<IResponse, T>>;
```

The type is self-descriptive, it defines a middleware as a function that takes a `Request` and asynchronously returns either an `IResponse` (in case of error) or an object of type `T`.  

We can now translate our initial example into a more functional middleware:

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
  if (users[id]) {
    return Promise.resolve(
      right(
        users[id]
      )
    );
  } else {
    return Promise.reject(
      left(
        ResponseError(404, 'failed to find user')
      )
    );
  }
}
```

## Functional request handlers

Now that we have `RequestMiddleware` for extracting arbitrary values from a `Request` in a typesafe way, how do we define a handler? Well, now handlers don't have to extract information from a `Request` anymore, so they just become pure functions that take some values and return an `IResponse`.  

We can now translate our initial example into a more functional handler:

``` ts
/**
 * A request handler that returns a User's name
 */
function getUserHandler(user: User): Promise<IResponse> {
  Promise.resolve(ResponseSuccess(user.name));
});
```

## Composing middleware and handler

Now we know how to extract values from a `Request` and how to generate an `IResponse` from those values. But how to we glue everything together into a function that can be used as an Express handler?

First of all we need a way to compose middlewares and handlers. In the previous example we used one single middleware to extract a value from the `Request` but in practice we want to be able to chain multiple middlewares that can extract several values from a single `Request`.[^2] 

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
      // attempt to extract a value from the request
      middleware(req).then(v => {
        if(isLeft(v)) {
          // if the result is an error response, resolve
          // the promise with the error response
          resolve(v.value);
        } else {
          // if it's a valid value, pass it to the handler
          // and resolve the promise with the handler response
          handler(v.value).then(resolve, reject);
        }
      });
    }, reject);
  };
}
```

The `withMiddleware` function looks a little bit complicated but is in fact very simple. It takes a `RequestMiddleware` that extracts a value of type `T` from a `Request` and a function (`handler`) that takes that value and returns an `IResponse`. The `withMiddleware` function returns a new function that takes an Express `Request` and returns an `IResponse` (we'll see later how to convert an `IResponse` into an Express response).

We can now compose the middleware and the handler to process a `Request` and have back an `IResponse`:

``` ts
const h = withMiddleware(userMiddleware)(getUserHandler);
// h: (Request) => IResponse
``` 

Now we need a way to transform the function returned by `withMiddleware` into an Express handler:

``` ts
function wrapRequestHandler(
  handler: (req: Request) => IResponse
): express.RequestHandler {
  return (request, response, _) => {
    // pass the Request to the handler
    handler(request).then(
      // if the Promise resolves to an IResponse, make it
      // generate the response
      r => r.apply(response),
      // if the Promise gets rejected, return a 500 error
      e => response.status(500).send()
    );
  };
}
```

So our final code will look like this:

``` ts
app.get('/user/:user', wrapRequestHandler(
  withMiddleware(userMiddleware)(getUserHandler)
));
```

## Multiple middlewares

Now that we have a way to compose a single middleware with a type safe handler to generate an Express request handler, let's try to extend this concept to support multiple middlewares that each can extract a different value from the request.

To do that, we need to extend `withMiddleware` to accept multiple middlewares that can product values of different types. For instance, here's an example of `withMiddleware` that can accept up to three middlewares:

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
      });
    }, reject);
  };
}
``` 

I'll leave you the exercise to extend `withMiddleware` to support more middlewares.

## Type safe responses

Now that we have a way to make sure that request handlers actually get the parameter they need in a type safe way, let's see if we can do the same with the responses. Until now we only had one single type of response (`IResponse`), so we would know what types a handler accept but we would not know what kind of responses it generates (does the handler generate only `200`s or also `404`s?).

To have this capability we need to the ability to define different response types. Let's extend the `IResponse` interface to have an associated `kind` that describes the kind of response:

``` ts
interface IResponse<T> {
  readonly kind: T;
  readonly apply: (response: Response) => void;
}
```

Now we can define specific response types and their constructors:

``` ts
// a successful response
interface IResponseOk 
  extends IResponse<"IResponseOk"> {};

function ResponseOk(value: any): IResponseOk {
  return {
    kind: "IResponseOk",
    apply: res => res.send(value) 
  };
}

// a Not Found response
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
// what we want our API to return
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

Know we know that the request handler can only respond with an HTTP 200 response containing a `UserJson` object represented as JSON.

Let's see how the middleware changes:

``` ts
function userMiddleware(
  req: Request
): Promise<Either<IResponseNotFound, User>> {
  const id = req.params.user;
  if (users[id]) {
    return Promise.resolve(
      right(
        users[id]
      )
    );
  } else {
    return Promise.reject(
      left(
        ResponseNotFound("failed to find user")
      )
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
// types and the handler response type
function withMiddleware<RH, R1, R2, T1, T2>(
  m1: RequestMiddleware<R1, T1>,
  m2: RequestMiddleware<R2, T2>
)(
  handler: (v1: T1, v2: T2) => Promise<IResponse<RH>>
): (req: Request) => Promise<IResponse<RH | R1 | R2>> {
  // ... body is unchanged
}
```


[^1]: Why can't we just use `Promise<T>` since promises can carry an `Error`? the problem is that you can't specify the type of your `Error`, so you'll have the same problem as having untyped params.

[^2]: We could have just one single middleware that extracts all the values we need for a handler, but that would become impractical as we would have to create a middleware specific for the data required by each handler - it is better instead to have a way of composing multiple simple middlewares.