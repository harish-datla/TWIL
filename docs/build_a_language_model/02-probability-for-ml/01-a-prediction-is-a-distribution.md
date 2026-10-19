# A prediction is a distribution

A spam filter reads an email and outputs "spam: 0.92, not spam: 0.08". A language model reads "The cat sat on the" and outputs a probability for every token in its vocabulary. Both outputs are a probability distribution: one non-negative number per possible outcome, with all of them adding up to 1. Every model in this series outputs one, and every loss in this series scores one against what actually happened.

The thing being predicted, the email's label or the next token, is a random variable: a quantity whose value is decided by an outcome you can't know in advance. When it can take only separate, countable values, like a token id, a class or a die roll, it's discrete. Its distribution is then a table with one probability per value, called a probability mass function.[1]

A quantity like a house price or a pixel's brightness can take any real value, so it's continuous. Any one exact value then has probability zero, and the distribution is described by a density instead: the probability of landing in a range is the area under the density over that range.[1] A classifier's softmax output is a mass function over classes. A regression model that assumes Gaussian noise around its prediction outputs a density.

A spam filter that learns from labelled email needs two kinds of probability. The joint probability P(spam, "free") is the chance that an email is spam and contains the word "free". The conditional probability P("free" | spam) is the chance that an email contains "free" once you know it's spam. Read the bar as "given".

Two rules connect them. The sum rule gets the probability of one variable on its own, its marginal, by adding up the joint probability over every value of the other variable. The product rule splits a joint into a conditional times a marginal.[1]

Worked example

Out of 100 emails, 30 are spam. 18 of the spam emails contain "free", and so do 7 of the 70 legitimate ones.

Joint: P(spam, free) = 18/100 = 0.18. Conditional: P(free | spam) = 18/30 = 0.6.

Sum rule: P(free) = P(spam, free) + P(not spam, free) = 0.18 + 0.07 = 0.25.

Product rule: P(spam, free) = P(free | spam) × P(spam) = 0.6 × 0.3 = 0.18.

So the chance that an email containing "free" is spam is P(spam | free) = 0.18 / 0.25 = 0.72.

That last step is Bayes' rule. It turns the probability you can measure, how often the evidence shows up when a hypothesis is true, into the one you want: how likely the hypothesis is now that you've seen the evidence. You get it by writing the product rule both ways round, P(free | spam) P(spam) = P(spam | free) P(free), and dividing by P(free).[1]
P(H | E) = P(E | H) P(H) / P(E)

In words: how likely the hypothesis is after the evidence equals how well it predicted the evidence, times how likely it was before, divided by the total chance of seeing that evidence at all.

The three parts have names. The prior P(H) is how likely the hypothesis was before the evidence. The likelihood P(E | H) is how often the evidence shows up when the hypothesis is true. The posterior P(H | E) is the updated answer. These are roles, not different kinds of number: all three are probabilities. The denominator P(E) comes from the sum rule, and it makes the posteriors of all the hypotheses add up to 1.

Bayes' rule matters most when the hypothesis is rare. Take a medical test for a condition that 1% of people have. It catches 90% of real cases, and it wrongly flags 5% of healthy people (its false-positive rate).

Worked example

Out of 10,000 people, 100 have the condition, and 90 of them test positive.

Of the 9,900 without it, 5% also test positive: 495 false alarms.

A positive result puts you in a pile of 90 + 495 = 585 people, and only 90 of them have the condition. P(H | +) = 90 / 585 ≈ 0.15.

A positive result is a false alarm more than five times as often as it's a real detection. The test isn't bad. The healthy group is 99 times bigger than the sick one, so 5% of it outnumbers 90% of the sick. Any detector for a rare event has the same problem, from fraud alerts to a classifier hunting for a rare class.
Move the prior and the error rates, read the posterior
Prior P(H), %
1
Sensitivity P(+|H), %
90
False positive P(+|¬H), %
5
Reset
Prior P(H) = 0.010; P(+|H) = 0.90; P(+|¬H) = 0.05. Joint: P(H,+) = 0.0090, P(¬H,+) = 0.0495. Evidence P(+) = 0.0585. Posterior P(H|+) = 0.0090 / 0.0585 = 0.1538. Per 10,000 people: 90 true positives, 495 false alarms.

The square is the whole population: drag the prior and the two error rates and watch the areas change. The prior splits its width into H (left) and ¬H (right), and within each column the shaded height is the chance of a positive test. The posterior is the purple area divided by the purple plus pink areas. The defaults are the worked example above: P(H|+) near 0.15, because of every 10,000 people, about 90 true positives share the positive pile with about 495 false alarms. Raise the prior to 20% and the posterior passes 0.8 with the same test.
