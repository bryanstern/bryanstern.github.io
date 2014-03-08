---
layout: post
title: Better ParseCallbacks 
---

I recently built an Android app using [Parse](http://www.parse.com) as my mBaaS (mobile backend as a service). It is a pretty coool service and while it does have it's issues, it certainly saves a lot of time when you are just worried about getting an MVP to market.

One common problem faced by Android developers is handling the callbacks from threads started from Activities and Fragments. Here is a little snippet I used to work around the issue I faced when ParseCallback would execute on an Activity or Fragment that has been garbage collected.

For example:

FindCallback.java

```java
package com.bryanstern.parsecallbacks;

import com.parse.ParseException;
import com.parse.ParseObject;

import java.lang.ref.WeakReference;
import java.util.List;

public class FindCallback <T extends ParseObject, U> extends com.parse.FindCallback {
    WeakReference<U> weakRef;

    public FindCallback(U object) {
        super();
        weakRef = new WeakReference<>(object);
    }

    @Override
    public void done(List list, ParseException e) {
        done(list, weakRef.get(), e);
    }

    // To be overridden
    public void done(List<T> list, U a, ParseException e) {

    }
}
```

Would be used like in your activity:

```java
ParseQuery<ParseUser> query = ParseUser.getQuery();
query.findInBackground(new FindCallback<ParseUser, Activity>(this) {
    @Override
    public void done(List<ParseUser> list, Activity a, ParseException e) {
        if (a != null) {
            if (e == null) {
                // display list of users
            } else {
                // handle error
            }
        }
    }
});
```

The Parse Android SDK does save a lof of work in most cases. However, if I were to tackle this project again, I would probably have used their REST API in combination with [Retrofit](http://square.github.io/retrofit/) or [Volley](https://android.googlesource.com/platform/frameworks/volley/).