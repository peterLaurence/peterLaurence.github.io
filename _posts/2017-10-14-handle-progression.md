---
title:  "Handle progression on configuration change"
date:   2017-10-14
categories: android design-pattern
---

# Handle progression on configuration change

A very common issue for Android developpers is to display a progression with a `ProgressBar` within an Activity or Fragment, and make it work properly on configuration change (for example screen orientation change).
Why so ?
Well, if things aren't done properly, when the screen is rotated, you might end up with the following issues :


* The progression stops
* The Activity/Fragment hangs while the task is running
* You might not even notice that behind the scene you are leaking an Activity


## A short explanation

By default, on configuration change,  an Android application has its main activity recreated. This is also the case for Fragments that are not retained. <br>
This has a lot of consequences because the developper has to keep in mind that every view can be destroyed at any time (e.g. the user rotates the screen). 

_What? You know you can prevent the activity from being re-created..._

If you're thinking about this :
```
<activity android:name=".MyActivity"
          android:configChanges="orientation|keyboardHidden"
          android:label="@string/app_name">
```

[This](https://developer.android.com/guide/topics/resources/runtime-changes.html#HandlingTheChange) explains why it isn't recommended. <br>

So yes, the spirit is to follow Android official guidelines.

## An example of mistake

Say we want to show a progress from 0 to 100% using a `ProgressBar`. We could do something like this :

```
public class AsyncActivity extends Activity {

    ProgressBar mProgressBar;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_async);
        mProgressBar = (ProgressBar) findViewById(R.id.progress_id);

        AsyncTask task = new BackgroundTask(mProgressBar).execute();
    }
}
```

```
public class BackgroundTask extends AsyncTask<Void, Void, Integer> {
    private final WeakReference<ProgressBar> mProgressBarReference;

    public BackgroundTask(ProgressBar progressBar) {
        mProgressBarReference = new WeakReference<>(progressBar);
    }

    @Override
    protected Integer doInBackground(Void... params) {
        // Do background work. Code omitted.
	return progress;
    }

    @Override
    protected void onPostExecute(Integer progress) {
        ProgressBar view = mProgressBarReference.get();
	if (view != null) {
            view.setProgress(result);
	}
    }
}
```

Here, the use of a `WeakReference` is to avoid a strong reference on a `ProgressBar` 
which has a reference on the activity `Context`. That way, the `BackgroundTask` won't
prevent the activty from being garbage collected.<br>
But if the user rotates the screen in the middle of the progression, the activity will
be destroyed and re-created -- so as the `ProgressBar`. Consequently, in the
`BackgroundTask`, `mProgressBarReference.get()` returns `null`, and our progression
stops.


Now if we want the `ProgressBar` to continue, there are other ways to achieve this, with
a background service and a broadcast receiver for example. But what if we want to keep things
simple, and not rely on services which are very resource demanding compared to bare `Thread`...<br>
So, assuming the activity has just been re-created, we could set the updated reference of the
`ProgressBar` to the `BackgroundTask`. But we can't have a reference on the `BackgroundTask`
since it was only known from the previous activity. Well actually we save this reference in a
singleton for example. But again no hacky solution is allowed.



## A solution

This post is about a solution to this common issue. There are other proper ways to tackle 
this.

**The idea**

Instead of passing a reference of ui object to the `BackgroundTask`, the graphical elements
will listen to messages coming from the running thread. So it doesn't matter whether the activity
or fragment is re-created or not, as long as they listen to the relevent messages. <br>
No need to reinvent the weel, this is the publish/subscribe design pattern and a very good library
available for Android is [greenrobot's EventBus](http://greenrobot.org/eventbus/).

{:refdef: style="text-align: center;"}
![Simple publish]({{ "/assets/simple_publish.jpg"}})
{: refdef}

For the rest of this post the publisher will be the background thread, and consumer will be a fragment. <br>
Now let's use more real world class example.

**Background thread**

```
public class DownloadTask extends Thread {
    private String mUrlToDownload;
    private File mOutputFile;
    private UrlDownloadListener mUrlDownloadListener;

    public DownloadTask(String urlToDownload, File outputFile, UrlDownloadListener listener) {
        mUrlToDownload = urlToDownload;
        mOutputFile = outputFile;
        mUrlDownloadListener = listener;
    }

    @Override
    public void run() {
        int percentProgress = 0;
        try {
            URL url = new URL(mUrlToDownload);
            URLConnection connection = url.openConnection();
            connection.connect();
            /* This will be useful so that you can show a typical 0-100% progress bar */
            int fileLength = connection.getContentLength();

            /* Download the file */
            InputStream input = new BufferedInputStream(connection.getInputStream());
            OutputStream output = new FileOutputStream(mOutputFile);

            byte data[] = new byte[1024];
            long total = 0;
            int count;
            while ((count = input.read(data)) != -1) {
                total += count;
                output.write(data, 0, count);

                /* Publishing the progress... */
                percentProgress = (int) (total * 100 / fileLength);
                mUrlDownloadListener.onDownloadProgress(percentProgress);
            }

            output.flush();
            output.close();
            input.close();
        } catch (IOException e) {
            e.printStackTrace();
        }

        if (percentProgress != 100) {
            //TODO : alert user, send an error signal to the receiver.
        }
    }

    public interface UrlDownloadListener {
        void onDownloadProgress(int percent);
    }
}
```

**The fragment**

```
import org.greenrobot.eventbus.EventBus;
import org.greenrobot.eventbus.Subscribe;
import org.greenrobot.eventbus.ThreadMode;

public class DownloadDialog extends DialogFragment {
    private ProgressBar mProgressBar;

    /* Constructor and component init ... */

    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onProgressEvent(UrlDownloadEvent event) {
        mProgressBar.setProgress(event.percentProgress);

        if (event.percentProgress == 100) {
            dismiss();
        }
    }

    @Override
    public void onStart() {
        super.onStart();
        EventBus.getDefault().register(this);
    }

    @Override
    public void onStop() {
        EventBus.getDefault().unregister(this);
        super.onStop();
    }
}
```

**Somewhere in a controler (parent fragment, activity...)**
```
DownloadTask downloadTask = new DownloadTask(url, outputFile, new DownloadTask.UrlDownloadListener() {
    @Override
    public void onDownloadProgress(int percent) {
        EventBus.getDefault().post(new UrlDownloadEvent(percent));
    }
});
downloadTask.start();
```

Now let me explain the code structure. The key points are :

* No `EventBus` dependency in `DownloadTask` -- use of listener pattern instead.
* The component that launches the `DownloadTask` creates a `DownloadTask.UrlDownloadListener` that uses
  the `EventBus`. <br>
  This allows the `DownloadTask` to ba agnostic of how progress flows into the application and improves
  reusability of this code.
* The `DownloadDialog` fragment subscribes to the `EventBus` in its `onStart` and unsubscribes in its
  `onStop` method.<br>
  This is very important to unsubscribe, to avoid memory leak.

And that's it. The screen can be rotated during the progress, the fragment's `ProgressBar` will continue
to show the progression.





