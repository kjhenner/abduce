---
layout: post
title:  'citation mining episode 1: the plan'
comments: true
---

## Can a computer tell me what I want to cite?

A better researcher rigorously defines a project's scope then diligently
consumes the topic's key, then minor, works, spurns tangents with an icy heart.
I, on the other hand, always seem to emerge from my reading tangled in
promising citations like some sea monster in fish nets and strands of kelp.

I've often fantasized about a tool that could help manage these. With a way to
efficiently store and query references, maybe I could really live in the
moment! Of course, there are entirely useful and mundane tools for this kind
of thing already available. It's way more fun, though, to build something new.

In this post and those to follow in the series, I'll talk about the background
of some of my ideas on exploring citations, then go into the actual process
of building out coding a system to experiment with these ideas. You can see the
current state of the project on [GitHub](https://github.com/kjhenner/pubmed_data).
Because the project is very much a work in progress, the code there may be
different than what I include here.

Around the time I was thinking about citation management, I discovered a paper
called "A solution to Plato's problem: the Latent Semantic Analysis theory of
acquisition, induction and representation of knowledge." (Landauer, T. K. and
Dumais, S. T., 1997) The "Plato's Problem" referred to in the title has its own
chain of citations. Plato's complaint is that the commonalities of human
knowledge and cognition seem far greater than what could be logically derived
from commonalities actual experience. This implies some inborn component these
capacities. Noam Chomksy applies this idea to linguistics: the complexity of
language a child acquires can't be fully explained by the child's actual
linguistic experiences, hence Chomsky's (1991) inference of an inborn Universal
Grammar.

While the complexity of debates over Universal Grammar seems far greater than
the actual evidence involved on either side—call this "Chomsky's
Problem"—Landauer and Dumais's solution to this "mystery of excessive learning"
caught my attention. What if there's no shortage of information for language
learners? Maybe it's all there, and we're just not looking hard enough. Though
looking at all the different contexts in which a word occurs may not provide a
language learner with enough information to accurately infer its meaning, these
contexts are not isolated.

If you'll forgive a somewhat contrived example, imagine a child learning
English who happens to never observe cooccurrence of "dog" and "tree" that can
hint any kind of relationship between dogs and trees. If, however, this child
is familiar with the propositions that "dogs bark," and the idiom of "barking
up a wrong tree," he or she may use this second order connection to infer a
possible relationship between dogs and trees: the former's potential to "bark
up" the latter.

This kind of indirect association is less concrete than a direct one, and one
easily imagine a child acquiring a confusing jumble of ideas about concept of
"barking up" and the relationship, if any, it implies between trees and dogs.
What such indirect relationship lack in specificity, however, they may make up
for in sheer quantity. The explosive nature of combinatorics means that the
number of indirect associations in language is astronomically larger than the
number of direct ones. 

The problem, then, transforms from a poverty of stimulus to an embarrassment of
riches. With so many possible associations, can one infer useful relationships?
Landauer and Dumais' paper describe a concrete algorithmic model to demonstrate
that kind of inference is technically possible.

I'll defer describing this model. For now, the point is this: If it's possible
to infer valuable information about the meaning of a word through its tangle of
indirect association, why not apply the same approach to analysis of citations?

We might expect a citation's context to be quite semantically rich, For
example, these contexts often include the author's concise statement of a topic
covered in the cited work, some further conclusion drawn from that work, and
the position of cited work in relation to other references in a similar domain.
This information may be a concise summary, or may emphasize how a paper is
recieved and used in ways that may not be apparent from the text of the
original. For instance, we could see a paper cited for its methods rather than
its results.

This brings me to my "wouldn't it be cool" proposition. With sufficiently
accurate predictions, could a trained model make useful suggestions about what
citaions should appear in a context? What if I could build a bibliography
of works I had read before beginning to write a paper, then generate a set of
suggested citations from among those works as I write? What if I could also
discover sources I had neglected to read, but which the model deemed to be
particularly relevant to a given passage?

## Vector semantics and latent semantic analysis

The model Landauer and Dumais describe is called latent semantic analysis.

The process begins with an occurrence matrix to describe each of the documents
in the broader corpus we're interested in. In this model, we build a matrix
where we have a row for each unique word across all our documents and a column
for each document. Each cell in this matrix shows the number of times a given
word occurs in a given document.

As an example, I'll take the following sentences (from Chow et. al 2012) as my
documents:

1. Recent studies revealed that RASSF1A is a substrate of Aurora and the
action of RASSF1A is regulated by Aurora A phosphorylation.

2. The level of Cyclin A was consistently lower in cells transfected with RASSF1A
siRNA than those transfected with control siRNA throughout the assay.

