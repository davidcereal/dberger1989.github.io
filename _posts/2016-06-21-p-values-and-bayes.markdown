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

In this case, the p-value is about .026, which means that 2.6 percent of the time, a fair coin flipped 30 times will yield 12 or more heads.


## The misconception
The problem is that people often conclude that the p-value is the probability that the null hypothesis is true. They stretch the meaning of p-values to mean that "because this outcome would only happen 2.6 percent of the time if the coin were fair, we can guess that the coin is weighted, and we'd only be wrong 2.6 percent of the time ."

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

- **`n flips`** - the number of flips per trial
- **`n_trials`** - the total number of trials in the study 
- **`coins_weighted`** - the proportion of coins which are weighted and not fair coins 
- **`weight_heads`** - the percent liklihood a weighted coin will turn up heads.

This gist contains the code for the experiment:
{% gist dberger1989/2e0c9dc3240d2aa554e21ad642a6815c %}

## Results
Now that we've defined the experiment, lets simulate it and see the resulting distributions:

``` python
weighted_distributions, fair_distributions = study_1.population_trial_distributions()
```
`weighted_distributions` and `fair_distributions` contain the outcomes from the trials. Since each trial was a coin flipped 30 times, each outcome is the number of times heads came up per 30 flips:

```python
print len(weighted_distributions)
#output: 1000

print len(fair_distributions)
#output: 9000

for trial in fair_distributions[:5]:
    print trial
#output:
#11
#14
#16
#13
#15
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

As we can see from the first chart, with a fair coin, the outcomes follow a pretty normal gaussian distribution. When the fair coins were flipped in trials of 30 tosses, they usually turned up heads 13-17 times. In the second chart, we have the simulated outcomes from only the weighted coins. With weighted coins, the trials usually yield 24+ heads. When we combine them we get the third chart, the heads distributions for all the trials. 

## P-values for hackers
Lets use simulation to determine how likely we would be to get 21/30 heads given that our coin was fair. This is our simulted version of the p-value. We can use our simulated experiment to arrive at something very close to the true p-value, without writing any equations, which I think is very cool. To do this we calculate how many trials had an outcome of 21 or more heads when the coin was fair:


``` python
## In a fair distribution, this is how many times you would get a distribution of 21 heads or more. 
fair_count = 0
for i in fair_distributions:
    if i >= 21:
        fair_count += 1
print '{} trials out of 9,000, or {}'.format(c, float(c)/9000)

#output: 193 trials out of 9,000, or 0.0214444444444
```

From our fair-coin trial sample, there is a 2.14 percent chance that a value of 21 or more would occur. This is close to the 2.6 p-value calculated earlier. We can conclude that 97.86 percent of the time, a fair coin would not have turned up 21/30 heads. But this isn't the probability that our coin is weighted! 

If we took the .0214 as the probability that our coin is not weighted, and the remaining 97.86 as the probability that it is, that would mean that out of all the coins in the trial, those with distributions of 21 or higher are weighted 97.86 (100-2.14) percent of the time. But we know that's not true, beacuse there is still all the weighted coins to factor in! 

``` python
## Simulate how many weighted coins would turn up 21/30 heads
weighted_count = 0
for i in weighted_distributions:
    if i >= 21:
        weighted_count += 1
