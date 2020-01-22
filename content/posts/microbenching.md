---
title: "Microbenchmarking for Learning"
date: 2020-01-22T07:29:42-05:00
draft: false
---

# Data Structures Make a Difference

I enjoy working on small puzzles to try to learn as well as keep my programming skills sharp. I find these puzzles in several different places such as [Code Wars](https://codewars.com), [Project Euler](https://projecteuler.net), or [Exercism](https://exercism.io). 

_Before going any further, do know there may be **spoilers** for your solution to the problem if you want to try Exercism: Scrabble Score._

I like to try different things when solving these problems to explore whether the code is more readable, use a different data structure than may normally be used, or a different programming paradigm (ex. functional approach instead of an imperative one). Additionally, I use the mentoring to hear other programmers opinions and learn more than I would just working in isolation. 

My first solution is as follows:

```python
letter_values = [
    (set("aeioulnrst"), 1),
    (set("dg"), 2),
    (set("bcmp"), 3),
    (set("fhvwy"), 4),
    (set("k"), 5),
    (set("jx"), 8),
    (set("qz"), 10),
]


def score(word):
    return sum(value 
        for letter in word 
        for st, value in letter_values 
        if letter.lower() in st
    )
```

Now if you are unfamiliar with many of the platforms that I mentioned, then you should know there are corresponding unit tests that determine the _correctness_ (input/output) for your solution. The above code passed all of the tests so I submitted it. All is good to go, right?

...

Well, yes it works based on the tests. However, the person reviewing the code made a mention that it was **SLOW**. 

![Shocked Owl](/blog/images/shocked-owl.gif)

Anytime someone says that, I give pause