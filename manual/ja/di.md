---
layout: default_ja
title: BEAR.Sunday | Dependency Injection
category: Manual
---


Dependency Injection フレームワーク
==============================

**Ray.Di**はGoogleのJava用DI framework [Guice]((http://code.google.com/p/google-guice/wiki/Motivation?tm=6)の主要な機能を持つアノテーションベースのDIフレームワークです。
DIを効率よく使用すると以下のようなメリットがあります。

* ロジックとコンフィギュレーションの分離を促進し、ソースコードを読みやすくします。
* コンポーネントの独立性と再利用性を強化します。コンポーネントは依存関係のあるインタフェースを宣言するだけになるため、他のコンポーネントとの関係を疎結合にし再利用性を高めます。
* コーディング量を減少させます。インジェクションの処理そのものはインジェクターが提供するためその分だけ実装するコードの量が減ります。多くの場合、依存を受け取る為のtraitの`use`文を記述するだけです。

Ray.Diは以下の特徴があります。

 * [JSR-250](http://en.wikipedia.org/wiki/JSR_250)のオブジェクトライフサイクル(`@PostConstruct`, `@PreDestroy`)のアノテーションをサポートしています。
 * [AOP Alliance](http://aopalliance.sourceforge.net/)に準拠したアスペクト指向プログラミングをサポートしています。
 * [Aura.Di](http://auraphp.github.com/Aura.Di)を拡張しています。
 * [Doctrine.Commons](http://www.doctrine-project.org/projects/common)アノテーションを使用しています。

概要
--------------

Ray.Diを使ったディペンデンシーインジェクション（[依存性の注入](http://ja.wikipedia.org/wiki/%E4%BE%9D%E5%AD%98%E6%80%A7%E3%81%AE%E6%B3%A8%E5%85%A5)）の一般的な例です。

```php
<?php
use Ray\Di\Injector;
use Ray\Di\AbstractModule;

interface FinderInterface
{
}

class Finder implements FinderInterface
{
}

class Lister
{
    public $finder;

    /**
     * @Inject
     */
    public function setFinder(FinderInterface $finder)
    {
        $this->finder = $finder;
    }
}

class Module extends \Ray\Di\AbstractModule
{
    public function configure()
    {
        $this->bind('MovieApp\FinderInterface')->to('MovieApp\Finder');
    }
}

$injector = Injector::create([new Module]);
$lister = $injector->getInstance('MovieApp\Lister');
$works = ($lister->finder instanceof MovieApp\Finder);
echo(($works) ? 'It works!' : 'It DOES NOT work!');

// It works!
```

これは **Linked Bindings** という束縛（バインディング）です。 Linked bindings はインターフェイスとその実装クラスを束縛します。

### Provider バインディング

[Provider bindings](http://code.google.com/p/rayphp/wiki/ProviderBindings) はインターフェイスと実装クラスの`プロバイダー`を束縛します。

シンプルでインスタンス（値）を返すだけの、Providerインターフェイスを実装したプロバイダークラスを作成します。

```php
<?php
use Ray\Di\ProviderInterface;

interface ProviderInterface
{
    public function get();
}
```

このプロバイダーの実装は自身にコンストラクターで`@Inject`とアノテートしている依存があります。
依存を使ってインスタンスを生成して`get()`メソッドで生成したインスタンスを返します。

```php
<?php
class DatabaseTransactionLogProvider implements Provider
{
    private ConnectionInterface connection;

    /**
     * @Inject
     */
    public DatabaseTransactionLogProvider(ConnectionInterface $connection)
    {
        $this->connection = $connection;
    }

    public TransactionLog get()
    {
        $transactionLog = new DatabaseTransactionLog;
        $transactionLog->setConnection($this->connection);

        return $transactionLog;
    }
}
```

このように依存が必要なインスタンスには **Provider Bindings** を使います。

```php
<?php
$this->bind('TransactionLogInterface')->toProvider('DatabaseTransactionLogProvider');
```

### Named バインディング

Rayには`@Named`という文字列で`名前`を指定できるビルトインアノテーションがあります。

```php
<?php
/**
 *  @Inject
 *  @Named("processor=Checkout")
 */
public RealBillingService(CreditCardProcessor $processor)
{
...
```

特定の名前を使って束縛するために`annotatedWith()`メソッドを使います。

```php
<?php
protected function configure()
{
    $this->bind('CreditCardProcessorInterface')->annotatedWith('Checkout')->to('CheckoutCreditCardProcessor');
}
```

### Instance バインディング

値を直接束縛することができます。依存のないオブジェクトや配列やスカラー値などの時だけ利用するようにします。

```php
<?php
protected function configure()
{
    $this->bind('UserInterface')->toInstance(new User);
}
```

PHPのスカラー値には型がないので、名前を使って束縛します。

```php
<?php
protected function configure()
{
    $this->bind()->annotatedWith("login_id")->toInstance('bear');
}
```

### Constructor バインディング

外部のクラスなどで`@Inject`が使えない場合などに、任意のコンストラクタに型を束縛することができます。

```php
<?php
class TransactionLog
{
    public function __construct($db)
    {
     // ....
```

変数名を指定して束縛します。

```php
<?php
protected function configure()
{
    $this->bind('TransactionLog')->toConstructor(['db' => new Database]);
}
```

## スコープ

デフォルトでは、Rayは毎回新しいインスタンスを生成しますが、これはスコープの設定で変更することができます。

```php
<?php
protected function configure()
{
    $this->bind('TransactionLog')->to('InMemoryTransactionLog')->in(Scope::SINGLETON);
}
```

## オブジェクトのライフサイクル

オブジェクトライフサイクルのアノテーションを使ってオブジェクトの初期化や、PHPの終了時に呼ばれるメソッドを指定する事ができます。

このメソッドは全ての依存がインジェクトされた後に呼ばれます。
セッターインジェクションがある場合などでも全ての必要な依存が注入された前提にすることができます。

```php
<?php
/**
 * @PostConstruct
 */
public function onInit()
{
    //....
}
```

このメソッドはPHPの **register_shutdown_function** 関数に要録されスクリプト処理が完了したとき、あるいは *exit()* がコールされたときに呼ばれます。

```php
<?php
/**
 * @PreDestroy
 */
public function onShutdown()
{
    //....
}
```

## 自動インジェクション

Ray.Diは`toInstance()`や`toProvider()`がインスタンスを渡した時に自動的にインジェクトします。
またインジェクターが作られたときにそのインジェクターはモジュールにインジェクトされます。依存にはまた違う依存があり、順に辿って依存を解決します。

## Install

モジュールは他のモジュールの束縛をインストールして使う事ができます。

 * 同一の束縛があれば先にされた方が優先されますが
 * `$this`を渡すとそれまでの束縛をインストール先のモジュールが利用することができます。そのモジュールでの束縛は現在の束縛より優先されます。

```php
<?php
protected function configure()
{
    $this->install(new OtherModule);
    $this->install(new CustomiseModule($this);
}
```

## モジュール内でのインジェクション

既存の束縛を使うモジュールの中でビルトインのインジェクターを利用できます。

```php
<?php
protected function configure()
{
    $this->bind('DbInterface')->to('Db');
    $dbLogger = $this->requestInjection('DbLogger');
}
```

## アスペクト指向プログラミング

Ray.Aopのアスペクト指向プログラミングが利用できます。インターセプターの束縛はより簡単になり、アスペクトの依存解決も行われます。

```php
<?php
class TaxModule extends AbstractModule
{
    protected function configure()
    {
        $this->bindInterceptor(
            $this->matcher->subclassesOf('Ray\Di\Aop\RealBillingService'),
            $this->matcher->annotatedWith('Tax'),
            [new TaxCharger]
        );
    }
}
```

```php
<?php
class AopMatcherModule extends AbstractModule
{
    pro
    protected function configure()
    {
        $this->bindInterceptor(
            $this->matcher->any(),                 // In any class and
            $this->matcher->startWith('delete'), // ..the method start with "delete"
            [new Logger]
        );
    }
}
```
