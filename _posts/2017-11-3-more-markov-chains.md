---
layout: post
title: 'generative poetics episode 3: more markov chains'
comments: true
---

In the last episode, I introced a Markov chain model. Markov chains are a
simple way to generate text that's more linguistically realistic than a
draw-from-a-bag model. By following the patterns of which words follow which
words in a text, it's possible to mimic much of linguistic structure without
dealing with complex hierarchical models of grammar.

When I left off last time, I raised the issue that a Markov chain can easily
regurgitate chunks of its source text. My goal is to generate novel poetry, so
I need a way to identify and minimize this kind of verbatim repititon.

To help illustrate the problem and demonstrate how it can be addressed, I add a
method to write the structure of the Markov model to a graph description
language called DOT. As an example, I use a model built from Shakespeare's
first sonnet:

>  From fairest creatures we desire increase,  
>  That thereby beauty's rose might never die,  
>  But as the riper should by time decease,  
>  His tender heir might bear his memory:  
>  But thou contracted to thine own bright eyes,  
>  Feed'st thy light's flame with self-substantial fuel,  
>  Making a famine where abundance lies,  
>  Thy self thy foe, to thy sweet self too cruel:  
>  Thou that art now the world's fresh ornament,  
>  And only herald to the gaudy spring,  
>  Within thine own bud buriest thy content,  
>  And tender churl mak'st waste in niggarding:  
>    Pity the world, or else this glutton be,  
>    To eat the world's due, by the grave and thee.  

Since the last episode, I've done a lot of work to clean up and structure the
code for this project. I've created a new MarkovModel class that makes it more
convenient to create and work with the Markov chain models I use in this
project. To create a new model from a source file, I call the `new_from_file`
class method. This does all the work of reading the file, parsing it into
tokens consisting of a single word or punctuation mark, sorting those tokens
into ngrams of the desired size (in this case, bigrams), and building a Markov
chain model from those ngrams.

```ruby
sonnet_one_model = MarkovModel.new_from_file('./sources/sonnet_one.txt')
```

Next, I use my `export_dot` method to convert the model's Hash structure to the
DOT format and save it to a file.

```ruby
sonnet_one_model.export_dot('sonnet_one.dot')
```

With this file created, I use the `dot` commandline tool to generate a PNG
image representation of the model's underlying graph structure.

![sonnet_one](/assets/sonnet_one.png)

Looking at this structure, you may be able to guess what will happen when I
generate text from the model. I've also created `MarkovEnumerator` and
`EdgeRenderer` classes to make generating and rendering text from models more
convenient. Before generating text, I initialize instances of these classes,
passing in my `sonnet_one_model` as an argument,

```ruby
renderer = EdgeRenderer.new(sonnet_one_model)
enum = MarkovEnumerator.new(sonnet_one_model)
```

With the enumerator and renderer initialized, I call the enumerator's `take`
method to generate the desired number of ngrams, and pass those ngrams to the
renderer to turn them back into text I can easily display.

```ruby
puts renderer.render_ngrams(enum.take(30)).join(" ")
```

I've added linebreaks back to the result to make it easier to read.

> creatures we desire increase ,  
> That thereby beauty's rose might never die ,  
> But as the riper should by time decease ,  
> His tender heir might bear his memory :  
> But thou contracted to thine own bud buriest thy content ,  
> And tender churl mak'st waste in niggarding :  
> Pity the world , or else this glutton be ,  
> To  

Most of what the model gives us here is directly from Shakespeare's original.
One line, however, diverges:

> But thou contracted to thine own bud buriest thy content ,

Looking back at the graph image, it's clear what happened here. While much of
the graph generated from the poem follows a single path, there's a split at
"thine own." The bigram can be followed either by "bright eyes," or "bud
buriest." In the original, the instance of "thine own" on the line beginning
with "But though contracted to" is followed by "bright eyes," but here our
enumerator chose a different path with "bud buriest."

![sonnet_one_thine_own](/assets/sonnet_one_thine_own.png)

Currently, there isn't enough information in the model to distinguish between
the path taken in the original and the divergent path. I had to look back at
the original poem to confirm that the divergence followed "thine own." To
determine which parts of a generated text are novel and which are lifted
directly from the source, the first step is to add some kind of metadata to
the model that will let me determine which part of the source text the
connection between two ngrams came from.

