---
layout: post
title:  "Welcome to Cabu !"
description: "I've open sourced one of my personal Python projects."
date:   2016-01-06 09:00:00
keywords: "welcome, cabu, python, framework, microservice, crawling, docker"
category: open-source
comments: true
---

Today, I decided to open source [Cabu][cabu], my first Open Source project.
It's a framework to create crawler as microservices.

<h2>Why another crawling framework in Python ?</h2>

It's been a while, that I was creating crawlers associated with schedulers
hosted on AWS. As many developers, I faced the lack of features of some actual
crawling projects, the time consuming configuration of Selenium webdrivers
(particularly when you want to use it headless), the lack of modern apps
architecture in some scraping projects, etc. I thought that maybe releasing the
architecture I'm using must be interesting for some Pythonistas.

So after few weeks refactoring the stack, ensuring that the codebase is well
tested and writing documentation, It's finally ready to be shown to the world !
There is a lot of things still to do but already enough to play around and avoid
you some headaches in the world of crawling :-)

<h2>How does it work ?</h2>

After installing the Python package using pip and a Selenium web driver, you can
create in a matter of seconds a crawler like in this example :

{% highlight python %}
@app.route('/gizmodo_last_articles_links')
def gizmodo_last_articles():
    app.webdriver.get('http://www.gizmodo.com')
    articles_links = [
        i.get_attribute('href') for i in app.webdriver.find_elements_by_css_selector('h1.headline>a')
    ]

    return jsonify({'articles': articles_links})

#=> http myapp.io/gizmodo_last_articles_links
{% endhighlight %}

And that's it !
There are already a bunch of features and lot more to come.
Actually, I'm already using some of the future features, but I want to release
them once I'm sure it's ready to be open sourced (Better be safe than sorry).
They are listed on the [changelog page][changelog] of the
[documentation][documentation].

Thanks to [Sofia][sofia-github] and Trisha for the patience to have a look to the
documentation and fixing my english.
Thanks to Maréva for the beautiful illustration of the robot.

I hope it will be useful to you. Feel free to create pull-requests and make any
improvements suggestions.

Cheers.


[cabu]:          https://github.com/thylong/cabu
[changelog]:     https://cabu.readthedocs.org/en/latest/changelog.html
[documentation]: https://cabu.readthedocs.org/en/latest/index.html
[sofia-github]:  https://github.com/sofiama
