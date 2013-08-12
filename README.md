walkabout.js
============

Walkabout.js is an automatic web application tester.  It randomly
simulates a user.

This code figures out what your application is paying attention to,
and does it.  It fills in fields with random values.  It finds the
events you listen for and fires those events.  It finds internal links
(like `<a href="#foo">`) and clicks them.  It's a little like a fuzz
tester for your app.

It has special support for jQuery, and uses source code rewriting to
detect what your application is listening for in the absence of
jQuery.  (jQuery leaves evidence on DOM nodes about what is being
listened for.)

You can use it like so (if you are using jQuery):

```javascript
// This makes something like $('#some-input').val() return random values:
jQuery.fn.val.patch();

// Now, fiddle around, do 100 random things:
Walkabout.runManyActions({
  times: 100
});
```

You can also give Walkabout hints about what's a valid input; this
lets it guess application-progressing values more often.

You can add `data-walkabout-disable="1"` to any element to suppress
activation of that element or any of its children.

You can use `data-walkabout-eventname="..."` to set attributes on the
event that is created, such as `data-walkabout-keyup="{which: 13}"`

You can use `data-walkabout-options="['a', 'b']"` to give the valid inputs
for a field.

You can use `data-walkabout-edit-value="type paste delete move"` to
have Walkabout do edit operations inside a `textarea` or `input
type=text`.  You give a space-separated list of the things to do: type
a key at the cursor (and fire keyup), paste at the cursor (and fire
paste), delete the selection or delete before or after the cursor, or
move the cursor around.


Bookmarklet
-----------

If you want to try walkabout.js on some random jQuery site, you can
use the [bookmarklet](http://ianb.github.com/walkabout.js).  This will
load Walkabout and also start a simple UI to start the runs.  In
addition to starting a run it'll also track any uncaught errors and
any calls to `console.warn()` or `console.error()`.


Non-jQuery Support
------------------

There's some support for working with code that doesn't use jQuery.
It *might* work with other frameworks (by catching their calls to
`addEventListener`), but is more intended to work with code that
doesn't use a framework.  (*Note:* if you know equivalent ways to
detect what other frameworks are listening for, patches to Walkabout
would be welcome, or issues with details and examples that can be
tested on.)

This technique uses code rewriting to capture calls to
`.addEventListener()` and `.value`.  The code can't be weirdly tricky,
it needs to actually use those literal names -- which is usually fine
in "normal" code, but might not be in fancy or optimized code.  E.g.,
`el["addEventListener"](...)` would not be found, nor would
`a="addEventListener";el[a](...)`

You can use `Walkabout.rewriteListeners(code)` to rewrite the code.

The code transformation looks like this:

```javascript
document.getElementById("el").addEventListener("click", function () {}, true);
var value = document.getElementById("textarea").value;

// Becomes:
Walkabout.addEventListener(document.getElementById("el"), "click", function () {}, true);
var value = Walkabout.value(document.getElementById("textarea"));
```

And to find actions you use `Walkabout.findActions(element)`.

