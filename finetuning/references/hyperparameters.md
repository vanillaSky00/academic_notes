

0:01
Hyperparameters are really important to good, stable training.
0:05
In post-training in particular, you may need to set or tweak the hyperparameters to be different from the defaults,
0:11
so that it actually fits your task.
0:13
So, one of the most important hyperparameters is learning rate, abbreviated LR, typically, and here.
0:20
The learning rate essentially controls how much your model is updating its weights with every single step.
0:25
So, choosing a good learning rate is super important to making sure that your model actually learns efficiently and effectively.
0:32
There is such thing as learning rate being too high, and then the model's learning is unstable,
0:37
like it's overcorrecting for every single example, or it can also be too low, and it learns painfully slow.
0:43
So, just to dig into that a little bit, you can visualize what this looks like when the model tries to complete the phrase,
0:48
the cat sat on, and with a learning rate that's too high, the model's updates are pretty chaotic,
0:55
and the loss jumps all over the place, and the output can be just gibberish.
0:59
You'll often see your loss reported as NAN, or not a number, or infinity, if the learning rate is just way too high.
1:06
A perfect, good learning rate is something like this.
1:09
You see a smooth, kind of steady decrease in the loss curve.
1:12
With every epoch, the loss is going down until it converges and stops going down.
1:17
The model learns pretty efficiently and produces a coherent, correct completion like the warm blanket here.
1:23
And then finally, if the learning rate is just too low, the model barely learns, the loss decreases, but super slowly.
1:29
The output might not be complete gibberish, but it's often nonsensical because the model isn't updating its weights enough to learn the patterns in the data.
1:37
And sometimes the model can also stall its learning if it's way too low.
1:41
And so these are kind of the different cases.
1:43
You'll see all different cases in between.
1:45
It really is highly empirical to figure out what your learning rate is.
1:49
However, there are some really good defaults.
1:51
You can also dynamically adjust the learning rate during training using something called a learning rate scheduler here.
1:58
Two popular strategies are cosine annealing, which you see here on the left.
2:02
And this is where you might want to learn more quickly early on when you're further from the goal, which provides that smooth decay.
2:09
And also linear decay with a warm-up, quote, warm-up period where the learning rate essentially starts slow, increases, and then steadily decreases after that warm-up.
2:18
And intuitively, this kind of gives time to get less random gently at first for the model.
2:25
And without it, training can diverge sometimes in the first steps too quickly.
2:28
So that's the warm-up it's trying to solve for by letting the model kind of stabilize its internal statistics before full-speed learning.
2:35
These are essentially some best practices that have been developed over time.
2:39
However, I found that the new stuff doesn't always make it necessarily that much better.
2:43
So maybe this area is kind of ripe for disruption, or it just means that we found some pretty good characteristics for how these models should learn.
2:50
Beyond the learning rate scheduler there, optimizers in different frameworks like Transformers and PyTorch will provide pretty sensible default learning rates for you.
2:59
And for most fine-tuning tasks, an optimizer like AdamW with a learning rate of 5e-5 is a great starting point.
3:06
That's the default. AdamW is just a variant of the famous Adam optimizer.
3:10
It uses kind of adaptive learning rates, which means it adaptively scales updates per parameter.
3:16
So it's different per parameter in the model.
3:18
And it decouples weight decay, which is just a way to regularize the loss.
3:23
And that's by encouraging kind of smaller weights overall, so the LLM learns more stably and isn't as prone to swings for particular examples.
3:31
It basically prevents overfitting.
3:33
There are a lot of great sensible defaults.
3:35
I would just start there and then empirically experiment with what works for your use case.
3:40
Next up on different hyperparameters is number of epochs.
3:44
One epoch means the model has seen every example in your training dataset once.
3:49
In pre-training, you might have heard that LLMs train for only a single epoch because there's so much data.
3:55
In post-training, however, you're typically training for multiple epochs on your high-quality data.
3:59
So that allows your model to see your high-quality data several times and basically allow it to refine its understanding.
4:06
So you typically train for multiple epochs here.
4:08
So let's see the first time, the second time, the third time.
4:11
That's pretty straightforward.
4:12
Choosing the right number of epochs is also empirical.
4:15
So if you stop too early, for example, after one epoch, the model might be underfit, right?
4:20
So it hasn't learned enough.
4:21
Its responses will still be like its previous checkpoint maybe.
4:25
And so here's a pre-trained model here.
4:27
The sweet spot is kind of an optimal stopping point where the model generalizes well.
4:32
The model has learned patterns from your examples that generalize to new inputs without memorizing them.
4:39
Because if you train for too many epochs, which is, you know, like 15 here, the model starts outputting memorized training examples verbatim.
4:47
It basically fails completely on new text because it's just recalling.
4:51
It's not generalizing from your examples.
4:53
And it even starts to forget previous capabilities.
4:56
And this is why monitoring validation loss and more on that later and stopping at the right point is pretty crucial.
5:02
So make sure to tune this.
5:03
You can also tune this by the number of steps or batches you take across your data.
5:07
That's more fine grained than number of epochs.
5:09
And you want to mix up your data across epochs so the model sees the data in a different order.
5:14
Quick aside, there's also a concept of overtraining.
5:16
And sometimes when overtraining, you can see improvements after it stops improving and seems to be overfitting.
5:21
This is a really interesting phenomenon being studied in research today, often known as double descent.
5:27
Another key hyperparameter is batch size.
5:29
So this is the number of training examples the model processes together before updating its weights with a small batch size like four.
5:36
Say the model makes frequent updates to get through the entire data set, right?
5:40
It's like a small set every single time.
5:42
And this leads to small updates, many of them, and potentially noisier training.
5:47
In the extreme case, you can definitely train on a batch size of one.
5:50
With a larger batch size like 2048 here, the model processes many examples at once and will make basically fewer updates, but they're going to be more stable.
5:59
A smaller batch size is slower per epoch because of all the updates, but it uses less GPU memory at a time.
6:05
And this makes it a good choice when you have a limited hardware.
6:09
And a large batch size is faster per epoch, but requires a lot more GPU memory.
6:14
So here, AMD has a GPU that has huge memory called high bandwidth memory, HBM, and that allows for larger batch sizes running on a single GPU.
6:24
It's pretty ideal for training on very large data sets if you have the hardware to then support it.
6:30
In addition to just batch size and data, exactly how much memory also depends, of course, on the size of your model and how many weights you're updating in that model.
6:38
If you set the batch size too large for your GPU, your training will crash with an out-of-memory error, OOM.
6:44
The fix is straightforward, though, so reduce your batch size until it basically fits in available GPU memory.
6:50
And you might have noticed that the batch size number here are all powers of two.
6:54
That's intentional.
6:55
Okay, so that's to optimally use your GPU.
6:58
Okay, so what does this look like in your code?
7:01
So here's how these hyperparameters show up in the training args that you saw before pass into the trainer.
7:07
So you can see different batch size terms for the training and also, of course, passing that for the eval and test sets.



