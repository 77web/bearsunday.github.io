---
layout: default
title: BEAR.Sunday | My First Web Page
category: My First - Tutorial
---

# My First Web Page

## Let's Create A Web Page

## Page Resource 

First without using an application resource, we will make the most basic page class possible.
(without using a model and only using a controller to create a **Hello World** page)

## Starting from the Most Basic of Pages 

Just like application resource creates the instance of a resource, 
a page resource can also create an instance of a page resource.

The greeting **Hello** is fixed in a static page.

{% highlight php startinline %}
<?php

namespace Demo\Sandbox\Resource\Page\First;

use BEAR\Resource\ResourceObject;
use BEAR\Sunday\Inject\ResourceInject;

/**
 * Greeting page
 */
class Greeting extends ResourceObject
{
    use ResourceInject;

    /**
     * @var array
     */
    public $body = [
        'greeting' => 'Hello.'
    ];

    public function onGet()
    {
        return $this;
    }
}
{% endhighlight %}

We store the string `Hello.` in the page contents `greeting` slot. 
When the get request is called it does nothing but return itself.

## Let's Check the Page Resource State from the Command Line 

Let's check this resource from the command line.

```
$ cd  {$PROJECT_PATH}/apps/Demo.Sandbox/bootstrap/contexts/
$ php api.php get page://self/first/greeting

200 OK
content-type: ["application\/hal+json; charset=UTF-8"]
cache-control: ["no-cache"]
date: ["Sun, 29 Jun 2014 09:11:01 GMT"]
[BODY]
greeting Hello.,
...
```

We have confirmed that in the `greeting` slot the string `Hello.` exists.

## Render the Page Resource State 

In order to render the state of the page resource as HTML we need a template. 
We save this in the same place as the resource and just change the suffix.

### File Path 

|URI|Resource Class| Resource Template |
|---|--------------|-------------------|
|page://self/first/greeting | apps/Demo.Sandbox/Resource/Page/First/Greeting.php | apps/Demo.Sandbox/Resource/Page/First/Greeting.tpl |

### Template

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <link href="//netdna.bootstrapcdn.com/bootstrap/3.1.1/css/bootstrap.min.css">
    <meta name="viewport" content="width=device-width, initial-scale=1">
</head>
<body>
<h1>{$greeting|escape}</h1>
</body>
</html>
```

## Check the HTML from the Command Line 

We assign the resource state to the template and the resource renders as HTML.
This is a HTML page and can also be checked via the command line.

Lets check.

```
$ php dev.php get /first/greeting
```

```html
200 OK
cache-control: ["no-cache"]
date: ["Fri, 01 Feb 2013 14:21:45 GMT"]
[BODY]
<!DOCTYPE html>
<html lang="en">
<head>
    <link href="//netdna.bootstrapcdn.com/bootstrap/3.1.1/css/bootstrap.min.css">
    <meta name="viewport" content="width=device-width, initial-scale=1">
</head>
<body>
<h1>Hello, anonymous</h1>
</body>
</html>
```

HTML all checked!

## Checking the Page HTML in a Web Browser 

```
$ php -S localhost:8088 dev.php
```

Browse http://localhost:8088/first/greeting 
Did you see the page OK?

## Role of the Page 

A page gathers clusters of information (resource), and configures the page itself.
Here a singular slot `greeting` has the string `Hello.` stored in it, however many pages will need multiple slots.

The pages role is to configure the page to gather other resources and to solidify the pages state. 
The resource state is composed with the resource template and rendered as HTML to be passed and shown to the user.
