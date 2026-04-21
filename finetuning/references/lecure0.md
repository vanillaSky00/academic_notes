Fine-tuning and reinforcement learning are really exciting techniques today to control
0:05
a language model's behavior, and they fall under this umbrella of post-training techniques,
0:10
which lets you control the model and get it from that raw intelligence to something usable
0:16
and practical. I'm really excited to see you learn about the overview of post-training and how it's
0:21
able to shift the model's behavior. Fine-tuning and reinforcement learning are super powerful
0:27
techniques. They are what got us from GPT-3 to chat GPT that everyone is familiar with. It is
0:34
what got us from just researchers playing with these models to millions of users actually being
0:40
able to use this very useful intelligence. And that's exactly it. It takes that raw intelligence
0:46
from the model and makes it usable and practical for all different types of applications that we
0:51
know and love today. Fine-tuning and RL are these techniques under the umbrella of post-training,
0:58
and post-training is key to most of the LLMs that you know and love today, such as chat GPT,
1:04
Claude, Gemini, Grok, you name it. Post-training has been evolving for quite some time, even before
1:10
chat GPT came along. So that started with fine-tuning. It went to the instruct GPT, RLHF,
1:16
which you might have heard of, reinforcement learning with human feedback. You'll learn about
1:19
that and preference learning. You'll learn about how these models can use tools, how they can reason,
1:25
and how they use chain of thought to actually be able to produce better answers and better code.
1:30
And so this is just the timeline of where post-training has gone over the years and the
1:35
models that have been post-trained to be better and better and better using, again, the set of
1:39
techniques across both reinforcement learning and fine-tuning. And you'll look into many of these
1:44
across this course. So just to actually get a little bit more concrete on post-training,
1:49
what does it look like before and after? So before, if you were to ask how to fix a car,
1:54
the model might just say, how to fix a bike? It thinks it might be in a survey. It'll just
2:00
answer your question with another question because it doesn't know it's supposed to be
2:04
a helpful assistant. So after it's been post-trained, after you've applied fine-tuning and reinforcement
2:09
learning to it, the model actually can become helpful. It can say, I'd be happy to help.
2:13
Could you tell me what specific issue you're experiencing with your car?
2:16
So much more helpful. It's able to actually give assistance. Now, diving into a more concrete,
2:21
deeper example here with code, maybe you have a question around, write a Python function for me
2:26
that takes the path to a .py file and returns a list of all function names defined in that file.
2:32
The base model is basically giving you that stream of consciousness. It hasn't gone through
2:37
that post-training yet. And so it just gives you, for example, if the file has this, it should
2:41
return this. It's kind of in the right general area, right? You can tell it's intelligent,
2:46
but it's not actually helpful. Whereas a model that has gone through post-training actually can
2:51
produce this helpful function, actually can give you something useful that you can take to put in
2:56
your code and execute. So very powerful techniques under the umbrella of post-training, fine-tuning,
3:01
reinforcement learning, and they're really here to control the model's behavior. It lets you
3:06
actually give the model the ability to chat with you, write that dialogue, be able to answer questions,
3:11
be able to handle interruptions when you're talking to it, topic changes, whether it's giving
3:15
something toxic or not, bias, it's able to handle these really tricky cases, become more helpful,
3:20
less harmful, handle typos even. That was an evolution through post-training. Handle ambiguity,
3:25
different prompt styles, make it consistent, reason, right? Think through things step-by-step,
3:30
be able to problem solve, be able to do deeper coding cases like you saw just now, as well as
3:35
be able to debug that code more effectively. Be able to actually give you the right type of
3:40
creative writing and style that you need, and of course, very deep domain expertise. So many,
3:44
many more techniques to basically control the model's behavior so the model can actually be
3:50
usable and practical. So just to dive into a few of those, this means that when you ask the model,
3:55
hello, how are you doing? It can say, hi, I'm doing great. What can I do for you? It's able
4:00
to answer that question, tell me how to build a bomb. It can actually have that guardrail and say,
4:04
sorry, I can't answer that. Maybe you are asking about the weather in SF on Sunday and you want it
4:10
to actually hit the weather API. It can learn to hit that weather API very accurately. If you ask
4:15
a question and you're using retrieval augmented generation, so you're attaching some document to
4:20
it and it misses, it actually doesn't have that information in there, it can actually be able to
4:24
recover from that and you can teach the model to recover from that and say, sorry, the information
4:28
provided doesn't include that date. It's very powerful. Maybe you ask a couple questions and
4:32
you want to get a consistent answer. This model, you can teach it now with post-training to give
4:37
that consistent answer across different variations of how you ask that question. And you can also get
4:43
the model to reason and think step-by-step. If you give it a difficult math problem, if you give it a
4:48
difficult coding problem, it can actually reason and think step-by-step. So these are all very,
4:52
very powerful techniques for you to control the model's behavior. Now that you learned how powerful
4:58
technique post-training is and exactly what it is, fine-tuning and reinforcement learning,
5:03
in this next lesson, you'll take a look at where it fits in in the LLM life cycle
5:08
of training the model, from pre-trained model to then post-training techniques.








