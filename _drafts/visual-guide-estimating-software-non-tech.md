---
title:  "A Visual Guide on Estimating Software Development for Non Tech-People"
layout: post
tags: [principles]
---

## The black art of estimating software projects

I've been developing software for more than 20 years now. I've learned and used
a lot of different programming languages, I've worked with a lot of different
technologies, I've developed frontend software, backend software, for DOS, UNIX,
Windows, the web, embedded applications, and mobile applications.

In despite of all this hands-on experience, there's still something I haven't
learned to do: estimating how much time it will take to develop non-trivial
features, even on software's I've been working on for some time.

A lot of been said on the topic of estimating software projects, many
articles and books have been written (I have probably not less than ten books
about the topic in my personal library, including classics like
_[The Mythical Man-Month](https://en.wikipedia.org/wiki/The_Mythical_Man-Month)_
by Fred Brooks and _Software Estimation: Demystifying the Black Art_ by Steve McConnell).

So even to this day I'm often in situations where I'm asked how much time it
will take to complete a certain feature or how much it would cost to deliver
a certain project.

## Why we need to estimate software projects

This is usually a genuine question, coming from sales or product people that
don't understand software but understand the way to communicate features and
benefits to customers. We basically need to agree on something that they don't
understand and we, as software developers, find almost impossible to predict.

So in this short article I've decided to share how I usually approach the task
of estimating software, from what I've read and from what I've learned in the
last 20 years - I believe that, as software developers, we have the duty to
explain the process to non-software people and teach them the right mindset for
approaching the problem. We know what never works: giving them binary answers
that we're not even confident ourself about, knowing that they will be
disappointed soon.

My hope is that my teaching them how to approach the problem and how to ask the
question will make everybody satisfied and help respect all the constraints,
leading to the best possible software can be developed in the needed time and
cost.

## It's all about constraints

When you develop software you usually have to deal with four variables:

  1. _Time_
  2. _People_
  3. _Scope_
  4. _Quality_

_Scope_ gives the features you want to implement and _Quality_ gives how good is the
software you are going to deliver. _Quality_ is usually linked to bugs and the
quality experience of the final user (its satisfaction), bad quality leads to
high risk and future [liabilities](https://www.wired.com/story/equifax-breach-no-excuse/).

Because of that, _Quality_ you really don't want to compromise on _Quality_, so
in practice _Quality_ is not a variable you can control (not always true but let's
make this assumption for now).

So you are left with three[^1] variables: _People_, _Time_ and _Scope_ and it
turns out that there is an additional constraint, you can only set the value
of two out of these three, the one left is the one you have to _predict_.

## Predicting Time, People and Scope

In the rest of the article I will describe the three scenarios in which we will
have to predict each one of the three variables.

To make things fun, I have also implemented a little game that you can play in
each of the three scenarios and by playing it you will understand how your
decisions will affect the outcome of the project and the resulting software.

<canvas id="pjs"> </canvas>

## Quality

In the beginning of the article I've lied a little bit when I've said that
_Quality_ is usually a variable that you can't control. That's not always true,
unfortunately, since there is a not so uncommon scenarios where quality gets
forgotten.

Say you're hiring a company to develop the software or even a single freelance
developer and you want decide _Scope_, _Time_ and maximum Cost (so that gives you
_People_ = _Cost_ / _Time_). You'll think that quality will not be afftected but
by deciding all the variables you are necessarily going to give up on _Quality_
and by giving up quality will you let the freelancer or the company decide
what's good enough for them to keep the project within the time, scope and cost
that you set: the result what is usually a [crappy product](https://twitter.com/FredBouchery/status/948903120956018688).

[^1]: Instead of _Time_ and _People_, you may have _Cost_ that you can map to _Time_
and _People_ through the relation _Cost_ = _Time_ x _People_.
