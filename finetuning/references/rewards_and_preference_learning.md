0:00
Rewards are one of the most important concepts in reinforcement learning, and preference learning
0:06
is a type of learning to train a reward model to output rewards for you.
0:11
So there are several key differences between reinforcement learning and fine-tuning.
0:16
So one, there's a loop of data gathering to training, to data gathering again, to training.
0:20
So it's not just first gather all the data, then train.
0:22
The next, the input is the same.
0:24
You need a bunch of inputs, but there's no target output.
0:27
The output here is just the model prediction, and it's needed to calculate the reward.
0:31
And reward is that key difference between RLM and fine-tuning
0:35
that you're going to explore in this segment.
0:37
So in fine-tuning, you have a specific correct target output,
0:42
and your goal is to reduce that gap between the target output and the model's predicted output.
0:48
And you're doing that by minimizing loss.
0:50
In reinforcement learning, that's pretty different.
0:52
There's no single correct or target output.
0:54
Instead, for a given input and given output, you provide a reward.
0:59
And that's a numerical score that tells the model how good that response was.
1:03
And the goal in RL is not to match a specific target,
1:06
but to learn behavior that generates outputs that will maximize this reward.
1:10
So what is reward exactly?
1:12
Reward is a single scalar number.
1:14
It can be positive or negative.
1:15
Typically, a good response will get a positive reward, like plus one,
1:19
and a bad response will get negative one, kind of like a penalty.
1:22
And one of the easiest ways to get rewards is using verifiable signals or verifiers.
1:28
And these verifiers rely on objective criteria.
1:31
For example, if the model is generating code,
1:34
verifier could check whether the code compiles or it runs without errors.
1:37
If it's generating math, it could see if it solves a math problem with a correct final answer.
1:42
So you can check these.
1:43
This might be confusing because fine-tuning feels like it checks whether the math problem
1:47
is correct too.
1:48
In fine-tuning, the model has to actually output the exact string of the response.
1:51
So if it includes reasoning,
1:52
it gets penalized for not predicting the exact reasoning trace in those think tags.
1:56
But in RL, the model can output basically nonsense in its reasoning
2:00
and still arrive at the correct answer and get the positive reward.
2:03
Verifiers are typically a script or a program, not a person, as you can see here.
2:08
So they're just kind of like checkers that give rewards.
2:10
The DeepSeek models pretty famously use verifiers.
2:13
One of the models, DeepSeek-R10, actually only use two verifiers
2:17
and only use reinforcement learning to train it.
2:19
So one is for checking if the answer to a math problem was correct.
2:22
And the other one is for format, to check if the model's response actually
2:26
used reasoning in those think tags.
2:28
So really powerful, just those two verifiers.
2:31
Another used in DeepSeek-R1, another model of theirs,
2:34
is a reward function that penalizes mixing languages.
2:38
So mixing English and Chinese specifically.
2:40
So the output is only in one language for human readability,
2:44
which was an issue without this extra verifier.
2:47
And just to see how these verifiers actually combine into a single reward signal,
2:52
it's just a simple sum here, right?
2:53
A simple sum of programmatic checks.
2:56
And that combined reward function shows how to balance multiple objectives
2:59
across accuracy, format, and consistency.
3:02
So in your lab code, you can assign a reward of one
3:06
if your model predicted the correct answer, even if its reasoning was off.
3:09
And if it was wrong, you can give it a zero or you can give it partial credit.
3:13
So you can give it that scaled reward based on how wrong it was.
3:16
To give the model some clue of how far off it was.
3:18
All right, this looks so easy.
3:20
What's the catch?
3:21
Verifiers only work for domains with clear verifiable criteria.
3:26
You can check if math's correct,
3:28
but you probably can't programmatically verify if the model is empathetic, if it's helpful.
3:33
The math checker here is freaking out a bit because the input says, how does Carly feel?
3:38
So the solution here is to train or use another model.
3:41
You can use an LLM judge or typically you can train a reward model,
3:44
which is common in RLHF or RL with human feedback used to train Chat GPT.
3:49
This model should give a scalar reward still based on the criteria it's learned.
3:54
And you want your reward model to output a scalar reward given an output like this.
3:58
So plus 1.3.
3:59
And you want that scalar to hopefully match some kind of quality you care about in the output,
4:03
whether it's helpful or nice in this case, but possibly safe and secure in others.
4:08
So just to go into exactly how the reward model is trained.
4:12
So the goal is to encode human preferences.
4:14
And it turns out to get the model to encode those human preferences,
4:18
there's a way to take human rankings of several model outputs and use that
4:22
as data to fine tune an LLM to be a reward model.
4:26
So first, you can show a person three model responses and a human labeler can rank them.
4:31
So you can see the rankings here of how good the responses are.
4:34
And the reward model is then trained on this data.
4:37
It doesn't learn how to write or output those thank you notes.
4:40
It just learns like this pattern of these rankings.
4:43
The raw rankings aren't usually immediately helpful.
4:46
That's just an easy way to get human preference data from a person.
4:50
You need to take those rankings and somehow turn it into a loss function
4:54
that can get the model to return those scalar rewards on every output.
4:58
So zooming in, you have your ranked preferences.
5:00
And you can turn those into preference pairs.
5:02
And this will be easier to use to achieve your goal of getting the reward model
5:06
to actually produce a reward.
5:07
You'll see in a bit.
5:08
So next, you have the reward model predict the reward for a model output A
5:13
and then for B in a pair where A is the preferred output by the human
5:16
and B is the less preferred one.
5:18
So it might look like this.
5:20
So scoring one output with a reward and then scoring the other output
5:23
with the same input but with another reward.
5:25
And now when you look at the relative preferences from the paired human preferences,
5:30
you can actually create a loss function between these two rewards.
5:33
So you know that the person, the labeler preferred A over B.
5:37
And you want to maximize the difference of the reward for A minus the reward of B.
5:42
And that means your reward model can apply high rewards
5:45
to the preferred A and low rewards to the less preferred B.
5:49
And to turn this into a probability,
5:50
you can actually use the sigmoid function of that difference,
5:53
which intuitively basically gets you the probability of A
5:57
being much better and more preferred over B.
5:59
And if the sigmoid is close to one or near 100%,
6:01
that means reward of A is much greater than B.
6:04
If they're the same, it's 0.5.
6:06
So that's how you actually get your loss function.
6:08
So now just to do the standard fine-tuning things to learn the model,
6:11
this process is called preference learning.
6:14
And typically it involves fine-tuning a language model head
6:17
to get it to output a scalar value.
6:19
You'll often see the reward model also be called a preference model too.
6:23
That's just the same model there.
6:24
And rankings don't have to be done with people.
6:27
They can actually be done by an LLM,
6:28
which you'll see an example of later as well.
6:30
So you can also get direct preference pairs by an LLM.
6:33
You don't have to go through the ranking process necessarily.
6:36
So this preference learning process is to get a reward model.
6:39
And it was also used in ChatGPT.
6:41
So you'll see here it's called RLHF or RL with human feedback.
6:45
And that's what it's popularized as.
6:47
So first in ChatGPT, what they did was you can sample multiple outputs from the model.
6:51
Here there are four outputs.
6:53
So A, B, C, D.
6:54
A human labeler actually ranks those four responses from best to worst.
6:58
Here the ranking is D, C, A, B.
7:00
That is turned into those preference pairs.
7:02
And that preference data is used to train a reward model
7:05
as you just learned to output that reward.
7:07
And that encodes that preference data.
7:09
After this, you'll see how this reward model is then used to update your main LLM with RL,
7:14
possibly in conjunction with those verifiers that you learned.
7:17
OK, so you've learned about verifiers.
7:19
You learned about the reward model.
7:20
It's time to compare them a little bit.
7:22
Often you'll use both, but to encode different things.
7:25
So for a reward model, the process is collect human preference data,
7:29
like rankings, train a reward model,
7:30
then use a reward model to generate learned reward signals of human preferences.
7:35
For verifiers, the process is somewhat similar.
7:37
You define those objective metrics.
7:39
You implement the text or functions.
7:41
And you want to generate those signals, but programmatically.
7:44
One of the best things about reward models is you can actually
7:47
use data to represent your objective if you can't actually write down the exact function,
7:51
which is what is needed for verifiers.
7:53
But one of the hard things is that your data might be
7:56
imperfectly learned by the reward model in some way.
7:59
And you'll learn about this later.
8:00
But since it's not objective, reward models can be famously, quote, reward hacked
8:04
or your main model is able to find a way to get high reward from your reward model,
8:08
even if the outputs aren't great.
8:10
And one of the hardest things about verifiers sometimes
8:12
is around the computational expensiveness of running a verifier.
8:17
Sometimes you need to run these on thousands and thousands of examples.
8:21
If it's very slow for a verifier to check a certain type of code, for example,
8:25
then it's not a very good signal to actually give back to the model in the RL loop.
8:30
So they're good for different things.
8:31
Reward models are essential for pretty subjective tasks like human value alignment,
8:37
improving dialogue, making the model more pleasant to interact with.
8:40
And verifiable rewards, verifiers are perfect for
8:43
more objective tasks like factual correctness, math problems or code.
8:47
So just to connect this with your RL data, where does this all come in?
8:51
So first, given your inputs, you get model outputs.
8:54
This is called a rollout and you get a bunch of rollouts.
8:57
If you're using a reward model, you might get preference data
9:01
and then convert to preference pairs.
9:02
And your data will look like this to train your reward model.
9:05
And your reward model can actually produce that scalar reward similar to your verifiers.
9:10
And in combination, those apply a reward to your rollout.
9:14
And that gets you a trajectory.
9:16
So that's the input model output and the reward.
9:19
And then you use that to train your LLM with reinforcement learning to maximize that reward.
9:24
And of course, you're looping here across, you know, generating that data,
9:28
applying the reward, getting the trajectories, and then maximizing the reward.
9:34
Now that you've learned about rewards and preference learning to get a reward model,
9:38
it's time to take a look at how those rewards actually inform
9:41
the LLM training objective in reinforcement learning.