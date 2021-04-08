# URLBarSizing
The mobile browser URL bar is such a pain! It interacts with the page in a different way in every browser.
This page hopes to explain the nuances of how the URL bar affects the web page, document the differences
between the browsers, and explain some ~~proposed~~ recent changes to Chrome to improve the current situation.

I've also provided a [demo page](http://bokand.github.io/demo/urlbarsize.html) to demonstrate the
issues. This page can be used to compare the behavior between different browsers.

Note: These URL bar issues are inherent only to mobile platforms so all the above discussion implicitly
applies only to the mobile versions of each browser.

Caveat: ~~I have not had the chance to try out Edge browser on mobile so it's not included below.~~ Edge does not have a movable URL bar.

## Update and Summary

This change shipped in Chrome M56. I've preserved the rest of this explainer for reference but here's the distilled details of what developers should know:

  * The initial containing block (ICB) is the root containing block for the page. When you give the `<html>` element `width: 100%; height: 100%`, its height is calculated from the ICB. Prior to this change, when the user hides the URL bar, Chrome would resize the ICB to fit the new visible area. With this change, the ICB will not change height in response to the URL bar. It will remain fixed to the size it would be when the URL bar is showing.
  * `vh` units are used to size elements on the page with respect to the "viewport" height. `100vh` means use 100% of the viewport height. Prior to this change, Chrome would take "viewport" to mean ICB, which also meant these elements would get resized if the URL bar was hidden or shown. With this change, `100vh` will remain fixed to the height it would have been when the URL bar was hidden. This is different from ICB height to match Safari's behavior.
  * `position: fixed` elements will continue to get their height from the dynamic visual viewport. That is, hiding/showing the URL bar will resize `position: fixed` elements with percentage height.
  
That is, there are three viewports used for sizing elements:

The "vh viewport" is used to size `vh` units. It doesn't change height in response to the URL bar and behaves as if the URL bar is always hidden.

The "visual viewport" is used as the root container for `position: fixed` elements. It is resized in response to the URL bar. 

The "initial containing block" is used as the root container for all elements on the page other than `positoin: fixed`. It doesn't change height in response to the URL bar and behaves as if the URL bar is always showing.


