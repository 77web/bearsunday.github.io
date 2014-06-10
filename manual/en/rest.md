---
layout: default
title: BEAR.Sunday | Resource
category: Resource
---

Hypermedia framework for object as a service
--------------------------------------------

**BEAR.Resource** Is a Hypermedia framework that allows resources to behave as objects. It allows objects to have RESTful web service benefits such as client-server, uniform interface, statelessness, resource expression with mutual connectivity and layered components.


In order to introduce flexibility and longevity to your existing domain model or application data you can introduce an API as the driving force in your develpment by making your application REST-Centric in it's approach.

### Resource Object

The resource object is an object that has resource behavior.

 * 1 URI Resource is mapped to 1 class, it is retrieved by using a resource client.
 * A request is made to a method with named parameters that responds to a uniform resource request.
 * Through the request the method changes the resource state and return itself `$this`.


{% highlight php startinline %}
<?php
namespace Sandbox\Blog;

class Author extends ResourceObject
{
    public $code = 200;

    public $headers = [
    ];

    public $body = [
        'id' =>1,
        'name' => 'koriym'
    ];

    /**
     * @Link(rel="blog", href="app://self/blog/post?author_id={id}")
     */
    public function onGet($id)
    {
        return $this;
    }

    public function onPost($name)
    {
        $this->code = 201; // created
        // ...
        return $this;
    }

    public function onPut($id, $name)
    {
        //...
    }

    public function onDelete($id)
    {
        //...
    }
{% endhighlight %}

### Resource request

Using the URI and a query the resource is requested.

{% highlight php startinline %}
<?php
$user = $resource
    ->get
    ->uri('app://self/user')
    ->withQuery(['id' => 1])
    ->eager
    ->request();
{% endhighlight %}

