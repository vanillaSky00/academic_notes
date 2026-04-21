0:01
Two of the most popular RL training algorithms are PPO and GRPO.
0:06
Let's take a look at them.
0:07
So here's where you're going to learn these new training objectives.
0:11
So first, revisiting the RL training objective you just learned,
0:14
you're trying to maximize the sum of some probability term multiplied by some reward term.
0:19
So for better performance, the probability term is a ratio between your model and reference LLM
0:24
that generated the data or the rollouts, and the reward term should be the advantage centered by a baseline.
0:28
And this leaves us with a question of how to compute the baseline, b here, to calculate the advantage.
0:33
There are a bunch of different ways, and you'll see a couple of them in this video.
0:37
So the standard approach in PPO, or proximal policy optimization, which is used in RLHF,
0:43
is to train another model for the baseline.
0:45
And this model's only job is to predict the expected reward for each token,
0:50
and not just the full output sequence, while the main LLM is generating,
0:54
so even halfway through generation.
0:56
And it's trained with regular supervised fine-tuning using the actual rewards r as the target labels.
1:02
Now, this lets you compute advantages per token step, not just on the full response,
1:06
so tokens that contributed more or less to the final reward will get weighted differently.
1:10
This is often implemented as the same LLM with a new head predicting the expected reward per token.
1:16
Now, for per token reward to be more accurate and smooth,
1:20
you'll want to use an advantage function estimator, like GAE, Generalized Advantage Estimation.
1:26
GAE uses the baseline model to backprop the final reward through earlier tokens.
1:32
So earlier tokens get credit or blame for the final outcome,
1:35
so it carries those prior rewards back with some discount factor, which is, again, another hyperparameter.
1:41
And as a result, GAE will smooth the rewards across tokens.
1:44
The exact math is shown here, but it's not important to know every single detail.
1:49
It's just that this is actually what's used in practice for that estimation of advantage.
1:54
So this brings us to PPO, proximal policy optimization, and that's the first algorithm here.
2:00
And there's just one final tweak on the other ratio term.
2:03
Sometimes that probability ratio can get really large, too large, and that can lead to huge unstable updates.
2:09
And PPO solves this by essentially clipping the ratio.
2:13
So this math term is really just picking which one is smaller, clipped or not clipped, update.
2:18
And this prevents the weight updates from exceeding a certain threshold, and large changes are therefore not allowed.
2:23
And this is the core idea of PPO, actually.
2:25
It takes small, stable steps to prevent the model from breaking during training.
2:30
All right. And that was the full PPO training objective.
2:33
So this is what PPO looks like in the whole RL loop.
2:36
It's part of the training piece. And just to look at it in a diagram, you have your training objective.
2:41
Great. And you add clipping here. You train another LLM to estimate the baseline for your advantage.
2:47
You then update your main LLM using that PPO training objective to maximize reward.
2:52
And you loop. All right. The full PPO setup requires several different models.
2:57
The main LLM you're training, the frozen reference model and the baseline estimation model that you're also training.
3:03
This is super computationally expensive and also complex to manage.
3:07
What's good is that this has led to newer algorithms that are more efficient.
3:11
And one of them is GRPO, or Group Relative Policy Optimization.
3:15
And that's been introduced by DeepSeek recently.
3:18
And GRPO's key innovation is that it gets rid of the separate baseline estimation model.
3:24
So the key innovation of GRPO over PPO is stated here in the paper.
3:29
And it's a change to how the baseline is computed to get that advantage.
3:33
And that is the main thing. But it is able to significantly reduce training resources.
3:37
All right. So exactly how does that happen?
3:39
Remember, the whole point of the baseline estimation model was to estimate the baseline reward so that the outputs are
3:44
clustered around zero and credit for the reward is distributed across the tokens more fairly.
3:49
And only tokens that did better or worse than expected get strong learning signals.
3:53
This definitely reduces that variance and distributes credit across the sequence more effectively.
3:58
GRPO asks, instead of training a whole separate model to predict the average or expected reward,
4:04
why not just generate a group of several outputs for a single input
4:08
and calculate the actual average reward of that group? That is the key idea of GRPO.
4:14
It's really simple and it's a direct way to estimate the baseline on the fly.
4:18
And specifically, GRPO computes this baseline by literally drawing many different outputs,
4:23
and that's part of a group, for every single input and then subtracting out the average reward
4:28
and dividing by the standard deviation. And there's obviously a need for that extra model, but it does require more sampling.
4:33
And so that's GRPO's advantage calculation.
4:36
Now, to go into it in code a little bit, you can specify a certain number of outputs you want to sample
4:41
and what temperature to use to get those variations. And temperature just helps you sample different outputs per input.
4:47
So pulling back to the RL training objective, GRPO's objective will be very similar to PPO,
4:52
but the baseline is no longer that prediction from another model.
4:56
Instead, the baseline is just the average of the raw rewards for all outputs of a single input.
5:02
Basically that group. And in code, this is what the baseline equation looks like,
5:06
just normalizing the rewards over different sequences. Pretty simple.
5:10
So note that the advantage estimation is now at the sequence level and no longer at the token level, like in PPO.
5:15
So just going through a concrete example, maybe you sample four outputs from a single input,
5:20
and then you have different graders like formatting or unit tests to give an aggregate reward for every single output, like this.
5:26
Then the baseline is the average of these scores.
5:29
So you can then calculate the advantage for each response by subtracting that baseline.
5:33
Note that in the raw rewards, it's all positive, right?
5:35
But then now the advantage, you're actually getting that distribution and they're centered around zero.
5:40
And then responses A and B will have that positive advantage.
5:42
So RL update will increase their probability and C and D will have negative advantage.
5:47
So their probability will be decreased. All right. So here's GRPO.
5:50
It keeps the clipping in PPO and just changes the advantage calculation.
5:54
Now going through it a bit here. So your GRPO objective here is to weight the probability of next token by advantage with clipping, similar to PPO.
6:02
But instead, your advantage calculation is going to be different.
6:06
So you want to normalize rewards in a group of outputs.
6:09
Then you update your LLM using this objective to maximize the reward, just like in PPO. And you lose.
6:15
In your code, this is what it'll look like. So for a particular input, you generate a group of outputs.
6:21
So just multiple outputs for a unique input. Then you compute the rewards for that group.
6:25
And then you get the average reward to then calculate the advantage. Pretty simple.
6:29
And then you pass this compute reward function as the reward function.
6:33
Pass your GRPO arguments in and your data sets in and then your model in and then you can train.
6:40
So GRPO essentially gets rid of this model out of these four different models,
6:44
which is great because you're also usually training this one in PPO.
6:48
And that takes up a lot of space, not to mention it can be some unstable as well.
6:52
All right. So here's the full comparison of PPO and GRPO.
6:55
There's still quite a few similarities overall, but the key differences are one, you see the baseline estimation model.
7:02
It's gone. And then you can see the advantage calculation being very different.
7:06
One using the baseline estimate or the expected reward per token and the other using that group computation.
7:12
OK, so this has been pretty thrilling to learn. Here's a bit of the history of the methods you've learned.
7:17
So from RLHF with PPO ahead of ChatGPT's launch to RL-AIF, where AI is giving feedback instead of humans to abide by that constitution,
7:26
to GRPO and using GRPO with verifiers and reward models to effectively reason.
7:31
So here's looking at the whole view again, where you put in PPO or GRPO.
7:35
It's really here where you're defining your RL training objective.
7:38
So going through that loop, your training objective can be either PPO or GRPO.
7:43
And you update your LLM again by maximizing the reward in your loop.
7:48
Congratulations on getting to the end of module two.
7:51
Oof, that was tough with all of that math, but I hope you got some intuition on fine tuning and reinforcement learning.
7:58
Next up, take a look at how evaluation, your test sets, your eval sets will actually help guide that training process as a North Star.