0:00
Fine-tuning and reinforcement learning are really exciting techniques today to control
0:05
a language model's behavior, and they fall under this umbrella of post-training techniques,
0:10
which lets you control the model and get it from that raw intelligence to something usable
0:16
and practical. I'm really excited to see you learn about the overview of post-training and how it's
0:21
able to shift the model's behavior. Fine-tuning and reinforcement learning are super powerful
0:27
techniques. They are what got us from GPT-3 to chat GPT that everyone is familiar with. It is
0:34
what got us from just researchers playing with these models to millions of users actually being
0:40
able to use this very useful intelligence. And that's exactly it. It takes that raw intelligence
0:46
from the model and makes it usable and practical for all different types of applications that we
0:51
know and love today. Fine-tuning and RL are these techniques under the umbrella of post-training,
0:58
and post-training is key to most of the LLMs that you know and love today, such as chat GPT,
1:04
Claude, Gemini, Grok, you name it. Post-training has been evolving for quite some time, even before
1:10
chat GPT came along. So that started with fine-tuning. It went to the instruct GPT, RLHF,
1:16
which you might have heard of, reinforcement learning with human feedback. You'll learn about
1:19
that and preference learning. You'll learn about how these models can use tools, how they can reason,
1:25
and how they use chain of thought to actually be able to produce better answers and better code.
1:30
And so this is just the timeline of where post-training has gone over the years and the
1:35
models that have been post-trained to be better and better and better using, again, the set of
1:39
techniques across both reinforcement learning and fine-tuning. And you'll look into many of these
1:44
across this course. So just to actually get a little bit more concrete on post-training,
1:49
what does it look like before and after? So before, if you were to ask how to fix a car,
1:54
the model might just say, how to fix a bike? It thinks it might be in a survey. It'll just
2:00
answer your question with another question because it doesn't know it's supposed to be
2:04
a helpful assistant. So after it's been post-trained, after you've applied fine-tuning and reinforcement
2:09
learning to it, the model actually can become helpful. It can say, I'd be happy to help.
2:13
Could you tell me what specific issue you're experiencing with your car?
2:16
So much more helpful. It's able to actually give assistance. Now, diving into a more concrete,
2:21
deeper example here with code, maybe you have a question around, write a Python function for me
2:26
that takes the path to a .py file and returns a list of all function names defined in that file.
2:32
The base model is basically giving you that stream of consciousness. It hasn't gone through
2:37
that post-training yet. And so it just gives you, for example, if the file has this, it should
2:41
return this. It's kind of in the right general area, right? You can tell it's intelligent,
2:46
but it's not actually helpful. Whereas a model that has gone through post-training actually can
2:51
produce this helpful function, actually can give you something useful that you can take to put in
2:56
your code and execute. So very powerful techniques under the umbrella of post-training, fine-tuning,
3:01
reinforcement learning, and they're really here to control the model's behavior. It lets you
3:06
actually give the model the ability to chat with you, write that dialogue, be able to answer questions,
3:11
be able to handle interruptions when you're talking to it, topic changes, whether it's giving
3:15
something toxic or not, bias, it's able to handle these really tricky cases, become more helpful,
3:20
less harmful, handle typos even. That was an evolution through post-training. Handle ambiguity,
3:25
different prompt styles, make it consistent, reason, right? Think through things step-by-step,
3:30
be able to problem solve, be able to do deeper coding cases like you saw just now, as well as
3:35
be able to debug that code more effectively. Be able to actually give you the right type of
3:40
creative writing and style that you need, and of course, very deep domain expertise. So many,
3:44
many more techniques to basically control the model's behavior so the model can actually be
3:50
usable and practical. So just to dive into a few of those, this means that when you ask the model,
3:55
hello, how are you doing? It can say, hi, I'm doing great. What can I do for you? It's able
4:00
to answer that question, tell me how to build a bomb. It can actually have that guardrail and say,
4:04
sorry, I can't answer that. Maybe you are asking about the weather in SF on Sunday and you want it
4:10
to actually hit the weather API. It can learn to hit that weather API very accurately. If you ask
4:15
a question and you're using retrieval augmented generation, so you're attaching some document to
4:20
it and it misses, it actually doesn't have that information in there, it can actually be able to
4:24
recover from that and you can teach the model to recover from that and say, sorry, the information
4:28
provided doesn't include that date. It's very powerful. Maybe you ask a couple questions and
4:32
you want to get a consistent answer. This model, you can teach it now with post-training to give
4:37
that consistent answer across different variations of how you ask that question. And you can also get
4:43
the model to reason and think step-by-step. If you give it a difficult math problem, if you give it a
4:48
difficult coding problem, it can actually reason and think step-by-step. So these are all very,
4:52
very powerful techniques for you to control the model's behavior. Now that you learned how powerful
4:58
technique post-training is and exactly what it is, fine-tuning and reinforcement learning,
5:03
in this next lesson, you'll take a look at where it fits in in the LLM life cycle
5:08
of training the model, from pre-trained model to then post-training techniques.

