# Linux GTK Theme Integration

The GTK+ port of Chromium has a mode where we try to match the user's GTK theme
(which can be enabled under Wrench -> Options -> Personal Stuff -> Set to GTK+
theme). The heuristics often don't pick good colors due to a lack of information
in the GTK themes.

Starting in Chrome 9, we're providing a new way for theme authors to control our
GTK+ theming mode. I am not sure of the earliest build these showed up in, but I
know 9.0.597 works.

## Describing the previous heuristics

The frame heuristics were simple. Query the `bg[SELECTED]` and `bg[INSENSITIVE]`
colors on the `MetaFrames` class and darken them slightly. This usually worked
OK until the rise of themes that try to make a unified titlebar/menubar look. At
roughly that time, it seems that people stopped specifying color information for
the `MetaFrames` class and this has lead to the very orange chrome frame on
Maverick.

`MetaFrames` is (was?) a class that was used to communicate frame color data to
the window manager around the Hardy days. (It's still defined in most of
[XFCE's themes](http://packages.ubuntu.com/maverick/gtk2-engines-xfce)). In
chrome's implementation, `MetaFrames` derives from `GtkWindow`.

If you are happy with the defaults that chrome has picked, no action is
necessary on the part of the theme author.

## Introducing `ChromeGtkFrame`

For cases where you want control of the colors chrome uses, Chrome gives you a
number of style properties for injecting colors and other information about how
to draw the frame. For example, here's the proposed modifications to Ubuntu's
Ambiance:

```
style "chrome-gtk-frame"
{
    ChromeGtkFrame::frame-color = @fg_color
    ChromeGtkFrame::inactive-frame-color = lighter(@fg_color)

    ChromeGtkFrame::frame-gradient-size = 16
    ChromeGtkFrame::frame-gradient-color = "#5c5b56"

    ChromeGtkFrame::scrollbar-trough-color = @bg_color
    ChromeGtkFrame::scrollbar-slider-prelight-color = "#F8F6F2"
    ChromeGtkFrame::scrollbar-slider-normal-color = "#E7E0D3"
}

class "ChromeGtkFrame" style "chrome-gtk-frame"
```

### Frame color properties

These are the frame's main solid color.

| **Property** | **Type** | **Description** | **If unspecified** |
|:-------------|:---------|:----------------|:-------------------|
| `frame-color` | `GdkColor` | The main color of active chrome windows. | Darkens `MetaFrame::bg[SELECTED]` |
| `inactive-frame-color` | `GdkColor` | The main color of inactive chrome windows. | Darkens `MetaFrame::bg[INSENSITIVE]` |
| `incognito-frame-color` | `GdkColor` | The main color of active incognito windows. | Tints `frame-color` by the default incognito tint |
| `incognito-inactive-frame-color` | `GdkColor` | The main color of inactive incognito windows. | Tints `inactive-frame-color` by the default incognito tint |

### Frame gradient properties

Chrome's frame (along with many normal window manager themes) have a slight
gradient at the top, before filling the rest of the frame background image with
a solid color. For example, the top `frame-gradient-size` pixels would be a
gradient starting from `frame-gradient-color` at the top to `frame-color` at the
bottom, with the rest of the frame being filled with `frame-color`.

| **Property** | **Type** | **Description** | **If unspecified** |
|:-------------|:---------|:----------------|:-------------------|
| `frame-gradient-size` | Integers 0 through 128 | How large the gradient should be. Set to zero to disable drawing a gradient | Defaults to 16 pixels tall |
| `frame-gradient-color` | `GdkColor` | Top color of the gradient | Lightens `frame-color` |
| `inactive-frame-gradient-color` | `GdkColor` | Top color of the inactive gradient | Lightents `inactive-frame-color` |
| `incognito-frame-gradient-color` | `GdkColor` | Top color of the incognito gradient | Lightens `incognito-frame-color` |
| `incognito-inactive-frame-gradient-color` | `GdkColor` | Top color of the incognito inactive gradient. | Lightens `incognito-inactive-frame-color` |

### Scrollbar control

Because widget rendering is done in a separate, sandboxed process that doesn't
have access to the X server or the filesystem, there's no current way to do
GTK+ widget rendering. We instead pass WebKit a few colors and let it draw a
default scrollbar. We have a very
[complex fallback](http://git.chromium.org/gitweb/?p=chromium.git;a=blob;f=chrome/browser/gtk/gtk_theme_provider.cc;h=a57ab6b182b915192c84177f1a574914c44e2e71;hb=3f873177e192f5c6b66ae591b8b7205d8a707918#l424)
where we render the widget and then average colors if this information isn't
provided.

| **Property** | **Type** | **Description** |
|:-------------|:---------|:----------------|
| `scrollbar-slider-prelight-color` | `GdkColor` | Color of the slider on mouse hover. |
| `scrollbar-slider-normal-color` | `GdkColor` | Color of the slider otherwise |
| `scrollbar-trough-color` | `GdkColor` | Color of the scrollbar trough |

## Anticipated Q&A

### Will you patch themes upstream?

I am at the very least hoping we can get Radiance and Ambiance patches since we
make very poor frame decisions on those themes, and hopefully a few others.

### How about control over the min/max/close buttons?

I actually tried this locally. There's a sort of uncanny valley effect going on;
as the frame looks more native, it's more obvious that it isn't behaving like a
native frame. (Also my implementation added a startup time hit.)

### Why use style properties instead of (i.e.) bg[STATE]?

There's no way to distinguish between colors set on different classes. Using
style properties allows us to be backwards compatible and maintain the
heuristics since not everyone is going to modify their themes for chromium (and
the heuristics do a reasonable job).

### Why now?

*   I (erg@) was putting off major changes to the window frame stuff in
    anticipation of finally being able to use GTK+'s theme rendering for the
    window border with client side decorations, but client side decorations
    either isn't happening or isn't happening anytime soon, so there's no
    justification for pushing this task off into the future.
*   Chrome looks pretty bad under Ambiance on Maverick.

### Details about `MetaFrames` and `ChromeGtkFrame` relationship and history?

`MetaFrames` is a class that was used in metacity to communicate color
information to the window manager. During the Hardy Heron days, we slurped up
the data and used it as a key part of our heuristics. At least on my Lucid Lynx
machine, none of the GNOME GTK+ themes have `MetaFrames` styling. (As mentioned
above, several of the XFCE themes do, though.)

Internally to chrome, our `ChromeGtkFrame` class inherits from `MetaFrames`
(again, which inherits from `GtkWindow`) so any old themes that style the
`MetaFrames` class are backwards compatible.
