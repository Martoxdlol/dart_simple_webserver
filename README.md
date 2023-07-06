# Dart Simple Web Server

## ¿What is this?

The idea is to build a simple (but powerful) file system driven web server using Dart.
Something like what php does but instead of '.php' files, we use '.dart' files.
Also with extra features and configurations (e.g., hide '.dart' in url).

## ¿Why?

I'm tired of complex and slow javascript frameworks and I want to use Dart.
I think dart is a great language, faster and more consistent than javascript.
It doesn't need to be transpiled, no tsconfig, no packing, no TypeScript server hell.

It is inspired by PHP, but much different (i still don't know if it will share anything with it). I want to take all the type-saftey ideas
and bring them to this web server. I want to use Dart's strong typing and code generation
for generating api documentation and client. Also I want a more rich url routing system
similar to modern frameworks but without all the js environment complexity.

## Requirements for designing this project

1. It must be simple to use. Really simple.
2. It must be fast. Faster than JS/TS.
3. It must be type-safe.
4. It must be easy to deploy. (single binary or something like php)
5. It must be easy to configure. (single yaml file)
6. It must bring something new to the table. (e.g., code generation, api documentation, or something...).
7. It must have some form of HTML templating 100% in Dart.
8. It must have some intelligent way to handle assets like images, css, js, etc.
9. It might have in the future a someting like components/islands for client side rendering.

## ¿How?

I don't know yet. I'm still thinking about it. But i have some ideas.

1. Use dart analyzer package to analize every file and generate a main file with all the imports. And all the other stuff...

## Design

### 1. Routing: I want to use something like this:

```
/.webserver (for internal use)
/lib
    /util.dart
    /some_code.dart
    /database.dart
    /post.dart
/pages
    /index.dart
    /about.dart
    /contact.dart
    /blog/
        /index.dart
        /post/[post]/
                /index.dart
    /[...catch_all]/
        /index.dart
/public
    /favicon.ico
    /assets
        /static_file.txt
    /js
/webserver.yaml
/pubspec.yaml
```

Or more simple:

```
/.webserver (for internal use)
/index.dart
/about.dart
/contact.dart
/favicon.ico
/blog/
    /index.dart
    /post/[post]/
            /index.dart
/[...catch_all]/
    /index.dart
/favicon.ico
/assets
    /static_file.txt
/js
/webserver.yaml
/pubspec.yaml
```

Things like pages and public directory, will be configurable in webserver.yaml.

### Handling requests

Requests are handled by `.dart` files. A handler file must have at least some of the following functions:

- onRequest: called with any php method (GET, POST, PUT, DELETE, etc).
- onGet: called with GET method.
- onPost: called with POST method.
- onPut: called with PUT method.
- onDelete: called with DELETE method.

Examples of on request handler:

```dart
// optional parameters (query, params, body, etc..) automatically injected and type safe.
// Represented using dart 3 records (they are nice).
Response onRequest(Request request, { query: ({ int id, bool? show_all }) }) async {
    // If id is not present, it will throw an error.

    if(show_all == true) {
        return Response.ok('Hello world! show_all: $show_all');
    }

    return Response.ok('Hello world! id: $id');
}
```

Exposing and using reponse instead of returning the response

```dart
// Route: /api/docs/:id

void onRequest(Request request, Response response, { params: ({ String id }) }) async {
    response.setHeaders({ 'Content-Type': 'text/plain' });

    final Stream<String> doc = readDoc(id);

    await for (final line in doc) {
        response.write(line);
    }

    response.end();
}
```

### Middleware

It must be a file in a directory that handles all the requests, for example `[...all].dart` or maybe
`(middleware).dart` (not sure yet). It will be called before any other handler in that directory.

```dart
Response onRequestHandler(Response Function(Request) handler, Request request) async {
    if(request.context['user'] == null) {
        return Response.forbidden('You must be logged in to access this page.');
    }

    return handler(request);
}

Response onPostHandler(Response Function(Request) handler, Request request) async {
    try {
        return await handler(request);
    } catch(e) {
        return Response.internalServerError('Cannot post data.');
    }
}
```

This functions may also be defined on the handler file. If defined on the handler file, they will be called before the middleware.

```dart
import 'package:my_app/middlewares/auth.dart';

// You can define it as a variable instead of a function.
final Response Function(Request) onRequestHandler = requireAuthenticationMiddleware;

// It will be execute after `onRequestHandler`
Response onPostHandler(Response Function(Request) handler, Request request) async {
    if(request.context['secret'] == null) {
        return Response.forbidden('You must present a secret to post this!.');
    }

    return handler(request);
}

Response onPost(Request request) async {
    // ...
}

Response onGet(Request request) async {
    // ...
}
```

### Request and context

I will probably end up copying the shelf library for this.

```dart
Response onRequest(Request<QueryRecordType, BodyRecordType> request) async {
    final User user = request.context['user'];

    final url = request.url; // e.g., /api/users/1?name=John&age=20

    final headers = request.headers; // e.g., { 'Content-Type': 'application/json' }
    final method = request.method; // e.g., GET, POST, PUT, DELETE, etc.
    final protocolVersion = request.protocolVersion; // e.g., 1.1
    final mimeType = request.mimeType; // e.g., application/json

    final Map<String, dynamic> = request.params; // From url
    // e.g., /api/users/:id -> { id: 1 }, /api/users/:id/:name -> { id: 1, name: 'John' }
    // e.g., /api/users/[...all] -> { all: ['1', 'John'] }

    final Map<String, dynamic> = request.query;
    // e.g., /api/users?name=John -> { name: 'John' }, /api/users?name=John&age=20 -> { name: 'John', age: '20' }
    // e.g., /api/users?name=John&age=20&friends[]=1&friends[]=2 -> { name: 'John', age: '20', friends: ['1', '2'] }

    final String body = await request.body.asString();
    final Map<String, dynamic> body = await request.body.asJson();
    final List<int> body = await request.body.asBytes();
    final Stream<List<int>> body = request.body.asStream();

    final data = await request.body.asParsed<BodyType>((Body body) => BodyType.fromJson(await body.asJson()));

    final ({String name, int age, List<String> friends}) body = request.body;

    // ...
}

// Another way of accessing context
Response onGet(Request request, Context context) async {
    // ...
}
```

### Response

```dart
Response onRequest(Request request) async {
    return Response.ok('Hello world!', headers: { 'Content-Type': 'text/plain' });
}

Stream<String> getLines() async* {
    yield 'Hello world!';
    yield 'Hello world!';
    yield 'Hello world!';
    yield 'Hello world!';
}

Response onRequest(Request request) async {
    return Response.stream(getLines(), status: Status.ok, headers: { 'Content-Type': 'text/plain' });
}
```

### Generating query, url, and body parameters as type safe records

```dart
// /api/users/:id/:name
Response onRequest(Request request, {
    params: ({ int id, String name }), 
    query: ({ int? limit, String? filter }),
    body: ({ String? name, int? age, List<String>? friends })
}) async {
    final id = request.context.parsed.params.id; // Same type as expected from function definition.

    assert(id == params.id); // true

    return Response.ok("Hi ${params.name}!");
}

typedef QueryRecordType = ({ int? limit, String? filter });

Response onRequest(Request request, { query: QueryRecordType }) async {
    final queryFromRequest = request.context.parsed.query as QueryRecordType; // Same type as expected from function definition.

    assert(queryFromRequest == query); // true

    return Response.ok("Hi ${params.name}!");
}
```