0:01
LLM training has multiple stages. Post-training is just one of those last stages.
0:06
Here you'll learn about pre-training, mid-training, which are stages that come before post-training.
0:12
So the first stage of LLM training is pre-training. That's where the model gets that raw intelligence.
0:18
So you're going to focus here for a little bit.
0:21
The model learns to generate text by looking at the next token, the next word, for example, in a giant corpus of text.
0:30
And that's all it's trying to do. It's just trying to predict that next token, the next word.
0:35
And, you know, a training example might look like this excerpt from The Raven by Edgar Allan Poe,
0:42
but it's essentially taking that input once, outputting, you know, upon, getting once upon,
0:47
trying to output a once upon a midnight, and it keeps going. Right.
0:51
So it's all it's trying to do is predict the future, the next little token that comes afterwards.
0:57
And what it's looking at is a bunch of text over the entire Internet, a huge amount of data inside of the pre-training data set.
1:05
And all it's trying to do is that very simple task of predicting the next token, the next word, for example.
1:10
And it starts with that untrained models, completely randomized weights.
1:14
It does not output anything sensical in the beginning.
1:17
And it takes a lot of time when it looks for all that data through pre-training to predict that next token.
1:23
And it's very, very expensive. It can take months of compute to get that pre-trained base model.
1:27
And intuitively what's happening in pre-training is that through this next token prediction, it's actually quite magical. Right.
1:33
It's able to learn concepts through this across a huge amount of data.
1:39
For example, if the sentence you get is the sky is, these are the probabilities it might see across blue or clear or dark or orange.
1:49
It's a very high probability on blue because it has learned that over the course of a huge amount of text that when it sees the sky is, it's very likely to be blue.
1:59
So it essentially has learned this concept of the sky being blue.
2:02
And it also has learned, you know, when you give it a different sequence, like the sun is setting, the sky is, therefore, then orange becomes a higher probability than blue.
2:11
You know, it might be a sunset here. Right. And so this is very, very powerful raw knowledge that the model knows.
2:17
But it only knows how to predict that next token, the next word, for example.
2:21
The next stage after pre-training is often called mid-training.
2:25
I'm kind of detangling this from regular pre-training because it is usually a different team inside of these frontier labs.
2:32
Not always, but it essentially is continuous pre-training, but on a more specific curated data set.
2:39
And it's still predicting the next token, the next word, but essentially it's used to target, for example, new languages.
2:44
So very curated, well-established data sets for new languages.
2:48
So the model already has raw intelligence. Maybe here it's learning Chinese.
2:52
It can also be a good place to add different modalities such as audio or images.
2:58
And finally, it's also been used to increase the context length for the model.
3:04
So the model previously maybe wasn't learning on huge context and now in mid-training, you could actually teach it to expand its context.
3:11
So that's essentially what happens after pre-training. It's kind of like a continuous pre-training phase that's more curated.
3:18
Okay, so there's pre-training. There's mid-training.
3:21
Finally, there's now post-training, which is the type of training that comes after mid-training and the focus of this course.
3:27
So post-training usually encompasses a couple different famous methods.
3:31
One is fine-tuning. So that's where you give the model different inputs, but also target outputs of exactly how it should respond to a particular input.
3:40
Fine-tuning can also be called superfine fine-tuning or SFT in the context of language models here.
3:46
So in this course, you'll be seeing it as fine-tuning.
3:49
Another very popular and increasingly popular technique in post-training is called reinforcement learning.
3:55
And that teaches the model, based on a certain input, whether its response was good or bad.
4:00
You give it some kind of reward or score for what it produced.
4:04
So in these next modules, you'll learn for fine-tuning how the model is actually supposed to learn from that target output
4:11
and how it essentially takes that signal and improves it over time.
4:14
And these are some of the output gradients that you'll learn about.
4:17
You'll also learn about how to make fine-tuning really efficient so that you can actually run this on very few GPUs, just one GPU or even locally.
4:26
And these are using LoRa adapters, essentially little weights in fine-tuning, where the model doesn't need to learn so many changes.
4:34
In reinforcement learning in later modules, you'll learn about how to actually get that reward or score using another model
4:41
and how to actually train that other model on human preferences in reinforcement learning with human feedback.
4:47
And you'll learn how many models it takes to actually do reinforcement learning for these language models.
4:53
Here are four different models to actually get us there and why that is computationally expensive.
4:58
Now, bringing it back to your lab, in this lab, you're going to learn about the base model, the fine-tuned model, and the reinforcement learned model.
5:07
And you'll be able to compare how these models do on an example prompt and how their behavior is very different.
5:13
So here's an example prompt that goes for a math problem, and you'll see how its behavior shifts.
5:18
You'll also get to look through an actual data set, a very popular data set for math,
5:23
where you'll see how the behavior shifts across the base model, which is a pre-trained model, as well as the two types of post-training techniques,
5:30
fine-tuning and reinforcement learning, and how the model behaves differently there.
5:34
So just to summarize the stages of model training, there's pre-training, where essentially the model is reading an entire library.
5:42
You know, there might be low-quality or high-quality books. It doesn't have a specific goal. It's just reading.
5:47
In mid-training, it's a little bit more curated. The model is reading a curated set of advanced books to actually be able to learn either new languages or new domains, but still reading a huge amount of books.
5:59
And finally, in post-training, which includes fine-tuning and reinforcement learning, the model's learning to be almost an effective tutor, right?
6:07
How to answer questions clearly, how to interact politely. It can actually get a job at this point and be useful and usable to the world and for you as an assistant.
6:16
Now that you've learned how post-training fits into LLM training overall, it's time to learn a little bit more about the intuition behind some of these post-training techniques,
6:25
like fine-tuning and reinforcement learning, and how they differ, and what makes them work.

