One way to address this problem is to restrict the model to filter out ngrams
that have only one possible following ngram in the corpus. This way, there will
always be at least two possibilities for the model to choose from.

~~~ruby
filtered_model = model.select{ |ngram, transitions| transitions.size >= 2}
~~~

I'll still have, at worst, a 50% chance that any six words in the poem are
lifted directly from Shakespeare, a 25% chance that any eight words words are,
etc.

By removing ngrams that don't have rich enough lists of followers, I'm also
introducing dead-ends into the model. Say, for example, "my mutest" can only be
followed "conscience to". This single possible follower disqualifies it from
the model. If I then remove "my mutest" from the model, I'll also
have to remove it from the list of ngrams that can follow "that from."

Following this process back up the chain, if "that from" formerly had two
possibilities, it now only has one, and I have to repeat the process here
as well.

This means I have to repeat my culling process iteratively until no followers
lists smaller than one exist in the model