### Known issues

 - Scrollable "overlays" can appear chopped-off at the bottom. This can occur when the page shows a fullscreen overlay while the URL bar is hidden. A common patter is to set `display:none` on the content while the overlay is up. This isn't seen in Safari since it's more aggressive about showing the URL bar. See this [demo page](http://bokand.github.io/overlay-bug.html) displaying the issue. [The solution](http://bokand.github.io/overlay-bug-fixed.html) is to make the overlay `position: fixed`.

## Differences between browsers

#### Resize Event

Browsers will fire a resize event on the window when it changes dimensions. On desktops, this would happen
when the user resizes the browser window. On mobile, this can happen when the URL bar is shown/hidden or 
when the user rotates their device.

So in terms of the URL bar hiding/showing, when is a resize event fired?

  + *Chrome* - When the user lifts their finger (final touchend). The URL bar will snap to either fully showing or
  fully hidden and a resize event will be dispatched.
  + *Safari* - When the controls reach their extent. That is, as soon as the controls are fully shown or fully
  hidden, a resize event will be dispatched even while the finger is down.
  + *Firefox* - When the user lifts their finger (final touchend). The URL bar will snap to either fully showing or
  fully hidden and a resize event will be dispatched.
  + *IE* - The URL bar doesn't move, simple enough.

#### Initial Containing Block size

Simply speaking, the *Initial Containing Block* (I'll use ICB for short) is the root block used for sizing
non-position:fixed elements on the page. That is, if an element specifies a percentage based size, it must use the
computed height of one of its ancestors. But what about the root element? It uses the ICB. According to the spec, the
ICB has the same dimensions as the *viewport*. However, this is a little vague as there are now a few notions of
"viewport".

So, how does the URL bar affect the ICB in each browser?

  + *Chrome* - ~~Chrome uses the viewable area that doesn't include the url bar, if it's showing, to size the ICB.
  This means that the ICB is made smaller when the URL bar is shown. It gets resized at the same time as the resize
  event is fired, when the user lifts their finger.~~ As of M56, Chrome uses a static ICB that doesn't change in
  response to the URL bar. The size used is the smallest possible,  i.e. the viewable area when the URL bar is fully shown.
  + *Safari* - Uses a static ICB that doesn't change in response to the URL bar. The size used is the smallest possible,
  i.e. the viewable area when the URL bar is fully shown.
  + *Firefox* - Firefox uses the viewable area that doesn't include the url bar, if it's showing, to size the ICB.
  This means that the ICB is made smaller when the URL bar is shown. It gets resized at the same time as the resize
  event is fired, when the user lifts their finger.
  + *IE* - Since the URL bar doesn't hide, it uses a static ICB which is the size of the viewable area.
  
#### position:fixed size

According to the spec, the containing block for position: fixed elements is the viewport. That is, the percentage based
sizes are based on the viewport's size. As discussed above, this can be somewhat ambiguous.

How do position:fixed elements get precentage-based sizes?

  + *Chrome* - Based on the viewable area not including top controls.
  + *Safari* - Based on the viewable area not including top controls.
  + *Firefox* - Same as the ICB area (smaller when top controls are visible; resizes when the resize event is fired).
  + *IE* - Based on the viewable area not including top controls.

This is the one bit of good news in this story. However, resizing elements on the page at 60fps is a bad idea, so when are
the sizes recalculated?

  + *Chrome* - At touchend (i.e. when the scroll ends).
  + *Safari* - At touchend (i.e. when the scroll ends).
  + *Firefox* - When the user lifts their finger, same time as resize event is fired.
  + *IE* - N/A
  
#### Viewport-units (vh/vw)

Viewport units allow the author to specify sizes in relation to the viewport size. e.g. If you want a box to be sized such
that it fills half the viewport height you'd specify height: 50vh;

How does each browser treat the viewport units with regard to the URL bar?

  + *Chrome* - ~~Uses the ICB to mean "viewport". This means that sizes based on vh units will change as top controls are
  shown/hidden.~~ As of M56, uses the largest possible viewable area (i.e. top controls hidden).
  + *Safari* - Uses the largest possible viewable area (i.e. top controls hidden).
  + *Firefox* - Viewport units are based on the ICB area (smaller when top controls are visible; resizes with the
  resize event).
  + *IE* - Top controls don't hide so it uses the static viewable area.

#### window.innerHeight and window.innerWidth

The innerWidth and innerHeight properties of the window object specify the size of the browser window, not including any
browser chrome (the UI concept, not the browser) like window borders and title bars. This *does* include scrollbars on
desktop browsers.

How does each browser treat the URL bar with regard to innerHeight?

  + *Chrome* - Treats the URL bar as browser chrome so that only the viewable area is counted in the innerHeight. That is,
  hiding the URL bar increases the size of innerHeight. Unlike the resize event and ICB resize, the innerHeight is
  updated in real-time as the URL bar is partially shown/hidden.
  + *Safari* - The same as Chrome.
  + *Firefox* - Similarly to the ICB, the window.innerHeight property is resized when the user lifts their finger.
  + *IE* - Since the URL bar doesn't hide window.innerHeight doesn't change. It does not include the top controls while
  outerWidth does.

## Changes in Chrome 56

Clearly this is an interop disaster. I propose we make some small changes to Chrome to help the situation:

1. Don't resize the ICB due to the URL bar. This is painful for users since the page get a relayout each time the user
changes the scroll direction. It's painful for developers because that's not how any of the other mobile browsers
work. For compatibilities sake, lets make the ICB sized statically to the *smallest possible viewable area*.
2. Make vh units relative to a static containing block. This means an author's 'vh' sized fonts wont change size each
time the user scrolls in a new direction. It might be conceptually nice to make this relative to the proposed-static
ICB (i.e. the smallest possible viewable area) but both Safari and Firefox already use the opposite. Web authors' lives are difficult enought; in the name of compatibility, I propose we do the same thing as Safari and Firefox, and make vh units relative to the *largest possible viewable area*.

Some notes:

\#1 means that, when the top controls are hidden, percentage-based heights will differ on elements based on whether they
are position:fixed or not. position:fixed elements with percentage based heights will resize when the top controls are hidden or shown.

\#2 doesn't apply to vh units set on a `position: fixed` element. They'll still resize in response to the URL bar.

It will be a little more work to make non-fixed elements fill the viewport after scrolling. I believe in most cases you'd want
this for position:fixed elements anyway. In the cases that remain, we can get back to todays behavior by explicitly sizing the
root element to window.innerHeight in the resize handler.

In general, the fixed-position viewport size can be read from window.innerHeight and the ICB size can be read from
documentElement.clientHeight.

## Demo

I've provided a [test page](http://bokand.github.io/demo/urlbarsize.html) that attempts to demonstrate
all the points talked about here. Try Chrome 56 (currently 
[Chrome Dev channel](https://play.google.com/store/apps/details?id=com.chrome.dev&hl=en) to see the changed behavior.
The four bars on the right of the page are all possible combinations of 99%, 99vh,
position:fixed and position:absolute provided on a scrollable page. Hiding the URL bar shows how it affects each. Resize events
are printed down the page.

