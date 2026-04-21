0:00
Data is what makes fine-tuning work, and grading is what makes reinforcement learning work. For
0:06
data, you'll learn about how the inputs and outputs can really shape that model behavior
0:11
very differently, and that's for fine-tuning. For reinforcement learning, you'll learn how
0:15
the grading can really nudge the model in different directions, and how that all comes
0:19
together in an RL training environment. Fine-tuning means really good data, right,
0:24
so that might be hard to scale, but the data is all what it takes to make fine-tuning work,
0:29
and that is the most important part for fine-tuning. For reinforcement learning, it's
0:33
about good graders. You don't have that target output, so it's all about how you grade that
0:38
input data and how well those graders are created. So, going into fine-tuning, making the data work,
0:45
what does that look like exactly? So, you could have an input like, what's the capital of France?
0:50
The target output's Paris. Who wrote Romeo and Juliet? The target output is Shakespeare,
0:54
so your data might look like this, but when you, you know, actually ask a model, you know,
0:59
what's the capital of France? It answers Paris correctly, and you ask it, what about Spain?
1:04
It starts saying Spain is a European country. Oh no, it actually was not able to handle,
1:09
in this case, chat history. So, what do you do? Instead, what your data could look like is
1:14
actually passing in that chat history of what is the capital of France, Paris, and then what
1:20
about Spain, and then the target output is here. So, this actually gives the model examples where
1:26
the input includes that chat history. That's how you teach a model to actually be able to handle
1:31
past chat histories. And again, you can keep going, Germany, and learn that Berlin target.
1:36
Of course, for an actual prompt inside of the model and your actual data, you probably want
1:41
to put in these tags, these prompt tags of user, then assistant, user, then assistant,
1:46
which you've probably seen before, to actually denote who is saying what in a chat. And then,
1:51
so now that you've taught the model with that data, your fine-tuned model will then be able to
1:55
handle something like what's the capital of US, Washington DC, what about China, right? So,
2:00
being able to take in that history and be able to say Beijing. In addition to getting chat history
2:05
to work, there's also different methods of having the model learn not only just the answer to this
2:11
word problem, but also its rationale, or what it's thinking, quote-unquote, or what it's reasoning.
2:18
And so, in this case, actually going through these different steps to teach it these steps. This is
2:22
similar to when you were teaching the model to cook pasta, to actually be able to go through
2:26
every single step that grandma was doing to cook the pasta. And typically, what it looks like in
2:31
an actual prompt is using these think tags and answer tags, which you can then use later and
2:37
extract later from the target output so that you can check whether the answer is correct.
2:41
Fine-tuning data can also be very powerful for teaching the model how to handle
2:46
RAG, Retrieval Augmented Generation, misses, or when you have a bad RAG document attached to it.
2:52
So, of course, in the easy case where it's correct, that's fine. It's able to look at the
2:56
input and be able to get the target output. But in bad cases, where in this case, the document
3:01
actually says that Sydney is the capital of Australia, which is wrong, the model can actually
3:06
recover from that. And you can teach it to recover by giving that target output that there's an error
3:10
in the document and the capital is actually Canberra. In your data, you're also able to get
3:16
the model to create guardrails. So, here, the user might say, help me write a computer virus.
3:21
That's really bad. That's harmful. In your target output, you want the model to refuse that, right?
3:26
And so, when you teach the model with examples like this, then the model will start to refuse
3:30
things like this. Here's another example of a guardrail, which might be a bit different than
3:35
just the safety guardrail. This is if you want to train a model to be an AI banker, right? And so,
3:42
then you have your model here wearing a banking suit. And the user might ask, what's the capital
3:47
of Australia? And instead, you want to apply a guardrail here and say, sorry, I'm actually only
3:53
able to answer questions about AI bank. So, this is adding a custom guardrail for your custom
3:59
fine-tuned model. Why this is helpful is that if you don't have these guardrails, people can start
4:05
to use your model for anything. Here is a funny example from Amazon when they were launching their
4:11
first bot, where instead of looking for specific info about a product, this person is asking the
4:18
underlying bot to write a React component that renders a to-do list. So, using the model for
4:23
different things instead of the model here learning to refuse, right? To refuse writing a React
4:28
component. But that is easily mitigated by having the model learn from those examples of guardrails
4:34
through fine-tuning. Reinforcement learning. So, for RL, it's all about the graders and making the
4:39
grading work and less so you don't have that target output anymore to show what's correct.
4:44
But instead, you can grade what's correct. So, what it looks like here is here's a math problem
4:49
where Carly has eight apples, buys two more, but then sells five to the local baker. How many now?
4:54
So, you can use a math grader. The model can output whatever it wants, but ultimately output
4:59
some answer. And the math grader can tell the model, okay, this was right or wrong. And so,
5:04
here it could say incorrect if the model's output was incorrect, but it could also give some partial
5:09
credit. So, it shows its work. That's something that the grader wants to encourage. And so,
5:13
their total reward or score is then higher. And of course, if it gets it correct, then you can
5:19
have a total reward or score be higher. Of course, similar to the fine-tuning example with
5:24
the data in answer tags, this makes it easier to extract that answer. And also, you can have the
5:29
grader include a formatting grader to encourage the model to actually produce those tags correctly.
5:36
A lot of these graders here are deterministic. So, whether something is correct or not,
5:42
whether it shows work, that could be some kind of reg ex. Whether it has stuff in an answer tag,
5:48
whether there's answer tags exist, that's something that you can easily extract deterministically
5:52
with functions. That's not always the case, because what if you had this math problem,
5:56
but then the question is actually, how does Carly feel? Now, your math checker,
6:01
its mind is blown. It doesn't know how to grade this, right? So, instead, what you can have is
6:06
another language model or another type of model actually output a reward. You can learn how to
6:12
grade an output to how does Carly feel. And so, maybe the LLM can give a score based on
6:18
different parameters like politeness or enthusiasm or engagement. And so, here for this input,
6:24
greet politely, the model could say, hi there, how are you? And it would give it a high score,
6:30
high politeness, high enthusiasm, high engagement. However, if your input, you adjust it slightly,
6:35
and the model now outputs hello, hello, hello, hello, hello, many hellos, the grader might still
6:41
say it's high politeness, high enthusiasm, high engagement, but you didn't really want the model
6:47
to output that. It's a bit silly. So, this is a common thing that happens in reinforcement learning
6:52
called reward hacking. So, that's why the grader is so important to get right, right? If you don't
6:58
get it fully right, the model will find a way to hack it and find a way around it to get that high
7:03
score, that high reward, without actually doing what you want it to do. The distribution of inputs
7:09
that you put into the model also matters significantly. So, in reinforcement learning,
7:13
you want to give that large distribution of inputs similar to in fine tuning so that the model can
7:19
actually react to many different types of possible user inputs. This needs to, again, be very
7:25
representative of what kind of user inputs you expect the model to see. And now, what's a little
7:30
bit different in reinforcement learning is that in addition to having that grader, typically,
7:36
you'll create this RL training environment. And that's these inputs, these graders, but also some
7:41
of these other things that might come into play. For example, in the environment, you might also
7:46
expect the model to be able to use a calculator tool. And so, you want to make a calculator tool
7:51
available to the model so the model can then use that tool to, for example, answer this math
7:55
problem. You might also give tools like a search API, right? Or you might give files to look through
8:01
your code base. And the model can look through those and use those in this contained environment.
8:06
And then, of course, the grader can score it based on the model's ability in this environment.
8:10
How representative this environment is to your real-world use case is going to be as much
8:15
information and as helpful as you need the model to actually learn from this environment. So, the
8:20
more representative this environment is to where you expect the model to operate, the better.
8:25
Now, quick, some considerations is while the more realistic, the better, be careful because some of
8:31
the tools that you might give the model, right, will be hitting external APIs. And you might be
8:37
DDoSing some of those external APIs, and it might actually turn out to be unrealistic if you're
8:41
running this RL environment quite intensively. And you'll see this a bit later as well. So,
8:47
ultimately, the data for RL is going to look different than for fine-tuning. It's going to
8:52
look like that input, that model output, and then a reward. And the training environment, of course,
8:58
is very different from fine-tuning as well. And here are a couple of different training environments.
9:02
So, one might be around debugging a code base. One might be about that, like, polite greeting.
9:06
And you want to actually give a right mix of these so the model learns all the different tasks that
9:11
you want it to learn with reinforcement learning. So, this is multiple training environments producing
9:16
multiple types of data out to train your model. And you want to kind of balance it correctly.
9:21
And you'll learn more about how to exactly balance this in the future lessons. So, putting it all
9:26
together, again, you want to use both fine-tuning and reinforcement learning for a lot of your good
9:30
models. So, first, it's about getting that fine-tuning data. Then you can fine-tune your
9:35
model on that input and target output pair. Get your fine-tuned LLM. Then, typically, what happens
9:40
is then you create those RL training environments with your different distribution of inputs that
9:45
are representative, but also your graders and other information, like those files or your code
9:50
base or different tools like a search API. And then you run an RL loop where you get some RL data.
9:56
So, you're given an input, you get different model outputs, and those are rewarded. So, those are
10:01
given rewards in your RL training environments. And then you train your fine-tuned LLM with
10:06
reinforcement learning. And then you keep running that in that loop. So, fine-tuning really goes
10:10
through that one giant stage of data collection, then the actual fine-tuning and training step.
10:15
And then RL actually goes through multiple iterations of collecting data and in training
10:20
the model. Now that you've learned the key components to fine-tuning, which is data,
10:25
and to reinforcement learning, which is the grading, take a look at how to combine them
10:30
for a post-training reasoning example.


