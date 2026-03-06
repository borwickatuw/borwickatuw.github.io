---
layout: default
title: "Manipulating MacOS HTML clipboard content"
---

Sometimes I need to change the HTML heading levels when I'm copying from one environment to another. Here's a different approach!

The end result is

    `pbpaste-html | shift-headings | pbcopy-html`

MacOS's clipboard actually stores a plaintext and an HTML version, so the `*-html` scripts manipulate the HTML versions.

# 1. `pbpaste-html`

```
#!/usr/bin/env swift
import AppKit
if let data = NSPasteboard.general.data(forType: .html),
   let html = String(data: data, encoding: .utf8) {
    print(html)
}
```

# 2. `shift-headings`

```
#!/usr/bin/env -S uv run --script
# /// script
# dependencies = ["beautifulsoup4"]
# ///
# Thanks, Claude Code! 2026-03-06
"""Shift HTML heading levels. Usage: shift-headings [N] < input.html"""
import sys, re
from bs4 import BeautifulSoup

shift = int(sys.argv[1]) if len(sys.argv) > 1 else 1
soup = BeautifulSoup(sys.stdin, "html.parser")

for level in (range(6, 0, -1) if shift > 0 else range(1, 7)):
    for tag in soup.find_all(f"h{level}"):
        new_level = min(max(level + shift, 1), 6)
        tag.name = f"h{new_level}"

print(soup)
```

# 3. `pbcopy-html`

```
#!/usr/bin/env swift
import AppKit
let input = FileHandle.standardInput.readDataToEndOfFile()
if let html = String(data: input, encoding: .utf8) {
    NSPasteboard.general.clearContents()
    NSPasteboard.general.setString(html, forType: .html)
}
```
