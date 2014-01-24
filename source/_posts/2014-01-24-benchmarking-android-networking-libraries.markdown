---
layout: post
title: "Benchmarking Android networking libraries - Part 1: Rest API Testbed."
date: 2014-01-24 08:39:46 -0600
comments: true
categories:
- android
- hack day
- experiments
author: Iain Connor
---

## Preamble

Over the last few months here at [Tipping Canoe](http://tippingcanoe.com/) we've really been trying to up our Android development game. Our primary web-language, [PHP](http://www.php.net/) has been going through somewhat of a renaissance, largely spurred by usage of [Composer](https://getcomposer.org/) to think of your Application as a set of small, reusable, and interchangeable components controlled by a proper dependency management system. The Android world seems to be lagging behind on this, with a vast majority of tutorials online (including [loads from Google](http://developer.android.com/tools/support-library/setup.html)) suggesting that manually copying `.jar` files around is a sane way of managing your Application's dependencies.

{% pullquote %}
Luckily, it seems there might be light at the end of the tunnel. Thanks to the ["new build system"](http://tools.android.com/tech-docs/new-build-system) powered by Gradle, Android developers can now declare their dependencies in a configuration file and have the build system be responsible for filling those dependencies. I'll hopefully be doing another blog post about our journey moving to this new way of doing things later, but the short version is, {" if you're using Eclipse and ANT, you're doing modern Android development wrong "}. Configuration-hell is a place you shouldn't have to live in.
{% endpullquote %}

Since this change, the quality and quantity of open source Android libraries is slowly slowly starting to grow, and it's an important part of our job to vet what's available out there to make sure we're making the right decisions -- which lead me to this experiment. A common theme across all of our current Applications is that they all rely on and communicate with a remote REST client and retrieve images and other resources from remote locations.

## Contenders

After a bit or lurking in [/r/androiddev](http://www.reddit.com/r/androiddev/), here are what I believe to be the contenders.

For networking/REST;

1. [Retrofit](http://square.github.io/retrofit/) by Square.
2. [Volley](http://developers.google.com/events/io/sessions/325304728) by Google.
3. [ion](http://github.com/koush/ion) by Koushik Dutta.
4. [Robospice](http://github.com/octo-online/robospice) by Octo Technology.

For image loading;

1. [Volley](http://developers.google.com/events/io/sessions/325304728) by Google.
2. [ion](http://github.com/koush/ion) by Koushik Dutta.
3. [Picasso](http://square.github.io/picasso/) by Square.
 
## Testbed

To accurately compare these libraries, we're going to need a simple testbed to act as a REST server. Lately, we've been using [Laravel 4](http://laravel.com/), a PHP framework, for lots of our API development, so I'm going to build a quick testbed using it.

``` bash
composer create-project laravel/laravel testbed --prefer-dist
```

### Tables

First, we're going to need a database to save our results to. For quick bootup, we're just going to go into the `/app/config/database.php` and setup a SQLite database.

``` php
'default' => 'sqlite',
```

Then we'll need to create some database tables for some Models we're going to need storage for.

``` bash
php artisan migrate:make create_users_table --create=users
php artisan migrate:make create_requests_table --create=requests
```

We'll add some information to those tables.

``` php
Schema::create('users', function(Blueprint $table)
{
	$table->increments('id');
	$table->string('username');
	$table->string('email');
	$table->boolean('is_admin');
	$table->timestamps();
});

Schema::create('requests', function(Blueprint $table)
{
	$table->increments('id');
	$table->integer('client_id')->unique();
	$table->enum('method', ['GET', 'POST']);
	$table->string('path');
	$table->string('parameters');
	$table->enum('engine', ['retrofit', 'volley', 'ion', 'robospice']);
	$table->timestamp('start_time');
	$table->timestamp('end_time');
	$table->timestamps();
});
```

And we'll run those migrations to actually create the tables.

``` bash
php artisan migrate
```

### Models

Now that we've got storage rolling, we'll need to make some Models.

``` php
class Request extends Eloquent {
	protected $table = 'requests';
	public $timestamps = true;
	protected $softDelete = false;
}

class User extends Eloquent {
	protected $table = 'users';
	public $timestamps = true;
	protected $softDelete = false;
}
```

### Routes 

And now we'll add some REST routes to the `/app/routes.php` file.  
*Note: If you add code into your routes file like this, it'll make web-developers cry. Use Controllers.*

``` php
$startTime = time();

Route::filter('requestEnd', function($route, $request, $response, $startTime) {
	$requestTiming = new RequestTiming(Input::all());
	$requestTiming->method = $request->server('REQUEST_METHOD');
	$requestTiming->path = $request->path();
	$requestTiming->parameters = json_encode(Input::all());
	$requestTiming->start_time = date("Y-m-d H:m:s", $startTime);
	$requestTiming->end_time = date("Y-m-d H:m:s");
	$requestTiming->save();
});

Route::group(['prefix' => 'api', 'after' => 'requestEnd:' . $startTime], function() {
	// Get some sample JSON
	Route::get('/', function() {

		return Response::json("Hello World!");
	});

	// Get an array of all Users
	Route::get('/user', function() {
		$users = User::all();

		return Response::json($users);
	});

	// Get a specific User
	Route::get('/user/{id}', function($id) {
		$user = User::findOrFail($id);

		return Response::json($user);
	});

	// Create a new User
	Route::post('/user', function() {
		$user = User::create(Input::all());

		if ( $avatar = Input::hasFile('avatar') ) {
			$avatar->move(storage_path(), "avatar_" . $user->id . "." . $avatar->getClientOriginalExtension());
		}

		return Response::json($user);
	});

	// Update a User
	Route::post('/user/{id}', function($id) {
		$user = User::findOrFail($id);
		$user->update(Input::all());

		return Response::json($user);
	});
});
```

## Repository

I've committed the source code for this example up to [Github](http://github.com/TippingCanoe/rest-api-testbed).

## Next steps

Next, we're going to move over to Android and get these libraries installed!