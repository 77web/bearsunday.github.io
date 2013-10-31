---
layout: default
title: BEAR.Sunday | My First Resource
category: My First - Tutorial
---

# My First Resource Object

## Application Resource

Here we will pass in a `name` and create a `greeting` resource which the greeting will return.
In terms of MVC the `Model` in BEAR.Sunday is called an `Application Resource`. 
An application resource is used as an internal API within the application.

## Resource Architecture 

A resource is a bundle of information. 
He we have a greeting (`greeting`) which is used as a `greeting resource`.
In order to create the resource object class the following is needed.

 * URI
 * Request Interface 

The pattern is as follows.

| Method | URI                         | Query      |
|--------|-----------------------------|------------|
| get    | app://self/first/greeting   |?name=Name  |

The expected greeting resource is as below.

Request

```php
get app://self/first/greeting?name=BEAR
```

Response

```php
Hello, BEAR.
```

## Resource Object 

Lets run the Sandbox application. The URI, PHP class and file layout is as follows. 


| URI | Class | File |
|-----|--------|-----|
| app://self/first/greeting | Sandbox\Resource\App\First\Greeting | apps/Sandbox/src/Sandbox/Resource/App/First/Greeting.php |

Implementing the request interface (method).

```php
<?php
namespace Sandbox\Resource\App\First;

use BEAR\Resource\ResourceObject;

class Greeting extends ResourceObject
{
    public function onGet($name)
    {
        return "Hello, {$name}";
    }
}
```

## Command Line Testing 

Lets try this out using the Command Line Interface (CLI). 
In the console we will enter some commands, starting with a *failure*.

```php
php api.php get app://self/first/greeting
```

400 Bad Request　is returned in the response.

```php
400 Bad Request
...
[BODY]
Internal error occurred (e613b4)
```

As you can see in the header information that an exception has been raised, 
you can decipher that in the query a `name` is required. 
Using the *`OPTIONS`Method* you can more accurately examine this.

```php
php api.php options app://self/first/greeting?name=BEAR
```

```php
200 OK
allow: ["get"]
param-get: ["name"]
```

This tells us that the resource has the `GET` method enabled and requires 1 parameter `name`.
If this `name` parameter was to be optional you would wrap it in parenthesis `(name)`.
Now we know about the required parameters via the options method lets try again.
 

```php
php api.php get app://self/first/greeting?name=BEAR
```

```php
200 OK
...
[BODY]
Hello, BEAR
```
Now the correct response is returned. Success!

## The resource object is returned 

This greeting resource returns a string when run, 
but if you alter it as below it will be handled in the same way.
Which ever method is used the request made by the client will return a resource object.

```php
<?php
public function onGet($name)
 {
    $this->body = "Hello, {$name}";
    return $this;
}
```

Lets change the `onGet` method like this and check that the response returned has not changed.
