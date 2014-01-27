---
layout: post
title: "Benchmarking Android Networking Libraries - Part 4: Looking at Robospice"
date: 2014-01-27 09:01:14 -0600
comments: true
categories:
- android
- hack day
- experiments
author: Iain Connor
---

## Preamble

Last time, we took a [look at ion](http://upthecreek.tippingcanoe.com/blog/2014/01/27/benchmarking-android-networking-libraries-part-3-looking-at-ion/). This time, we're going to take a closer look at the syntax and features of another contender, Robospice.

After looking at [some samples](https://github.com/octo-online/RoboSpice-samples) provided by the Robospice team, it's becoming apparent that Robospice is a bit different than the other contenders. Whereas something like Retrofit is concerned with making requests and processing the results, Robospice is more concerned with the mechanics of where and how those requests take place, and what is done with the responses to make future requests easier. Robospice proposes that [Android Services](http://developer.android.com/guide/components/services.html) are where networking should occur, rather than in a simple separate thread.

Robospice can be used with one of many of what it calls `SpiceServices` and each of those bring support for particular features to the table. Interestingly enough, there's a pre-built [`SpiceService` for Retrofit](https://github.com/octo-online/robospice/wiki/Retrofit-module), so we're going to jump on that and see what Robospice really brings to the table ontop of Retrofit.

## Common setup

Setup for our POJO objects is the same as described in [part 3](http://upthecreek.tippingcanoe.com/blog/2014/01/27/benchmarking-android-networking-libraries-part-3-looking-at-ion/).

## Setup

Stepping back slighty, in [part 2](http://upthecreek.tippingcanoe.com/blog/2014/01/24/benchmarking-android-networking-libraries-part-2-installing-libraries/) we actually installed the wrong version of Robospice. There's a version specifically built for using alongside Retrofit.

``` groovy
compile 'com.octo.android.robospice:robospice-retrofit:1.4.2'
```
 
Since we're using Retrofit under the hood, we'll need to create an interface for our API.  
*Note: This code snippet returns the result directly, as opposed to using a callback. Retrofit uses this as a mechanic to determine wheter to execute the request synchronously (which it does when values are returned) or asynchronously (when values are consumed by a callback). In the Robospice world, the request is already being done in a Service, so there is no need to spawn a separate thread.*

``` java
public interface Testbed {
	@GET("/")
	List<String> getIndex();

	@GET("/user")
	List<User> getUsers();

	@GET("/user/{id}")
	User getUser(@Path("id") int userId);

	@POST("/user")
	User createUser(@Body User user);

	@POST("/user/{id}")
	User updateUser(@Path("id") int userId, @Body User user);
}
```

And we'll also need a Robospice Service class to represent our API. The overriden `protected Converter createConverter()` method is used to inject the `new GsonBuilder()` that we made in the common setup of [part 3](http://upthecreek.tippingcanoe.com/blog/2014/01/27/benchmarking-android-networking-libraries-part-3-looking-at-ion/).

```
public class TestbedSpiceService extends RetrofitGsonSpiceService {
	private final static String BASE_URL = "@TODO";

	@Override
    public void onCreate() {
        super.onCreate();
        addRetrofitInterface(Testbed.class);
    }

    @Override
    protected String getServerUrl() {
        return BASE_URL;
    }

    @Override
    protected Converter createConverter() {
    	return new GsonConverter(gson);
    }
}
```

Note we can also use our Jackson converter with Robospice via Retrofit. Here's how to set it up, but we'll be omitting these examples for the moment. The overriden `protected Converter createConverter()` method is used to inject the `new ObjectMapper()` that we made in the common setup of [part 3](http://upthecreek.tippingcanoe.com/blog/2014/01/27/benchmarking-android-networking-libraries-part-3-looking-at-ion/).

``` java
public class TestbedSpiceServiceJackson extends RetrofitJackson2SpiceService {
	private final static String BASE_URL = "@TODO";

	@Override
    public void onCreate() {
        super.onCreate();
        addRetrofitInterface(Testbed.class);
    }

    @Override
    protected String getServerUrl() {
        return BASE_URL;
    }

    @Override
    protected Converter createConverter() {
    	return new JacksonConverter(jackson);
    }
}
```

That new service will need to be declard in our `AndroidManifest.xml`.

``` xml
<service
	android:name="TestbedSpiceService"
	android:exported="false" />
```

## Forming a request

Robospice uses classes to represent requests and responses, so we'll need to create one for each pair.

Requests go into their own Class files.

``` java
public class GetUsersRequest extends RetrofitSpiceRequest<List<User>, Testbed> {
	public GetUsersRequest() {
		super(List<User>.class, Testbed.class);
	}

	@Override
	public List<User> loadDataFromNetwork() {

		return getService().getIndex();
	}
}
```

## Handling a response

Responses go into each Activity (or Fragment) that's listening for the response. Conceptually, this makes a lot of sense - requests are general purpose and likely reused throughout the Application, and responses need UI work that's specific to that situation.

Robospice requires some additions to our Activities in order for that to function. The idea of having boilerplate code laying around to support a Library doesn't sound that great, but it's minimal for now. Of course, we're going to be clever developers and add this into an abstract Activity subclass so that we don't have to copy/paste it into all of our Activities down the line.

``` java
public abstract class SpicedActivity extends Activity {
	private SpiceManager spiceManager = new SpiceManager(TestbedSpiceService.class);

	@Override
    protected void onStart() {
        spiceManager.start(this);
        super.onStart();
    }

    @Override
    protected void onStop() {
        spiceManager.shouldStop();
        super.onStop();
    }

    protected SpiceManager getSpiceManager() {
        return spiceManager;
    }
}
```

And then we'll add the handling of the response from the service in our Activity that extends the `SpicedActivity`.

``` java
public class @TODO extends SpicedActivity {
	private GetUsersRequest getUsersRequest;

	public final class GetUsersRequestListener implements RequestListener<List<User>> {
		@Override
		public void onRequestFailure(SpiceException spiceException) {

		}

		@Override
		public void onRequestSuccess(final List<User> result) {

		}
	}
}
```

## Executing a request

And then we just need to add a line to instantiate and trigger the request and pipe the result to the response.

``` java
getUsersRequest = new GetUsersRequest();
getSpiceManager().execute(getUsersRequest, "getUsers", DurationInMillis.ONE_MINUTE, new GetUsersRequestListener());
```

## Other features

#### Cacheing
That last line has a couple of neat features. First, we can set a key by which to recognize future requests of this type in a cache (we're just using a dumb string `"getUsers"`, but you could implement your own mechasim of course) and second, we can state how long that cache is valid for. In this example, if we were to execute the same request 2 times within a minute, the n+1 requests wouldn't actually trigger any network activity.

#### Priority
Requests can have a priority attached to them, and Robospice will attempt to execute higher priority ones first.
``` java
getUsersRequest.setPriority(SpiceRequest.PRIORITY_HIGH);
```

#### Cancelling
Requests can be cancelled in a few ways. The first is to use the request itself.
``` java
getUsersRequest.cancel();
```	
But that could be suboptimal because the Object may have left your context. A better way is by using the cache key.
``` java
spiceManager.cancel("getUsers");
```
	
#### Retrying
If requests fail (for example, the user's network connectivity drops in the middle of sending a request), Robospice can automatically queue those requests again. A default [`DefaultRetryPolicy`](https://github.com/octo-online/robospice/blob/release/robospice-core-parent/robospice/src/main/java/com/octo/android/robospice/retry/DefaultRetryPolicy.java) exists for requests by default, but you can implement your own by implementing [`RetryPolicy`](https://github.com/octo-online/robospice/blob/release/robospice-core-parent/robospice/src/main/java/com/octo/android/robospice/retry/RetryPolicy.java).
``` java
getUsersRequest.setRetryPolicy(YourRetryPolicy);
```
	
#### Image loading
Robospice *does* support downloading images, but does so by requiring you to use a [`UI SpiceList`](https://github.com/octo-online/robospice/wiki/UI-SpiceList-module) that imposes several requirements on your ListViews/Adapters. After trying to work with the sample, we decided this implementation was not worth persuing.

## Thoughts

That was a lot of work and, possibly, a lot of duplication between Retrofit and Robospice. Let's imagine that the '/users' route of our API suddenly changes and adds a parameter to toggle some functionality. In this example, we would need to touch `public interface Testbed`, `public class GetUsersRequest`, and our Activity. Here be dragons.

On the other hand, it's possible that this work is worth it. The Robospice team seems [confident that Services are the way to go](https://raw.github.com/octo-online/robospice/master/gfx/RoboSpice-InfoGraphics.png) for networking request, and the cacheing features are a nice bonus.

The fact that Robospice can work with a variety of Services makes it so their documentation is quite difficult to parse. You'll find some examples designed for Spring, some for OkHTTP, etc. We found that the guides and documentation do not make it clear how to swap between these Services, and can easily lead you astray.