To store this metadata, I add an instance variable called `@edge_properties` 
to the `MarkovModel` class. In the context of a graph data structure, an edge
is the connection between two nodes. In the case of the Markov chain model,
each ngram is a node. These are shown by the ovals in the DOT visualization.
The connections between these nodes that indicates which ngrams can follow
which are the edges. In the visualization, the edges are shown as arrows
connecting the oval nodes.

I need to label each of these edges with an array of indices representing the
positions of matching ngram pairs in the source text. Beause I want to
support models built from multiple sources, I separate these index lists by
source. To store this data and provide some helper methods to manage it
effeciently, I've created an `EdgeLabel` class.

The `MarkovModel`'s `@edge_properties` variable, then, is a hash of ngram
pairs, each pointing to a corresponding `EdgeLabel`.

For example, the label for the edge corresponding to the ngram pair
`[["thine", "own"], ["bud", "buriest"]]` looks like this:

```ruby
model.edge_properties[[["thine", "own"], ["bud", "buriest"]]]
=> #<EdgeLabel:0x007ffdc4070020 @sources={"sonnet_one.txt"=>{:indices=>[86], :weight=>1.0}}>
```

Here, `:indices=>[86]` tells us that the string "thine own bud buriest" occurs
only at the 86th token of the parsed source text.

This makes it easy to see that `[["thine", "own"], ["bright", "eyes"]]` comes
from a different place earlier in the source text:

```ruby
model.edge_properties[[["thine", "own"], ["bright", "eyes"]]]
=> #<EdgeLabel:0x007ffdc4e31830 @sources={"sonnet_one.txt"=>{:indices=>[36], :weight=>1.0}}>
```

Adding these labels to the DOT graph visualization lets me see the same
information in a graphical format.

![sonnet_one_labeled](/assets/sonnet_one_labeled.png)

Now that this index data is included in the model, it's possible to mark points
in the generated text where the sequence of ngrams diverges from the original
sequence in the source text. In the example below, the renderer uses a double
forward-slash (`//`) to mark the point of divergence from the original
sequence.

> And only herald to the gaudy spring ,  
> Within thine own // bright eyes ,  
> Feed'st thy light's flame with self-substantial fuel ,  
> Making a famine where abundance lies ,  

Of course, with such a small corpus, the frequency of this kind of disjuncture
is very low. However, returning to the full Shakespeare corpus yields a better
result.

```ruby
model = MarkovModel.new_from_file_list(['./sources/shakespeare.txt'], {tokenizer: lambda{ |string| string.scan(/[\n\w'-]+|[[:punct:]]+/) }})
enum = MarkovEnumerator.new(model)
renderer = EdgeRenderer.new(model)
puts renderer.render_ngrams(enum.take(200), show_indices=false).join(" ")
``` 

> MENENIUS // . He wrought better // that made the slaughter // is  
> not without // a storm ; quaff'd // off the great Cham's // beard ; and to // him put me to // be no further harmful // than in all  
> // Thy by-gone fooleries were but spices of it // .  
> Do you // speak in print , // for in her ray and brightness  
> The // worms were hallow'd that did breed the silk // ,  
> Then deputy // of Ireland , your // ink to blood , // a pin ,  
> // squints the eye , // heaven in my young // rover , he's  
> // so full of songs // , sirrah ,  
> // Sword , hold thy // tongue .  
> I // have broke mine eyestrings // , crack'd  
> As // aconitum or rash gunpowder // .  
> This gentle and unforc'd accord of // Hamlet  
> ( For so this side of our known world esteem'd him )  
> Did // run pure blood , // but a patch'd fool // , if thou thy // sins enclose !  
> // O most happy hour // ;  
> Fire burn and cauldron bubble . //  
> ALL . Thanks // , sweet Kate , // I  
> believe that // these applauses are  
> // That war , or // shall they be divided // ? Must I  
> // desire you of the // field  

In the next post in this series, I'll expand on this to create a weighted
model that prefers selecting disjunctures when they're available. This method
for weighted selection will pave the way for some much more interesting ways
to expand on the Markov chain model, such as mixing corpora of different
authors, and adding preferential weighting for poetic characteristics such
as rhyme and meter.
