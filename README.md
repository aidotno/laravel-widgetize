
### When to use it ?

>This package helps you in stuations that you want to create crowded web pages with multiple widgets (on sidebar, menu, ...) and each widget needs seperate sql queries and php logic to be provided with data for its template. If you need a small application with low traffic this package is not much of a help.Anyway installing it has minimal overhead since surprisingly it is just a small abstract class and Of course you can use it to refactor your monster code and tame it into managable pieces or boost the performance 4x-10x times faster. ;)



### So what is our problem ?
#### Problem 1 : Controllers easily get crowded :(
>Imagine An online shop like amazon which shows the list of products, popular products, etc (in the sidebar), user data and basket data in the navbar and a tree of product categories in the menu and etc... In traditional good old MVC model you have a single controller method to provide all the widgets with data. You can immidiately see that you are violating the SRP (Single Responsibility Priciple)!!! The trouble is worse when the client changes his mind over time and asks the deveploper to add, remove and modify those widgets on the page. And it always happens. Clients do change their minds.The developoer's job is to be ready to cope with that as effortlessly as possible.

#### Problem 2 : Page caching is always hard :( 
>Trying to cache the pages which include user specific data (for example the username on the top menu) is a often fruitless. Because each user sees slightly different page from other users. Or in cases when we have some parts of the page(recent products section) which update frequently and some other parts which change rarly... we have to expire the entire page cache to match the most most frequently updated one. :(
AAAAAAAAAhh...


#### Problem 3 : View templates easily get littered with if/else blocks (&_&)
>We ideally want our view files to be as logicless as possible and very much like the final output HTML. if/else blocks and other computations are always irritating within our views. specially for static page designers in our team. We just want to print out already defined variables wiout the to decide what to print. Anyway the data we store in database are sometimes far from ready to be printed on the page.

========================

#### So, How to fight against those ? ;(
>__The main idea is simple, Instead of one controller method to handle all widgets of the page, Each widget should have it's own `controller class`, `view partial`, `view presenter class` and `cache config`, isolated from others.__
>That's it !! :)

### How this package is going to help us ? (@_@)

1. It helps you to conforms to SRP (`single responsibility principle`) in your controllers (Because each widget class is only responsible for one and only one widget of the page but before you had a single controller method that was resposible for all the widgets. Effectively exploding one controller method into multiple widget classes.)
2. It helps you to conforms to `Open-closed principle`. (Because if you want to add a widget on your page you do not need to touch the controller code. Instead you create a new widget class from scratch.)
3. It optionally `caches the output` of each widget. (which give a very powerful, flexible and easy to use caching opportunity) You can set different cache config for each part of the page. Similar to `ESI` standard.
4. It executes the widget code `Lazily`. Meaning that the widget's data method `public function data(){` is hit only and only after the widget object is forced to be rendered in the blade file like this: `{!! $widgetObj !!}`, So for example if you comment out `{!! $widgetObj !!}` from your blade file then all database queries will be disabled automatically. No need to comment out the controller codes anymore...
5. It optionally `minifies` the output of the widget. Removing the white spaces. (In order to save cache storage space and bandwidth)
6. It support the `nested widgets` tree structure. (Use can inject and use widgets within widgets.)



### Installation:

`composer require imanghafoori/laravel-widgetize`

>Add `Imanghafoori\Widgets\WidgetsServiceProvider::class` to the providers array in your config/app.php
>Now you are free to extend the `Imanghafoori\Widgets\BaseWidget` abstract class which enforces to implement the public `data` method in your sub-class.

### Configuration:
you can set the variables in your .env file to globally set some configs for you widgets and override them if needed.

__WIDGET_MINIFICATION=true__ (you can turn off HTML minification in development)

__WIDGET_CACHE=true__ (you can turn caching on and off for all widgets.)

__WIDGET_IDENTIFIER=true__ (you can turn off widget identifiers in production)


###Guideline:

>1. So we first extract each widget into it's own partial. (recentProducts.blade.php)
>2. Use `php artisan make:widget` command to create your widget class.
>3. Set configurations like __$cacheLifeTime__ , __$template__, etc and implement the `data` method.
>4. Your widget is ready to be instanciated and be used in your view files. (see example below)



###Example : How to create a Widget?

>__You can use : `php artisan make:widget MyWidget` to make your widget class.__

Sample widget class :
```php
namespace App\Widgets;

use Imanghafoori\Widgets\BaseWidget;


class RecentProductsWidget extends BaseWidget
{
    protected $template = 'widgets.recentProducts.blade.php'; // referes to: views/widgets/recentProducts.blade.php
    protected $cacheLifeTime = 1; // 1(min) ( 0 : disable, -1 : forever) default: 0
    protected $friendlyName = 'A Friendly Name Here'; // Showed in html Comments
    protected $context_as = '$recentProducts'; // you can access $recentProducts in view file (default: $data)
    protected $minifyOutput = true; // minifies the html before storing it in the cache to save storage space.

    // The data returned here would be available in widget view file.
    public function data($param1=5)
    {
        // It's the perfect place to query the database for your widget...
        return Product::orderBy('id', 'desc')->take($param1)->get();;

    }
}
```

==============

__Tip__ : If you do not set the `$template` by default it looks for a template file within the app\Widgets folder with the same name as the class name.
This means that you can put your view partials beside your widget class. like this:

| app\Widgets\RecentProductsWidget.php
| app\Widgets\RecentProductsWidget.blade.php

you can group the in folders if you want to: 

| app\Widgets\Homepage\RecentProductsWidget.php
| app\Widgets\Homepage\RecentProductsWidget.blade.php

=================

views/widgets/recentProducts.blade.php

```blade
<ul>
  @foreach($recentProducts as $product)
    <li>
      <h3> {{ $product->title }} </h3>
      <p>$ {{ $product->price }} </p>
    </li>
  @endforeach
</ul>
```

Ok, Now it's done! We have a ready to use widget. let's use it...


###How to leverage a Widget Object?

In your typical controller methods (or somewhere else) we may instanciate our widget classes and pass the resulting object to our view like this:
```php

use \App\Widgets\RecentProductsWidget;

public function index(RecentProductsWidget $recentProductsWidget)
{
    return view('home', compact('recentProductsWidget'));
}
```

And then you can render it in your view (home.blade.php) like this:
```blade
<div class="container">
    <h1>Hello {{ auth()->user()->username }} </h1>
    <br>
    {!! $recentProductsWidget !!}
    <p> if you need to pass parameters to data method :</p>
    {!! $recentProductsWidget(10) !!}
</div>
```

=============

In order to easily understand what's going on here...
Think of `{!! $recentProductsWidget !!}` as `@include('widgets.recentProductsWidget')` but more sophisticated.
The actual result is the same piece of HTML, which is the result of rendering the partial.

Pro tip: After `{!! $myWidget('param1') !!}` is executed in your view file, the `data` method is called on your widget class with the correspoding parameters. `But only if it is Not already cached` or the `protected $cacheLifeTime` is set to 0.
If the widget HTML output is already in the cache it prints out the HTML with out executing `data` method (hence performing database queries) or even rendering the blade file.
You may want to look at the BaseWidget source code and read the comments for more information.

=============