0:01
So how do you know the choices you're making are actually working?
0:05
So the key is to monitor the model's learning progress, and you learned about loss and monitoring it earlier,
0:10
but when training LLMs, monitoring the loss curves is pretty crucial.
0:16
It's the central point for understanding model performance, and this helps us tune the hyperparameters
0:21
and empirically know what's off and when things are going well.
0:24
It's basically the dark arts of AI training.
0:27
So you typically track two very important metrics.
0:30
One is training loss and one is validation loss.
0:33
Training loss is the error on the data the model is currently trained on,
0:37
and validation is on that separate held-out set of data that the model hasn't seen before,
0:42
but you use to see if the model is generalizing to that new set.
0:46
And it's specifically for tuning hyperparameters to optimize that validation loss.
0:51
You want to make sure that all your hyperparameters are tuned to make validation loss appropriate and low as possible.
0:57
This is not your eval or test set, which is also held-out and not going to be used to tune your hyperparameters
1:03
so that there's another set held-out.
1:05
Okay, so as training progresses, we basically expect the model's outputs on the validation set to improve
1:10
and get more coherent, correct as the loss goes down.
1:13
So here, as the loss goes down, you'll see checkpoints from the model to assess how it performs on both training and validation losses.
1:20
For example, when you ask the model, you know, explain quantum computing briefly,
1:24
which is not present in the training data, you can see the progression of outputs getting better,
1:28
and early in training, responses are maybe incomplete or repetitive or whatever the pre-trained model is.
1:33
And as the model learns, the outputs become more coherent and comprehensive.
1:38
So plotting these two loss curves will tell you a lot,
1:42
though you'll see in later videos your test set or evals will be potentially even more crucial for post-training success.
1:48
But getting the model to learn stably is a given, table stakes.
1:52
So here, in a good training run, both losses will decrease and converge.
1:56
If the validation loss starts to increase while the training loss continues to decrease,
2:00
that's actually a clear sign of overfitting.
2:03
The model's memorizing the training data and losing its ability to generalize to the validation set.
2:08
And if both losses plateau at a high value, the model is underfitting.
2:12
It's not complex enough or hasn't trained long enough.
2:16
I will say that, you know, small note that in research,
2:19
we are finding that actually the overfitting line isn't as bad and actually gets the model to do better sometimes.
2:26
And we call it often a double descent.
2:28
So it will go down, go back up again and down again.
2:31
But that is still in research in terms of understanding these models.
2:34
Here are some realistic plots, more realistic than the ones that you just saw there,
2:38
that you'll be plotting directly in your lab.
2:40
The left is underfit and too high, and the right one is converging nicely.
2:44
So note that the smoothness is just not perfect here.
2:46
And that depends on how often you're plotting the loss and how often you're checkpointing the model
2:50
to run on the validation set and get the validation loss.
2:53
Great. OK, so now you can start hyperparameter tuning.
2:56
It's super empirical. Every single hyperparameter is pretty empirical.
2:59
So one of the biggest tips is reproducibility.
3:02
So you know whether your experiment was actually successful or not.
3:05
And so you know whether to scale your new finding or technique or if it's just random luck.
3:09
And random luck can happen when training a lot.
3:12
So on randomness, maybe you run kind of two different experiments here,
3:15
and you get two different accuracies, 86% and 91%.
3:19
Without setting any kind of random seeds, these could just be variants from your runs.
3:23
Random seeds are a way to control randomness in training,
3:26
like weight initialization and data shuffling.
3:28
And controlling that randomness is really important for debugging,
3:32
for comparing experiments, for just reproducibility, scientific reproducibility.
3:37
So here's how you can set the same seed.
3:40
That means hopefully getting ideally identical results across runs for these frameworks.
3:44
And please note that there's still other randomness lower down in the stack
3:48
that you might not be able to control directly with Python, NumPy, or PyTorch,
3:51
kind of at the kernel level that runs on the GPU.
3:54
You won't learn this here, but there are some super cool papers about determinism in LLM training
3:59
to make it more reproducible if you want to dig more into that research.
4:02
Now that you've learned how to tune hyperparameters for good, stable training,
4:07
next up, you are going to take a look at how you train more efficiently
4:12
with parameter-efficient fine-tuning.