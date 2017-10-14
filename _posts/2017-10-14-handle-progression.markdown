## Handle progression on configuration change

A very common issue for Android developpers is to display a progression with a `ProgressBar` within an Activity or Fragment, and make it work properly on configuration change (for example screen orientation change).
Why so ?
Well, if things aren't done properly, when the screen is rotated, you might end up with the following issues :

<ul>
<li>The progression stops</li>
<li>The Activity/Fragment hangs while the task is running</li>
<li>You might not even notice that behind the scene you are leaking an Activity</li>
</ul>

## A short explanation

By default, on configuration change,  an Android application has its main activity recreated. This is also the case for Fragments that are not retained. <br>
This has a lot of consequences because the developper has to keep in mind that every view can be destroyed at any time (e.g. the user rotates the screen). 

"What? You don't prevent the activity to be re-created with :"
```
<activity android:name=".MyActivity"
          android:configChanges="orientation|keyboardHidden"
          android:label="@string/app_name">
```

[This](https://developer.android.com/guide/topics/resources/runtime-changes.html#HandlingTheChange) is why i don't do that. <br>
And throughout this article i am assuming that we follow official Android guidelines.

## A solution



