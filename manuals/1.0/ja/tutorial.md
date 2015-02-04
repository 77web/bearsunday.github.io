---
layout: docs-ja
title: チュートリアル
category: Manual
permalink: /manuals/1.0/ja/tutorial.html
---

## Webサービス作成


プロジェクトを作成します。

{% highlight bash %}
mkdir ~/bear-workshop
cd ~/bear-workshop/
...
{% endhighlight %}

フレームワークをインストールします。

{% highlight bash %}
cd Koriym.Work
composer install
...
{% endhighlight %}

アプリケーションリソースファイルを`src/Resource/App/Weekday.php`に作成します。

{% highlight php %}
<?php

<?php

namespace Koriym\Work\Resource\App;

use BEAR\Resource\ResourceObject;

class Weekday extends ResourceObject
{
    public function onGet($year, $month, $day)
    {
        $date = \DateTime::createFromFormat('Y-m-d', "$year-$month-$day");
        $this['weekday'] = $date->format("D");

        return $this;
    }
}
{% endhighlight %}

作成したリソースにコンソールでアクセスしてみましょう。
まずはエラー、必要な引き数を渡していません。

{% highlight bash %}
php bootstrap/api.php get '/weekday'

400 Bad Request
...
{% endhighlight %}

40Xのエラーはリクエストに問題があるエラーです。
次は引数をつけて正しいリクエスト。

{% highlight bash %}
php bootstrap/api.php get '/weekday?year=2001&month=1&day=1'

code: 200
header:
body:
{
    "weekday": "Mon",
    "_links": {
        "self": {
            "href": "/weekday?year=2001&month=1&day=1"
        }
    }
}
{% endhighlight %}

曜日が`weekday`に入り、このリソースのURIは`/weekday?year=2001&month=1&day=1`として表されてます。

これをWeb APIサービスにしてみましょう。

{% highlight bash %}
php -S 0.0.0.0:8080 bootstrap/contexts/api.php
{% endhighlight %}

