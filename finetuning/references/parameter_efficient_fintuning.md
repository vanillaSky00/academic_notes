0:01
One of the most popular ways to fine-tune models is actually through parameter-efficient
0:04
fine-tuning techniques. Let's take a look. So fine-tuning your whole model is pretty expensive.
0:10
You're updating all the weights, which means you need an extra amount of memory to hold
0:15
the gradients in the GPU. That's double the memory. And then there's actually states for
0:19
your optimizer to store as well. So you're looking at two to three times memory of the
0:23
model itself just to fine-tune. You also need more compute to calculate those gradients,
0:28
and that translates to just a lot more money overall to run all of that.
0:31
So if you need multiple fine-tuned models for different tasks like you see here,
0:35
you're actually going to need to run them on different GPUs, right? They're all very big,
0:40
and they can't fit maybe on a single GPU. It might be three separate GPUs here. So each model is not
0:45
very easily portable. It's hard to move between the different servers. So is there a better way?
0:51
Well, actually, yes. And it comes from a really, really interesting finding. It turns out that
0:56
the change in the LLM weights, that delta W you see there, during fine-tuning actually have a lot
1:02
of redundancy. So this doesn't mean the LLM weights themselves have a lot of redundancy. It's actually
1:07
just the weight updates or the matrices to update the weights based on the gradient and based on the
1:11
optimizer. So a lot of the updates from new fine-tuning data that are applied to the weights
1:17
are basically uninformative and don't have to happen. There's more noise and signal.
1:21
Maybe to put it another way, you can represent the LLM changes during fine-tuning pretty accurately
1:26
with a lot fewer parameters in that weight update matrix. So it's if you represented it with more
1:32
signal and then you throw out the noise. To visualize this empirically, you can actually
1:36
take that weight update matrix and do singular value decomposition or SVD on it. And you can
1:42
see that most of the information can actually be represented in the first few singular values. And
1:47
that means the remaining directions contribute very little and can often be approximated away.
1:53
So in linear algebra, that means the weight update matrix can be approximated as, quote,
1:58
low rank. So a much smaller matrix. And that's a huge savings on parameters. And what that means
2:03
is that the updates can actually be smaller and the base model can just actually remain frozen.
2:08
And when trained, the smaller update weights, they're often called adapters.
2:11
More intuitively, this basically means that updates to the weights during fine-tuning can
2:16
be broader strokes rather than fine-grained. And, you know, maybe it should be called
2:20
broad-tuning rather than fine-tuning. Okay, so here's the analogy. On the left,
2:26
you have multiple individual fine-tuned models. But on the right, you get multiple adapters,
2:31
and they can share that same base model. Again, huge savings on compute and storage.
2:35
Adapters are lightweight to fit multiple on the same GPU. There's also, in terms of inference
2:41
latency, if you're running them all on the same GPU, you can swap adapters, not just full models.
2:46
Okay, so since the savings are so good, how does actually getting to that low rank matrix work?
2:51
So it's just some basic matrix math called rank decomposition. It's essentially lossy
2:56
compression or a way to denoise the matrix and only keep the parts that are high signal.
3:01
And a large matrix, for example, here could be 1000 by 1000 with a million parameters total,
3:06
but it can actually be decomposed into these two smaller matrices with rank two, which reduces it
3:11
to only 4000 parameters. And that's insane, because that's a 250x fewer parameters to train
3:17
and store in memory. You can generalize the two to just R, right? So that's just the rank. R is
3:22
a hyperparameter, of course, you can use for the max rank of the decomposed matrices. Ideally,
3:28
the rank is much less than your original matrix dimensions, right? If it's the same, then you're
3:32
back to actually full fine-tuning. R equals four is often called a good starting point on the
3:37
original LoRA paper, but realistically, it really changes per task. What's really crazy is a rank of
3:43
one will also work, and even crazier for reinforcement learning, that rank of one often
3:49
has shown to work well. Okay, you also don't need to have the same rank for every single
3:54
LoRA that you put in, but that's research for another time. And as all hyperparameters,
3:58
you find the right R empirically based on how many LoRAs, where you're putting the LoRAs,
4:02
and most importantly, your data size. So smaller data sets and changes are probably able to get
4:06
away with smaller R's, and larger updates probably need larger R's. Your other hyperparameters will
4:11
change too, for example, using a 10x higher learning rate than full fine-tuning is often
4:16
recommended for LoRAs. So needless to say, the benefits are pretty clear, and the impact
4:21
accuracy pretty minor when you consider how you get to a better model is through iteration and
4:25
smaller, faster updates, and this will get you faster time to accuracy anyways. And you can
4:30
always dial up the R if your task is too big and requires a major update later. So now you know
4:35
what low rank decomposition is, here's how it's actually implemented. So zooming into one weight
4:40
matrix here, the matrix gets an input x, it's modified by all the layers, and the output is
4:45
this hidden state which is then processed by subsequent layers, calculate the loss, and backprop.
4:49
Okay, so this is just regular full fine-tuning. At some point, the weights of this matrix gets
4:54
updated. In full fine-tuning, all weights are basically changed, so that delta w is the size
4:59
of all the weights. So now to understand what's going on in LoRA, you can move that out separately
5:04
and visualize that the full weights of delta w you can actually approximate with those LoRA matrices.
5:09
These are dot product together to create a delta w matrix, which then takes the input x and outputs
5:16
that hidden state just the same. The rest is the same, but what's really interesting is your main
5:20
weights are all frozen, and when back prop actually happens, you only do it through that delta w, right.
5:26
You only do it through your LoRA adapters, and that saves you so much here. Okay, so maybe
5:32
you get how LoRA adapters work now, but where is it actually happening in the model? So if you look
5:38
into a standard LLM architecture, you can see decoder blocks, and there you can see feed-forward
5:42
layers and self-attention layers, and the original paper for LoRA looks at adding adapters to this
5:48
self-attention block. The recent work has also shown to apply LoRA on all layers, not just
5:53
attention, so you can also experiment with what works best for you. But specifically, the LoRA
5:58
paper looked at adding LoRA on the query and value matrices visualized here, and the remaining
6:04
weights are frozen. Okay, so you know where LoRA goes. If you're curious about digging more into
6:09
LoRA, you'll often see the LoRA diagram represented by these trapezoids, like in the original paper.
6:15
It's to signal that the matrices actually get smaller rank, but technically the matrices
6:18
should be more accurately depicted as what you saw previously with the matrix rectangles. But
6:23
anyways, using this diagram, you can imagine having LoRA adapters for a task on custom code
6:29
getting generated, or different adapters on custom unit tasks. And it's shown for all fine-tuning
6:35
tasks here. Keep in mind that you can also update weights of a model in RL using LoRA as well.
6:40
All right, one more important hyperparameter of LoRA that matters is alpha. Alpha scales how much
6:45
LoRAs matter versus the original weights when updating the weights, and empirically you need
6:50
to increase this when the rank increases. And the default is 1. Again, it's something to tune here.
6:55
All right, so you heard all this talk about saving GPU memory. It's time to compare. So,
7:00
the top is regular full fine-tuning. The bottom is LoRAs. First, you have to fit the whole base
7:05
model into the GPU memory no matter what, so that's the same. Second, you want to add space
7:10
for your LoRA adapters. It's usually way smaller than what's shown here, but this is just one case
7:14
here just to show and visualize. Then you need memory for your gradients. For the full model,
7:20
it's all of it. And for LoRAs, it can be tiny, way tinier than what's shown here, right. Even
7:24
that 0.1 percent. But here's kind of a fat LoRA at 25 percent. The next piece depends on the optimizer,
7:30
but essentially the optimizer state, which depends on the gradient memory. Finally, the forward pass
7:35
will need a little bit more for LoRA, especially if you expect to hot swap them. The LoRAs can
7:39
also be fused back into the model for the same computational efficiency as just a regular forward
7:44
pass like that. So, clearly, LoRAs actually enable you to save a lot here. This is a very
7:50
generous depiction for full fine-tuning. LoRAs can actually save you significantly more than this.
7:55
So, now that you understand LoRAs, go build with them. There are a ton of open-source frameworks
8:01
to help you. The pros are that you can get started very quickly, sometimes even locally.
8:07
And typically, LoRAs are the way to go for getting started with fine-tuning anyways.
8:11
The cons of LoRAs is the hyperparameter tuning. There are fewer good defaults out there,
8:16
although that's changing than regular full fine-tuning. And local training is typically
8:21
only on smaller models, of course. So, here's what it looks like in Hugging Phase Transformers and
8:26
their PEFT library, or Parameter Efficient Fine-Tuning library. You can see the rank R and
8:30
alpha hyperparameters here. You can also see the query and value being where the LoRAs are getting
8:35
attached. You can add your own. And all you need to do is take any model, then wrap LoRA model
8:40
around it with a LoRA config. LoRAs are a part of a much, much broader set of Parameter Efficient
8:46
Fine-Tuning or PEFT techniques that make fine-tuning or just generally updating LLMs much
8:50
more efficient. And that's both during and after training, actually. So, more often, you'll see it
8:54
for fine-tuning, but it's also getting used in RL, which you'll learn about next. Switching gears to
8:59
reinforcement learning a bit, you'll learn how rewards inform LLM updates instead of that target
9:05
output in fine-tuning.