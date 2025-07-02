---
layout: post
title:  Exceptional JUnit4 - How Does A Test Fail Successfully?
header: header-eng
---

I've spent a lot of time writing JUnit4 tests, but never really spent time to understand how they work behind the
scenes. As a user, I know that if I add "junit:junit:4.13.2" and throw in a few `@Test fun`, then I'm up and running.
This works very similarly on Android as well, you just also need to add `@RunWith(AndroidJUnit4::class)` on the test
class.

A few days ago I ran into a case where a test contained the following:

```kotlin
for (i in 0 until frameTimes.size) {
    mainHandler.postAtTime(
        {
            assertThat(dispatchedEventsSinceLastFrame)
                .isEqualTo(expectedDispatchedEventsPerFrame[i])
            dispatchedEventsSinceLastFrame = 0
            allFramesPassedLatch.countDown()
        },
        frameTimes[i],
    )
}
```

where `mainHandler` is `Handler(Looper.getMainLooper())`.

We started seeing this test fail on the `assertThat().isEqualTo()`, but not only it was failing this test method, but
it was also crashing the whole test process, and thus never running all the tests that followed. From my experience,
I knew that adding an assertion outside the instrumented thread can cause failure. Knowing that, the fix was obvious -
propagate the failure back to the instrumentation thread. However, my curiosity got me and I decided to figure out why
this is needed and how does it work in a normal case.

First thing to know is that JUnit4 uses exceptions to indicate test failures. If you look at `Assert.assertTrue` is
effectively a call to `throw AssertionError(message)` and you can validate that by writing a test like:

```kotlin
@Test
fun test1() {
    if (!myCondition) throw AssertionError("omg")
}
```

and this will have the same results as `assertTrue(myCondition, "omg")`.

In a JVM application if you just threw an exception the whole JVM process would exit. This does not happen in JUnit4
tests. `BlockJUnit4ClassRunner.methodBlock` is where the magic try/catch happens.

```java
    protected Statement methodBlock(final FrameworkMethod method) {
        Object test;
        try {
            test = new ReflectiveCallable() {
                @Override
                protected Object runReflectiveCall() throws Throwable {
                    return createTest(method);
                }
            }.run();
        } catch (Throwable e) {
            return new Fail(e);
        }

        Statement statement = methodInvoker(method, test);
        statement = possiblyExpectingExceptions(method, test, statement);
        statement = withPotentialTimeout(method, test, statement);
        statement = withBefores(method, test, statement);
        statement = withAfters(method, test, statement);
        statement = withRules(method, test, statement);
        statement = withInterruptIsolation(statement);
        return statement;
    }
```

This allows the runner to catch any exceptions can carry on running the rest of the tests.

Going back to my failure earlier, we had a piece of test code, that was effectively throwing an exception on the main
Android thread, and because the test runner only try/catches exceptions on the instrumentation thread, that causes the
Android application to crash. When the app crashes, it takes down the instrumentation process along with it!

Luckily for you, `androidx.test:runner` already ships `UiThreadStatement.runOnUiThread` that allows you to safely assert
as it does a similar exception forwarding via 

```java
  public static void runOnUiThread(final Runnable runnable) throws Throwable {
    if (Looper.myLooper() == Looper.getMainLooper()) {
      Log.w(
          TAG,
          "Already on the UI thread, this method should not be called from the "
              + "main application thread");
      runnable.run();
    } else {
      FutureTask<Void> task = new FutureTask<>(runnable, null);
      getInstrumentation().runOnMainSync(task);
      try {
        task.get();
      } catch (ExecutionException e) {
        // Expose the original exception
        throw e.getCause();
      }
    }
  }
```

There is also a `@UiThreadTest` annotation that allows to run the whole method on the UI thread.

Happy testing!
