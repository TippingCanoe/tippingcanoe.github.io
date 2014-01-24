---
layout: post
title: "Benchmarking Android Networking Libraries - Part 2: Installing Libraries"
date: 2014-01-24 12:22:20 -0600
comments: true
categories:
- android
- hack day
- experiments
author: Iain Connor
---

## Preamble

Where we [last left our heroes](http://upthecreek.tippingcanoe.com/blog/2014/01/24/benchmarking-android-networking-libraries/), we'd setup a [testbed](http://github.com/TippingCanoe/rest-api-testbed) using [Laravel 4](http://www.laravel.com) to compare the speeds Android networking libraries. This time around, we're going to look at creating an Android Application to test these libraries out.

## Creating the Application

As mentioned in [part one](http://upthecreek.tippingcanoe.com/blog/2014/01/24/benchmarking-android-networking-libraries/) of this discussion, using modern tools to build our Applications has become a big focus for us at [Tipping Canoe](http://tippingcanoe.com/). The new hotness of [Gradle](http://tools.android.com/tech-docs/new-build-system) and [Android Studio](http://developer.android.com/sdk/installing/studio.html) have replaced the old bustedness of Eclipse and ANT for us, so having libraries available in a format that gels with proper dependency management is a big win and, in our opinion, a litmus test for the quality of a library.

We'll use the Android Studio new project wizard to create our project.

![Android Studio new Application]({{ site.url }}/assets/new_project.png)

Once the project has booted, a majority of the work will be done in the `build.gradle` file.  
*Note: There are actually two `build.gradle` files, one in the outer project and one in the inner module for our Application. For the moment, the inner one is where we will be concentrating. However, at the time of writing a new update to Android Studio was pushed which introduced version `0.8` of the gradle plugin, which may have changed this.*

![Android Studio new Application]({{ site.url }}/assets/new_project_done.png)

The one improvement that I'd suggest making is changing the `.gitignore` file. If you're working in a team all using Android Studio, there are actually a few settings in projects that you might want to share between your developers. Most `.gitignore` files will have you ignoring the entire `/.idea` folder, so you'll lose things like code style settings, which, in our mind, make sense to enforce and share amongst your team.

{% gist 8605514 %}

### Installing Retrofit

``` groovy
dependencies {
    compile 'com.android.support:appcompat-v7:+'
	compile 'com.squareup.retrofit:retrofit:1.3.0'
}
```

### Installing Volley

[Volley](http://developers.google.com/events/io/sessions/325304728) isn't available in Maven at the time of writing. In fact, finding a download for it at all is a little tricky. For shame, Google. Thankfully [Vince Mi](https://github.com/vinc3m1) has done us a great service in wrapping Volley in Maven repository he calls [Ark](https://github.com/ark/ark).

So, we just need to add Ark.

``` groovy
repositories {
	mavenCentral()
	maven {
		url 'https://raw.github.com/ark/ark/master/releases/'
	}
	maven {
		url 'https://raw.github.com/ark/ark/master/snapshots/'
	}
}
```

And then add Volley.

``` groovy
dependencies {
    compile 'com.android.support:appcompat-v7:+'
	compile 'com.android.frameworks:volley:master-SNAPSHOT'
}
```

### Installing ion

``` groovy
dependencies {
    compile 'com.android.support:appcompat-v7:+'
	compile 'com.koushikdutta.ion:ion:1.2.1'
}
```

### Installing Robospice

``` groovy
dependencies {
    compile 'com.android.support:appcompat-v7:+'
	compile 'com.octo.android.robospice:robospice:1.4.2'
}
```

### Installing Picasso

``` groovy
dependencies {
    compile 'com.android.support:appcompat-v7:+'
	compile 'com.squareup.picasso:picasso:2.1.1'
}
```

## Thoughts

Two things came up as a result of this,

First, it's interesting to note that the only library that fails at being dead simple to install and manage is the "offical" one provided by Google. As tempting as it is to assume that Google's solutions are always going to be the best, there's somewhat of a track record of projects being abandonded without warning to developers using or relying on them. The optimist would say that Google knows Gradle/Maven are going to be replaced by some secret project in the near future. The pessimest would say that this is a sign that Volley might not be a long-term supported project.

Second, this may be an extremely temporal thing that only exists at the time of writing, but Robo Spice generated an error while compiling;

``` bash
./gradlew build

...
UNEXPECTED TOP-LEVEL EXCEPTION:
com.android.dex.DexException: Multiple dex files define Landroid/support/v4/accessibilityservice/
...
```

What the heck? From the looks of that error message, something in Robo Spice is trying to include something that's conflicting with our version of the Android support library.

``` bash
./gradlew -q dependencies networkLibraryTest:dependencies

...
_debugApk
+--- com.android.support:appcompat-v7:+ -> 19.0.1
|    \--- com.android.support:support-v4:19.0.1
+--- com.squareup.retrofit:retrofit:1.3.0
|    \--- com.google.code.gson:gson:2.2.4
+--- com.android.frameworks:volley:master-SNAPSHOT
+--- com.koushikdutta.ion:ion:1.2.1
|    +--- com.google.code.gson:gson:2.2.4
|    \--- com.koushikdutta.async:androidasync:1.2.1
+--- com.octo.android.robospice:robospice:1.4.2
|    \--- com.octo.android.robospice:robospice-cache:1.4.1
|         +--- org.apache.commons:commons-lang3:3.1
|         +--- org.apache.commons:commons-io:1.3.2
|         |    \--- commons-io:commons-io:1.3.2
|         \--- com.google.android:support-v4:r7
\--- com.squareup.picasso:picasso:2.1.1
...
```

The solution was to top it from attempting to compile that module.

``` groovy
compile ('com.octo.android.robospice:robospice:1.4.2') {
	exclude module: 'support-v4'
}
```

## Scorecard

| Library   | Score | Notes                                     | Running Tally |
|-----------|------:|-------------------------------------------|--------------:|
| Retrofit  |    +1 |                                           |             1 |
| Volley    |    -1 | Not available in modern dependency tools. |            -1 |
| ion       |    +1 |                                           |             1 |
| Robospice |     0 |                                           |             0 |
| Picasso   |    +1 |                                           |             1 |

## Repository

I've committed the source code for this example up to [Github](https://github.com/TippingCanoe/network-library-test/releases/tag/step-2).

### Next steps

Next, we're going to look at performing basic REST requests in the different frameworks.