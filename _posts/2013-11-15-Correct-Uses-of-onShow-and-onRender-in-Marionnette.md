---
layout: post
title:  "Correct Uses of onShow and onRender in Marionette"
date:   2013-11-15
categories: marionettejs
---

For the past few months, I have been consistently stymied by Marionette's built-in onShow and onRender methods. 

In a single-page app, you often manipulate the DOM Tree instead of doing full page reloads. [Marionette.js](http://marionettejs.com/) is a handy [backbone](http://backbonejs.org/) library that provides a clean way to execute this manipulation in the form of its Layouts and Views. It is often convenient to know when a View's DOM subtree has successfully been inserted into the DOM and is jQuery accessible. 

This is where the `onShow()` and `onRender()` methods can come in handy.

What the docs say
=================

onRender
--------

>"render" / onRender event

> Triggered after the view has been rendered. You can implement this in your view to provide custom code for dealing with the view's el after it has been rendered.

_from [Marionette ItemView Docs](https://github.com/marionettejs/backbone.marionette/blob/master/docs/marionette.itemview.md#render--onrender-event)_

onShow
------

>A region will raise a few events when showing and closing views:

> "show" / onShow - Called on the view instance when the view has been rendered and displayed.

> "show" / onShow - Called on the region instance when the view has been rendered and displayed.


_from [Marionette Region Docs](https://github.com/marionettejs/backbone.marionette/blob/master/docs/marionette.region.md)_


This does not present us a full picture of why, when, and how we should use these methods.

What the docs should say
========================

onRender
--------

`onRender()` was a misleading term for me. "onRender", I said to myself, "The sounds like it would be invoked when DOM elements have been _rendered_ by the browser." 

![false meme]({{ site.url }}/images/false-meme.jpg)

**`onRender()` gets triggered when the View's DOM subtree is prepared for insertion into the DOM but before those elements are viewable.** Putting code in the onRender function of your view will NOT guarantee that you will have jQuery access to these newly created elements.
 

onShow
------

The `onShow()` method is used in the context of Layouts and Regions. A Marionette Layout may have many "Regions" that will later be populated with sub-Views. These sub-Views are instantiated when a Layout invokes its region's `.show()` method with the View to be rendered.

When `.show()` is invoked, the DOM subtree for that sub-View is constructed and inserted into the DOM. A "show" event then triggers any code in the `onShow()` methods in both the Layout and the sub-View. 


Proper Use
==========

Code in `onRender()` will be executed reliably **every** time `this.render()` is called by the view. You should include code that needs to be called every time you would update that View. `onRender()` does not assure you jQuery access to the View's DOM elements. Code that relies on DOM elements should be in the `onShow()` method.

Code in in both the Layout and the sub-View's `onShow()` methods will be executed **every** time `someRegion.show(subView)` is called by the Layout. The Layout may not call `.show()` every time you update that sub-View. Code that needs to be run on every rendering should be in the sub-View's `onRender()`.

Improper Use
============

The triggering of `onShow()` methods will behave unexpectedly if you do not handle handle nested `.show()` method calls carefully. Below is a diagram of a concrete example of deeply nested layouts and views that illustrate this:
![Deeply nested views]({{ site.url }}/images/marionette-on-show.png)

Layout A Snippet
----------------

{% highlight javascript %}
regions: {
  regionAOne: ".region-a-one",
  regionATwo: ".region-a-two"
},

onRender: function() {
  this.regionAOne.show(LayoutB);  // wrong
  this.regionATwo.show(ViewA);
}
{% endhighlight %}

Layout B Snippet
----------------

{% highlight javascript %}
regions: {
  regionBOne: ".region-b-one",
  regionBTwo: ".region-b-two"
},

onRender: function() {
  this.regionBTwo.show(ViewC);  // wrong
  this.regionBOne.show(ViewB);
}
{% endhighlight %}

View C Snippet
--------------

{% highlight javascript %}
onShow: function() {
  this.$el.addClass("class-name");
}
{% endhighlight %}

In this example, in **View C** we will run into a javascript error: _cannot call addClass of undefined_. But wait! Shouldn't **View C**'s `onShow()` method only be invoked if **View C**'s DOM Tree elements are properly in the DOM? 

We see this unexpected behavior because in the deeply nested layouts, rather than waiting for **Layout B**'s DOM subtree to be viewable, **View C** is instantiated in the `onRender()` method. We see the same in **Layout A** in regards to **Layout B**.

In this case the order of events is as follows:

 1. **Layout A** comes into being (probably from a `.show()` call by its parent.
 
 1. When **Layout A**'s DOM subtree has been constructed **Layout B** and **View A** are instantiated using **regionAOne** and **regionATwo**'s `.show()` methods.
 
 1. When **Layout B**'s DOM subtree has been constructed **View B** and **View C** are instantiated using **regionBone** and **regionBTwo**'s `.show()` methods.
 
 1. **Layout A**'s DOM subtree is visible in the DOM.
 
 1. `onShow()` in **Layout A**, **Layout B**, **View C**, etc. are all triggered at the same time.
 
 1. **Layout B**'s DOM subtree is visible in the DOM.
 
 1. **View C**'s DOM subtree is visible in the DOM.

Corrected Layout A Snippet
--------------------------

To ensure that each DOM subtree is inserted into the DOM before the `onShow()` method is called, move the region `.show()` code to the `onShow()` method of the Layout:

{% highlight javascript %}
regions: {
  regionAOne: ".region-a-one",
  regionATwo: ".region-a-two"
},

onShow: function() {
  this.regionAOne.show(LayoutB);  // correct
  this.regionATwo.show(ViewA);
}
{% endhighlight %}

If you are unable to access jQuery DOM elements in an `onShow()` method, trace your way through your Layout/View hierarchy and find the culprit whose region calls `.show()` in a non-`onShow()` method.

