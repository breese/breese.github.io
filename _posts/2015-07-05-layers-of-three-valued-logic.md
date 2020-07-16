---
layout: post
title: Layers of three-valued logic
tags: logic, tribool
---

[Three-valued logic](http://en.wikipedia.org/wiki/Three-valued_logic) has three states:

* True
* False
* Unknown (or indeterminate)

The unknown state means that we have to device new truth tables for the various logic operators. As there are several different logics of indeterminacy we will end up with different sets of truth tables. This becomes important when we use three-valued logic in composite conditions.

<!--more-->

In composite conditions we combine two or more conditions by logic operators such as AND and OR. When dealing with three-valued logic, we need to define the truth tables for these logic operators. We are going to look at three different alternatives that differ in how they prioritize the unknown state.

Truth tables can be illustrated with [Venn diagrams](http://en.wikipedia.org/wiki/Venn_diagram), wherein the logic states are drawn as overlapping circles. As the logic operators are binary operators, we are only interested in the overlap between two circles. We will draw the true state as a green disc, the false state as a red disc, and the unknown state as a blue disc. These discs are stack on top of each other in their order of priority.

## I do not care

For the first alternative we only want the composite condition to evaluate to unknown if all individual conditions evaluate to unknown. The unknown state will behave like a [don't care](http://en.wikipedia.org/wiki/Don%27t-care_term) value.

|![Lower AND](/assets/tribool-lower-and.png){: style="width:70%"}|
|_Figure 1_: AND with unknown below.|
|![Lower OR](/assets/tribool-lower-or.png){: style="width:70%;"}|
|_Figure 2_: OR with unknown below.|
{: class="normalimage"}

The truth tables and Venn diagrams can be seen in Figure 1 and 2. We have given lowest priority the the unknown state for both logic operators. This is illustrated by placing the unknown disc below the other discs in the Venn diagrams.

This kind of three-valued logic is useful when we have an optional value that we do not want to influence the result if it is missing.

Consider a situation where we have several input streams that each provides a sequence of boolean values. We want to combine these input streams through an AND gate and deliver the result to an output stream. If the input streams are finite then we want the calculation to end when all input streams become empty, but not before. If one of the input streams becomes empty before the others, then we want it to return a "don't care" value so that the values from the others input streams will continue to be processed.

Another use of this three-valued logic is to represent a veto vote, where all participants must vote in favour. The final result is then found by combining all votes with the AND logic operator. Unknown is used for abstained votes.

## I may care

For the second alternative we regard the unknown state as something that will become true or false in the future, but until then we reserve judgement in the matter. In some cases we can make a decision without knowing the future value of the unknown state, such as F &and; U &rArr; F and T &or; U &rArr; T.

|![Middle AND](/assets/tribool-middle-and.png){: style="width:70%;"}|
|_Figure 3_: AND with unknown middle.|
|![Middle OR](/assets/tribool-middle-or.png){: style="width:70%;"}|
|_Figure 4_: OR with unknown middle.|
{: class="normalimage"}

We can seen in the Venn diagrams in Figure 3 and 4 that the unknown disc has been placed in the middle, which corresponds to the Kleene's strong logic of indeterminacy. This logic is implemented by [Boost.Tribool](http://www.boost.org/doc/html/tribool.html).

A use case for this kind of three-valued logic is as a state variable for asynchronous operations that can indicate whether the operation resulted in a value, an error, or is still pending.

## I care

In the third alternative the unknown state acts like the [Not-A-Number](http://en.wikipedia.org/wiki/NaN quantity) in IEEE floating-point numbers. If one of the conditions evaluate to unknown, then the entire composite condition must evaluate to unknown.

|![Upper AND](/assets/tribool-upper-and.png){: style="width:70%;"}|
|_Figure 5_: AND with unknown above.|
|![Upper OR](/assets/tribool-upper-or.png){: style="width:70%;"}|
|_Figure 6_: OR with unknown above.|
{: class="normalimage"}

Figure 5 and 6 shows the unknown disc on top in the Venn diagrams, which corresponds to Kleene's weak logic of indeterminacy.

One use case is similar to the before-mentioned boolean streams, but this time we want the calculations to terminate as soon as the first input stream ends.
