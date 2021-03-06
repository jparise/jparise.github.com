---
layout: post
title: Password Composer for iPhone
category: ios
---

I often use [Password Composer][pwdcomposer] (written by Johannes la Poutré)
to generate unique, per-site passwords.  It does an excellent job because it's
simple, unobtrusive, and reliable.  The one downside is that you need to have
it available in order to (re)generate the password for a given web site, and
that isn't always convenient, despite the large number of existing Password
Composer implementations.

The main place I find myself missing Password Composer is on my iPhone.  The
only current solution that works from Mobile Safari is the [public web-based
interface][web], and I'm security-conscious enough to avoid using that version
whenever possible.  I decided the only reasonable solution would be to find a
way to run Password Composer locally on my iPhone.

I first considered writing a full-blown native iPhone application.  I know a
bit of Objective-C and have experimented with the iPhone SDK, but, additional
learning opportunities aside, this approach felt like overkill.

I then discovered the option of writing an iPhone-enhanced offline web
application.  That would let me reuse Password Composer's existing static web
form without having to work through the iPhone SDK.  I found the early release
of Jonathan Stark's [Building iPhone Applications with HTML, CSS, and
JavaScript][stark] and set out to adapt Password Composer to the iPhone.

## Basic Application

The basic application started as a single HTML file containing a form and some
JavaScript.

{% highlight html %}

<html>
<head>
    <title>Password Composer</title>
</head>
<body>

<script type="text/javascript">
// Password Composer JavaScript
</script>

<h1>Password Composer</h1>

<form>
<table>
<tr>
    <td>Master Key:</td>
    <td><input type="password" id="masterpwd1" onkeyup="mpwd_generate()"
        onchange="mpwd_generate()"/></td>
</tr>
<tr>
    <td>Domain:</td>
    <td><input type="text" id="domain1" onkeyup="mpwd_generate()"
        onchange="mpwd_generate()"/></td>
</tr>
<tr>
    <td>Password:</td>
    <td><input type="text" id="genpwd1" onkeyup="mpwd_generate()"
        onchange="mpwd_generate()"/></td>
</tr>
</table>
</form>

</body>
</html>

{% endhighlight %}

The JavaScript and form were copied directly from Password Composer's [public
web-based interface][web] and worked just fine in Mobile Safari.

## Styling the Application

Next, I set about styling the look and feel of the page to more closely
resemble an iPhone application.  I started by adding a stylesheet file.  I
could have used inline CSS, and maybe I'll merge the styles back into the main
page at some point in the future, but using a separate file keeps things more
manageable for the time being.

{% highlight css %}

body {
    background-color: #ddd;
    color: #222;
    font-family: Helvetica; 
    font-size: 14px;
    margin: 0;
    padding: 0;
}

h1 {
    background-image: -webkit-gradient(linear, left top, left bottom,
                                        from(#ccc), to(#aaa));
    background-color: #ccc;
    border-bottom: 1px solid #666;
    color: #222;
    font-size: 20px;
    font-weight: bold;
    padding: 10px 0;
    text-align: center;
    text-decoration: none;
    text-shadow: 0px 1px 0px #ddd;
}

form {
    background-color: #FFFFFF;
    border: 1px solid #999999;
    color: #222222;
    font-size: 15px;
    padding: 12px 10px;
    margin: 10px;
    -webkit-border-radius: 8px;
}

td, input {
    font-size: 14px;
}

{% endhighlight %}

Most of these styles are influenced by examples from [Jonathan Stark's
book][stark].

### Viewport Definition

The next change is the addition of a `viewport` meta tag:

{% highlight html %}

<meta name="viewport" content="user-scalable=no, width=device-width" />

{% endhighlight %}

This `viewport` definition convinces Mobile Safari not to rescale the HTML
content as though it were a "normal" 980px-wide web page.

### Input Field Types

I was also interested in configuring the form's `<input>` fields to pop up the
most appropriate type of on-screen keyboard on the iPhone.  *Master Key* was
already being recognized as a password field (`type=password`), but I wanted
the *Domain* field to use the iPhone's special URL keyboard.

[A little research][keyboards] indicated that Mobile Safari recognizes a small
subset of the HTML5 input types, including the `url` type.

{% highlight html %}

<input type="url" id="domain1">

{% endhighlight %}

## Custom Icon

Because I want Password Composer to feel like a natural, first-class iPhone
application, it deserves to get a real icon.  This icon is used when a link to
the application is saved to the user's home screen.  Home screen icons are
57×57 pixel PNG images.

There are two ways for a web application to specify its home screen icon:

1. Provide a file named `apple-touch-icon.png` in the site's root.
2. Add a `<link rel="apple-touch-icon" href="icon.png" />` tag to the page's
   `<head>` block.

I chose the second approach because I wasn't planning on using an entire
domain to serve the Password Composer application.  It also feels like better
encapsulation for a single-page web application.

## Offline Application Cache

The last bit of work adds support for the iPhone's offline application cache.
This cache allows the files used by a web-based applications to be stored
locally on the iPhone so that they can still be accessed when the phone
doesn't have an active network connection.  This "offline mode" is described
quite well in Stark's book: [Chapter 6: Going Offline][ch6].

Briefly, all that's required is the creation of a "manifest" file, [as
specified by HTML5][html5].  My manifest file is named `pwdcomposer.manifest`:

    CACHE MANIFEST
    index.html
    iphone.css
    icon.png

The manifest lists all of the files used by the web application.  Manifest
files can be much more involved than this example.  They can specify fallback
images for use when network-based files are inaccessible, for example.  But
for my current purposes, the simple manifest above works great.

Manifest files must be served using the `text/cache-manifest` MIME type.
For Apache-like web servers, this can be configured using a directive like:

{% highlight apacheconf %}

AddType text/cache-manifest .manifest

{% endhighlight %}

Lastly, the web page needs to specify its associated manifest file using the
`manifest` attribute of the `<html>` element:

{% highlight html %}

<html manifest="pwdcomposer.manifest">

{% endhighlight %}

## Deployment

All that is left to do is deploy the application.  Simply serve up the source
files using a web server, and iPhone users can choose to bookmark the
application's link just like they would for any other web page.  The offline
application caching system will work automatically.  When the iPhone is
connected to a network, Mobile Safari will attempt to fetch the original
manifest file to check for updates, but the application will otherwise be
running entirely from the cache, essentially making it a local iPhone
application.

My version is deployed here: <https://www.indelible.org/pwdcomposer/>

The [complete source code][source] is available on [GitHub][github].

[pwdcomposer]: http://www.xs4all.nl/~jlpoutre/BoT/Javascript/PasswordComposer/
[web]: http://www.xs4all.nl/~jlpoutre/BoT/Javascript/PasswordComposer/password_composer_form.html
[stark]: http://building-iphone-apps.labs.oreilly.com/
[keyboards]: http://www.bennadel.com/blog/1721-Default-To-The-Numeric-Email-And-URL-Keyboards-On-The-iPhone.htm
[ch6]: http://building-iphone-apps.labs.oreilly.com/ch06.html
[html5]: http://www.w3.org/TR/offline-webapps/
[source]: https://github.com/jparise/pwdcomposer
[github]: http://github.com/
