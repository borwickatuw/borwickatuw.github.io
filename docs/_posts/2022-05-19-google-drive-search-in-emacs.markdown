---
layout: default
title: "Finding recent Google Drive documents with Emacs/elisp"
---
I regularly look at recently changed Google Drive documents so I can see what I need to work on. (For example, I check meeting minutes to see what commitments I made in the meeting.)

I do this enough that I built an elisp function to find recently-modified documents. I move to the end of this statement and type `C-x C-e` to execute it. That opens a Google Drive search that shows documents modified in the last 14 days!

{% highlight elisp %}
    (browse-url (concat "https://drive.google.com/drive/search?q=type:document%20after:"
			(format-time-string "%Y-%m-%d"
					    (time-subtract (current-time) (days-to-time 14)))))
{% endhighlight %}
