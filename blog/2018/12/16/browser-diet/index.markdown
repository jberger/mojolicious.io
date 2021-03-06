---
title: Day 16: A pre-Christmas Diet for Mojolicious - A Children's Story
disable_content_template: 1
tags:
    - advent
    - caching
    - headers
author: Boyd Duffee
images:
  banner:
    src: '/blog/2018/12/16/browser-diet/squirrel.jpg'
    alt: "Too many nuts. I'm gonna need to slim down"
    data:
      attribution: |-
        <a href="https://www.flickr.com/photos/55426027@N03/16915881989">Image</a> by <a href="https://www.flickr.com/photos/55426027@N03">Peter G Trimming</a> <a href="https://creativecommons.org/licenses/by/2.0"> CC BY 2.0 </a>
data:
  bio: duffee
  description: 'Setting the HTTP CacheControl header for your static files'
---

You've just read
[How to lose Weight in the Browser](https://browserdiet.com)
and you want to know to slim down your Mojo app.
Part of that process is preventing the browser from requesting files
that hardly change.
I spent a well-caffeinated afternoon trying to do that with
Mojolicious.
I've been 'round the houses, and _spoiler alert_ I didn't find
the answer until the very end, kind of like your favourite Christmas
animated special with a small woodland creature narrating
"The Gruffalo's HTTP header".

# A Children's Story

Our beloved small woodland creature needed to display a web calendar
with forest events pulled from a database.
Perl could get the event data and package it as a JSON feed.
Mojolicious could prepare the webpages with the correct JSON feed for each user.
With some JavaScript libraries to display the web calendar,
all would be well in the forest.

Everything except the JavaScript libraries are lightweight.
And everyone knows a page reload goes _so_ much faster if it doesn't have to download the
JavaScript every time.  Those libraries won't change for months!
If only the client browser knew that it could use the file that it had downloaded
last time.

The secret, of course, is to set the `Cache-Control` field of the HTTP header, but _how_?

---

## First, there was a [Horse](https://httpd.apache.org/) ...

Everybody using Apache would be thinking about using
[mod_expires](https://httpd.apache.org/docs/2.4/mod/mod_expires.html)
which looks quite easy, except that Apache wasn't being used to serve the webpages.

... but the Horse mentioned where there were some sweet
[Cache-Control directives](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control)
to munch on and while continuing to graze on some
[HTTP Caching](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching)
pages that had been downloaded earlier.  The small creature moves on.

## ... and _then_ there was a [Toad](https://perlmaven.com/deploying-a-mojolicious-application)

The forest creatures used the
[Hypnotoad](https://github.com/mojolicious/mojo/wiki/Hypnotoad-prefork-web-server)
web server that comes with Mojolicious to serve their pages.
They found it a good fit for their arboreal production environment.

<img class="align-center" src="Hypnotoad.gif" title="All glory to the Hypnotoad">

It can set the HTTP headers to turn it into a
[reverse proxy](https://mojolicious.org/perldoc/Mojolicious/Guides/Cookbook#Hypnotoad),
but a popular setup is sitting Hypnotoad behind
[Nginx](https://www.mind-it.info/2014/09/27/running-hypnotoad-behind-nginx/)
or Apache/mod_proxy.
Those servers should let you play with the `Expires` header.
But the Toad didn't _quite_ have what this particular rodent was looking for.

_An aside_ - No, I didn't mention
[Plack](https://metacpan.org/pod/Plack).
Maybe if I'm good this year, Santa will
[tell me how](http://blogs.perl.org/users/aristotle/2018/11/modern-perl-cgi.html)
I should be using it.  Probably something to do with

    Plack::Response->header('Expires' => 'Tue, 25 Dec 2018 07:28:00 GMT');

but I wouldn't know and neither did our narrator.

## ... and _then_ there was a [Unicorn](https://mojolicious.org)

Well, that was easy.  Just use the standard
[Mojo::Headers](https://mojolicious.org/perldoc/Mojo/Headers#expires)
module to set the `Expires` header.

But, wait!  That sets it for a page which isn't that big at all.
Our furry friend only wants to stop JavaScript files from reloading every single time
which were killing the Sciuridae mobile experience.  Hmmmm.

## ...  and _then_ there was a [Rhino](http://shop.oreilly.com/product/9780596805531.do)

<img class="pull-right" src="rhino.jpg" alt="JavaScript rhino image">

If the JavaScript lives in the `<head>` tag, then the page body won't be parsed
until the script is downloaded.  Some people get around this by putting the script just
before the end `</body>` tag, but the JS still has to download.
There are a couple of attributes that are hiding in the bushes.
`Async` and `defer` tell the browser to continue parsing the HTML while the
JavaScript is downloading.  These two
[wise](https://flaviocopes.com/javascript-async-defer/)
[owls](https://bitsofco.de/async-vs-defer/)
can help decide which one to use.

Tell your script to load after the main page
with this
[tag helper](https://mojolicious.org/perldoc/Mojolicious/Plugin/TagHelpers#javascript)
in your template

    %= javascript '/js/lib/jquery.min.js', defer => undef

which produces

    <script defer src="/path/to/script.js"></script>

If you want it in the `<head>`, you'll have to put it in your layout
otherwise the template will do.

## ... and finally, there was a [Man in a Hat](https://metacpan.org/author/LDIDRY)

<img class="pull-right" alt="Luc Didry gravatar" src="/static/ldidry.png">

"But the JavaScript needs to load **FIRST**!", squeaked the nut-botherer.

Sigh - this really is one demanding Sciurus vulgaris.

Well ... it _is_ Christmas, but you'll need to install the
[StaticCache plugin](https://metacpan.org/pod/Mojolicious::Plugin::StaticCache)
written for you only last year by
[Luc Didry](https://fiat-tux.fr/).
It sets the `Control-Cache` header for all static files served by Mojolicious.
With the **nut.js** and **nut.css** files in the `public` directory
(properly [minified](https://www.minifier.org/) of course),
they should only be downloaded once and use the cached version until it expires.
The default **max-age** is 30 days and
if you want you can even cache during development with `even_in_dev => 1`.

<img class="pull-right" src="speedtest_before_StaticCache.png">

The magpies in the forest had cluttered the calendar with 3 JavaScript libraries,
3 CSS files and 4 logos.  Sure, the biggest and shiniest was only 66 kB
and the whole collection was a paltry 164 kB, but bandwidth is precious in the wilderness.
Before using the StaticCache plugin, the calendar rated a
**92** on Google's PageSpeed Insights.

With the StaticCache plugin loaded

    sub startup {
        my $self = shift;

        $self->plugin('StaticCache' => { even_in_dev => 1 });
        ...
    }

<img class="pull-left" src="speedtest_with_StaticCache.png">

page speeds are now **93** **!!!!**
WOW!  It's <a href="https://xkcd.com/670">one faster!</a> said Nutgel Tufty-tail
and everyone in the forest cheered.

And _that_, dear readers, is how the squirrel learned to store nuts for the winter
... in a _cache_.  Good night, little kids, good niiiight.

## Try it out for yourself

The [Browser calories](https://github.com/zenorocha/browser-calories)
plugin for Firefox, Chrome and Opera breaks down the file sizes of your web page
into a nice little traffic light report measuring the HTML, images, CSS, JavaScript
and other parts of your page against
user-configurable limits on what you think is acceptable.

Google's [PageSpeed Insights](https://developers.google.com/speed/pagespeed/insights)
measures performance on both mobile and desktop.

Hopefully, the increasingly awkward attempt at writing in a narrative style
didn't get in the way of a new idea or two.  [Let me
know](https://github.com/duffee/Mojolicious_session_example) if I've missed
something.
