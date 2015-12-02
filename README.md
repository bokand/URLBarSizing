# URLBarSizing
The mobile browser URL bar is such a pain! It interacts with the page in a different way in every browser.
This page hopes to explain the nuances of how the URL bar affects the web page, document the differences
between the browsers, and explain some proposed changes to Chrome to improve the current situation.

I've also provided a [demo page](http://bokand.github.io/demo/urlbarsize.html) and [custom Chrome build](https://github.com/bokand/URLBarSizing/blob/master/ChromePublic.apk) for Android to demonstrate the
proposal.

Note: These URL bar issues are inherent only to mobile platforms so all the above discussion implicitly
applies only to the mobile versions of each browser.

Caveat: I have not had the chance to try out Edge browser on mobile so it's not included below.

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
  + *Firefox* - Firefox does not send a resize event at all in response to the URL bar.
  + *IE* - The URL bar doesn't move, simple enough.

#### Initial Containing Block size

Simply speaking, the *Initial Containing Block* (I'll use ICB for short) is the root block used for sizing
non-position:fixed elements on the page. That is, if an element specifies a percentage based size, it must use the
computed height of one of its ancestors. But what about the root element? It uses the ICB. According to the spec, the
ICB has the same dimensions as the *viewport*. However, this is a little vague as there are now a few notions of
"viewport".

So, how does the URL bar affect the ICB in each browser?

  + *Chrome* - Chrome uses the viewable area that doesn't include the url bar, if it's showing, to size the ICB.
  This means that the ICB is made smaller when the URL bar is shown. It gets resized at the same time as the resize
  event is fired, when the user lifts their finger.
  + *Safari* - Uses a static ICB that doesn't change in response to the URL bar. The size used is the smallest possible,
  i.e. the viewable area when the URL bar is fully shown.
  + *Firefox* - Uses a static ICB that doesn't change in response to the URL bar. The size used is the largest possible,
  i.e. the viewable area when the URL bar is fully hidden.
  + *IE* - Since the URL bar doesn't hide, it uses a static ICB which is the size of the viewable area.
  
#### position:fixed size

According to the spec, the containing block for position: fixed elements is the viewport. That is, the percentage based
sizes are based on the viewport's size. As discussed above, this can be somewhat ambiguous.

How do position:fixed elements get precentage-based sizes?

  + *Chrome* - Based on the viewable area not including top controls.
  + *Safari* - Based on the viewable area not including top controls.
  + *Firefox* - Based on the viewable area not including top controls.
  + *IE* - Based on the viewable area not including top controls.

This is the one bit of good news in this story. However, resizing elements on the page at 60fps is a bad idea, so when are
the sizes recalculated?

  + *Chrome* - At touchend (i.e. when the scroll ends).
  + *Chrome* - At touchend (i.e. when the scroll ends).
  + *Firefox* - When top controls hit their extent (i.e. are maximally shown/hidden).
  + *IE* - N/A
  
#### Viewport-units (vh/vw)

Viewport units allow the author to specify sizes in relation to the viewport size. e.g. If you want a box to be sized such
that it fills half the viewport height you'd specify height: 50vh;

How does each browser treat the URL bar with regard to the URL bar?

  + *Chrome* - Uses the ICB to mean "viewport". This means that sizes based on vh units will change as top controls are
  shown/hidden.
  + *Safari* - Uses the largest possible viewable area (i.e. top controls hidden).
  + *Firefox* - Uses the largest possible viewable area (i.e. top controls hidden).
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
  + *Firefox* - Similarly to the ICB, the window.innerHeight property isn't affected by the URL bar. The size used is the
  largest possible viewable area (i.e. with the URL bar hidden).
  + *IE* - Since the URL bar doesn't hide window.innerHeight doesn't change. It does not include the top controls while
  outerWidth does.

## Proposed Changes to Chrome

Clearly this is an interop disaster. I propose we make some small changes to Chrome to help the situation:

1. Don't resize the ICB due to the URL bar. This is painful for users since the page get a relayout each time the user
changes the scroll direction. It's painful for developers because that's not how any of the other mobile browsers
work. For compatibilities sake, lets make the ICB sized statically to the *smallest possible viewable area*.
2. Make vh units relative to a static containing block. This means an author's 'vh' sized fonts wont change size each
time the user scrolls in a new direction. It might be conceptually nice to make this relative to the proposed-static
ICB (i.e. the smallest possible viewable area) but Safari already uses the opposite. Web authors' lives are difficult
enought; in the name of compatibility, I propose we do the same thing as Safari and make vh units relative to the
*largest possible viewable area*.

Some notes:

\#1 means that, when the top controls are showing, percentage-based heights will differ on elements based on whether they
are position:fixed or not. position:fixed elements with percentage based heights will resize when the top controls are hidden
or shown.

\#2 means that vh units will not resize as a result of top controls, regardless of what their position: property is.

It will be a little more work to make non-fixed elements fill the viewport after scrolling. I believe in most cases you'd want
this for position:fixed elements anyway. In the cases that remain, we can get back to todays behavior by explicitly sizing the
root element to window.innerHeight in the resize handler.

In general, the fixed-position viewport size can be read from window.innerHeight and the ICB size can be read from
documentElement.clientHeight.

## Demo

You can try out the proposed changes in a custom [build of Chromium](https://github.com/bokand/URLBarSizing/blob/master/ChromePublic.apk)
provided in this repo. I've provided a [test page](http://bokand.github.io/demo/urlbarsize.html) that attempts to demonstrate
all the points talked about here. The four bars on the right of the page are all possible combinations of 99%, 99vh,
position:fixed and position:absolute provided on a scrollable page. Hiding the URL bar shows how it affects each. Resize events
are printed down the page.