 * This request passes 1 to the **onGet($id)** method in the **Sandbox\Resource\App\User** class that conforms to [PSR0](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-0.md).
 * The retrieved resource has 3 properties **code**, **headers** and **body**.

{% highlight php startinline %}
<?php
var_dump($user->body);

// Array
// (
//  [name] => Athos
//  [age] => 15
//  [blog_id] => 0
//)
{% endhighlight %}


## Hypermedia

A resource can contain [hyperlinks](http://en.wikipedia.org/wiki/Hyperlink) to other related resources.
Hyperlinks are shown by methods annotated with **@Link**.

{% highlight php startinline %}
<?php
use BEAR\Resource\Annotation\Link;

/**
 * @Link(rel="blog", href="app://self/blog?author_id={id}")
 */
{% endhighlight %}

The relation name is set by **rel** and link URI's are set by **href** (hyper reference).
The URI can assign the current resource value using the [URI Template](http://code.google.com/p/uri-templates/)([rfc6570](http://tools.ietf.org/html/rfc6570)).


Within a link their are several types **self**, **new**, **crawl** which can be used to effectively create a resource graph.

### linkSelf

`linkSelf` retrieves the linked resource.

{% highlight php startinline %}
<?php
$blog = $resource
    ->get
    ->uri('app://self/user')
    ->withQuery(['id' => 0])
    ->linkSelf('blog')
    ->eager
    ->request();
{% endhighlight %}
The result of the  **app://self/user** resource request jumps over the the **blog** link and retrieves the **app://self/blog** resource.
Just like clicking a link a the webpage it is replaced by the next resource.

### linkNew

`linkNew` adds the linked resource to the response.

{% highlight php startinline %}
<?php
$user = $resource
    ->get
    ->uri('app://self/user')
    ->withQuery(['id' => 0])
    ->linkNew('blog')
    ->eager
    ->request();

$blog = $user['blog'];
{% endhighlight %}
In a web page this is like 'opening a page in a new window', while passing the current resource but also retreiving the next.

### Crawl

A crawl passes over a list of resources (array) in order retrieving their links, with this you can construct a more complictated resource graph. Just as a crawler crawls a web page, the resource client crawls hyperlinks and creates a resource graph.

Let's think of author, post, meta, tag, tag/name and they are all connected together by a resource graph.
Each resource has a hyperlink. In ths resource graph add the name **post-tree**, on each resource add the hyper-reference *href* in the @link annotation.

In the author resource there is a hyperlink to the post resource. This is a 1:n relationship.

{% highlight php startinline %}
<?php
/**
 * @Link(crawl="post-tree", rel="post", href="app://self/post?author_id={id}")
 */
public function onGet($id = null)
{% endhighlight %}
In the post resource there is a hyperlink to meta and tag resources. This is also a 1:n relationship.

{% highlight php startinline %}
<?php
/**
 * @Link(crawl="post-tree", rel="meta", href="app://self/meta?post_id={id}")
 * @Link(crawl="post-tree", rel="tag",  href="app://self/tag?post_id={id}")
 */
public function onGet($author_id)
{
{% endhighlight %}

There is a hyperlink in the tag resource with only an ID for the tag/name resource that corresponds to that ID. It is a 1:1 relationship.

{% highlight php startinline %}
<?php
/**
 * @Link(crawl="post-tree", rel="tag_name",  href="app://self/tag/name?tag_id={tag_id}")
 */
public function onGet($post_id)
{% endhighlight %}

Set the crawl name and make the request.

{% highlight php startinline %}
<?php
$graph = $resource
  ->get
  ->uri('app://self/marshal/author')
  ->linkCrawl('post-tree')
  ->eager
  ->request();
{% endhighlight %}

The resource client looks for the crawl name annotated with @link using the **rel** name connects to the resource and creates a resource graph.

```
var_export($graph->body);

array (
    0 =>
    array (
        'name' => 'Athos',
        'post' =>
        array (
            0 =>
            array (
                'author_id' => '1',
                'body' => 'Anna post #1',
                'meta' =>
                array (
                    0 =>
                    array (
                        'data' => 'meta 1',
                    ),
                ),
                'tag' =>
                array (
                    0 =>
                    array (
                        'tag_name' =>
                        array (
                            0 =>
                            array (
                                'name' => 'zim',
                            ),
                        ),
                    ),
 ...
```

### HETEOAS Hypermedia as the Engine of Application State

The resource client next then takes the next behavior as hyperlink and carrying on from that link changes the application state.
For example in an order resource by using **POST** the order is created, from that order state to the payment resource using a **PUT** method a payment is made.

Order resource

{% highlight php startinline %}
<?php
/**
 * @Link(rel="payment", href="app://self/payment{?order_id, credit_card_number, expires, name, amount}", method="put")
 */
public function onPost($drink)
{% endhighlight %}

Client code

{% highlight php startinline %}
<?php
    $order = $resource
        ->post
        ->uri('app://self/order')
        ->withQuery(['drink' => 'latte'])
        ->eager
        ->request();

    $payment = [
        'credit_card_number' => '123456789',
        'expires' => '07/07',
        'name' => 'Koriym',
        'amount' => '4.00'
    ];

    // Now use a hyperlink to pay
    $response = $resource->href('payment', $payment);

    echo $response->code; // 201
{% endhighlight %}

The payment method is provided by the order resource with the hyperlink.
There is no change in client code even though the relationship between the order and payment is changed,
You can checkout more on HETEOAS at [How to GET a Cup of Coffee](http://www.infoq.com/articles/webber-rest-workflow).

### Resource Representation

Each resource has a renderer for representation. This renderer is a dependency of the resource, so it is injected in using an injector.
Apart from `JsonModule`you can also use the `HalModule` which uses a [HAL (Hyper Application Laungage)](http://stateless.co/hal_specification.html) renderer.


{% highlight php startinline %}
<?php
$modules = [new ResourceModule('Sandbox'), new JsonModule]:
$resource = Injector::create(modules)
  ->getInstance('BEAR\Resource\ResourceInterface');
{% endhighlight %}

When the resource is output as a string the injected resource renderer is used then displayed as the resource representation.

{% highlight php startinline %}
<?php
echo $user;

// {
//     "name": "Aramis",
//     "age": 16,
//     "blog_id": 1
// }
{% endhighlight %}

In this case `$user` is the renderers internal `ResourceObject`.
This is not a string so is treated as either an array or an object.

{% highlight php startinline %}
<?php

echo $user['name'];

// Aramis

echo $user->onGet(2);

// {
//     "name": "Yumi",
//     "age": 15,
//     "blog_id": 2
// }
{% endhighlight %}
### Lazy Loading

{% highlight php startinline %}
<?php
$user = $resource
  ->get
  ->uri('app://self/user')
  ->withQuery(['id' => 1])
  ->request();

$smarty->assign('user', $user);
{% endhighlight %}

In a non `eager` `request()` not the resource request result but a request object is retrieved.
When this is assigned to the template engine at the timing of the output of a resource request `{$user}` in the template the `resource request` and `resource rendering` is executed and is displayed as a string.


## Signal Parameter

In order to execute a method parameters are needed. Normally the following parameters are available in priority order:

  * Use of a consumer that calls the method ```$obj->method(1, 2, ...);```
  * Use of default method signature ```function method($a1 = 1)```
  * When null is present in a method instantiate internally. ```function method($cat = null) { $cat = $cat ?: new Cat;```

In order to seperate the provision responsibility of parameters from the method and consumer we use the `signal parameter`.
This only fires when the consumer and method does not provision the needed parameters.

The name `signal parameter` comes from the [Signal and Slots](http://en.wikipedia.org/wiki/Signals_and_slots) design pattern.
When a parameter is not available a `signal` is dispatched in the variable name and missing value is resolved by a signal parameter that is registered as a `slot`.

### Registering a Parameter Provider

Assign the variable names and provider in the resource client.

{% highlight php startinline %}
$resource = $injector->getInstance('BEAR\Resource\ResourceInterface');
$resource->attachParamProvider('user_id', new SessionIdParam);
{% endhighlight %}

In this case the when the parameter that has the variable name `$user_id` is needed, `SessionIdParam` is called.

### Parameter Provider Implementation

{% highlight php startinline %}
<?php
class SessionIdParam implements ParamProviderInterface
{
    /**
     * @param Param $param
     *
     * @return mixed
     */
    public function __invoke(Param $param)
    {
        if (isset($_SESSION['login_id'])) {
            // found !
            return $param->inject($_SESSION['login_id']);
        };
        // no idea, ask another provider...
    }
}
{% endhighlight %}

`SessionIdParam` implements the `ParamProviderInterface` interface and recieves parameter data, **when possible** it then prepares the actual parameters and returns them in `$param->inject($args)`.

The parameter provider can register multiple parameters with a matching variable name, the registered provider will then be called by each of them. When none of the providers can prepare all parameters then `BEAR\Resource\Exception\Parameter` exception is thrown.

### The `onProvides` Method

By not setting a variable name and assigning `OnProvidesParam` to '*' then setting up a provided is not needed, it is possible to inject parameters into a class method following a single pattern.

{% highlight php startinline %}
<?php
class Post
{
    public function onPost($date)
    {
        // $date is passed by the onProvidesDate method.
    }

    public function onProvidesDate()
    {
        return date(DATE_RFC822);
    }
}
{% endhighlight %}
In this resource when `$date` is not specified in the client `onProvidesDate` is called, the returned value is passed to the `onPost` method.
In the `onPost` method only the values passed to it are used, which has a clear separation of concerns and gives you a vast improvement in testability.

To use the `onProvides` method functionality simply register the `OnProvidesParam` parameter provider.

{% highlight php startinline %}
<?php
$resource->attachParamProvider('*', new OnProvidesParam);
{% endhighlight %}

### A Clean Layered Architecture

A resource is built up from other resources. Although a resource is a service, a layered architecture can be acheived by the resource also becoming a resource client. A resource is handled with injection and aspect wrapping from the Ray.Di injector, meaning a resource can be built as a clean object with a separation of concerns.

{% highlight php startinline %}
<?php

class News extends ResourceObject
{
    /**
     * @Inject
     */
    public function __construct(ResourceInterface $resource)
    {
        $this->resource = $resource;
    }

    /**
     * @Auth
     * @Cache(60)
     */
    public function onGet()
    {
        $this['domestic'] = $this->resource->get->uri('app://self/news/domestic')->request();
        $this['international'] = $this->resource->get->uri('app://news/international/')->request();
        $this['breaking'] = [
            $this->resource->get->uri('app://self/news/domestic/breaking')->request();
            $this->resource->get->uri('app://self/news/international/breaking')->request();
        ];

        return $this;
    }
}
{% endhighlight %}
In this way the variables in the resource are not `eager`, even when resource contains a request, the resource request made inside the the resource is lazily loaded.
