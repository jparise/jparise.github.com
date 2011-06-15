---
layout: post
title: Manually Loading nib Files
category: iphone
---

[Interface Builder][ib] is the standard Mac OS and iOS user interface design
tool.  Layouts are saved as "nib" (NeXT Interface Builder) files.  Recent
versions of Interface Builder write these files as XML archives using the
`.xib` file extension.

At runtime, nib files are loaded and assigned an "owner" object that forms the
basis of the layout's object hierarchy.  This "owner" is very often a view
controller object.  In this article, we'll focus specifically on UIKit and its
[`UIViewController`][UIViewController] class.

The nib loading process is generally abstracted away as part of the standard
`UIViewController` initialization process.  Here's a simple example:

{% highlight objectivec %}

- (id)init
{
    if ((self = [super initWithNibName:@"SomeNib" bundle:nil])) {
        // additional view controller initialization
    }

    return self;
}

{% endhighlight %}

`-initWithNibName:bundle:` handles all of the nib loading and instantiation
work.  While this is quite convenient, it has one major limitation: the nib
file must have been compiled and bundled (in the `NSBundle` sense) along with
the application itself.  Fortunately, it is possible to manually load a nib
from arbitrary data and associate its contents with a view controller.

## The UINib Class

The [`UINib`][UINib] class provides all of the functionality needed to
manually load nib files.  One reason to do this is to pre-cache the contents
of a nib file in memory, ready for quick deserialization and instantiation
later in the application's lifetime.  Another reason (and our focus here) is
to load the raw nib data from an alternate (non-`NSBundle`) source.

The `UINib` class provides two methods for loading nibs:

- `+nibWithNibName:bundle:` - load a bundled nib file
- `+nibWithData:bundle:` - load raw nib data

Once loaded, the nib's objects can be instantiated and assigned an owner using
the `-instantiateWithOwner:options:` instance method.

## Manual View Controller Loading

How can the `UINib` class be used in conjunction with a view controller?

First, the view controller needs to call `UIViewController`'s designated
initializer *without* a nib name.  Note that passing `nil` as the nib name
will trigger an automatic lookup for a nib named the same as the view
controller class, so take care not to fall into that case unintentionally.

{% highlight objectivec %}

[super initWithNibName:nil bundle:nil]

{% endhighlight %}

Because this view controller hasn't loaded a view during initialization, we
must implement the `-loadView` method.  This is where all of the manual
loading is performed.

{% highlight objectivec %}

- (void)loadView
{
    [super loadView];

    UINib *nib = [UINib nibWithNibName:@"SomeNib" bundle:nil];
    [nib instantiateWithOwner:self options:nil];
}

{% endhighlight %}

This code effectively reproduces the standard nib loading process that was
previously being handled by `-initWithNibName:bundle:` itself.

## Loading Raw nib Data

At this point, our code is now manually loading the nib's objects, but it's
still reading the nib itself from our application bundle.  We can go a step
further and load arbitrary nib data using the `+nibWithData:bundle:` method.

{% highlight objectivec %}

- (void)loadView
{
    [super loadView];

    NSData *data = [NSData dataWithContentsOfFile:@"SomeNib.xib.bin"];

    UINib *nib = [UINib nibWithData:data bundle:nil];
    [nib instantiateWithOwner:self options:nil];
}

{% endhighlight %}

The nib data can come from pretty much anywhere, such as from a bundled file
or a network-fetched resource.  The only requirement is that the nib data be
compiled.  To compile an existing `.xib` file, use [`ibtool`][ibtool]:

    ibtool SomeNib.xib --compile SomeNib.xib.bin

## Additional Considerations

Now that the basic approach has been established, there are a few practical
considerations to keep in mind.  First, we've largely ignored the `bundle`
parameter in the above code examples.  This argument instructs the nib loader
where to look for dependent resources, such as images.  Passing `nil` implies
that the main application bundle (`[NSBundle mainBundle]`) should be used.
There unfortunately doesn't appear to be a way to redirect these lookups to
non-bundle locations, however.

Next, the nib data needs to be available by the time the view controller is
initialized.  All I/O needs to block, and any stalls could result in a poor
user experience.  Given that, any non-bundled nib data needs to be fetched
*before* the view is loaded.

Lastly, nibs can contain references to "external" objects.  One such example
is the "owner" object, which is explicitly resolved by the call to
`-instantiateWithOwner:options:`.  If the nib references any additional
external objects, the loading code must provide those objects via a dictionary
mapping passed through the `options:` interface's `UINibExternalObjects` key.

[ib]: http://en.wikipedia.org/wiki/Interface_Builder
[UIViewController]: http://developer.apple.com/library/ios/#documentation/uikit/reference/UIViewController_Class/Reference/Reference.html
[UINib]: http://developer.apple.com/library/ios/#documentation/uikit/reference/UINib_Ref/Reference/Reference.html
[ibtool]: http://developer.apple.com/library/mac/#documentation/Darwin/Reference/ManPages/man1/ibtool.1.html
