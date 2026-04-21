
0:00
Fine-tuning has a lot of interesting neural network math behind it.
0:04
In this lesson, you'll learn about all the different intricacies,
0:07
including calculating loss, the gradients, and also how to update your weights,
0:11
and also some of the little intricacies for fine-tuning in particular.
0:16
So first, a quick refresher on training a neural network,
0:18
because training a neural network is roughly all the same steps you need for fine-tuning.
0:22
So your model can start at any point. In this case, it's outputting garbage predictions.
0:27
You have your training data. You can calculate loss on some objective between your model's
0:31
predictions and the target. So that's how far off the model was. In this case, it's going to be a
0:36
lot. And you calculate gradients for the entire model via backpropagation. So that's understanding
0:41
how each weight in the model actually influenced that loss. You then update your weights in the
0:46
direction of minimizing the loss with some algorithms to help you make that update more
0:51
stable and effective. In fine-tuning, you're starting with a pre-trained model. So it's
0:55
already going to be better than a random model. It might output night here instead of those random
1:00
characters. And what's particular about the steps is two main things. One, the training data
1:05
specifically has an input and a target output. So there's a target again. And your loss to calculate
1:12
the difference between the model's predicted output and the target outputs, and only on those
1:17
target outputs, not the input. And the backprop and weight updates are the same. You'll see some
1:22
review here as well. So first, take a look at the training data. Lots of current examples like this
1:28
of how you want the model to behave, like what's the capital of California, Sacramento, of course in
1:32
the next modules, and you've seen a little bit of this already, You'll get to see more sophisticated
1:36
pairs, like inputs and outputs with custom tags, for example, to hook up to tools and other things.
1:41
You'll also learn how to craft your examples, which will impact the model's performance pretty
1:45
heavily, very, very heavily. And later here, you'll just assume your data has that input-target-output
1:50
paired structure. Okay, so next, you want to run your model through the inputs, get predicted
1:55
outputs, and then compute the loss on the outputs against that target output. So just an example here,
2:01
given this input, what's the capital of California? The model is going to output that distribution of
2:06
predicted tokens over its vocabulary, its entire vocabulary. So here are the token probabilities
2:11
across this model's predictions. And then it samples SF as the predicted token with greedy
2:17
sampling because it was the highest probability. This token had a 0.6 probability here. That's
2:22
actually wrong. The answer is actually Sacramento. But the probability on Sacramento is much lower,
2:27
as you see here, at 0.12. Now, the right answer is actually a distribution that looks like this.
2:33
It's one on Sacramento and zero on everything else. So it's one hot encoded target output.
2:38
So to calculate the loss on this predicted token, you want to measure how well the predicted
2:44
probability distribution, so those predicted tokens distribution, matches the true distribution,
2:49
which is that one hot distribution of target output. And you want to maximize the likelihood
2:54
or probability of the correct token, in this case, Sacramento. Thankfully, there are algorithms for
2:59
this. And one of the most popular ones is actually cross-entropy loss in language modeling, or also
3:04
known as negative log likelihood. And you can see here, cross-entropy directly optimizes the log
3:09
likelihood of the correct token. And the log part essentially penalizes the low confidence. So
3:15
instead of just telling the model to, "pick the right token", the model also needs to be as
3:21
confident as possible on that right token. So this is why LLMs don't just learn to rank tokens,
3:26
but to assign good probability distributions over that entire predicted token's distribution.
3:32
So as an example, take a look at Sacramento with this prediction. So you put 0.12 in there,
3:37
and the loss is around 2.12. Okay, so let's say the model had actually predicted the 0.68 on
3:43
Sacramento instead of SF, and it got it right. What would the loss actually be? Much lower, right? So
3:48
the loss would be 0.39. So that's just showing if it predicted correctly, it would do much, much
3:54
better. And the more correct it is, the better it is, and the lower that loss is. And this is why,
3:59
through loss, you can actually get models to learn not just how to rank these tokens, which one is
4:03
better, but to assign good probability distributions for that predicted token distribution.
4:08
Okay, so let's actually go through the entire target output to see how loss is calculated on
4:14
the output. The target output here is Sacramento is the capital, and that's the answer to what's
4:19
the capital of California. So for this loss calculation, that means there's a loss mask
4:24
on the input. So the calculations of loss on the input isn't used at all in the final loss
4:30
calculation. And that is exactly what that step had meant around only calculating loss on the
4:36
outputs. So let's compute the loss on the next token. One pretty notable thing here is we put
4:42
Sacramento in the input sequence now for the next token. And whether the model predicted Sacramento
4:47
or not, we're actually going to put the correct output token here. And this is something called
4:51
teacher forcing, because we're forcing the right output in there. And teacher forcing basically
4:55
enables really efficient training, because this is what makes these language models really
5:00
parallelizable. You can actually know what the target output is, because you basically can do
5:05
this computation in parallel. And this computation can happen with the last computation as well as
5:10
with the next computation. You don't need to wait for the model to predict something to do the next.
5:14
So this is really important, because then you can just do one forward pass of the entire model,
5:19
one big matrix multiply, and essentially that's one fused kernel for the whole loss.
5:24
So next one is, it keeps predicting the here correctly, and then capital, maybe also correctly
5:30
here, but not as much. And then it predicts a stop token, which is a good practice to know when
5:35
a sequence is complete. To essentially get all the probabilities of the outputs together, you
5:39
normally would multiply probabilities, right? That's like the probability of multiple probabilities.
5:44
But because of that log term in the cross-entropy loss, that log turns that product of probabilities
5:50
into a sum. And this makes optimization much easier by not having a product of probabilities
5:56
get too small or large for computers to handle, because it could explode when you multiply many
6:01
very large numbers or very small numbers together. And this is one of those subtler points, but
6:06
pretty deep and important points of why log likelihood is pretty much everywhere in AI.


