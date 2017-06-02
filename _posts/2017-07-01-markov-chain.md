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
larger source texts and more easily build complex models. I do find these
features appealing for the avenues of exploration they open, but I should be
clear that I don't see scale and effeciency as positives in their own right.
For me, much of the appeal of creating art is in the process.

As with the Tzarian model, the first step is to select some source material.
The nature of the model demands something longer than a news article. In this
case, I begin with [the complete works of Shakespeare](http://www.gutenberg.org/ebooks/100).

After downloading the text to the directory where I'm working, I can use Ruby
to load its contents into memory as a simple text object called a *string*.
I assign this string to the variable `text` so I can use it later.

~~~ruby
text = File.read('./shakespeare.txt')
~~~

Next, I need something like a virtual pair of scissors to cut this text into
its constituent words. In Ruby, a string has a `split` method I can use to
break the text into an array. When I use this `split` method, I provide an
argument that tells it what delimiter to use to find the boundaries between
words. To define this delimiter, I use something called a regular expression,
or regex, for short. The regular expression code `\s` matches with any
white space in the text. This lets me split on newline and tab characters as
well as plain spaces. Adding a `+` modifier extends the match to include
one or more of these whitespace characters.

I assign this array of words to a `tokens` variable. In linguistics, the word
'token' is used to refer to a given occurrance of a word in text or speech.

~~~ruby
tokens = text.split(/\s+/)
~~~

It would be overwhelming to list the full array of tokens here, but I can
sample a range to get a sense for what I'm working with.

~~~ruby
tokens[1176 .. 1188]
=> ["From", "fairest", "creatures", "we", "desire", "increase,", "That", "thereby", "beauty's", "rose", "might", "never", "die,"]
~~~

This array of words is the raw material I use to build a Markov chain model.
For each unique word in the list, I make a list of every word that can follow
it.

~~~ruby
def build_ngram_markov_transition_model(tokens, n=2, join_char='%', model={})
  model.default = []
  (0 .. tokens.length - 2*n).each do |i|
    this_ngram = tokens[i .. i+n-1]
    next_ngram = tokens[i+n .. i+(2*n)-1]
    model[this_ngram.join(join_char)] = model[this_ngram.join(join_char)]
      .dup
      .push(next_ngram.join(join_char))
  end
  model
end
build_ngram_transition_model
~~~

Below, for example, is a list of all the words that follow the word
'creatures.'

~~~ruby
["we", "broke", "of", "Turn", "as", "heartily.", "vile,", "and", "else", "works!", "feed", "as", "yet", "as", "ours,", "are,", "kings.", "that", "are", "as", "living,", "Whose", "on", "Of"]
~~~

If I select a word from this list, say, 'heartily.' I can see the list of words
that follow it as well.

~~~ruby
["sorry", "glad", "well", "will", "beseech", "forgive", "entreats", "to", "as", "request", "I", "and", "wish", "I", "solicit", "consented", "That", "indeed", "he"]
~~~




