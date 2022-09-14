---
layout: default
title: "Replacing URLs with their redirects for text files"
---
If you are super duper sure that the input is safe, here's a Perl script that will update org-mode URLs based on where they redirect:

        perl -i.bak -pe 's|\[(http[^\]]*?itconnect.*?)\]|"[".`curl -Ls -o /dev/null -w %{url_effective} $1`."]"|eg' *.org

This find URLs denoted inside brackets, e.g. `[https://itconnect.uw.edu]`, and calls curl on them to get their effective URL.

Thanks very much to https://stackoverflow.com/a/3077316 for the curl part.

This could definitely be redone as a proper Python or Perl script, but this seemed like the easiest way to update a bunch of content all at once when the itconnect.uw.edu link structure changed.