See [Proxy Server](#proxy-server) for an option to use this.


Options
-------

By default Walkabout will only go to links on the current page (i.e.,
`href="#something"`).  This is because it will lose its context and
potentially not load on the next page.  But if you want Walkabout to
move to other pages you can use `Walkabout.options.anyLocalLinks =
true`

Or set it to something like `"/dir/"` which will only follow links
under `/dir/`.

If you use `Walkabout.options.loadPersistent = true` then it will save
the state in localStorage and continue where it left of when another
page is loaded.


<a name="proxy-server"></a>

Proxy Server
------------

An application could include `walkabout.js` on its own, and if it
doesn't use jQuery it could also run `Walkabout.rewriteListeners()` on
all its code.  But maybe you don't feel like doing that, maybe you
want to try it out without all that work.  Also, by default Walkabout
will try to stay on the current page, but in the proxy mode because it
knows Walkabout will load on each page, it will freely move about the
site.

There is a proxy server `node-proxy.js` that makes this easier.  As
you might guess, you have to install Node.js to use it.

The server expects to receive requests for the website you want to
actually access.  To do this you have to edit `/etc/hosts` to point
the request locally, e.g.:

```
127.0.0.1 site-to-test.com
```

Then when you access `http://site-to-test.com` it will connect
locally, and the proxy server in turn will forward the request to the
actual server.  Any HTML responses will have the Walkabout Javascript
added, and Javascript will be rewritten.

Note that it binds to port 80, and so you must run it as root.  This
isn't awesome, pull requests to drop root welcome.  A tool like
[authbind](http://en.wikipedia.org/wiki/Authbind) also could help.

When you are done you should definitely stop the server, and undo the
`/etc/hosts` entry.

Note many live sites seem to notice the proxy, though I don't know
how.  My IP got blocked for a week from news.ycombinator.com after
using it in my own testing.  Any ideas welcome.

You can control the proxy with environmental variables: `$PORT` for
the port to bind to (default 80), `$BIND` for the interface to bind to
(default 127.0.0.1; 0.0.0.0 means all interfaces), and `$PORT_ALIASES`
which looks like `domain1:8088;domain2:8080` which lets you proxy from
one port to another (helpful sometimes when you need to proxy to a
server running locally).


License
-------

This is available under the
[Mozilla Public License](http://mozilla.org/MPL/2.0/) or the GPL.

To Do
-----

Lots of stuff, of course.  But:

- Sometimes the validation options (like `data-walkabout-options`)
  should be obeyed, and sometimes they should be ignored.  Obeying
  them progresses you through the site, disobeying them does some fuzz
  testing.

- Not all form controls get triggered, I think.  E.g., checkboxes
  don't get checked.

- `.val()` should figure out better what's a reasonable return value.
  E.g., a textarea returns multi-line strings, a checkbox returns
  true/false.

- Probably `.val()` hacking should just go away, and only do "real"
  editing of the fields.

- The generator could have smarts about what scripts (i.e., action
  sequences) are successful and which are not.  Starting with a
  successful script (that progresses you through the application), and
  then tweaking that script and increasing randomness at the end of
  the script.

- We could look at code coverage as a score.  Or we could even just
  look at event coverage - hidden elements often have events bound,
  but we know they have to be visible to be triggered.  We'd like to
  explore paths where they are visible.

- Something more Gaussian with the values generated.  Like, sometimes
  you should make a 100 character input.  But maybe not as often as a
  10 character input.  And sometimes a 1 character input.  The values
  should be biased towards "fencepost" values.  E.g., a bug that isn't
  triggered when you enter `43` is unlike to be triggered by `42`, but
  more likely with `0`, `-1`, or `1000000`.

- `data-walkabout-options` should be treated more as *additional*
  suggestions instead of *the only* suggestions.  They represent valid
  inputs that will keep the application going, but invalid inputs are
  still sometimes interesting.  Still the given options should be
  generally preferred when picking random options.

- Once a path has shown itself to get us to "new" code (via code
  coverage or that we see new action options) we might want to explore
  that specific code path further.  Maybe instead of a single random
  number seed, we could have a sequence of seeds: `[[30491, 10],
  50492]` meaning use seed 30491 for 10 numbers, then reseed,
  exploring a different path but with the same start as a
  known-successful path.

- A corollary of the above: we should have a signature for a set of
  actions, so we can quickly see if a set is unique.

- Configuration should be possible externally, not just on attributes
  on the DOM.  This would be fairly simple stuff, like:

      Walkabout.options.elements = {
        "#name": {
          suggest: "Bob"
        },
      };

- Some way of indicating a bug-detector.  Like, you know in some
  circumstances some element doesn't show when it should or something.
  These would be functions that would be run in-between actions.
  Simple assertions might work okay, but some tests are expensive or
  tricky.  But in cases when you don't get an "error" but have noticed
  that invalid things can happen (without an exception) a detector
  might help create a reproducable case.  This could also be a kind of
  "pause when" detector.

- QuickCheck has this notion of making a test as small as possible --
  basically seeing if you can get to the same error case with a
  smaller set of actions.  This could be pretty straight forward for
  Walkabout as well.  Though in cases like "do something, cancel
  something, do something again, do error thing" it might be tricky:
  we want to trim the first two actions, but trimming either one alone
  won't make the path possible.  So we have to look for chunks we can
  pull out.  With action signatures it might be easier to determine
  such chunks.

- With code coverage it would be interesting to see which scripts had
  "changed" by seeing if any of the code they touch "changed".  Then
  those scripts could be re-run.

- Making "screencaps" at various points would be interesting.  These
  would be a freeze of the visible DOM (BrowserMirror-style).  We
  could do them only when we've determined a path that is
  "interesting" (i.e., maybe shows an error), by rerunning from the
  start.

- Having a "reset app" hook would be good, so we can truly rerun.

- Combining signatures, we could try to determine if the app wasn't
  deterministic despite our efforts.

- We should patch `Math.random()`

- We could consider timeouts an actionable thing.  We could keep our
  own fake internal clock (mocking Date and setTimeout/setInterval),
  and moving it ahead in a predictable way.  We could simulate
  conditions like a computer going to sleep this way.

- Simulate `pagehide` and `pageview`.

- Simulate [WebAPIs](https://developer.mozilla.org/en-US/docs/WebAPI),
  at least the non-deterministic ones.  This involves mocking each one
  out (except event ones, which work kind of like other events).

- Maybe hook up to
  [WebDriver](http://www.w3.org/TR/2013/WD-webdriver-20130117/) so we
  can issue events that we aren't generally allowed to issue.
  `.dispatchEvent()` doesn't work with keyboard events, for instance.
  OTOH, we could also reimplement the event dispatch process.

- Put in touch events on mobile, and suppress hover events.
