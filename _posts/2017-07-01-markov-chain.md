---
layout: post
title:  "generative poetics episode 2: markov chains"
---

In the last episode, I discussed the simplest model of generative poetry, what
I called the naive Tzarian model after the dadaist Tristan Tzara. Tzara's poem
*How to Make a Dadaist Poem* describes a method by which the aspirant poet
cuts a newspaper article into words and draws them randomly from a bag to
create a poem.

Of course, this method doesn't work very well. Setting aside the question of
poetic merit, a text generated with this method frustrates its reader with
syntactical impossibilities like "with of hairdresser stags" and "the to the."
Even if I wanted to create a poem that would challenge our expectations of
language, I'd need to do it in a more interesting way—"the to the" is worse
than wrong, it's *boring*.

In this post, I discuss a *Markov chain* model of text generation. This model
improves on the naive Tzarian approach by eliminating some of the most
egregious syntactical violations.

Building a Markov chain isn't feasible with scissors and paper. Instead, I'll
use a programming language called Ruby to build an algorithmic model. Replacing
my newspaper article with a digital text file and defining my model in code
changes the process of generative poetry significantly. It lets me process much
larger source texts and more easily build complex models. These features of
computer models open avenues of exploration that I find very compelling, but I
should be clear that I don't see see bigger and faster as necessarily better.
For me, much of the appeal of creating art is in the process, so the fact that
I can create something faster is compelling as an exploration of process, not
because I'm getting to some product more quickly.

As with the Tzarian model, our first step it to select some source material.
The nature of the model demands something longer than a news article. In this
case, I begin with [the complete works of Shakespeare](http://www.gutenberg.org/ebooks/100).

After downloading the text, I can use Ruby to load its contents into memory as
a simple text object called a *string*. I'll assign this string to a variable
called `text`, so I can use it later.

```
text = File.read('./shakespeare.txt')
test
test
```