print '{} trials out of 1,000 weighted coins, or {}'.format(weighted_count, float(weighted_count)/10000)
#output: 990 trials out of 1,000 weighted coins, or 0.099
```
99 percent of the weighted coins showed 21 or more heads. But if we were going hypothesize that our 21/30 coin was rigged, we wouldn't have been right 99 percent of the time. Because as we saw above, 2.14 percent of the fair coins showed this result too. To determine how many trials with outcomes of 21/30, we simply add together the weighted trials yeliding this result and the fair trials. 990+ 193 comes out to 1183. 1183 coin total coin tosses yielded 21 or more heads. However, only 990 of that outcome is weighted. Thus, if you would have seen 21/30 heads and you guessed the coin was rigged, you would have been correct 990 times out of 1183, or a rate of 83.68 percent of the time.

83.68 is the number we set out to find from the outset. While the p-value told us that an outcome of 21+ heads was very rare, happening only 2.57 percent of the time under randomness (2.14 in our simulation), you would still be wrong 16.32 percent of the time (100-83.68).

## But what about the original question?

In our original question, we didn't know how many, if any, coins were weighted, and we didnt know how heavily a weighted coin would turn the outcome to heads. We couldn't have run the simulation performed above. To get to the answer, we now need to go through bayes theorem:

To answer why, lets continue to try and answer the question. With a distirbution of 21 heads, can we determine the probability that our coin is weighted? 

The answer is no. We can only us p-scores to determine how out of the ordinary our result is under conditions of randomness. But that doesn't tell us what the probability is of seeing an out of the ordinary result to begin with. 
For example, if we were in the coin-flipping olympics, we would probably be very confident that there would be regulations in place to ensure that our coin was not weighted. In such a case, the probability would be much closer to zero. It would take a much more skewed distribution to convince us that our coin is biased.

However, if we were partaking in a coin flipping match in a cantina in Mos Eisely, of which some say you'll never find a more wretched hive or scum and villainy, there would be a much greater probability that 21/30 heads would be an indicator of the coin being tampered with. 

In the experiment outlines above, the probability if seeing a weighted coin was 10%. Thus, although 21 heads out of 30 is a rare occurance when using a fair coin, we still needed to factor in that coins in general were only 10 percent likly to be weighted. To take this prior probabilty into consideration, we'll be using bayes theorem.

Bayes Theorem is defined as:

$$ \color{RubineRed}{P(A|B)} \color{black}= \frac{ \color{BlueGreen}{P(B|A)}\color{purple}{P(A)} } { \color{BlueGreen}{P(B|A)}\color{purple}{P(A)} + \color{orange}{P(B|not A)}\color{orangered}{P(not A)} } $$

\\(A\\) is the probability of a coin being weighted.

\\(B\\) is the probability of the coin turning up heads 21/30 times.

So lets start plugging in values. 

\\(\color{RubineRed}P(A\|B)\\) is what we're trying to find: the probability of the coin being unfair given that we had a trial inwich 21/30 turned up heads. 

\\(\color{BlueGreen}P(B\|A)\\) is the probability of the coin turning up heads 21/30 times given that the coin is weighted. In our study, the weighted coins were weighted such that heads would come up 75 percent of the time. If you know this, you can do the sampling simulation we did above, inwhich we saw that out of 10,000 coins weighted 75-25 in favor of heads, 98.55 percent of them got a score as extreme of 21/30 heads or more. 

\\(\color{purple}P(A)\\) is the probability of a coin being unfair to begin with. Like the weight just mentioned, we didnt know this value in the original question. Here we'll use it for the sake of illustration, and then talk about what happens when we don't. The percent of weighted coins we used to run the trials was 10,000/100,000, or 10%. 

The first set of terms in the denominator is equivelant to the numerator. 

\\(\color{orange}P(B\|not A)\\) is the probability that the coin would turn up heads 21/30 times given that the coin is fair. This was the probability of 21/30 given complete randomness, akin to a p-value. We had this at 0.0214. 

We multiply this by \\(\color{orangered}P(not A)\\): The probability that the coin is not unfair, which in our case is .90, since 90% of the coins were fair. Again, we'll pretend for the sake of this example that we knew the weighted/fair ratio.

Pluggint in the values, this is our result:

$$ \color{RubineRed}{0.8346} \color{black}= \frac{ \color{BlueGreen}{(.99)}\color{purple}{(.10)} } { \color{BlueGreen}{(.99)}\color{purple}{(.10)} + \color{orange}{(.02144)}\color{orangered}{(.90)} } $$

## The intuition

The probability of an outcome can be defined as:
$$ \frac{ \color{blue}{possibilities for the outcome} } { \color{ForestGreen}{all possibilities} } $$

In our simulation, the outcome is the coin being weighted, given the fact that we have 21+ heads. 

The outcome we're testing for is a weighted coin that turns up 21+30 heads. How often does this happen? We know that in our experiment, a \\(\color{blue}{weighted coin will turn up 21+ heads 99 percent of the time, but we also know that only 10 percent of coins are weighted}\\). So there is a \\(\color{BlueGreen}{(.99)}\color{purple}{(.10)}\\) probability of our coin being weighted and turning up 21+ heads. 

The denominator, all possible outcomes, is \\(\color{ForestGreen}{the probability that *any* coin would turn up 21+ heads}\\). So we add the probability of a weighted coin turning up 21+ heads to the probability of a non-weighted coin turning up 21+ heads: \\(\color{BlueGreen}{(.99)}\color{purple}{(.10)} + \color{orange}{(.02144)}\color{orangered}{(.90)}\\)


