---
layout: post
title:  "Asynchronous GUI Update in GTK"
date:   2019-01-23
categories: gtk
---

**TL;DR: [`gdk_threads_add_idle()`][gdk-threads-add-idle] / [`gdk_threads_add_timeout()`][gdk-threads-add-timeout]**

[gdk-threads-add-idle]: https://developer.gnome.org/gdk3/stable/gdk3-Threads.html#gdk-threads-add-idle
[gdk-threads-add-timeout]: https://developer.gnome.org/gdk3/stable/gdk3-Threads.html#gdk-threads-add-timeout

GTK+ is not thread safe[^1]. As is described in GDK 3 reference manual:

> GLib is completely thread safe... GTK+, however, is not thread safe. You should only use GTK+ and GDK from the thread `gtk_init()` and `gtk_main()` were called on. This is usually referred to as the "main thread".
>
> Signals on GTK+ and GDK types, as well as non-signal callbacks, are emitted in the main thread.

Consider the following case:

- We wait for an event from user (e.g. a button "clicked" by the user)
- We then have a task that may take long time to finish (e.g. network request, expensive computation)
- We want to do the above asynchronously (so we do not block the main UI thread, because signal callbacks are executed in the UI thread ("main thread"))
- Meanwhile, we want to show user a [pulsing progress bar][pulsing-progress-bar] because we have no idea how long the procedure will last

[pulsing-progress-bar]: https://developer.gnome.org/gtk3/stable/GtkProgressBar.html#gtk-progress-bar-pulse

We need:

- A thread to perform the task itself ("task thread")
- A thread to update UI elements
- A communication method for the task thread to tell the UI thread about the result

As GTK+ is not thread safe, we cannot update UI elements on a thread other than the "main thread", or race condition will occur, the program will crash. Thus, we need a way to put our UI update function in the "main thread".

To achieve this, use:

```c
gdk_threads_add_idle_full(ui_update_func, user_data);
```

... to spawn the UI updating thread, where

- `ui_update_func` is the function name of our UI update code
- `user_data` is the extra argument to be passed to `ui_update_func`

In this way, `ui_update_func` will be executed "whenever there are no higher priority events pending"[^2] (e.g. no signals are emitting), so in the UI update code we can (busy-)wait for the task thread to finish, receive the result, and display it on the UI.

Another similar way to achieve asynchronous UI update is:

```c
/* 1000 in millisecond, so here ui_update_func will be run every 1 second */
gdk_threads_add_timeout(1000, ui_update_func, user_data);
```

This allows us to manipulate widgets (e.g. call `gtk_progress_bar_pulse`) from time to time safely in the UI update function, because it is run in the "main thread" with a time interval. I think this is most useful for pulsing progress bars because this seems to be the only widget that requires manual advancing!

As you can see, these are utilities provided by GDK, not GTK nor GLib. There are, however, GLib "equivalents":

```c
guint
g_idle_add (GSourceFunc function,
            gpointer data);

guint
g_timeout_add (guint interval,
               GSourceFunc function,
               gpointer data);
```

Previously, I use these, and it works. However, as is described in the GDK 3 developer manual, using them is not recommended:

> You should use `gdk_threads_add_idle()` and `gdk_threads_add_timeout()` instead of `g_idle_add()` and `g_timeout_add()` since libraries not under your control might be using the deprecated GDK locking mechanism. If you are sure that none of the code in your application and libraries use the deprecated `gdk_threads_enter()` or `gdk_threads_leave()` methods, then you can safely use `g_idle_add()` and `g_timeout_add()`.

(It looks like my program is so modern, wow)

---

Another issue here is that we need a mechanism for the two threads to communicate with each other, so that:

- When the task is finished, the UI thread can receive the result and display it on the UI
- When the task fails, the UI thread can display an error message on time

The [Asynchronous Queue][async-queue] functionality provided by GLib seems to be the best solution. With asynchronous queue created and passed to the two threads described above, the task thread can pass result or error through the queue, and the UI thread can check the queue from time to time in order to display the result or error to user.

[async-queue]: https://developer.gnome.org/glib/stable/glib-Asynchronous-Queues.html

## Notes

[^1]: [https://developer.gnome.org/gdk3/stable/gdk3-Threads.html#gdk3-Threads.description](https://developer.gnome.org/gdk3/stable/gdk3-Threads.html#gdk3-Threads.description)
[^2]: [https://developer.gnome.org/gdk3/stable/gdk3-Threads.html#gdk-threads-add-idle-full](https://developer.gnome.org/gdk3/stable/gdk3-Threads.html#gdk-threads-add-idle-full)