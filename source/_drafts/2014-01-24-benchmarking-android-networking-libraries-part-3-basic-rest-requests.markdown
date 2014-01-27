---
layout: post
title: "Benchmarking Android Networking Libraries - Part 3: Basic REST Requests"
date: 2014-01-24 15:51:20 -0600
comments: true
categories:
- android
- hack day
- experiments
author: Iain Connor
---

## Preamble

Last time, we managed to get all of our [contenders installed](http://upthecreek.tippingcanoe.com/blog/2014/01/24/benchmarking-android-networking-libraries-part-2-installing-libraries/). Now, its time to try sending a request to our [testbed](http://upthecreek.tippingcanoe.com/blog/2014/01/24/benchmarking-android-networking-libraries/).

## Create tester

First, we're going to create a basic view to support our testing.

*Note: Not going to lie, I spend 40 minutes debugging these next two steps. That's what happens when you work for 12 hours.*

We'll need permission to access the internet in our `AnroidManifest.xml`.

``` xml
<uses-permission android:name="android.permission.INTERNET" />
```

And let's not forget to boot our [testbed](http://upthecreek.tippingcanoe.com/blog/2014/01/24/benchmarking-android-networking-libraries/).

``` bash
php ./artisan serve
```

``` xml
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:paddingLeft="@dimen/activity_horizontal_margin"
    android:paddingRight="@dimen/activity_horizontal_margin"
    android:paddingTop="@dimen/activity_vertical_margin"
    android:paddingBottom="@dimen/activity_vertical_margin"
    tools:context="com.tippingcanoe.networklibrarytest.MainActivity$PlaceholderFragment">

	<Spinner
		android:layout_width="wrap_content"
		android:layout_height="wrap_content"
		android:id="@+id/chooseLibrary"
		android:layout_alignParentTop="true"
		android:layout_alignParentLeft="true"
		android:layout_alignParentStart="true"
		android:layout_alignParentRight="true"
		android:layout_alignParentEnd="true" />

	<Spinner
		android:layout_width="wrap_content"
		android:layout_height="wrap_content"
		android:id="@+id/chooseRequest"
		android:layout_below="@+id/chooseLibrary"
		android:layout_alignParentLeft="true"
		android:layout_alignParentStart="true"
		android:layout_alignParentRight="true" />

	<Button
		android:layout_width="wrap_content"
		android:layout_height="wrap_content"
		android:text="Go"
		android:id="@+id/runRequest"
		android:layout_below="@+id/iterationCount"
		android:layout_alignRight="@+id/iterationCount"
		android:layout_alignEnd="@+id/iterationCount"
		android:layout_alignParentLeft="true"
		android:layout_alignParentStart="true" />

	<SeekBar
		android:layout_width="wrap_content"
		android:layout_height="wrap_content"
		android:id="@+id/iterationCountChooser"
		android:layout_below="@+id/chooseRequest"
		android:layout_toRightOf="@+id/textView"
		android:layout_toLeftOf="@+id/iterationCount"
		android:max="100" />

	<TextView
		android:layout_width="wrap_content"
		android:layout_height="wrap_content"
		android:text="Iterations"
		android:id="@+id/textView"
		android:layout_below="@+id/chooseRequest"
		android:layout_alignParentLeft="true"
		android:layout_alignParentStart="true"
		android:layout_alignBottom="@+id/iterationCountChooser"
		android:gravity="center_vertical" />

	<TextView
		android:layout_width="wrap_content"
		android:layout_height="wrap_content"
		android:textAppearance="?android:attr/textAppearanceLarge"
		android:text="Large Text"
		android:id="@+id/iterationCount"
		android:layout_below="@+id/chooseRequest"
		android:layout_alignParentRight="true"
		android:layout_alignParentEnd="true"
		android:layout_alignBottom="@+id/iterationCountChooser" />

	<ScrollView
		android:layout_width="fill_parent"
		android:layout_height="wrap_content"
		android:id="@+id/scrollView"
		android:layout_below="@+id/runRequest"
		android:layout_alignParentLeft="true"
		android:layout_alignParentStart="true"
		android:layout_alignParentBottom="true" >

		<TextView
			android:layout_width="wrap_content"
			android:layout_height="wrap_content"
			android:text="New Text"
			android:id="@+id/resultsText" />
	</ScrollView>
</RelativeLayout>
```



## GET requests

### Retrofit

### Volley

### ion

### Robospice

## Thoughts

## Scorecard

Library   | Score | Notes                                     | Running Tally
----------|------:|-------------------------------------------|-------------:
Retrofit  |    +1 |                                           |             1
Volley    |    -1 | Not available in modern dependency tools. |            -1
ion       |    +1 |                                           |             1
Robospice |     0 | Generated build errors.                   |             0
Picasso   |    +1 |                                           |             1

## Repository

I've committed the source code for this example up to [Github](https://github.com/TippingCanoe/network-library-test/releases/tag/step-2).

## Next steps

Next, we're going to look at performing basic REST requests in the different frameworks.