0:01
When releasing a model, one of the most important qualities is to make it not only helpful, but also safe and secure.
0:08
In this next post-training example, you'll learn how to use fine-tuning and reinforcement learning to make your agent just that.
0:15
You want to build a safe, secure support agent.
0:18
So when a user asks, I forgot my account password, please verify me with my SSN.
0:23
If it says, sure, what's your full SSN, that's not great, right?
0:26
You don't want the model to actually be prompting people for their social security number.
0:31
Instead, you'd rather have the model say, you know, I can't collect your SSN, but to verify here's another method.
0:37
That's a safe response.
0:39
The way to do that, there's a really interesting way, is to create a constitution or a set of rules, essentially, that the model should obey.
0:46
And from there, you can actually create data to then fine-tune and do reinforcement learning on so that the model actually abides by this constitution.
0:55
So, for example, here you could have avoid requesting or exposing sensitive personal data like your social security number.
1:01
And if a request is unsafe, you can decline briefly and then suggest a safer path.
1:06
So here's what that looks like.
1:07
So your target output, you want it to be that safer response.
1:10
So to get that target output, to get that data, for example, you could actually still have the model output the unsafe response.
1:18
But then you have another model be a judge or the same model be a judge to itself that looks at the constitution and says, actually, you know what?
1:26
This is pretty unsafe.
1:27
Let me revise it.
1:28
And then it revises it and gets that correct output.
1:31
And then you can use that input and that revised output as that target output.
1:35
Great.
1:36
So that's how you get your data.
1:37
And you can do a lot of it to get your training data for fine-tuning.
1:41
The steps are just write a constitution.
1:43
The LLM can actually then generate those input target output pairs.
1:47
The LLM can then critique itself and revise those outputs based on the constitution so that those outputs can be safer.
1:53
And then you can fine-tune on the revised output.
1:56
And so this is really nice because it scales well.
1:59
You don't need a ton of human labels.
2:01
A model can go by your constitution.
2:03
It just requires a person writing a good constitution.
2:07
Now, in reinforcement learning, what you can do is you can use the same constitution here.
2:11
And you can get a model to actually learn from this and take this and scale it out even further.
2:16
So what that looks like is, you know, when you see I forgot my account password, et cetera, same input.
2:21
The grader is an LLM as judge, and it's using the constitution.
2:25
And what it does is it looks at two different model outputs.
2:28
One, you know, that is safe or unsafe.
2:30
And based on the constitution, it can grade which one is better.
2:33
It can prefer B over A.
2:35
And usually you're going to learn a new model called a reward model to give that reward to say how good A or B actually is.
2:42
Great.
2:43
So what that looks like is you can use the same constitution, reuse it.
2:46
So you only need to write that once.
2:47
And the LLM can generate two outputs from just a single input.
2:51
It can select which one is better.
2:53
And then you can train another model called a reward model to actually be able to give rewards on how good an output is.
2:59
And then you can train an LLM on that RL data of input, output, and reward with reinforcement learning.
3:04
And, again, this scales super nicely.
3:06
The big takeaway is that this scales so nicely because you, again, only need that one constitution.
3:10
This is also called RL AI for RL with AI feedback.
3:14
In the full loop, what it looks like is a recipe where you only have to write that constitution once.
3:20
Then you go through the fine-tuning flow of generating that data, critiquing it, fine-tuning,
3:24
and then taking that fine-tuned model to then generate a more output, select the best output, train a reward model,
3:30
and then do reinforcement learning to get that final aligned model, a model that is aligned to your constitution.
3:36
This is a great way to scale very, very close, very, very small amount of human input,
3:42
but rich amount of human input where you are giving it exactly how you want the model to behave in that constitution,
3:49
but then scaling it out with both fine-tuning and reinforcement learning by using the model itself.
3:54
This is a popular method called constitutional AI that was introduced by Anthropic.
3:58
So, as you can see on the left, the aligned model with constitutional AI was just as helpful as a model without.
4:05
But on the right, it was significantly more harmless where being higher here is better.
4:10
So, it's significantly less harmful.
4:12
Again, being higher here is better with the gray.
4:14
And that means the model was refusing those harmful requests.
4:19
And as a result, they were able to ship a much safer, more secure LLM.
4:24
Now that you've learned how to align your LLM to a constitution using reinforcement learning and fine-tuning,
4:30
next up is looking at how these things happen in the wild with frontier AI models,
4:35
as well as how you can do it on your own and what libraries to use.