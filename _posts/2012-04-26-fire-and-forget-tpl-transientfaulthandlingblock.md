---
layout: post
title: Fire and forget with retries using TPL and Transient Fault Handling Application Block
tags : [tpl, http, topaz]
---
{% include JB/setup %}

These are exciting days. We have HTTP APIs for almost every service our applications rely on: storage, db, security, logging, just to mention some. One of the things that I've found myself doing from time to time is some code to call an HTTP endpoint without worrying about its result. The typical scenario is logging/auditing: you have a web application and you want to log every call to a certain action. The challenges we have here are:

* How to do that without blocking the request?
* How do you make sure that the http call actually succeeded?

It turns out that using TPL and the the [Transient Fault Handling Application Block from patterns & practices](http://msdn.microsoft.com/en-us/library/hh680934.aspx) (wow, that's a big name) it's quite simple to do that. Here is a simple class and sample usage:

This will simply execute something and if that "something" fails, it will retry 10 times every one second. If after 10 seconds it fails it will simply ignore the exception.

    // simple fire and forget (it will retry 10 times every one second)
    FireAndForget.ExecuteWithRetry(() =>
    {
        // do something that takes a couple of seconds and you don't care abt the result
    });

Or you can pass some state and handle the error in case it failed after 10 times

    // same but passing state + handling the error if it failed after 10 times
    FireAndForget.ExecuteWithRetry((state) =>
    {
        var url = ((dynamic)state).RequestUrl;

    }, 
    state: new { RequestUrl = HttpContext.Request.Url }, 
    onErrorAfterRetries: (exception) =>
    {
        // log somewhere
    });

If the default retry policy doesn't make sense to you, you could use a different one (exponential, progressive, or even create your own).

    // redefine the default retry policy
    FireAndForget.DefaultRetryPolicy = RetryPolicy.DefaultExponential;
    FireAndForget.DefaultRetryPolicy = RetryPolicy.DefaultProgressive;
    FireAndForget.DefaultRetryPolicy = new RetryPolicy(...) // or create your own;

This is the small piece of code that will do the magic:

    public static class FireAndForget
    {
        public static RetryPolicy DefaultRetryPolicy = RetryPolicy.DefaultFixed;
    
        public static void ExecuteWithRetry(Action action)
        {
            ExecuteWithRetry(action, null, DefaultRetryPolicy);
        }
    
        public static void ExecuteWithRetry(Action action, RetryPolicy retryPolicy)
        {
            ExecuteWithRetry(action, null, retryPolicy);
        }
    
        public static void ExecuteWithRetry(Action<object> action, object state)
        {
            ExecuteWithRetry(action, state, DefaultRetryPolicy);
        }
    
        public static void ExecuteWithRetry(Action<object> action, object state, RetryPolicy retryPolicy)
        {
            ExecuteWithRetry(action, state, null, retryPolicy);
        }
    
        public static void ExecuteWithRetry(Action action, Action<Exception> onErrorAfterRetries)
        {
            ExecuteWithRetry(action, onErrorAfterRetries, DefaultRetryPolicy);
        }
    
        public static void ExecuteWithRetry(Action action, Action<Exception> onErrorAfterRetries, RetryPolicy retryPolicy)
        {
            Task.Factory.StartNew(() =>
            {
                try
                {
                    retryPolicy.ExecuteAction(() =>
                    {
                        action();
                    });
                }
                catch (Exception ex)
                {
                    if (onErrorAfterRetries != null)
                        onErrorAfterRetries(ex);
                }
            });
        }
    
        public static void ExecuteWithRetry(Action<object> action, object state, Action<Exception> onErrorAfterRetries)
        {
            ExecuteWithRetry(action, state, onErrorAfterRetries, DefaultRetryPolicy);
        }
    
        public static void ExecuteWithRetry(Action<object> action, object state, Action<Exception> onErrorAfterRetries, RetryPolicy retryPolicy)
        {
    
            Task.Factory.StartNew((taskState) =>
            {
                try
                {
                    retryPolicy.ExecuteAction(() =>
                    {
                        action(taskState);
                    });
                }
                catch (Exception ex)
                {
                    if (onErrorAfterRetries != null)
                        onErrorAfterRetries(ex);
                }
            }, state);
        }
    }

I should make it a nuget, but I am too lazy :S. If you improve it, fork the gist: <https://gist.github.com/2490815>




