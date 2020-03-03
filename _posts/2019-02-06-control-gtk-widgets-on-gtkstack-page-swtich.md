---
layout: post
title:  "Controlling GTK Widgets on GtkStack Page Switch"
date:   2019-02-06
categories: en
---

**TL;DR: Connect callbacks to the signal `GtkWidget::map` of the stack children.**

In some cases, you may have a `GtkStack` without a `GtkStackSwitcher` to easily create several "interfaces" in a region, but as pages switch other actions are required to perform in order to cooperate with the stack switching. For example, [_PhoneBook_][pb], a course project of my group on networking in the past semester, has several interfaces switched according to user actions like button press and searching result reaching.

[pb]: https://github.com/DRJ31/COMP3003-Project

![PhoneBook Main Page](/assets/pb-main.png)

_Figure 1. Main Page of The PhoneBook Application_

![PhoneBook About Page](/assets/pb-about.png)

_Figure 2. About Page of The PhoneBook Application_

As you can see, in Figure 1 we have an about button on the right of the header bar (on the left of "Login" button), but in Figure 2 we have a "go back" button on the left instead, without any button on the right. ~~We~~ I designed these two button as:

- Left button, acting as a "back" button when in need.
- Right button, acting as either an "about" button, an "edit" button, or a "submit" button.

You can see the full logic [here in the code](https://github.com/DRJ31/COMP3003-Project/blob/master/src/gui.c#L29-L39).

Now, what we want to do is to let GTK calls our callback to change the states of these buttons once the stack is switched. However, `GtkStack` does not provide such signal for us to connect to (it does not even emit signals by itself)[^1].

As almost all GTK widgets are sub-classes of `GtkWidget`, I looked through all signals emitted by `GtkWidget` (it is a quite long list), and tried to find a signal that is emitted once the widget **shows up** on the screen. Finally, I found the `::map` signal and the `::map-event` signal of `GtkWidget`:

```c
/* Prototype of callback of ::map, Run First */
void
user_function (GtkWidget *widget,
               gpointer   user_data)

/* Prototype of callback of ::map-event, Run Last */
gboolean
user_function (GtkWidget *widget,
               GdkEvent  *event,
               gpointer   user_data)
```

Signal `::map` is emitted when a widget **is going to be drawn**[^2], while signal `::map-event` is emitted when a widget **has been drown** (`::map` has occurred). In addition, to receive `::map-event`, a proper GDK mask is required to be set.[^3] Since we only want to associate buttons to stack page switches, these two signals make no difference to me. I chose to connect my callback to the `::map` signal to **children of `GtkStack`**. The following is a piece of sample code snippet showing how to write the callback function:

```c
/**
 * The following callback function requires:
 *
 * - GtkBuilder to be used as the constructing method of UI;
 * - the "swap" attribute be set in Glade (i.e. GtkBuilder UI definition file);
 * - the default user data of callbacks be the GtkBuilder;
 */
void stack_switch_cb(GtkBuilder *builder)
{
  /* Get widgets we want to operate on from GtkBuilder */
  GtkWidget *btn_l = GTK_WIDGET(gtk_builder_get_object(builder, "btn_left"));
  GtkWidget *btn_r = GTK_WIDGET(gtk_builder_get_object(builder, "btn_right"));

  /* Get the name of the stack page to switch to */
  const char *stack_page_name = gtk_stack_get_visible_child_name(
    GTK_STACK(gtk_builder_get_object(builder, "stack"))
  );

  /* Determine what to do according to the name of stack page (must be set in prior) */
  if (g_strcmp0(stack_page_name, "main") == 0) {
    /* ... */
  } else {
    /* ... */
  }
}
```

(See [my last blog post](2019-01-22-control-all-widgets-in-gtk-callbacks-with-gtkbuilder.md) for my reasons to do the above)

In this way, when a stack page is going to be shown, the function is called and actions are performed with the switch of `GtkStack`. Either the page is switched with `GtkStackSwitcher`, by user actions, or by other means, these actions are guaranteed to be performed, and thus widgets can be associated with the show of corresponding stack pages!

## Notes

[^1]: [https://developer.gnome.org/gtk3/stable/GtkStack.html](https://developer.gnome.org/gtk3/stable/GtkStack.html)
[^2]: [https://developer.gnome.org/gtk3/stable/GtkWidget.html#GtkWidget-map](https://developer.gnome.org/gtk3/stable/GtkWidget.html#GtkWidget-map)
[^3]: [https://developer.gnome.org/gtk3/stable/GtkWidget.html#GtkWidget-map-event](https://developer.gnome.org/gtk3/stable/GtkWidget.html#GtkWidget-map-event)
