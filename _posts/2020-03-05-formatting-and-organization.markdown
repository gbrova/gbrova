---
layout: post
title: "Code Formatting and Code Organization"
date: 2020-03-05 18:50:20 +0000
tags: python coding
---


I recently came to the conclusion that nicely formatted code tends to be better organized. Now I even think that automatic linters can (occasionally) expose ways to reorganize code and eliminate some code smells.


## Automatic formatting with Black

I recently decided to adopt [Black](https://github.com/psf/black) for a Python project I'm working on with several other people.

If you didn't know, Black is an opinionated code formatter for Python, similiar to Prettier for JavaScript.

It's generally accepted that a codebase with consistent formatting is easier to read and navigate.
Additionally, adopting an automatic, opinionated code formatter like Black brings us two advantages. First, we don't have to think about adhering to any consistent formatting rules - just save a file and Black takes care of it. Second, style-related comments should disappear almost entirely from code review discussions.

More recently, I came across a more surprising benefit.

## Prettier Code is Better Organized

Or, to phrase it less controversially: all else being equal, it's harder to prettify disorganized code.

To illustrate with a short example, consider this snippet:

```
assert isinstance(start_time, datetime) and isinstance(end_time, datetime), "Pass datetime objects to 'start_time' and 'end_time'"
```

This line is too long ("ugly"), so Black automatically reformatted it for me:

```
assert isinstance(start_time, datetime) and isinstance(
    end_time, datetime
), "Pass datetime objects to 'start_time' and 'end_time'"
```

Yuck! Now the line is split up, but in an arbitrary and asymmetric way. My first instinct was to fight the tool or maybe just disable auto-formatting for this line. My second instinct was to try to fix Black: I agree this line is too long and needs to be split up, but Black should pick a _better_ way to split it. I just couldn't find a more natural way to split it.

Taking a step back, I realized the formatting is telling me something else: _this single line of code is doing too many things_!

The snippet becomes clearer if split into two lines:

```
assert isinstance(start_time, datetime), "Pass a datetime object to 'start_time'"
assert isinstance(end_time, datetime), "Pass a datetime object to 'end_time'"
```

Now, the code for handling start_time and end_time looks the same, so it's clearer at a glance that the two lines are doing basically the same thing[^1]. As a bonus, the error messages are more precise.

I came to this realization because the original line was too long, and there was no good natural way to split it. This is a surprisingly deep insight to come from a simple formatting tool!

---

[^1]: Depending on circumstance, this might also be an opportunity to extract a function for the repeated functionality.
