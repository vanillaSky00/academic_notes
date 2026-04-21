0:02
The goal of training and reinforcement learning is to maximize the reward.
0:05
But how do you actually define the training objective?
0:08
Take a look in this next video.
0:10
So first, if you've seen ChatGPT, you've likely seen this interface where you get two responses,
0:16
response A versus B, and it's asking for your preference.
0:19
This is the first step of RLHF, or reinforcement learning with human feedback,
0:23
and it's collecting that human feedback data.
0:26
RLHF is the original flagship process that ChatGPT used, so time to take a closer look at it.
0:32
You've seen this before.
0:33
So essentially, RLHF uses the reward model that was learned using preference learning
0:38
from those rankings of a human labeler.
0:41
But then what happens after that?
0:42
So what happens after that is you might sample an input, the LLM might generate an output,
0:47
and then the reward model applies reward to that output, getting that trajectory.
0:51
And then that reward is used in an RL training objective to update the LLM.
0:55
How exactly does that happen, though?
0:57
How is that reward actually used?
0:59
And how do you get that RL training objective?
1:01
So the goal is to backprop the reward information to update the model so the model can maximize the reward.
1:08
But the main problem you have here is the reward is not directly differentiable.
1:13
You can't naively backprop.
1:14
The reward is not in the gradient.
1:16
So if the reward is coming from verifiers, there's no weights to backprop through.
1:21
In this case, you do have weights with a reward model,
1:23
but the reward is still not a differentiable function of the LLM output.
1:27
So how do you actually nudge the model so that it produces outputs that can get a higher reward?
1:31
Even if it were possible to nudge things a little bit more, the LLM output is a sample,
1:36
which is passed through to the reward function, whether it's a reward model or verifiers,
1:40
and backpropping from the reward model weights through a sampling operation to get the LLM weights is tricky.
1:46
So instead, your training objective can be to somehow increase the probability of generating tokens that led to a high reward
1:54
and decrease the probability of tokens that led to a low reward.
1:57
So you can basically weight your next token probabilities for an output by the reward on that output.
2:02
And that's exactly how to do it.
2:04
So to recap, there's this loop, right?
2:06
So you're getting data that's the same as before, and then you're doing training.
2:09
And in training, you want to define that training objective.
2:11
And that's what you'll go into here.
2:13
So again, you get your rollouts, you apply reward, you get your trajectories.
2:17
In RLHF, you're training a reward model with preference learning to apply that reward.
2:21
And then you train your LLM on those trajectories with RL.
2:25
And in here, you're defining that training objective using rollouts to backprop reward info.
2:30
And then you update your LLM with your training objective to maximize that reward.
2:33
And you loop.
2:34
So let's dig into that training objective.
2:37
So you're weighting the LLM output with the reward.
2:40
So just to get a tad more technical, you're weighting the probability of an output token inside of that LLM output
2:46
with kind of how good that token is, which is going to be based on that reward.
2:50
Looking more closely at this first term, probability of next token, this part is pretty straightforward.
2:56
It comes from your LLM since its output probabilities for the next token.
3:00
And theoretically, this is fine.
3:02
But actually, practically, this is not the case for compute reasons.
3:05
It's actually a bit unstable given how you're actually updating the LLM and collecting data.
3:10
It's worth looking into here.
3:12
So practically, first, the LLM will read all the input prompts, generate answers, outputs, and get rewards for each one.
3:19
So you get this data set of trajectories.
3:21
This phase is typically, quote, offline, meaning it's done on its own without a live updating model.
3:27
And you generate a bunch of outputs or rollouts.
3:29
And you use your reward model to score a bunch of those.
3:32
So you get your trajectories.
3:33
And you can pretty efficiently generate this data set with large batches, right?
3:37
All you're doing is running inference and then running your reward model inference on top of those rollouts.
3:42
In phase two, you typically are then training on the static data set of trajectories.
3:47
And the objective is still this at the bottom.
3:49
So it's to maximize the probability of a given token weighted by the reward or how good it was.
3:54
And you're typically looping over in much smaller batches than in the data collection in phase one.
3:59
And this is because of memory and compute intensiveness of training.
4:02
And that's why practically it's usually two phases.
4:05
And you'll see soon just how computationally expensive and intensive RL training is because it has so many models.
4:10
So you collect your data in phase one.
4:12
That's generated by the model as it existed at that time.
4:15
But in phase two, you are constantly updating the model.
4:18
The model is changing with every single batch.
4:20
And this means the probabilities it would assign now are different from the ones that actually generated the data.
4:26
This creates a mismatch and makes the training process very unstable.
4:30
The solution to this is to use a quote reference model.
4:33
So you save a copy of the original pre-update model LLMref here.
4:37
And that's the data collection model.
4:39
Instead of using the current model's probabilities, you'll use the ratio of the current model's probability relative to the reference model's probability.
4:45
And intuitively, this ratio just tells you how much the model is changing and how it's changing.
4:51
If the ratio is greater than one, it means your new model is more likely to produce this token versus a reference model.
4:56
If it's less than one, it's less likely.
4:58
And this stabilizes training by anchoring the updates to a fixed reference point.
5:01
And stability is going to be a recurring theme in RL training since it's so unstable most of the time.
5:08
So now you've got your rollouts from a reference LLM.
5:10
That's where it comes in for that data phase.
5:12
Now take a look at the other half of this equation.
5:15
It's the reward term, a.k.a. how good the token was.
5:18
So using the raw reward R works, but it can be inefficient and cause large noisy gradients, which causes swings in training.
5:25
Also, it's unclear how to distribute credit for the reward across tokens.
5:29
The reward is full sequence level, not token level.
5:31
And it might be high variance to penalize a token that didn't negatively harm the resulting reward and vice versa.
5:38
Anyways, this is the reason why you estimate a baseline for every token, which is how you expected the model to do, what reward you expected for a particular token.
5:48
And then you backprop on how much it exceeded or didn't meet expectations.
5:52
This also keeps learning more stable.
5:54
So imagine if all your rewards were positive, the model would try to increase the probability of all tokens all the time, just a little bit more on others.
6:01
Learning is slow because it's kind of hard to distinguish across them.
6:04
And same if all the rewards are negative.
6:06
It's the best practice to have rewards clustered at zero.
6:09
So you have a mix of positive and negative rewards.
6:12
There is definitely some research challenging this.
6:14
So please try it for yourself.
6:16
This here is just doing it how RLHF originally did it.
6:19
So the way to center rewards around zero is to introduce some type of baseline and subtract that baseline from the reward to kind of center it.
6:27
And again, this baseline is kind of the average or expected reward for every single token.
6:31
So the centered reward you get are how much token succeeded or didn't meet expectations.
6:36
In RL literature, this is typically called the value function.
6:39
And the calculation here is typically called the advantage.
6:43
And the advantage is how good your token was.
6:45
So now if you go back and look at subtracting a baseline from the reward to get the resulting value called the advantage, the advantage is positive.
6:54
If a response was better than average and negative, it was worse than average or expected.
6:58
And this gives you a much clearer signal to the model about what to do.
7:02
So increased probability for positive advantage tokens decrease for negative.
7:06
And note that there are many ways to compute advantage.
7:09
And you'll explore another way in the next video.
7:11
So it's a simple trick to center rewards around zero by estimating expected rewards.
7:15
And it's shown pretty dramatic speed ups in learning.
7:18
This is a diagram, a graph from Sentinem's reinforcement learning textbook.
7:22
And in this case, their RL algorithm was called reinforce.
7:25
And with a baseline, it learned much more quickly.
7:27
If you look at the green here, it got a higher reward much faster than red.
7:31
All right. Now you've got your training objective to update your LLM with the reward.
7:35
So just to take stock a little bit.
7:37
So far, you have two models to keep track of.
7:39
Your LLM that's learning.
7:41
So it takes up more memory because of gradients and optimizer states.
7:44
And then the reference LLM, which is frozen, that's used to generate rollouts.
7:48
And just a peek to the next video, you'll have two more models at play.
7:51
You learned how to train the reward model already.
7:54
And the baseline estimation will come from a fourth model.
7:56
And this is why RL is so memory intensive and needs a ton of computer memory on GPUs to run,
8:01
especially compared to that LoRA fine tuning that you saw previously.
8:05
Hoping this will change over time as we find more efficient methods.
8:08
OK. So now you've got your training objective. Great.
8:11
The data here is the same. You're defining your training objective.
8:14
So you're defining your training objective by weighting the probability of next token by advantage.
8:18
You're using your reference LLM to stabilize the measure for the probability of next token.
8:23
And you're measuring advantage by adjusting the reward with a baseline.
8:26
Then you're updating your LLM with this training objective to maximize the reward.
8:30
And then you loop. Great.
8:31
So this is the general and basic training objective for RL.
8:35
In the next video, you'll explore how to define a few modern training objectives, including the exact one used for RLHF.
8:42
So you've learned about how to define the training objective in a basic way.
8:46
Now take a look at modern RL algorithms, PPO and GRPO, for how they define the RL training objective.