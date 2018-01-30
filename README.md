# inner-class-handler-memory-leak
inner-class-handler-memory-leak

Consider the following code:

    public class SampleActivity extends Activity {

      private final Handler mLeakyHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
          // ... 
        }
      };
    }
While not readily obvious, this code can cause cause a massive memory leak. Android Lint will give the following warning:

In Android, Handler classes should be static or leaks might occur.

But where exactly is the leak and how might it happen? Let’s determine the source of the problem by first documenting what we know:

When an Android application first starts, the framework creates a Looper object for the application’s main thread. A Looper implements a simple message queue, processing Message objects in a loop one after another. All major application framework events (such as Activity lifecycle method calls, button clicks, etc.) are contained inside Message objects, which are added to the Looper’s message queue and are processed one-by-one. The main thread’s Looper exists throughout the application’s lifecycle.

When a Handler is instantiated on the main thread, it is associated with the Looper’s message queue. Messages posted to the message queue will hold a reference to the Handler so that the framework can call Handler#handleMessage(Message) when the Looper eventually processes the message.

In Java, non-static inner and anonymous classes hold an implicit reference to their outer class. Static inner classes, on the other hand, do not.

So where exactly is the memory leak? It’s very subtle, but consider the following code as an example:

    public class SampleActivity extends Activity {

      private final Handler mLeakyHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
          // ...
        }
      };

      @Override
      protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // Post a message and delay its execution for 10 minutes.
        mLeakyHandler.postDelayed(new Runnable() {
          @Override
          public void run() { /* ... */ }
        }, 1000 * 60 * 10);

        // Go back to the previous Activity.
        finish();
      }
    }
When the activity is finished, the delayed message will continue to live in the main thread’s message queue for 10 minutes before it is processed. The message holds a reference to the activity’s Handler, and the Handler holds an implicit reference to its outer class (the SampleActivity, in this case). This reference will persist until the message is processed, thus preventing the activity context from being garbage collected and leaking all of the application’s resources. Note that the same is true with the anonymous Runnable class on line 15. Non-static instances of anonymous classes hold an implicit reference to their outer class, so the context will be leaked.

To fix the problem, subclass the Handler in a new file or use a static inner class instead. Static inner classes do not hold an implicit reference to their outer class, so the activity will not be leaked. If you need to invoke the outer activity’s methods from within the Handler, have the Handler hold a WeakReference to the activity so you don’t accidentally leak a context. To fix the memory leak that occurs when we instantiate the anonymous Runnable class, we make the variable a static field of the class (since static instances of anonymous classes do not hold an implicit reference to their outer class):

    public class SampleActivity extends Activity {

      /**
       * Instances of static inner classes do not hold an implicit
       * reference to their outer class.
       */
      private static class MyHandler extends Handler {
        private final WeakReference<SampleActivity> mActivity;

        public MyHandler(SampleActivity activity) {
          mActivity = new WeakReference<SampleActivity>(activity);
        }

        @Override
        public void handleMessage(Message msg) {
          SampleActivity activity = mActivity.get();
          if (activity != null) {
            // ...
          }
        }
      }

      private final MyHandler mHandler = new MyHandler(this);

      /**
       * Instances of anonymous classes do not hold an implicit
       * reference to their outer class when they are "static".
       */
      private static final Runnable sRunnable = new Runnable() {
          @Override
          public void run() { /* ... */ }
      };

      @Override
      protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // Post a message and delay its execution for 10 minutes.
        mHandler.postDelayed(sRunnable, 1000 * 60 * 10);

        // Go back to the previous Activity.
        finish();
      }
    }
The difference between static and non-static inner classes is subtle, but is something every Android developer should understand. What’s the bottom line? Avoid using non-static inner classes in an activity if instances of the inner class could outlive the activity’s lifecycle. Instead, prefer static inner classes and hold a weak reference to the activity inside.
