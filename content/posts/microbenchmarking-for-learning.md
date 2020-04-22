---
title: "Microbenchmarking for Learning"
date: 2020-01-22T07:29:42-05:00
draft: false
tags:
    - python
    - benchmarking
    - performance
    - data structures
---

I enjoy working on small puzzles to try to learn as well as keep my programming skills sharp. I find these puzzles in several different places such as [Code Wars](https://codewars.com), [Project Euler](https://projecteuler.net), or [Exercism](https://exercism.io). 

_Before going any further, do know there may be **spoilers** for your solution to the problem if you want to try Exercism: Scrabble Score._

I like to try different things when solving these problems to explore whether the code is more readable, using a different data structure than may normally be used, or utilize different programming paradigm (ex. functional approach instead of an imperative one). Additionally, I use the mentoring or peer to peer interactions on these sites to hear other programmers opinions and learn more than I would just working in isolation. 

My first solution is as follows:

```python
# Scrabble Score
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

Well, yes it works based on the unit tests. However, the mentor reviewing the code mentioned that the solution was **SLOW**. 

![Shocked Owl](/blog/images/shocked-owl.gif)




Anytime someone says that, I give pause. Sometimes the difference in performance, whether it be runtime, memory, network, etc., is small enough to not matter. However, I like to measure to put my mind at ease.

![Thoughtful Scientist](/blog/images/thoughtful-scientist.gif)

---
 
### Benchmarking the Original Solution

**Caution:** _Microbenchmarking results can be difficult to reason about so use your own judgement about the following results._

How does one benchmark a small function? I chose to use the `timeit` module that is part of the Python standard library, but there are other approaches to making these measurements such as using the `time` command in bash or utilizing a plugin for `pytest`.

Let's look at the results:
```bash
python -m timeit -s 'from scrabble_score import score' 'score("awesome")' 
#50000 loops, best of 5: 7.42 usec per loop
python -m timeit -s 'from scrabble_score import score' 'score("chimichanga")'
#20000 loops, best of 5: 11.7 usec per loop
```

We can see that we are in the microsecond range for scoring a scrabble word. Now that seems fairly fast to me because I have a hard time understanding the microsecond scale, but maybe I am wrong.

Let's rewrite the solution to see if we can make things **FASTER**.

---

### Rewrite Using Dictionaries
```python
scores = (
    (set("aeioulnrst"), 1),
    (set("dg"), 2),
    (set("bcmp"), 3),
    (set("fhvwy"), 4),
    (set("k"), 5),
    (set("jx"), 8),
    (set("qz"), 10),
)

lookup = dict(
    (letter, value) 
    for letters, value in scores 
    for letter in letters
)


def score(word):
    return sum(lookup[letter] for letter in word.lower())
```

As you can see, I leveraged some of the previous solution, but created a lookup dictionary for determining the value for a given letter.

But did that change anything?


...

Let's find out!!

```bash
python -m timeit -s 'from scrabble_score import score' 'score("awesome")' 
#500000 loops, best of 5: 962 nsec per loop
python -m timeit -s 'from scrabble_score import score' 'score("chimichanga")'
#200000 loops, best of 5: 1.25 usec per loop
```


### What did we learn?
Well look at that. Changing to a dictionary as the lookup for a given letter provided between **~7x** speed up. Now that sounds like a great performance impact, but remember these are very small sample sets for testing so we should probably test more if our program is bottlenecking on the `score` function. 

The real take away though. I think you should explore data structures, paradigms, and different solutions when learning. However, picking an appropriate data structure (such as a dictionary) can provide better performance and many times readability. If I wanted to further refine the solution above, then I would take some time to change how the `lookup` dictionary is constructed such as:
```python

lookup = {
    'a': 1, 
    'b': 3, 
    'c': 3,
    'd': 2,  
    #...
    'z': 10, 
}

```

Well that was interesting to see, and I enjoy the perplexity that crops up in random places. I guess it is back to work. Let's see what other trouble, er... I mean, work I can get into.





