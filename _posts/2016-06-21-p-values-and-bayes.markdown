---
title: "Why P-Values are not the Error Rate"
layout: post
date: 2016-06-21 22:48
image: /assets/images/markdown.jpg
headerImage: false
tag:
- experiment design
- probability
- bayes
blog: true
author: davidberger
description: Or, when to conclude you're being swindled in the Mos Eisley cantina   
---
## What's a p-value?

P-values are pretty controversial, perhaps in part due to the fact that they are often-times misused and misunderstood. Lets use some coin flips to help explain the concept behind a p-value:

Lets say we have a coin and we flip it 30 times. Heads turns up 21 out of the 30 flips. We think this might mean that the coin is weighted. Looking at this series of coin flips as an experiment, we would consider the coin being fair our null hypothesis. We'll first look to p-values. The p-value answers the question, "What are the odds, given that the coin is fair, that an outcome this extreme or would happen?"


``` python
from scipy import stats

coin_flip_outcomes = [1]*21 + [0]*9
test_stats = stats.ttest_1samp(one_sample_data, .5)

print "The t-statistic is %.3f and the p-value is %.6f." % test_stats

#output: The t-statistic is 2.350 and the p-value is 0.025774
```

In this case, the p-value is about .026, which means that 2.6 percent of the time, a fair coin flipped 30 times will yield 22 or more heads.


## The misconception
The problem is that people often conclude that the p-value is the probability that the null hypothesis is true. They stretch the meaning of p-values to mean that "because this outcome would only happen 2.6 percent of the time if the coin were fair, we can guess that the coin is weighted, and 97.4 percent of the time we'll be right."

This is false. The p-value is not the error rate! 

In a nutshell, it's false because p-values only tell us the probability of the event occuring under the assumption that the null hypothesis (in this case a random fair coin flip) is true. But When we start talking about the probability of the coin being weighted, we are making 2 new assumptions not factored in by the p-value:
1. There are weighted coins that occur with some degree of frequency.
2. Those weighted coins will have some effect that skews the outcome.  

If this makes perfect sense to you, then that's great! Thanks for reading this post. To me though, the above logic didn't quite click at first. If p-values aren't explaining the error rate, what are they explaining? After watching a pretty snazzy presentation by Jake Vanderplas on [using hacking methods](https://www.youtube.com/watch?v=Iq9DzN6mvYA) to simulate statistical methods, I decided to try to understand this solution by programming coin flipping simulations. The effort was well worth it, and I now feel much better about the whole thing; I hope you will too.

## Simulating an experiment

The following simulation will instruct us how we can approach the question "If i flipped a coin 30 times and got heads, what is the probability my coin is rigged?" 

In doing so I hope it will illustrate with concrete examples what p-values can and cannnot tell us.

Our simulation will have the 2 things mentioned above that a p-value alone doesnt consider: The effect frequency and effect size. In the original question we don't have this information, but it's necessary to use it here to illustrate how to answer the question without using it later.

## The Experiment
We have 10,000 coins to flip. 9,000 are fair, and 1,000 are not. Lets assume that if the coin is not fair, that is to say, if it's weighted, the probability of it turning up heads is .75 and the probability of tails is .25. 

Each coin is flipped 30 times, and this constitutes a trial.

``` python
study_1 = CoinTossTrials(n_flips=30, n_trials=10000, coins_weighted=.1, weight_heads=75)
```

- **n flips** - the number of flips per trial
- **n_trials** - the total number of trials in the study 
- **coins_weighted** - the proportion of coins which are weighted and not fair coins 
- **weight_heads** - the percent liklihood a weighted coin will turn up heads.

## Results
Now that we've defined the experiment, lets simulate it and see the resulting distributions:

``` python
weighted_distributions, fair_distributions = study_1.population_trial_distributions()
```
``` python 
import matplotlib.pyplot as plt
import plotly.plotly as py

plt.hist(fair_distributions, bins=[n for n in range(30) if n%1==0])
plt.title("Distribution of Heads in Only Fair")
plt.xlabel("Number of Heads")
plt.ylabel("Frequency")
plt.show()

plt.hist(weighted_distributions, bins=[n for n in range(30) if n%1==0])
plt.title("Distribution of Heads in Only Weighted Trials")
plt.xlabel("Number of Heads")
plt.ylabel("Frequency")
plt.show()

plt.hist(weighted_distributions+fair_distributions, bins=[n for n in range(30) if n%1==0])
plt.title("Distribution of Heads in All Trials")
plt.xlabel("Number of Heads")
plt.ylabel("Frequency")
plt.show()
```

![Distribution across all fair coin flip trials](https://raw.githubusercontent.com/dberger1989/dberger1989.github.io/master/assets/images/post_images/dist_heads_fair.png)
![Distribution across weighted coin clips](https://raw.githubusercontent.com/dberger1989/dberger1989.github.io/master/assets/images/post_images/dist_heads_weighted.png)
![Distribution across total coin clips](https://raw.githubusercontent.com/dberger1989/dberger1989.github.io/master/assets/images/post_images/dist_heads_total.png)



``` python
## In a fair distribution, this is how many times you would get a distribution of 21 heads or more. 
fair_count = 0
for i in fair_distributions:
    if i >= 21:
        fair_count += 1
print '{} trials out of 9,000, or {}'.format(c, float(c)/9000)

#output: 193 trials out of 9,000, or 0.0214444444444
```






