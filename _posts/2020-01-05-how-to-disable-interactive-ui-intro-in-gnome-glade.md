---
layout: post
title:  "How to Disable the Interactive UI Introduction in GNOME Glade"
date:   2020-01-05
---

**TL;DR Add the following lines in your `~/.config/glade.conf`:**

```ini
[Intro]
intro-done=true
```

---

[GNOME Glade][glade] is essential for developers using GTK, as writing GTK C API calls directly in their applications is painful. Glade provides an easy way for UI designers to design GTK GUIs interactively, by allowing drag and drop interactions like most of the GUI designers, e.g. Qt Designer, and Visual Studio Blend.

Glade 3.22 redesigned the interface to better conform [GNOME HIG][hig] (I guess). They designed an interactive UI introduction on Glade so that new users can follow the guides to quickly start using Glade. To be honest, the interactive design is not as good as I think: it automatically plays at some stages, but it requires user interaction at some other stages.

The worst thing is that until a user finishes the interactive introduction, the button for continuing the interactive introduction **blinks**, which consumes resources. As I install Glade at different time on different computers, I have been annoyed by having to finish this interactive introduction several times. I can't even find anyone mentioning about this on the Web... do you all tamely walk through the introduction?

Glade must remember whether the user have finished the interactive UI introduction somewhere so that the button to the interactive introduction never shows up. After an intense search in the source code repository of Glade, it turns out that Glade dynamically creates a configuration file (`glade.conf`) in the user configuration directory (this is typically `~/.config`). I compared the `glade.conf` before and after finishing the interactive introduction, and the difference turns out to be the section I wrote above in TL;DR.

I hope Glade developers can either improve the experience of the interactive UI introduction or provide a disable option in the preference dialog!

[glade]: https://glade.gnome.org/
[hig]: https://developer.gnome.org/hig/stable
