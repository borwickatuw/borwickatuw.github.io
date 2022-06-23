---
layout: default
title: "Creating and using JavaScript bookmarklets"
---
A bookmarklet is a browser bookmark that contains JavaScript. These are like little utilities that you can call for a given web page.

For example, I use:

- https://gist.github.com/n-st/0dd03b2323e7f9acd98e - bookmarklet to call archive.org for the current page. This lets you see anything the Wayback Machine has for the current page.
- https://chafel.github.io/bookmarklets/ "inject jQuery" - This lets me add jQuery to the page I'm on, so I can use the browser console with access to jQuery functions.

I recently found <https://caiorss.github.io/bookmarklet-maker/>, which will convert JavaScript into its URL encoded version. For example, the code

{% highlight javascript %}
    var answer = prompt('ID?');
	window.location.href = "http://example.com/path/to/id/" + answer;
{% endhighlight %}

gets turned into

    javascript:(function()%7Bvar answer %3D prompt('ID%3F')%3B   window.location.href %3D %22http%3A%2F%2Fexample.com%2Fpath%2Fto%2Fid%2F%253D%22 %2B answer%3B%7D)()%3B

You can add this to your browser bookmarks, and then when you click it, you'll be asked for an ID. When you populate the ID, you'll then be redirected to a URL that starts with `http://example.com/path/to/id/` and then gets whatever you put as the ID added to the end of it.