cross entropy can refers to
https://www.youtube.com/watch?v=FjizPOgzplA&t=485s
why we want to minimize q(x) to p(x)
KL divergent (p,q entropy difference, what is the meaning)
K() indicate the current encoding method can still minmize how many bits(since the ideal p disctribution is unknow and also lower bound)
K is not distance!!

in sumarry 
entropy H(p) is the minimum information the system send
cross entropy H(p,q) is using q(x) to send the information from p the information we have to send
KL divergence is the current encoding q(x) information can still drop down how many bit

a good qustion in yt
謝謝作者的分享，有個地方想請教一下
在影片約10:40處計算 H(p,q) 的部分
因為 p(x) = [0.35, 0.40, 0.05, 0.20] 是未知的
那在 p(x) 分布未知的情況下要怎麼計算 H(p,q) = -0.35*log(0.32) - 0.40log(0.38) -... 呢？
還是應該是 -0.32*log(0.32) - 0.38log(0.38) -... 呢？
謝謝用心的影片解說~




Reply


@tony0731
3 months ago
Hi 謝謝你的回覆，在機器學習中，p(x) 會來自於我們觀察、搜集到的 data 哦！資料越多，分布就會越接近真實的 p(x)


0:03
So, next up, the goal is to get the model to actually minimize this term.
0:07
This is the loss representing what was off about the model's predictions overall, and
0:12
yes, the smaller the loss, the better.
0:14
And so that next step is exactly that.
0:16
So with that loss term, you can now calculate basically how much or what direction to update
0:21
every single weight in the model to minimize that loss.
0:24
You want to put more mass on outputting Sacramento and less on SF, and that's what Backprop does.
0:29
It will compute backwards into the model from the very last layer, closest to the final
0:35
loss calculation, to the very first one, and it's going to calculate the direction and
0:39
how much, the magnitude, each weight influences that loss.
0:43
So going through this a bit, it computes the gradients of the loss with respect to every
0:47
single weight in the model, and the gradients are telling you how to essentially change
0:53
the weights to make the LLM more likely to output the correct answer, in this case Sacramento,
0:57
and less likely on all the other ones.
0:59
So less likely on SF, less likely on Boston, LA, and the gradient is just a derivative
1:04
calculation of the loss with respect to each weight, and it's very iterative.
1:08
So you keep doing the gradient backwards through the model, and that's why it's called Backprop.
1:12
To think of it more intuitively, if you're standing on a mountain, say, you can think
1:17
of your loss as your altitude, like how high up you are, and your weights essentially as
1:22
your XY position, where you're standing, and then the gradients are your slope, right,
1:27
derivative, so like the steepness and the direction.
1:29
And so that's kind of the direction you need to go in for gradient descent.
1:33
So for cross-entropy loss, the derivative is pretty simple here, so it's just negative
1:38
log likelihood, and that's the probability of the token minus the target probability.
1:42
So if we actually do the calculation here, you can take a look at this column of gradients,
1:46
and you'll see that Sacramento, the correct one, has a negative gradient, while everything
1:50
else has that positive gradient.
1:51
Okay, you understand how much and kind of in what direction change of weights, so let's
1:55
actually update the weights themselves.
1:57
It's not as simple as just using the gradients directly.
2:00
You have your weights with respect to your output, and then let's say your last hidden
2:05
state is here.
2:06
Basically, your input transformed by your model turns into this.
2:10
This is a toy example, so the hidden state is usually larger, but it essentially represents
2:14
a semantic compression of your input, and that gets then projected and turned into a
2:20
distribution over your output token predictions over the vocabulary.
2:24
Every single token here has a row of weights that connects the hidden state, and the gradient
2:31
of the loss with respect to every single weight is the output gradient times that hidden state,
2:36
and the hidden state really scales how much to update each weight row here, so it tells
2:40
us how to adjust the weights to improve our predictions.
2:43
And then you can update your weights with this information, so gradients are ultimately
2:48
somewhat local signals, right?
2:49
You can say, like, push this weight up a bit or push this weight down a bit, but if you
2:53
updated by the full gradient from one example, the model would probably overfit to that one
2:58
example and potentially forget others, so raw gradients are often just, like, too small
3:03
or too large, and that could lead to pretty unstable learning, and that's where something
3:06
called optimizers come in.
3:08
Optimizers basically help you decide how to use these gradients to update the weights.
3:12
Here is just showing one of the simplest ones called stochastic gradient descent, SGD, and
3:17
you update each weight by subtracting your gradient scaled by some kind of learning rate
3:22
called eta, and here eta is just set at 0.1, so you can see here for Sacramento, it's scaling
3:26
that here.
3:27
In practice, LLMs are going to use more advanced optimizers, Adam or AdamW, which you'll see
3:32
kind of tricks later for.
3:34
In an LLM with billions of parameters, this update happens in parallel for all weights
3:37
using gradients averaged across your batches of data.
3:40
Now that you've updated your weights, one really cool thing is you can take a look at
3:44
the change from just a single update.
3:46
So this graph here is showing the distribution of all your weights in the model, with blue
3:51
being the old ones, orange being the new ones, and showing how the entire distribution
3:55
slightly shifts after just one update step, and the orange should get the LLM closer to
3:59
Sacramento, hopefully.
4:00
So these days, it's kind of easy, and the steps are really wrapped up into a Hugging
4:05
Face trainer class if you're using Hugging Face, so you can just specify the model, your
4:09
training arguments, your datasets, and SFT, it's called SFT trainer, SFT stands for supervised
4:14
fine-tuning, again that is the type of fine-tuning we're doing here.
4:17
In your hyperparameters for fine-tuning, it's important to specify completion only
4:21
loss equals true, and that means that you'll only be calculating loss on the outputs, which
4:26
are also called completions, typically, and not on your inputs.
4:30
And then you can just call trainer.train to kick off the training loop that just walked
4:33
through.
4:34
So that's that, but just taking a look quickly inside of trainer, I think it's important
4:37
to kind of drop down a little bit into PyTorch just to understand what's going on in there.
4:42
So inside of that .train, what it looks like is for every training epoch, so that's across
4:47
all of your dataset, your whole dataset, and for a small batch, a set of input target
4:51
output pairs, the model is going to predict some outputs, it's going to calculate loss
4:56
on the outputs versus the target outputs.
4:59
With that cross-entropy loss from before, the loss is then used in backprop to calculate
5:04
gradients for all of the weights.
5:06
And then finally, the weights are going to be updated using the gradients, and the optimizer
5:10
is going to decide what direction to take a step in and how large of a step to essentially
5:14
take.
5:15
And that cycle is repeated for the next batch, and you'll walk through all the data, in this
5:19
case, num epoch times.
5:20
So that's it.
5:22
Next up would be probably diving into those hyperparameters and how to tune them so your
5:26
model can actually effectively learn.
5:28
And that's it.
5:29
You've gone through the nitty-gritty math of how to do fine tuning.
5:32
Next up is deep diving into those hyperparameters to tune your model effectively.