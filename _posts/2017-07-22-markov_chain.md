---
layout: post
title:  'generative poetics episode 2: markov chains'
comments: true
---

In the last episode, I discussed a very simple model of generative poetry:
slice a newspaper article into words and draw these at random from a bag to
form a poem.  Unsurprisingly, this method is likely to frustrate even the most
syntactically generous of the poetic avant-garde with jarring word combinations
like 'of the of'.

In this post, I discuss a *Markov chain* model of text generation. This model
improves on the bag-of-words approach by eliminating some of the most egregious
syntactical violations.

Building a Markov chain isn't feasible with scissors and paper. Instead, I'll
use a programming language called Ruby to build an algorithmic model.  This
lets me process larger source texts and follow a more complex process to turn
them into poetry. There are aesthetic differences between analog and digital
methods of poetry generation. For now, I won't discuss the quality of quantity
of those differences—suffice to say only that each approach has its proper
merits.

The first step is to select some source material. The nature of the model
demands something longer than a news article. In this case, I begin with [the
complete works of Shakespeare](http://www.gutenberg.org/ebooks/100).

After downloading the text to a directory on my files sytem and trimming out
some of the front matter, I use Ruby to load its contents to a variable called
`text`.

~~~ruby
text = File.read('./shakespeare.txt')
~~~

Next, I use the `split` method to break this text into array of words. I assign
this array of words to a `tokens` variable. (In linguistics, 'token' refers to
a specific occurrence of a word in text or speech.)

~~~ruby
tokens = text.split(/[\s[[:punct:]]+/)
~~~

It would be overwhelming to list the full array of tokens here, but I can
sample a range to get a sense for what I'm working with. The 6th to 19th
tokens in the array are the first two lines of Shakespeare's first sonnet.

~~~ruby
tokens[6 .. 19]
=> ["From", "fairest", "creatures", "we", "desire", "increase", "That", "thereby", "beauty", "s", "rose", "might", "never", "die"]
~~~

This array of words is the raw material I use to build a Markov chain model.
For each unique word in the list, I create a new array of every word that shows
up after an instance of that word in the text.

~~~ruby
def build_markov_transition_model(tokens)
  model = Hash.new([])
  (0 .. tokens.length - 2).each do |i|
    model[tokens[i]].push(tokens[i+1])
  end
  model
end
markov_model = build_markov_transition_model(tokens)
~~~

Below, for example, is an array of all the words that follow the word
'creatures'.

~~~ruby
markov_model['creatures']
=> ["we", "broke", "of", "not", "Turn", "as", "heartily", "would", "Gods", "But", "vile", "sitting", "and", "else", "their", "works", "feed", "That", "get", "as", "yet", "as", "ours", "are", "kings", "that", "Against", "are", "as", "living", "Whose", "want", "Which", "on", "Of"]
~~~

Using the `sample` method on this array picks out a word at random.  Because
some words show up after 'creatures' in multiple places in Shakespeare's work,
these words are listed multiple times in the array. This means that the list
represents the relative frequencies of all words following 'creatures.' For
example, the word 'as' shows up 4 times out of the 35 possible words, while
'feed' appears only once, giving 'as' a frequency of 0.114 and 'feed a
frequency of 0.029.

~~~ruby
markov_model['creatures'].sample
=> 'feed'
~~~

Next, I'll take the result of this `sample` method and repeat the process. In
this case, because `sample` picked the token 'feed', I'll get the list of
all the words that follow 'feed'.

~~~ruby
markov_model['feed']
=> ["on", "mine", "myself", "but", "with", "their", "Yea", "Here", "Are", "on", "and", "ORLANDO", "on", "IMOGEN", "again", "on", "and", "capons", "upon", "And", "Even", "A", "on", "like", "contention", "in", "and", "upon", "you", "it", "on", "upon", "their", "this", "That", "on", "were", "I", "grow", "fat", "upon", "nothing", "my", "my", "it", "upon", "on", "upon", "my", "it", "upon", "st", "my", "Although", "Most", "Than", "st", "not", "Tis", "him", "the", "on", "on", "his", "me", "SATURNINUS", "his", "TAMORA", "too", "for", "your", "on", "on", "on", "upon"]
~~~

To repeat this process, I use an object in Ruby called an enumerator.

~~~ruby
def build_markov_enumerator(model, init_value=nil)
  Enumerator.new do |yielder|
    value = init_value || model.keys.sample
    loop do
      yielder.yield value
      value = model[value].sample || break
    end
  end
end
markov_enumerator = build_markov_enumerator(model)
~~~

Each time the enumerator runs, it takes another step in the random path of
connected words. Making this process into an enumarator gives an easy method
to repeat this process as many times as desired.

~~~ruby
markov_enumerator.take(5)
=> ["Fitzwater", "shall", "rue", "with", "plagues"]
markov_enumerator.take(10)
=> ["tempts", "and", "wind", "Yourself", "held", "them", "but", "what", "say", "nay"]
~~~

Generating a desired number of tokens and splitting them into lines gives us
a poem. (Or at least this model's best approximation of one!)

~~~ruby
def generate_lines(enumerator, line_length=8, lines=10, join_char=' ')
  enumerator.take(line_length*lines)
    .each_slice(line_length)
    .map{ |line| line.join(join_char) }
    .join("\n")
end
~~~

Here's an example of the output:

> cuique is this of our esteem Yet I  
> have by my wife a canary with you  
> Brutus as it and loud trumpet Let it  
> As he did I proceeded Or else imputation  
> of cheer for the watery kingdom Which as  
> it soon out than his son of moan  
> What black for with the realm of mine  
> own rede Laer A very lips Where Caesar  
> O notable shame you Why beg of your  
> own impatience hath lost the soldiers With pestilent  

Though the Shakespearian vocabulary adds a touch of interest, this hardly does
better drawing words from a bag. Remember, the theory behind this model is that
we can guarantee that any pair of words in our generated poem must be possible
in English because we've actually seen it somewhere in the corpus used to
generate the model. Clearly, however, this isn't sufficient. After a few words
phrases generated by this model tend slide out of coherence. Syntax demands
more than the pairwise matches this model offers.

But let's not give in! With a little refactoring, we can build a model that
does a lot better. If matching single tokens with those that can follow
guarantees that any two words in our poem are attested in our Shakespeare
corpus, matching pairs of tokens with pairs that can follow guarantees that
any four words in a generated poem are attested in the corpus.

These token pairs I use as the base of the next model are called 'bigrams'.
Actually, while I'm at it, I'll generalize the method to use 'ngrams', that is,
groups of tokens of a specified length *n*.

~~~ruby
def build_ngram_markov_transition_model(tokens, n=2, join_char='%', model={})
  model.default = []
  (0 .. tokens.length - 2*n).each do |i|
    puts "#{i} of #{tokens.length}"
    this_ngram = tokens[i .. i+n-1]
    next_ngram = tokens[i+n .. i+(2*n)-1]
    model[this_ngram.join(join_char)] = model[this_ngram.join(join_char)]
      .dup
      .push(next_ngram.join(join_char))
  end
  model
end
model = build_ngram_markov_transition_model(tokens)
~~~

Now instead of looking at all the tokens that follow 'the', for example, I can
look at all the bigrams that follow a pair like 'the woods'. Note that I'm
using the `%` character to squish my ngrams together so each can be represented
as a single string. This is a little inelegant, but keeps the data structure
simple.

~~~ruby
model['the%wood']
=> ["s%boldness", "no%assembly", "our%wizards", "a%league", "where%often", "will%he", "HELENA%Ay", "Enter%TITANIA", "And%to" "Enter%OBERON", "go%swifter", "Seeking%sweet", "I%measuring", "There%is"]
~~~

I also adjust the `generate_lines` so it can clean up these `%` delimiters
before outputting the generated text.

~~~ruby
def generate_lines(enumerator, line_ngram_length=4, n=2, lines=10, join_char=' ', split_char='%')
  .map{ |ngram| ngram.split(split_char) }
  .flatten
  .each_slice(line_ngram_length*n)
  .map{ |line| line.join(join_char) }
  .join("\n")
~~~

Here's a poem generated with this bigram Markov model:

> commandment I will run mad So much she  
> doteth on her Mortimer Exit Mort Fie cousin  
> Percy how you cross my father and all  
> That from my mutest conscience to my tongue  
> may utter forth The forms of things unknown  
> the poet here describes By nature prov d  
> worth a welcome lord The Earl of Salisbury  
> and Warwick We thank you lords My title  
> s good that s gone made himself much  
> sport out of him I gather he is  

The output is still quite rough, but using some discretion in punctuation and
capitalization, I find a degree of syntactical and semantic coherence that,
though imperfect, is notable for such a simple model:

> Commandment, I will run mad! So much she  
> Doteth on her Mortimer, Exit Mort. Fie cousin  
> Percy, how you cross my father and all  
> That from my mutest conscience to my tongue  
> May utter forth. The forms of things unknown,  
> The poet here describes by nature prov'd.  
> Worth a welcome, lord The Earl of Salisbury  
> And Warwick, we thank you lords. My title's  
> Good that's gone made himself much  
> sport out of him I gather he is.  

Some of this modest success, however, is a little deceptive. The larger the
ngrams are the fewer possible followers I'll find for each.  This can lead our
model to lift chunks of text wholesale from the initial corpus. For example,
the well-formed line "That from my mutest conscience to my tongue" is actually
a full line spoken by Iachimo in the play *Cymbeline*.

In my next post, I'll discuss some ways to address this problem. I'll also
explore some of the more compelling possibilities the Markov chain method
offers, such as building corpora from multiple authors.