0:01
Fine-tuning and reinforcement learning are both important techniques for post-training.
0:05
You'll dive into the intuitions behind both of them, how they differ, and how they're alike,
0:11
using a really fun example with pasta. So, to first understand the difference between fine-tuning
0:16
and RL, or reinforcement learning, take a look at this example. So, if you have an input, like,
0:21
how do I cook pasta? In fine-tuning, what it'll look like is it wants to nudge the output of the
0:27
model to match some kind of target. Perhaps the target output is bring salted water to boil,
0:32
add pasta, follow package timing. So, that's the target. The model outputs bring, okay,
0:37
that matches well, salted, that matches well, water to boil, etc. So, it's all about matching
0:43
that target. In reinforcement learning, it doesn't actually matter what the output of the model is.
0:49
Maybe it's put pasta in boiling water with salt, then follow package instructions, or something
0:54
like that. It doesn't really matter as long as it gets scored later on and gets these rewards,
1:00
whether it's helpful, whether it's accurate, whether it's safe, and it gets a total reward.
1:05
And that reward is then how the model learns whether its output was good or not. So, just to
1:11
go a little bit more into this pasta intuition, fine-tuning is like watching your grandma cook
1:17
step-by-step, and your goal is to mimic your grandma at every single step. So, the model is
1:22
trying to mimic every single step that grandma is doing in making this pasta. And, of course,
1:28
the final pasta dish, hopefully, is correct here as well. But it's all about mimicking every single
1:32
step. In reinforcement learning, it's all about just getting actually the right dish. So, it
1:37
doesn't matter what any of the steps are. You can do any wacky thing in between, like here's
1:41
throwing all the pasta up in the air. And if you still get the right pasta dish at the end,
1:47
it doesn't matter. It'll still get a positive reward and say that was right. And, of course,
1:51
this can cause both interesting things like enable the model to learn new things, like new ways,
1:58
other than what grandma has done, maybe better than grandma's recipe or better than grandma's
2:02
steps. It can also introduce weird issues where the model might do some weird thing and think
2:08
that was actually the right thing to do because they got the right dish in the end. The major
2:12
upsides of fine-tuning versus reinforcement learning is that fine-tuning, often people just
2:17
say, you know, it just works. It actually mimics my data, right? It mimics the steps correctly and
2:22
understands the overall distribution of the data I'm giving it. So, it's able to mimic that target
2:27
output. And it's much more stable in that sense. Reinforcement learning, the major upside is that
2:33
it can develop these almost superhuman capabilities. It can think of things that are
2:37
better than the steps that we asked it to learn. And so, that's really powerful, but it is less
2:41
stable. So, to summarize a little bit about the needs, fine-tuning, you really need that good
2:46
data, right? The data has to be so good because it's trying to mimic that target output and you
2:51
need that data upfront. And that sometimes is harder to gather at scale. Sometimes it's actually
2:56
easier. So, that really depends on the type of domain. RL, you need that same input data, like
3:00
how do I cook pasta, but you don't need that target output data anymore. Instead, you need
3:05
good graders or good ways to get that score or reward at the end to say, yes, that pasta dish
3:10
did match the exact pasta dish, a good pasta dish, right? And these good graders are hard to
3:16
create and they're hard to tune and there are ways to game them, right? And so, this is one of the
3:21
hardest parts of RL to get right. For stability, fine-tuning has been around much longer. So,
3:26
there are more stable solutions in terms of getting the model to fine-tune. And as a result,
3:31
there are also more efficient methods. So, you need less compute. And in some future videos and
3:36
lessons, you'll see how fine-tuning can be significantly more efficient. RL, on the other
3:42
hand, suffers from a lot less stable methods and requires a significant amount of compute investment
3:48
to get there. But again, it's able to get that superhuman capability if you're able to keep it
3:53
stable for a long amount of time so that the model can learn. Ultimately, Frontier Labs combine the
3:58
best of both worlds. Usually, a base model, pre-trained model will go through fine-tuning
4:03
to learn some of the patterns first and then go through reinforcement learning to actually
4:08
learn some additional ways to solve the same problems and to solve it in its own way,
4:12
maybe more efficiently or better, and ultimately get that upgraded model in the end. So,
4:18
fine-tuning and reinforcement learning have both been very popular through the years. Fine-tuning,
4:22
again, much more mature. Here's just a graph plotting the papers published in both of those
4:27
areas. And you can see that reinforcement learning has really been picking up in excitement,
4:31
especially as it applies to language models. Now that you've learned about the intuitions behind
4:38
reinforcement learning versus fine-tuning, you'll dive a little bit deeper into what
4:43
makes each of them really work. For fine-tuning, it's the data. And for reinforcement learning,
4:48
it's the grading.