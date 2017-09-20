---
layout: post
title:  "Selenium Click Capturing"
date:   2014-10-28
categories: selenium
---

If you've worked with Selenium for any length of time you'll have encountered a test where Selenium occassionally errors out with:

```
Element is not clickable at point (x, x). Other element would receive the click
```

In my experience with Selenium, I've found the click capturing to be one of the more finicky processes. Let's take a look at what's happening.

Why this happens
================

There are a couple of reasons that another element is recieving a mis-intentioned click. One is that the test is legitimately failing. That is, if a user were to try to click on the jQuery element, another element (like a menu) is over the intended target, obscuring it in some way.

With tests that are failing sporadically, especially if the click happens soon after page load, there may be another mechanic at work. 

In our tests, when a page was first loading, Selenium would find a button to click and in the time between when the button coordinates were located and the mouse was clicked, CSS reflowing caused the position of the button to be relocated.

{<1>}![Selenium Click Capturing](/content/images/2013/Nov/Screen_Shot_2013_11_14_at_11_51_04_PM.png)


Possible (python selenium) Solutions
====================================

Use jQuery to click
-------------------

If you are using jQuery in your project you can leverage the jQuery click method to click on the button.

{% highlight python %}
def jquery_click(driver, css_selector):
    driver.execute_script("$(\"" + selector + "\").click();")
{% endhighlight %}

Wait until it's clickable
--------------------------

If you prefer to use the UI (which, after all, is how your end user will be accessing the element), you have to be a little more clever. 

One option is that you can wait until the UI stops moving as demonstrated in code below:

{% highlight python %}
import time
from selenium.common import exceptions as s_exc
import selenium.webdriver.support.ui as ui

def wait_until_clickable(driver, css_selector):
  driver.wait = ui.WebDriverWait(driver, 30)

  def stopped_moving(driver):
    try:
      element = driver.wait.until(EC.element_to_be_clickable((By.CSS_SELECTOR, css_selector)))
      initial = (element.location, element.size)
      time.sleep(0.05)
      element = driver.wait.until(EC.element_to_be_clickable((By.CSS_SELECTOR, css_selector)))
      final = (element.location, element.size)
      return initial == final
    except s_exc.StaleElementReferenceException:
      return False

  driver.wait.until(stopped_moving)
  return driver.wait.until(EC.element_to_be_clickable((By.CSS_SELECTOR, css_selector)))
{% endhighlight %}

elaborate click function
------------------------

In our integration tests we use a combination of the two methodologies above creating an elaborate click function that we use instead of the simple `click()` provided by selenium. This has drastically cut down the flakeyness of our tests.

Other reasons click() may not work
==================================

Make sure the middle is clickable
---------------------------------

In selenium, the `click()` function clicks on the middle of an element. If your element is unclickable at that point you're going to have a problem.

Make sure the window containing the clicked element is in front
---------------------------------------------------------------

Sometimes when you're not looking the window you are trying to interact with will be backgrounded. Check out this article on foregrounding the window- [Working with Multiple Windows](http://elementalselenium.com/tips/4-work-with-multiple-windows)