3. Deletion of the D-box clusters abolishes the inhibitory effect of RASSF1A
in vitro.

4. Phosphorylation of RASSF1A by Aurora kinases is essential for Cdc20-dependent
degradation.

Across these documents, I get a list of unique words. I've also removed
stopwords—that is, common words such as "a" and "the" that don't contribute to
semantic differentiation. Using these words as my rows, I can build a matrix of
document-word occurrences:

|                 | d1 | d2 | d3 | d4 |
|-----------------|----|----|----|----|
| recent          | 1  | 0  | 0  | 0  |
| studies         | 1  | 0  | 0  | 0  |
| revealed        | 1  | 0  | 0  | 0  |
| RASSF1A         | 2  | 1  | 1  | 1  |
| substrate       | 1  | 0  | 0  | 0  |
| Aurora          | 2  | 0  | 0  | 1  |
| action          | 1  | 0  | 0  | 0  |
| regulated       | 1  | 0  | 0  | 0  |
| phosphorylation | 1  | 0  | 0  | 1  |
| levels          | 1  | 0  | 0  | 0  |
| Cyclin A        | 0  | 1  | 0  | 0  |
| consistently    | 0  | 1  | 0  | 0  |
| lower           | 0  | 1  | 0  | 0  |
| cells           | 0  | 1  | 0  | 0  |
| transfected     | 0  | 2  | 0  | 0  |
| siRNA           | 0  | 2  | 0  | 0  |
| control         | 0  | 1  | 0  | 0  |
| assay           | 0  | 1  | 0  | 0  |
| deletion        | 0  | 0  | 1  | 0  |
| D-box           | 0  | 0  | 1  | 0  |
| clusters        | 0  | 0  | 1  | 0  |
| abolishes       | 0  | 0  | 1  | 0  |
| inhibitory      | 0  | 0  | 1  | 0  |
| effect          | 0  | 0  | 1  | 0  |
| vitro           | 0  | 1  | 0  | 0  |
| kinases         | 0  | 0  | 0  | 1  |
| essential       | 0  | 0  | 0  | 1  |
| cdc20-dependent | 0  | 0  | 0  | 1  |
| degredation     | 0  | 0  | 0  | 1  |  

<br> 

Even with simple model, you can see a direct connection between the articles
in the term "RASSF1a".

In addition to removing stopwords, there are a few steps I could take at this
point to clean up this data further. For example, I'd likely want to use a
stemmer to remove things like tenses and suffixes. That way I could treat
"study" and "studies" as the same word. I'd likely also want to use something
called a term frequency-inverse document frequency (tf-idf) transformation to
weight relatively rare words over more common ones. If I were looking at a
larger number of documents and "RASSF1A", occurred *only* in these four shown
in the table, I could weight it higher, letting it signify a closer connection
between the documents, while if it seemed to pop up everywhere, the tf-idf
would give it a lower weight.

To get an actual mathematical measure of document similarity with this model,
I can treat each document's column as a vector in an *n*-dimensional space,
where *n* is the number of unique words or rows in the matrix. I can then take
the cosine of the angle between any two documents to get a measure of their
similarity. If this is difficult to imagine in a high dimensional space,
picture a simpler model with only two dimensions. Let's say we were looking at
articles about pets, and only interested in the relative mentions of the words
"cat" and "dog" in our documents.

|                 | d1 | d2 | d3 |
|-----------------|----|----|----|
| cat             | 8  | 2  | 4  |
| dog             | 1  | 6  | 6  |

<br>

Because we're only working with two dimensions, these vectors can be plotted on
our familiar cartesian plane, with "cat" assigned to the *y*-axis and "dog" to
the *x*-axis:

![vectors](/assets/vectors.png)

The semantic distance between any two articles with respect to these two topics
can be seen as the angle between the vectors. Taking the cosine of this angle
then gives a value from 0 to 1, where 0 means that they share no common words
and 1 means that their proportion of words is identical.

The latent semantic analysis model builds on this by using linear algebra to
find a *low rank approximation* of this document-word matrix. That is, a way of
collapsing invividual words into a smaller number of topics while preserving
as closely as possible the angles among the documents. The key insight of
latent semantic analysis is that while this rank-lowering loses some
information, it also surfaces latent information as words that may not occur
in two documents are collapsed into topics that do. This allows the model to
capture the indirect associations among documents that may not manifest
directly in actual word co-occurrences.

My plan, then, is to apply this method to analysis to a collection of academic
articles, but rather than at articles themselves as documents, look at
collections of paragraphs that include a common reference.

![citation_context](/assets/citation_context.png)
