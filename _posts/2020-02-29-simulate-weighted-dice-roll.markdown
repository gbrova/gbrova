---
layout: post
title: "Simulating a Weighted Dice"
date: 2020-02-29 18:50:20 +0000
tags: programming python
---

Doing and re-doing the same thing in different ways tends to be a good way to build expertise. By asking the same question numerous times in technical interviews, I realized repetition can open new sights even with very simple tasks.  In this post I'll walk through one of my favorite interviewing questions and several quite different ways of approaching it.


## Problem description

The task is straightforward: write some code to simulate rolling a multi-sided, weighted dice. An example input might be the 4-sided dice with probabilities `[0.1, 0.1, 0.3, 0.5]`, where the output should be an integer between 1 and 4, such that 4 is 5x as likely as 1.

A common approach is to assign each outcome to a unique a segment between 0 and 1. For example, consider the following assignment:

{% highlight python %}
[0, 0.1) -> 1
[0.1, 0.2) -> 2
[0.2, 0.5) -> 3
[0.5, 1) -> 4
{% endhighlight %}

Since the length of each segment is proportional to its desired probability, we can just generate a random number between 0 and 1, see which segment it falls in, and return the corresponding side label.

In Python 3, you might write the above approach like this:

**Attempt 1**
{% highlight python %}
import random
from itertools import accumulate
def roll(probs):
cumulative_weights = list(accumulate(probs))
position = random.random()
for side, end_position in enumerate(cumulative_weights):
if position < end_position:
return side + 1 # Dice are 1-indexed

# Usage example

print(roll([0.1, 0.1, 0.3, 0.5]))
{% endhighlight %}

## What about rolling the same dice many times?

The above approach takes O(N) time for each dice roll - it takes linear time to build `cumulative_weights`, and also linear time in the `for` loop to find the right position. Can we save some time if our users want to roll the same dice many times? It turns out that we can. First, since the cumulative weights are sorted, we can use a binary search to eliminate the `for` loop. Second, we can compute the cumulative weights only once, and reuse it across multiple dice rolls.

At this stage, most candidates write something like this:

**Attempt 2**
{% highlight python %}
import bisect
def compute_cumulative(probs):
cumulative_weights = list(accumulate(probs))
return cumulative_weights

def roll(cumulative_weights):
position = random.random()
return bisect.bisect(cumulative_weights, position) + 1 # Dice are 1-sided

# Usage example

cumulative = compute_cumulative([0.1, 0.1, 0.3, 0.5])
print(roll(cumulative))
{% endhighlight %}

## Can we do better?

One problem with the above approach is that we exposed some implementation details to the user. Users now have to think about what to do with cumulative weights, which they didn't have to think about before.

The natural object-oriented way to improve this is to introduce a class:

**Attempt 3**
{% highlight python %}
class Dice:
def **init**(self, probs):
self.cumulative_probs = self.\_compute_cumulative(probs)

def compute_cumulative(self, probs):
cumulative_weights = list(accumulate(probs))
return cumulative_weights

def roll(self):
position = random.random()
return bisect.bisect(self.cumulative_probs, position) + 1 # Dice are 1-sided

# Usage example

d = Dice([0.1, 0.1, 0.3, 0.5])
print(d.roll())
{% endhighlight %}

A fair number of the candidates I interview struggle to come up with a solution that takes less than linear time for each roll and also doesn't expose implementation internals.

While discussing this problem with a group of friends, someone accused me of being "too OOP" when I explained that I nudge candidates towards the class solution above. Fair enough. Here's another way using generators:

**Attempt 4**
{% highlight python %}
import bisect
from itertools import accumulate
from collections import Counter
def gen_rolls(probs):
cum_probs = list(accumulate(probs))
while True:
yield bisect.bisect(cum_probs, random.random()) + 1

# Usage example

rolls = gen_rolls([0.1, 0.1, 0.3, 0.5])
print(next(rolls))
{% endhighlight %}

There are surely many other approaches as well. Challenging yourself to come up with other approaches could be a good way to discover new programming paradigms - for example, how would you reimplement this using closures?

## Extensions

If you enjoyed thinking about this question, or perhaps are considering asking it in your own interviews, here are some follow-on questions to think about:

1. What if we no longer have access to a random number generator like `random.random()`, and our only source of randomness is something like `randomCoinToss() -> boolean`?
2. What if `randomCoinToss()` is very expensive? How can we minimize the number of times we call it?
3. How would you test your `roll()` function? What are some challenges to writing tests for a nondeterministic function, and how might we overcome them?

## Conclusion

I like interview questions that cover algorithms and problem solving, and also have a code organization or design element. In my experience, the roll-a-weighted-dice question gives candidates an opportunity to show off both these skills, which is probably why I ask it quite often these days.