RESTクライアント（Chromeアプリの [Advanced REST client](https://chrome.google.com/webstore/detail/advanced-rest-client/hgmloofddffdnphfgcellkdfbfbjeloo/) など）で
`http://0.0.0.0:8080/weekday?year=2001&month=1&day=1` にGETリクエストを送って確かめてみましょう。

GET以外のメソッドは用意されていないので`405 Method Not Allowed`が返されるはずです。これも試してみましょう。

## DI

[monolog](https://github.com/Seldaek/monolog) を使って結果をログする機能を追加してみましょう。
[composer](http://getcomposer.org)で取得します。

{% highlight bash %}
composer require monolog/monolog "~1.0"
{% endhighlight %}

monologログオブジェクトを`new`で直接作成しないで、作成されたログオブジェクトを受け取るようにします。
このように必要なものを自らが取得するのではなく、外部からの代入に期待する仕組みを [DI](http://ja.wikipedia.org/wiki/%E4%BE%9D%E5%AD%98%E6%80%A7%E3%81%AE%E6%B3%A8%E5%85%A5) といいます。

ロガーインターフェイスは`vendor/psr/log`の [PSR-3](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-3-logger-interface.md) のロガーインターフェイスを使用します。


インターフェイスを
BEAR.Sundayの使うDIコンテナ`Ray.Di`ではインターフェイスとその実装クラスを束縛するとバインディングする方法をいくつか提供していますがここでは`Provider`バインディングを使います。詳細は [BEAR.Sundayのマニュアル](http://bearsunday.github.io/manual/ja/di.html) や [Ray.DiのREADME](https://github.com/koriym/Ray.Di) をご覧ください。

最初にインスタンスを用意する`Provider`というファクトリーを作成し、インターフェイス実装に必要な`get`メソッドでインスタンスを返します。

**src/Module/Provider/MonologLoggerProvider.php**

{% highlight php %}
<?php

namespace Koriym\Work\Module\Provider;

use Monolog\Handler\StreamHandler;
use Monolog\Logger;
use Ray\Di\ProviderInterface;

class MonologLoggerProvider implements ProviderInterface
{
    public function get()
    {
        $log = new Logger('monolog');
        $log->pushHandler(
            new StreamHandler(__DIR__ . '/../../../var/log/debug.log',
            Logger::DEBUG)
        );

        return $log;
    }
}
{% endhighlight %}

次にインターフェイス束縛の定義を`AppModule`の`configure`メソッド内で`bind`メソッドを使い行います。

**src/Module/AppModule.php**

{% highlight php %}
class AppModule extends AbstractModule
{
    // ...
    protected function configure()
    {
        $this->install(new StandardPackageModule('Koriym\Work', $this->context, dirname(dirname(__DIR__))));

        $this->bind('Psr\Log\LoggerInterface')
            ->toProvider('Koriym\Work\Module\Provider\MonologLoggerProvider');

        // ...
    }
}
{% endhighlight %}

この定義はキャッシュされる事に注意してください。Webサーバーで確認する時には変更が反映されるようにキャッシュをクリアします。

```
$ php bin/clear.php
```

アプリケーションスクリプトの中で毎回クリアするためにはコメントアウトを外します。

**bootstrap/contexts/dev.php**
{% highlight php %}
//
// The cache is cleared on each request via the following script. We understand that you may want to debug
// your application with caching turned on. When doing so just comment out the following.
//
require $appDir . '/bin/clear.php';
{% endhighlight %}


`Add`クラスのコンストラクタで`monolog`オブジェクトを受け取りプロパティに格納します。ログ出力ではそのロガーを使います。

**src/Resource/App/Add.php**
{% highlight php %}
<?php

namespace Koriym\Work\Resource\App;

use BEAR\Resource\ResourceObject;
use Psr\Log\LoggerInterface;
use Ray\Di\Di\Inject;

class Add extends ResourceObject
{
    private $logger;

    /**
     * @Inject
     */
    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    public function onGet($a, $b)
    {
        $this['result'] = $a + $b;

        $this->logger
            ->debug(sprintf('%d + %d = %d', $a, $b, $this['result']));

        return $this;
    }
}
{% endhighlight %}

フレームワークはメソッドについた`@Inject`を発見すると「ここには依存が必要だ」という事を理解し、モジュールで束縛したインスタンスを代入します。そのメソッドがコンストラクタの時はコンストラクタインジェクション、その他のセッターメソッドのときはセッターインジェクションと呼ばれます。

実行してみて、`var/log/debug.log`に結果が出力されていることを確認しましょう。

## AOPで実行時間を計測

メソッドの実行時間を出力するインターセプターを作成しましょう。

インターセプターは指定されたメソッドを横取り（**intercept**)します。インターセプターから**元のメソッド**を呼びだす時に前後に処理を記述することができます。

メソッドの実行時間を計測するインターセプターはこのようになります。

**src/Interceptor/BenchMarker.php**
{% highlight php %}
<?php

namespace Koriym\Work\Interceptor;

use Ray\Aop\MethodInterceptor;
use Ray\Aop\MethodInvocation;

class BenchMarker implements MethodInterceptor
{
    public function invoke(MethodInvocation $invocation)
    {
        $start = microtime(true);
        $result = $invocation->proceed(); // 元のメソッドの実行
        $time = microtime(true) - $start;

        var_dump($time);

        return $result;
    }
}
{% endhighlight %}

[MethodInvocation](http://www.bear-project.net/Ray.Aop/build/apigen/class-Ray.Aop.MethodInvocation.html) インターフェイスの`$invocation`に「メソッド実行オブジェクト」が渡され`$invocation->proceed()`とすると対象のメソッドを実行します。

例えばこのインターセプターを`Add`クラスの`onGet`メソッドにバインドすると`$invocation->proceed()`で足し算の結果が返ります。

対象メソッドの指定には**matcher**を使います。matcherで指定した条件でメソッドが検索されマッチしたメソッドに、1つまたは複数のインターセプターがバインドされます。まずはクラスやメソッドを名前で検索する`startsWith()`を使用してみましょう。前方一致文字列でマッチします。

**src/Module/AppModule.php**
{% highlight php %}
<?php

namespace Koriym\Work\Module;

use BEAR\Package\Module\Package\StandardPackageModule;
use Koriym\Work\Interceptor\BenchMarker;
use Ray\Di\AbstractModule;
use Ray\Di\Di\Inject;
use Ray\Di\Di\Named;

class AppModule extends AbstractModule
{
    // ...
    protected function configure()
    {
        // ...

        // マッチするメソッドにインターセプターをバインド
        $this->bindInterceptor(
            $this->matcher->startsWith('Koriym\Work\Resource\App\Add'), // クラスの指定
            $this->matcher->any(), // メソッドの指定
            [$this->requestInjection('Koriym\Work\Interceptor\BenchMarker')] //　インターセプター
        );
    }
}
{% endhighlight %}

`AbstractModule::requestInjection()`メソッドはここでは、`new Koriym\Work\Interceptor\BenchMarker()`と同じ動作をします。

これで、`Add`クラスのすべてのメソッドに対して`BenchMarker`がバインドされました。実行して画面に実行時間を表示させましょう。

### アノテーションでバインド

名前によるマッチングでメソッドを指定しましたが、次は`@BenchMark`とアノテーションをつけたメソッドにインターセプターをバインドするように変更しましょう。

まずはアノテーションクラスを実装します。BEAR.Sundayでは [Doctrine Annotation](http://docs.doctrine-project.org/projects/doctrine-common/en/latest/reference/annotations.html) を使います。

アノテーションファイルを作成します。

**src/Annotation/BenchMark.php**
{% highlight php %}
<?php

namespace Koriym\Work\Annotation;

use Ray\Aop\Annotation;

/**
 * @Annotation
 */
class BenchMark implements Annotation
{
}
{% endhighlight %}

バインドの定義を`annotatedWith`を使ったものに変更します。

{% highlight php %}
$this->bindInterceptor(
    $this->matcher->any(),
    $this->matcher->annotatedWith('Koriym\Work\Annotation\BenchMark'),
    [$this->requestInjection('Koriym\Work\Interceptor\BenchMarker')]
);
{% endhighlight %}

計測するメソッドにアノテートします。

**src/Resource/App/Add.php**
{% highlight php %}
<?php

namespace Koriym\Work\Resource\App;

use BEAR\Resource\ResourceObject;
use Psr\Log\LoggerInterface;
use Ray\Di\Di\Inject;
use Koriym\Work\Annotation\BenchMark;

class Add extends ResourceObject
{
    private $logger;

    /**
     * @Inject
     */
    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    /**
     * @BenchMark
     */
    public function onGet($a, $b)
    {
        $this['result'] = $a + $b;

        $this->logger->debug(sprintf('%d + %d = %d', $a, $b, $this['result']));

        return $this;
    }
}
{% endhighlight %}

アノテーションはuse文を使ってフルパスで指定する必要があります。
実行してみて同じように結果が出力されることを確認しましょう。

> AOP機能の詳細は [BEAR.Sundayのマニュアル](http://bearsunday.github.io/manual/ja/aop.html) や [Ray.AopのREADME](https://github.com/koriym/Ray.AOP) をご覧ください。

### インターセプターにDI

実行時間の出力先を画面ではなくログにするように変更しましょう。
ロガーインターフェイスのバインディングがされているので、新たに設定をすることなく`@Inject`をアノテートするだけでログオブジェクトを受け取ることができます。

**src/Interceptor/BenchMarker.php**
{% highlight php %}
<?php

namespace Koriym\Work\Interceptor;

use Psr\Log\LoggerInterface;
use Ray\Aop\MethodInterceptor;
use Ray\Aop\MethodInvocation;
use Ray\Di\Di\Inject;

class BenchMarker implements MethodInterceptor
{
    private $logger;

    /**
     * @Inject
     */
    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    public function invoke(MethodInvocation $invocation)
    {
        $start = microtime(true);
        $result = $invocation->proceed(); // 元のメソッドの実行.
        $time = microtime(true) - $start;

        $this->logger->debug(sprintf('%f sec elapsed', $time));

        return $result;
    }
}
{% endhighlight %}

今までは`$this->requestInjection('Koriym\Work\Interceptor\BenchMarker')`は`new`と同じ動作でしたがここでは依存解決してインスタンスを生成しました。

実行して`var/log/debug.log`に実行時間のログが出力されることを確認しましょう。

## リソース表現

Addリソースは**値**を提供するだけでしたが**表現**も提供するアプリケーションリソースを作成してみましょう。

**src/Resource/App/User.php**
{% highlight php %}
<?php

namespace Koriym\Work\Resource\App;

use BEAR\Resource\ResourceObject;
use BEAR\Sunday\Annotation\Cache;

class User extends ResourceObject
{
    private $data = [
        0 => ['name' => 'BEAR',   'age' => 10],
        1 => ['name' => 'Sunday', 'age' => 3]
    ];

    /**
     * @Cache(10)
     */
    public function onGet($id = 0)
    {
        $this->body = $this->data[$id];

        return $this;
    }
}
{% endhighlight %}

次にリソースを表現にするためのテンプレートを作成します。

**src/Resource/App/User.twig**
{% highlight php %}
<div>
    <h2>User</h2>
    Name: {{ name }}<br>
    Age: {{ age }}<br>
</div>
{% endhighlight %}

ユーザーリソースの値はテンプレートと合成されページにセットされます。**@Cache**とアノテートしているので指定した10秒間テンプレートの合成も含めてキャッシュされます。

## リソースの埋め込み

次に`@Embed`アノテーションを使ってユーザーリソースを埋め込んで（セットして）みましょう。

**src/Resource/Page/User.php**
{% highlight php %}
namespace Koriym\Work\Resource\Page;

use BEAR\Resource\ResourceObject;
use BEAR\Resource\Annotation\Embed;

class User extends ResourceObject
{
    /**
     * @Embed(rel="user", src="app://self/user{?id}")
     */
    public function onGet($id)
    {
        return $this;
    }
}
{% endhighlight %}

`rel`で指定された名前で`src`で指定したリソースが埋め込まれます。URIには`onGet`で渡された`$id`をそのままappリソースに渡しています。

> URIに動的要素を加える時は [RFC-6570](http://tools.ietf.org/html/rfc6570) の [URI Template](https://github.com/ioseb/uri-template) を用います。

この`@Embed`アノテーションによるリソースのセットは以下のコードと同じ事を行っています。

{% highlight php %}
    public function onGet($id)
    {
        $this['user'] = $this->resource
            ->get
            ->uri('app://self/user')
            ->withQuery(['id' => $id])
            ->request();

        return $this;
    }
{% endhighlight %}

HTMLの`<img>`や`<script>`、それに`<iframe>`タグをイメージしてみてください。`src`で指定される他のリソースを自身のリソースに埋め込んでいます。`@Embed`タグも同じように埋め込んでいます。

**src/Resource/Page/User.twig**
{% highlight html %}
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="utf-8">
    <title>Wlecome to {{ user.name }} page</title>
    <link href="//netdna.bootstrapcdn.com/bootswatch/3.0.0/united/bootstrap.min.css" rel="stylesheet">
</head>
<body>
<div class="container">
    <div>{{ user }}</div>
</div>
</body>
</html>
{% endhighlight %}

ページのテンプレートでは`{{ user }}`でユーザーリソースのプレースフォルダを指定します。プレースフォルダにはアプリケーションリソースがテンプレートと合成されたマイクロコンテンツ（部分的なHTML）が展開されます。

`<title>`タグでは`{{ user.name }}`としてユーザーリソースのnameという要素を利用しています。このようにリソースは他のリソースに埋め込まれ文字列として評価される（`{{ user }}`）と表現（マイクロコンテンツ）になり、配列として評価される（`{{ user.name }}`）とその値が取り出されます。

## マイクロコンテンツ

作成したリソースは他のシステムから容易に利用することができます。

**my_system.php**
{% highlight php %}
<?php
$app = require __DIR__ . '/bootstrap/instance.php';
$user = $app->resource
    ->get
    ->uri('app://self/user')
    ->withQuery(['id' => 0])
    ->eager->request();
?>

<!DOCTYPE html>
<head>
    <title>"<?php echo $user['name']; ?>" in my system</title>
</head>
<body>
<?php echo $user; ?>
</body>
</html>
{% endhighlight %}

`instance.php`はアプリケーションインスタンスで`require`で取得します。初期化は必要ありません。次にリソースクライアントを使って`app://self/user`リソースを取得しています。

Symfony等他のフレームワークやWordPress等他のCMSからBEAR.Sundayで作成したリソースを利用することができます。

コンソールで試してみましょう

{% highlight bash %}
$ php my_system.php
{% endhighlight %}

{% highlight html %}
<!DOCTYPE html>
<head>
    <title>"BEAR" in my system</title>
</head>
<body>
<div class="app-user">
    <h2>User</h2>
    Name: BEAR<br>
    Age: 10<br>
</div>
</body>
</html>
{% endhighlight %}
