0:01
The way a model consumes text is through something called tokens, which are these
0:05
different pieces of text that are very efficiently encoded. In post-training, what's important to
0:11
know is that these tokenizers, basically these modules that are able to convert text into these
0:18
byte-size encodings, they are trained with the model, and as a result you sometimes need to
0:23
freeze or unfreeze them to get the right result. So you have your text, how do you actually encode
0:29
it into numbers efficiently for the model to actually read and process it, and then output
0:34
text? So you could encode text like people do, you could use words like in a dictionary,
0:40
or you can use characters like in an alphabet. But are there more efficient ways to encode text?
0:47
Yes, there are, and they're called tokens. You could do definitely word-level or character-level
0:51
tokens, but there are really interesting algorithms like BVE, byte-pair encoding,
0:55
that compress your training text into more efficient strings, like ING, for example. So
1:02
you can think of swimming or going, it'll repeat and reuse that ING string. And all the tokens a
1:08
model has in its vocabulary is, again, called its vocabulary, and for GBV3, it has 50,000
1:16
byte-pair encoded tokens. So its vocabulary size is about 50,000, which is enormous if you think
1:22
about it, but it's very efficient to encode it into these small pieces. Tokenization also
1:27
introduces some fun problems. So one of the most popular problems has been, count the number of
1:31
Rs in strawberry. That's a very difficult problem for language models because they're tokenized,
1:37
they operate on tokens. So the tokens might be straw and berry, and as a result, those are two
1:42
different numbers, and each of them, the model can't really see inside of a token, so it might
1:46
count only two Rs, and this has been a long-running joke in AI that the model can't handle this. And
1:52
of course, you can alleviate this by making sure that the tokens, you know, individual Rs or the
1:58
tokens will actually be able to handle that effectively. And you can see in this graph here
2:03
that when you tokenize with BPE versus tokenizing with just characters, you can actually get the
2:09
number of tokens per sentence to be very low, right? Like this is the same data set, and you
2:13
can see the distribution shift all the way to the left over there. That is showing just how efficient
2:18
BPE is at tokenizing text. So how tokens really fit in. So you have your text, you turn it into
2:24
tokens, and then the LLM is able to process those tokens and produce the next token. And then from
2:29
that token, you can also process it back into text. And of course, that next token then gets fed in
2:35
to append to that whole set of tokens going into the model to predict the next token in a loop.
2:39
What's used to encode text into tokens and then to decode tokens into text is something called a
2:45
tokenizer. So just looking at a tokenizer a bit more, you can have the text, what words are
2:51
indivisible. And you might see these tokens, right? So you might see that this whole sequence is
2:56
subdivided into different pieces. And each of those pieces is called a token, and they get mapped
3:02
onto an ID in the vocabulary. And that's just a simple lookup table. But essentially, it gets
3:07
mapped into a number. And then those numbers are then fed into the model. So the model can then
3:11
operate on those numbers using math that you'll take a look at in a sec. But essentially, the
3:16
tokenizer in code kind of looks like this. It's just tokenized, you pass in the text, and it's
3:20
able to give you these IDs. Same thing for decode, just in the opposite direction. You give it those
3:25
IDs, and you just call tokenizer.decode, and it turns it back into that text for you. Next are
3:31
embeddings. So how do the tokens actually go into the model? So they're basically semantic
3:37
representations of every single token. So when you have your token IDs, those get mapped onto the
3:42
first layer of the model that are the embeddings. And the embeddings are the size of the vocabulary.
3:46
So you can think of the token IDs as essentially indexing into this embedding matrix. And then this
3:52
embedding matrix is trained with the model to represent each token semantically initially as it
3:57
enters into the model. Next, the model will produce probabilities. So it's going to produce
4:01
these probabilities over the entire vocabulary. And then it's going to take from those probabilities
4:08
a token ID. And the simplest way is to just do some type of greedy sampling or greedy selection,
4:14
which is pick the token ID with the highest probability. And this is by default just in
4:19
the model.generate function in HuggingFace. So you don't need to specify anything. It's just
4:23
greedily picking the highest probability token, which in this example is that 0.5 token ID 0.
4:29
You can also sample from this distribution of probabilities over the vocabulary. So you could
4:34
sample the ID 0. But through sampling, you might also sample this ID 2. And when you're sampling,
4:41
you can pick essentially different things using your sampling algorithm. And what's really
4:45
interesting is you can modulate how much you want your sampling algorithm to be more random or to be
4:52
more deterministic. And by more deterministic, I mean you can actually set the temperature here,
4:57
which is a parameter. You can set it to zero, which means you don't want any variation in
5:02
terms of how you're sampling. And that actually just makes sampling the same as doing that greedy
5:06
selection. But you could also turn it up. So here is just visually showing what temperature does.
5:11
On the left, you see temperature 0.25, mostly selecting vanilla. There's a little bit of
5:16
strawberry if you look there. But essentially, it's constraining what can actually be sampled.
5:21
And then in temperature 2, which is much higher temperature, you can see that the distribution
5:26
across the different flavors is much more even. So then a model will be able to sample across
5:31
this distribution, get a little bit more randomness in terms of what it gets for its next token.
5:35
Finally, another method to get the next token ID from these probabilities over your vocabulary
5:42
is something called beam search. And what beam search does is it's able to track multiple
5:48
different sequences as you sample them. And these branches will branch off, right? Things will get a
5:53
bit different as you sample different tokens, but it's keeping track of the top beams. So the top
5:59
candidates, essentially. So for two beams, you might sample two different token IDs and you
6:04
keep the top ones in memory. And so if you were to have three beams, you might actually ultimately
6:09
end up with something like this as an output. So when a user asks, you know, write a poem,
6:14
the model might have the sun is setting soon and the sun rose above the hills
6:18
and then three roads diverge. So it might keep those three beams as the top candidates.
6:23
So we have an idea of how we go from those end probabilities that ultimately are processed out of
6:29
the model to then pick that next token ID and keep looping that for the next token. All of this can be
6:35
processed in a batch for efficiency on GPUs so that we can parallelize processes and get
6:40
outputs much faster across one text, but also many texts. So when you batch multiple different
6:48
sequences together, you might see something like this where, you know, multiple sets of token IDs
6:54
to embeddings to do the embedding lookup. But realistically, they're going to be different
6:58
lengths. So typically what's done is adding padding tokens. So these padding tokens are
7:03
basically like null tokens in there to make sure the sequences are the same length so that we can
7:09
process this efficiently on a GPU, which requires things to be put in a batch of the same size so
7:14
that we can do matrix multiplications on it and operations on it much more efficiently. And so
7:19
here's an example of taking a tokenizer and actually looking at how it's padding things. So this is
7:25
padding things ahead of, you know, prepending or prefixing the padding tokens on the sequences so
7:31
that they are the same length with the ID zero. So you saw how tokenizers work. Tokenizers often
7:37
are tied to a model or rather models tied to their tokenizers. So typically how you instantiate
7:43
it in HuggingFace is there is an AutoTokenizer feature, which basically is able to take any
7:49
model name and map it to the appropriate tokenizer it used. As you probably can tell, because these
7:54
tokenizers are associated with different numbers for vocabulary, it really matters like what
7:59
tokenizer you use for a particular model and what it was trained with. And this will become really
8:03
important for post-training. So many tokenizers, just to compare a few, you can use through the
8:08
Transformers library here. So here is the tokenizer for BERT based on case. So there's a BERT model
8:14
and you can see that the tokens produced here have this hash symbol, right? And so it does that when
8:21
it chunks up a word. So using HuggingFace is pretty manageable. So if it's kind of chunking up
8:26
and saying it's the same word, it uses that hash. Here is another tokenizer for a model called T5
8:31
small, and this uses underscores for spaces and includes the spaces in the tokens. It also
8:37
prefixes the whole sequence with an underscore as kind of a sequence symbol. And then this
8:42
tokenizer might look really weird from a DeepSeek model, but essentially uses this really strange
8:47
G character for spaces and it doesn't merge spaces into a single space, but it does group them into
8:52
single tokens. The main takeaway here is not to memorize these different tokenizers and what they
8:57
output, but essentially to see that there's a lot of variation in how these tokenizers
9:02
tokenize and choose basically substrings to represent different sequences and not to be
9:06
alarmed if you see something like that weird G character. So how does this all play into post
9:11
training? So in post training, when you're making really small changes, you can actually just get
9:15
away with freezing the embeddings. You don't need to change much. When you're keeping the same
9:19
vocabulary, for example, in your RL phase, you can freeze the tokenizer as well. You don't need to
9:23
continue training that because the vocabulary doesn't change. However, you do need to make
9:28
changes when you're making larger changes, right? You'll be inclined to do that. You'll want to
9:33
train your embeddings because those semantic representations might change. Let's say you're
9:37
training a legal LLM and it learns all this legal vocabulary and jargon and maybe there's some new
9:44
acronyms in there. And to learn that, you really, really need to continue training those embeddings
9:50
so that it can capture that semantic meaning more effectively and efficiently. When adding new terms
9:55
or special tags, you'll want to train your tokenizer because your vocabulary will change.
10:00
You might not be representing those tags very efficiently if you retrain your tokenizer,
10:05
if you keep it frozen. And so this is just something to consider. When you're changing
10:08
your vocab size, you'll also want to resize your models embedding layer because that's dependent
10:12
on the vocabulary size. And typically what people do is kind of do a warmup or train the new
10:17
embedding first and then train everything. Now that you learned how tokens are created,
10:22
it's time to get into some of the fun stuff with fine-tuning math.
