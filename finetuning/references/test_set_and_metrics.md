0:01
Time to take a look at evaluation test sets, as well as metrics, like Pass at K, Calibration, and Uncertainty.
0:08
So, in pre-training, loss and metrics like perplexity are super important to making sure the model is doing well.
0:16
While this is important to the stability of training and post-training, in fact, evals are probably more important in post-training.
0:23
And these are massive data sets, as well as massive prep work.
0:27
But they help guide us to understand how well the model actually does in changing its behavior.
0:32
So, the reason why loss isn't sufficient in post-training is that while it's great for predicting the next token, like in pre-training,
0:38
it's fairly meaningless to user experience. And that is what really matters in usable intelligence for post-training.
0:45
Loss can go down, but that's basically mainly an indicator that stable training is occurring.
0:50
So, what exactly are evals? Well, evals encompass a bunch of things. One of them is evaluation sets, or test sets.
0:57
And these are held-out data sets that the language model hasn't seen before.
1:01
And it represents the desired behavior that you want the model to actually have.
1:06
So, in fine-tuning, it'll look like that fine-tuning data. So, essentially, there's an input, and then there's a desired output.
1:12
And then you can compare it to what the model's output is.
1:14
In preference learning, very similarly, there's going to be an input, there's going to be two model outputs, and then a preference label.
1:19
And you can compare that to what your reward model actually outputs and see how close that is. In this case, it's a match.
1:25
Beyond eval test sets, you can also evaluate the models in different ways using different metrics.
1:31
So, for example, you could get relative scores between two different models.
1:35
And you could use ELO ratings, which is common in chess, by looking at the win, lose, and draws across a lot of different examples.
1:42
If you're dealing with rankings and preferences, there's actually a lot of rank correlation metrics that can tell you how close a ranking is to another ranking.
1:49
Basically, agreement on rankings.
1:51
When correctness is verifiable, you can do similar things to your graders in RL and actually use eval graders that check for correctness or formatting.
1:59
Here's another example with JSON output.
2:02
Another way to evaluate is to output, say, three different outputs from the model and see if any of them are actually correct.
2:08
This is called pass at three.
2:10
So that means out of all three examples, one of them, at least one of them, was correct.
2:14
It's more lenient and suggests if you sample a lot more from the model, it can do well on your task.
2:18
And you can generalize this to pass at K.
2:21
And you'll see this a lot in papers reporting results.
2:23
Beyond just correctness, you want your model to behave well under uncertainty.
2:27
So calibration is one of those measures.
2:29
It's about making sure what the model predicts as a probability actually lines up with that probability in reality.
2:36
The famous example of calibration is when a model predicts 80 percent rain.
2:40
Does it actually rain 80 percent of the time?
2:42
Like if model confidence is trustworthy, especially if you use those token probabilities for downstream applications or even ensembling models, for example, like 70 percent might mean different things for different models.
2:53
I'm sure you know people who are overconfident versus underconfident.
2:57
You want to be able to measure how well calibrated your model is.
3:00
There are different ways to evaluate calibration.
3:02
One, you could just look at the token probability matching the empirical token occurrence.
3:07
So how often the token actually occurs.
3:09
You could also do something interesting with multiple choice.
3:12
So with multiple choice, you could have, you know, say four different tokens, A, B, C, D, and extract the token probability for a multiple choice token.
3:20
Finally, you can also aggregate across the entire sequence and normalize over token probabilities there to get an understanding of calibration at the sequence level.
3:29
And finally, metrics like expected calibration error here will help you quantify how well calibrated your model is.
3:36
So fun fact about calibration, weather people are actually often not calibrated and that's so they don't disappoint you when it actually rains.
3:44
So more on uncertainty beyond just calibration.
3:47
If you get the model to say, I don't know, it's probably better than it hallucinating at different times.
3:53
This is called a refusal.
3:54
So a way to assess this in the model is to have a confidence threshold like point seven.
3:59
And if the probability is under that, then the model just says, I don't know, instead of outputting what it thought.
4:05
And you can find that on, let's say, a hundred math questions.
4:09
If you don't allow it to say, I don't know.
4:11
So no refusals, just regular way of testing it.
4:14
It can answer, let's say, 65 percent correct.
4:17
But if you allow it to say, I don't know, and refuse some of those answers, and maybe it refused 20 of them, it could actually get a higher score.
4:25
And so this is really interesting.
4:26
This means that your model can actually do well and you can actually use the confidence threshold probabilities to help you filter when it should actually be outputting something or not.
4:36
Finally, you can't really miss efficiency metrics.
4:38
So there's a bunch of different ways to evaluate how fast you can actually get an output from a model.
4:43
So that's latency.
4:44
And a couple of really important latency measures are time to first token.
4:48
So that's the time it takes to get to that first token that it outputs.
4:51
And time per output token, or TPOT.
4:54
And that is kind of a steady state decoding speed of how fast it's able to output an entire sequence for you.
5:00
There's other different measures in throughputs.
5:02
You can get tokens per second per GPU.
5:04
You can aggregate throughput across concurrent requests in terms of scalability.
5:09
Cost per token is also really important.
5:10
Cost per sample as well.
5:12
Input and output tokens are going to be weighted differently often in different APIs and just the cost it takes to do them.
5:18
Token efficiency is also important.
5:20
This is kind of like, what is the effective token that you're getting from the model?
5:25
How verbose is it versus concise it is to get to the answer that you need?
5:29
So, wow, that's a lot of metrics.
5:30
It turns out evaluating on all of them is best practice.
5:34
So you get a holistic sense of how well the model actually is doing.
5:39
HELM is one example from Stanford.
5:41
And it stands for Holistic Evaluation of Language Models to evaluate across many different dimensions.
5:46
And what HELM does is it evaluates models across different diverse tasks like question answering, summarization, classification, like spam detection.
5:55
And these aren't isolated tests.
5:56
They represent 42 different scenarios spanning different domains like medical, legal, news, finance.
6:02
And what makes HELM nice is that it's a living benchmark.
6:05
They're continually updating it with new scenarios.
6:07
So while HELM is great and there are a bunch of those types of composite metrics,
6:12
traditional benchmarks up until recently kind of test the model with simple tasks like those exam questions, coding puzzles, lab scenarios, or shorter prompts.
6:20
And they're relatively toy benchmarks.
6:22
And this is useful for progress tracking, especially in the early days of language models.
6:26
But we're starting to measure real world work now.
6:29
And GDPval from OpenAI actually moves evaluation to this professional level of tasks.
6:34
So they include real deliverables, real reference files, and real expert professionals who've contributed to the eval.
6:41
And specifically, GDPVal has a ton of different tasks.
6:44
So 1,300 tasks, more than that.
6:47
And it's from 44 different types of jobs across nine different industries.
6:51
And the contributors are professionals with over 14 years of experience in their industry, which is huge.
6:57
And the deliverables are real.
6:58
So they're documents, slides, blueprints, care plans that actually matter in their jobs.
7:02
And the environment is real too.
7:03
The complexity of real files, context in a work environment.
7:07
And then the grading, they try to make it fair.
7:09
So they try to do blind expert grading when they release this, comparing the AI and human work.
7:14
And those blind expert graders are, again, professionals who hadn't previously seen the outputs before.
7:20
So what's really interesting is that OpenAI ran GPT-4.0 and GPT-5 on GDPVal.
7:26
And the performance more than tripled in a year.
7:28
That's a sign that these models are better at real world complexity and are just getting better and better.
7:33
Expert graders compared the generated deliverables to human deliverables for this.
7:38
And it's really just approaching the quality of work produced by industry experts.
7:42
They also found that Claude Opus 4.1 produced outputs rated as good as or better than humans in nearly half the tasks.
7:49
So as GDPval saturates, we'll need to devise new benchmarks for harder real world tasks.
7:55
And I'm excited to see what those are because I think we'll be constantly producing new benchmarks to meet the current next frontier of these models.
8:04
Now that you've seen some of the metrics in eval benchmarks for fine tuning and preference learning,
8:11
it's time to take a look at RL test environments and how they work.