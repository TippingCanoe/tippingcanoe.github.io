---
layout: post
title: "Benchmarking Android Networking Libraries - Part 3: Looking at ion"
date: 2014-01-27 09:00:59 -0600
comments: true
categories:
- android
- hack day
- experiments
author: Iain Connor
---

## Preamble

Last time, we managed to get all of our [contenders installed](http://upthecreek.tippingcanoe.com/blog/2014/01/24/benchmarking-android-networking-libraries-part-2-installing-libraries/). This time, we're going to take a closer look at the syntax and features of one of our contenders, ion.

ion comes from the creator of [AnroidAsync](https://github.com/koush/AndroidAsync) and uses that library for its low-level networking. This is actually the library that we're currently using in our legacy Applications.

## Common setup

#### POJO 
A POJO is, simply, a plain-old-java-object. These are just your standard, existing business objects.

#### Gson
[Gson](https://code.google.com/p/google-gson/) is a library used to convert between POJOs and JSON.

#### Jackson 
[Jackson](https://github.com/FasterXML/jackson) is an alternative library used to convert between POJOs and JSON. [Some](http://blaazinsoftwaretech.blogspot.ca/2013/08/jackson-2-vs-gson-performance-comparison.html) [developers](http://daniel-codes.blogspot.ca/2013/12/streaming-json-parsing-performance-test.html) report it being much faster than GSON, so we'll be testing both out.

So, we're going to need a POJO for Gson & Jackson to use to represent our User object.

``` java
public class User {
	protected int id;
	protected String username;
	@SerializedName("email");
	protected String emailAddress;
	protected boolean isAdmin;

	// Constructor/getters/setters ommitted
}
```

The one line there that might not look familliar is the `@SerializedName` annotation. If you remember back to [part one](http://upthecreek.tippingcanoe.com/blog/2014/01/24/benchmarking-android-networking-libraries/), our testbed's User object had the property for the email address as `email`, not `emailAddress`. This annotation will let us override it.

You'll also remember that the User object had a property called `is_admin`, not `isAdmin`. We could fix this in the same way as the email address, but Gson provides a nice way of altering underscore/camelcaseing in a standard way.

``` java
Gson gson = new GsonBuilder()
	.setFieldNamingPolicy(FieldNamingPolicy.LOWER_CASE_WITH_UNDERSCORES)
	.create();
```

Of course, there are loads more features, so you should be familliar with [how to work with Gson](https://sites.google.com/site/gson/gson-user-guide) as many of the REST libraries we're going to be testing use it.

And just to keep the annotations separate in our minds, we'll create a separate class for Jackson.

``` java
@JsonIgnoreProperties(ignoreUnknown = true)
@JsonAutoDetect
public class UserJackson {
	@JsonProperty
	protected int id;
	@JsonProperty
	protected String username;
	@JsonProperty(value = "email")
	protected String emailAddress;
	@JsonProperty
	protected boolean isAdmin;

	// Constructor/getters/setters ommitted
}
```

A few more annotations required there, so let's go through them.
* `@JsonIgnoreProperties(ignoreUnknown=true)` simply flags Jackson to say that if there are additional properties in the JSON that aren't a part of this Object to not worry about it. Just ignore them.
* `@JsonAutoDetect` signals Jackson that we want it to automatically find mappings between our POJO and JSON.
* `@JsonProperty` signals that this property is mapped to JSON. We can also override the name using the `value = "foo"` syntax.

Same as with GSON, we'll need to apply a strategy to overcome `is_admin` to `isAdmin`.

``` java
ObjectMapper jackson = new ObjectMapper();
jackson.setPropertyNamingStrategy(PropertyNamingStrategy.CAMEL_CASE_TO_LOWER_CASE_WITH_UNDERSCORES);
```

## Setup

The one thing that we'll need to do is inject our `new GsonBuilder()` that we built above.

``` java
Ion.getDefault(activity).setGson(gson);
```

## Executing a request

ion makes it really nice and easy to just work with the basic string response from the API.

``` java
Future<String> indexRequest = Ion.with(getApplicationContext(), "@TODO")
	.asString()
	.setCallback(new FutureCallback<String>() {
		@Override
		public void onCompleted(Exception e, String result) {
		}
	});
```

Or as a JSONObject or JSONArray using `.asJsonObject()` or `.asJsonArray()`. Or as a POJO using Gson.  
*Note: At the time of writing, ion does not have built-in support for Jackson.*

``` java
Future<List<User>> getUsers = Ion.with(getApplicationContext(), "@TODO")
	.as(new TypeToken<List<User>>(){})
	.setCallback(new FutureCallback<List<User>>() {
		@Override
		public void onCompleted(Exception e, List<User> result) {
		}
	});
```

## Other features

#### Progress
ion has some really nice plugins for retrieving the progress of any request.
``` java
indexRequest
	.progressBar(yourProgressBarHere)
	.progressDialog(yourProgressDialogHere)
	.progress(new ProgressCallback() {
		@Override
		public void onProgress(int downloaded, int total) {
		}
	});
```	
	
#### Image loading
``` java
Ion.with(yourImageViewHere)
	.placeHolder(R.drawable.yourPlaceholderGraphicHere)
	.error(R.drawable.yourErrorGraphicHere)
	.animateLoad(yourAnimationHere)
	.animateIn(yourAnimationHere)
	.centerInside()
	.resize(100, 200)
	.transform(MyTransformation)
	.load("http://example.com/image.png");
```	
The `MyTransformation` must be an implementation of [`Transform`](https://github.com/koush/ion/blob/master/ion/src/com/koushikdutta/ion/bitmap/Transform.java)

#### Cancelling requests
Requests can be canncelled individually.	
``` java
getUsers.cancel();
```	
Or per-activity. Doing this call in the `onStop()` of an Activity makes a lot of sense.
``` java
Ion.getDefault(activity).cancelAll(activity);
```
Or per-group.	
``` java
Object usersGroup = new Object();
Future<List<User>> getUsers = Ion.with(getApplicationContext(), "@TODO")
	.as(new TypeToken<List<User>>(){})
	.group(usersGroup)
	.setCallback(new FutureCallback<List<User>>() {
		@Override
		public void onCompleted(Exception e, List<User> result) {
		}
	});
Ion.getDefault(activity).cancelAll(usersGroup);
```

## Thoughts

The syntax for ion is certainly attractive. There is very little boilerplate code involved and there are some very attractive helpers that it gives. Of special note is the ability to "drop" between parsers so easily - in a perfect world, your API returns beautiful JSON that is consumed by Gson magically... in the real world, there are perfectly valid scenarios where API responses aren't JSON at all.

However, the fact that ion works in such a loose fashion may come to hurt a project after a long time. In our legacy code, the fact that AndroidAsync events are so "easy" to program (and ion events seem even easier) has lead to far too much thinking of functions as scripts and not objects - the proof of this is copy-pasted versions of AndroidAsync events throughout the sourcecode, instead of thinking of the API as an object the way that most of these other libraries do.