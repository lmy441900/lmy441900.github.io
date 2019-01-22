---
layout: post
title:  "Control All Widgets in GTK Callbacks with GtkBuilder"
date:   2019-01-22
categories: gtk
---

**TL;DR: `gtk_builder_connect_signals(builder, builder)`**

`GtkBuilder` allows one to conveniently create graphical user interface (GUI) using an interface designer (i.e. Glade), which is convenient because we can draw GUI by dragging an dropping, without writing a loong piece of code in a GUI initialization function.

In a `GtkBuilder` UI definition file, widget signals can also be assigned callbacks in the designer so that one can conveniently connect them with a single function call (this is what previously I wrote):

```c
gtk_builder_connect_signals(builder, NULL);
```

Callbacks are declared by the caller, and thus forms of callbacks are fixed for implementors. In GTK (and almost all G\* stuff), one `gpointer user_data` parameter is reserved for extra argument passing. It is natural if we only pass at most one object to the callbacks. If we want to pass more to a callback, we may want to do the following:

1. Pack all things into one data structure. In C, `struct` is certainly used.
2. Dynamically allocate a piece of memory somewhere in code.
3. Put (pointers of) all things to pass in that memory.
4. Tell `GObject` to pass the pointer to that piece of memory to the callback.

```c
struct _PrivateFoo {
  GtkWidget *btn_left;
  GtkWidget *btn_right;
  GtkWidget *entry;
};

void btn_xxx_clicked_cb(GtkWidget *self, gpointer data);

/* ... */

struct _PrivateFoo *data = malloc(sizeof(struct _PrivateFoo));

data->btn_left  = btn_foo;
data->btn_right = btn_bar;
data->entry     = entry_baz;

/* Connect "clicked" signal of a button to the callback declared before */
g_signal_connect(G_OBJECT(btn_xxx), "clicked", G_CALLBACK(btn_xxx_clicked_cb), data);
```

This must be written in code, and preferably during the initialization process of a program. If we have to do these for all callbacks that controls on multiple widgets[^1], we need to declare many private `struct`s and write a long piece of code in order to initialize the signals. This eliminates benefits that `GtkBuilder` brings us.

At some point, I noticed the prototype of `gtk_builder_connect_signals`[^2]:

```c
void
gtk_builder_connect_signals (GtkBuilder *builder,
                             gpointer user_data);
```

In which:

- `builder`: a `GtkBuilder`
- `user_data`: user data to pass back with all signals

After some search and experiments, I found out that this `user_data` is **what being passed to all callbacks by default**. If we pass `NULL` here, all callbacks without `user_data` declared in Glade will receive `NULL`. So, if we do:

```c
/* GtkBuilder used to connect signals are also passed to all callbacks */
gtk_builder_connect_signals(builder, builder);
```

Then all callbacks written in the UI definition file without a designated widget as `user_data` will receive a `GtkBuilder *`. We can, therefore, extract multiple widgets from the `GtkBuilder` in callbacks:

```c
void some_cb(GtkWidget *self, GtkBuilder *builder) {
  /* Widget IDs are declared in Glade */
  GtkWidget *btn_left  = GTK_WIDGET(gtk_builder_get_object(builder, "btn_foo"));
  GtkWidget *btn_right = GTK_WIDGET(gtk_builder_get_object(builder, "btn_bar"));
  GtkWidget *entry     = GTK_WIDGET(gtk_builder_get_object(builder, "entry_baz"));

  /* Operate on these widgets! */
}
```

This is a graceful way to pass multiple widgets to all callbacks, with the help of `GtkBuilder`.

## Notes

[^1]: Note that if non-widget data is also required by a callback, then we still need to do extra work for the callback to access the data.
[^2]: [https://developer.gnome.org/gtk4/stable/GtkBuilder.html#gtk-builder-connect-signals](https://developer.gnome.org/gtk4/stable/GtkBuilder.html#gtk-builder-connect-signals)