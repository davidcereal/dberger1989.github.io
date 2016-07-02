---
title: "Using Python and Bayes to Better Explain P-Values"
subtitle: "or, how to determine if you're being swindled in a space-port cantina"
layout: post
date: 2016-06-21 22:48
image: /assets/images/markdown.jpg
headerImage: false
tag:
- experiment design
- probability
- bayes
blog: true.
author: davidberger
description: Or, when to conclude you're being swindled in the Mos Eisley cantina   
---
## P-primer

P-values are pretty controversial, perhaps in part due to the fact that they are often-times misused and misunderstood. But at face value, they're actually pretty simple. Here's a quick coin-flip example to help explain the concept behind a p-value:

Let's say we have a coin and we flip it 30 times. Heads turns up 21 out of the 30 flips. We think this might mean that the coin is weighted to land on heads. Looking at this series of coin flips as an experiment, our coin being fair would be the null hypothesis. The alternate hypothesis, is that the coin is weighted to land on heads.

A P-value is the probability that given our null hypothesis, an outcome as extreme or more would occur. In our context, it answers the question "What is the probability, given that the coin is fair, an outcome this extreme (21/30 heads) or more would occur?"


``` python
from scipy import stats

coin_flip_outcomes = [1]*21 + [0]*9
test_stats = stats.ttest_1samp(one_sample_data, .5)

print "The t-statistic is %.3f and the p-value is %.6f." % test_stats

#output: The t-statistic is 2.350 and the p-value is 0.025774
```

In this case, the p-value is about .026, which means that 2.6 percent of the time, a fair coin flipped 30 times will yield 21 or more heads. 


## The big misconception
The problem is that people often conclude that the p-value is the probability that the null hypothesis is true. They stretch the meaning of p-values to mean that "because this outcome would only happen 2.6 percent of the time if the coin were fair, we can guess that the coin is weighted, and we'd only be wrong 2.6 percent of the time ."

This is false. The p-value is not the error rate! 

In a nutshell, it's false because p-values only tell us the probability of the event occurring under the assumption that the null hypothesis (in this case a coin being fair) is true. But when we start asking about the probability of the coin being *weighted*, we are making 2 new assumptions not factored in by the p-value:

1. There are weighted coins that occur with some degree of frequency.
2. Those weighted coins will have some measurable effect that skews the outcome of a flip.  

The only way to measure the probability that our alternate hypothesis is true is by having some idea of how frequently weighted coins are found in general, and to know the degree to which they effect the outcome. P-values ignore this nuance, because they only tell us how likely or unlikely an outcome would occur under the null hypothesis.  

When I first learned about p-values in this way, it made logical sense, but there was an intuitive element I was still missing. P-values are often used to quantify statistical significance, but in our coin flipping scenario, they don't seem capable of explaining much with regards to our hypothesis that the coin is weighted. If they aren't explaining the error rate, why are they so important? 

After watching a pretty snappy presentation by Jake Vanderplas on [using hacking methods](https://www.youtube.com/watch?v=Iq9DzN6mvYA) to simulate statistical methods, I decided to try to better understand the limitations and purpose of p-values by programming coin flip simulations. The effort was well worth it, and I now feel much better about p-values, error rates, and even probability in general. So keep reading, and I hope you will too. By using a real life experiment of our own design we'll have a clear view of the role p-values play in evaluating probabilities.

## Simulating coin flips with python

The question our simulation will be answering is: 

*If we rejected the null hypothesis (that the coin is fair) when a trial results in 21+ heads, what would our error rate be?* 

Said in a less sciencey way, it would be:

*If we flipped a coin 30 times and got 21+ heads, what is the probability our coin is rigged to land on heads?*

Our simulation will use the 2 things mentioned above that a p-value alone doesnt consider: The frequency weighted coins occur, and how heavily a weighted coin influences the probability a coin will turn up heads. In the original question we don't have this information, but it's necessary to use it here so that we can later illustrate how to answer the question without using it.

## Defining the experiment
In our experiment we have 10,000 trials, each using a coin that is either weighted or fair. 9,000 coins are fair, and 1,000 are not. Let's assume that if the coin is weighted, the probability of it turning up heads is .75 and the probability of tails is .25. 

Each coin is flipped 30 times, and this constitutes a single trial.

``` python
study_1 = CoinTossTrials(n_flips=30, n_trials=10000, coins_weighted=.1, weight_heads=75)
```

- **`n flips`** - the number of flips per trial
- **`n_trials`** - the total number of trials in the study 
- **`coins_weighted`** - the proportion of coins which are weighted and not fair coins 
- **`weight_heads`** - the percent likeliyhood a weighted coin will turn up heads.

This gist contains the code for the experiment:
{% gist dberger1989/443e02e11d14a34f45e8b71bea5b5f44 %}

## Results
Now that we've defined the experiment, let's simulate it and see the resulting distributions:

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

![Distribution across all fair coin flip trials](/assets/images/post_images/dist_heads_fair.png)
![Distribution across weighted coin clips](/assets/images/post_images/dist_heads_weighted.png)
![Distribution across total coin clips](/assets/images/post_images/dist_heads_total.png)

As we can see from the first chart, with a fair coin, the outcomes follow a pretty normal gaussian distribution. In trials using fair coins, heads usually came up 13-17 out of the 30 flips. In the second chart, we have the simulated outcomes from only the weighted coins. With weighted coins, the trials usually yield around 23+ heads. When we combine them we get the third chart, the heads distributions for all the trials. 

## P-Values for hackers
Let's use the results from our trials to determine how likely we would be to get 21+/30 heads given that our coin was fair. Since our coin being fair is the 2-sided coin equivelent to randomness, this is our simulated version of the p-value. We can thus use our simulated experiment to arrive at something very close to the true p-value, without writing any complicated equations, which I think is very cool. To do this we calculate how many trials had an outcome of 21 or more heads when the coin was fair:


``` python
## In a fair distribution, this is how many times you would get a distribution of 21 heads or more. 
fair_count = 0
for trial in fair_distributions:
    if trial >= 21:
        fair_count += 1
print '{} trials out of 9,000, or {}'.format(fair_count, float(fair_count)/9000)
#output: 214 trials out of 9,000, or 0.0237777777778
```

From our fair-coin trial sample, there is a 2.37 percent chance that a value of 21 or more occurred. This is close to the 2.57 p-value calculated earlier. We can conclude that about 97.63 percent of the time, a fair coin would not turn up 21+/30 heads. But remember, this isn't the probability that our coin is weighted! Let's illustrate why:

If we took .0237 as the probability that our coin is not weighted, and the remaining 97.63 percent as the probability that it is, that would mean that out of all the coins in the trial, those with distributions of 21 or higher are weighted 97.63 percent of the time. But that can't possibly be true, because *all of the coins we are currently discussing are only fair coins*. If you would have guessed that any of these coins were weighted, you would have been wrong 100% of the time. We need to analyze the weighted coin outcomes as well to get to the full picture:

``` python
## Simulate how many weighted coins would turn up 21/30 heads
weighted_count = 0
for trial in weighted_distributions:
    if trial >= 21:
        weighted_count += 1
print '{} trials out of 1,000 weighted coins, or {}'.format(weighted_count, float(weighted_count)/10000)
#output: 800 trials out of 1,000 weighted coins, or 0.8
```
80 percent of the weighted coins showed 21 or more heads. But if we were going to hypothesize that our 21/30 coin was rigged, we wouldn't have been right 80 percent of the time. Because as we saw above, 2.37 percent of the fair coins showed this result too. To determine how many trials with outcomes of 21/30, we simply add together the weighted trials yeilding this result and the fair trials. 800 + 214 comes out to 1014. 1014 coin total coin tosses yielded 21 or more heads. However, only 800 of that outcome is weighted. Thus, if you would have seen 21/30 heads and you guessed the coin was rigged, you would have been correct 800 times out of 1014, or a rate of 78.90 percent of the time.

78.90 is the number we set out to find from the outset. While the p-value told us that an outcome of 21+ heads was very rare, happening only 2.57 percent of the time under randomness (2.37 in our simulation), if we predicted that our coin was weighted, we would still be wrong 21.10 percent of the time (100-78.90).

## Enter uncertainty

In our original question, we didn't know how many, if any, coins were weighted, and we didnt know how heavily a weighted coin would turn the outcome to heads. We thus wouldn't have been able to run the simulation performed above. So given this lack of prior knowledge, what could we have done?

All we have to work with in our original question is the number of flips and the number of heads. This alone is enough to yield a p-value. But we couldn't use p-values alone to arrive at our answer, because we can only use p-values to determine how out of the ordinary our result is under conditions of randomness, and that doesn't tell us what the probability is of seeing an out of the ordinary result when conditions are not necessarily random, which is precisely what we suspect if we are wondering if the coin is weighted. 

In our simulation, this probability was 10 percent, but again, we are now trying to work out the problem without having this prior knowledge. Given this inherent uncertainty, the only way to go forward is to start making some educated guesses.

For example, if we were in the Galactic Coin-Flipping Olympics, we would probably be very confident that there would be regulations in place to ensure that our coin was not weighted. In such a case, the probability would be much closer to zero. It would take a much more skewed distribution to convince us that our coin is biased.

However, if we were partaking in a coin flipping match in a place where our opponents are less trustworthy, say the Mos Eisely cantina, there would be a much greater probability that 21/30 heads would be an indicator of the coin being tampered with. 

![Markdowm Image](https://raw.githubusercontent.com/dberger1989/dberger1989.github.io/master/assets/images/cantina.jpg)
<figcaption class="caption" style="margin-top:-20px"><i>"You'll never find a more wretched hive of scum and villainy"</i><br><br></figcaption>

In order to factor in this estimated prior probability of the coin being rigged, we need to use Bayes' theorem. 

## Plugging in to Bayes 

Every article that explains Bayes' theorem is obligated to include a drawing of his likeness:

![Bayes](https://raw.githubusercontent.com/dberger1989/dberger1989.github.io/master/assets/images/post_images/Thomas_Bayes.gif){:height="420px" width="560px"}
<figcaption class="caption" style="margin-top:-20px">more like bae's theorem, amiright?<br><br></figcaption>


Bayes' theorem is defined as:

$$ \color{RubineRed}{P(A|B)} \color{black}= \frac{ \color{BlueGreen}{P(B|A)}\color{purple}{P(A)} } { \color{BlueGreen}{P(B|A)}\color{purple}{P(A)}~\color{black}{+}~\color{orange}{P(B|not~A)}\color{orangered}{P(not~A)} } $$

\\(A\\) is the probability of a coin being weighted.

\\(B\\) is the probability of the coin turning up heads 21/30 times.

\\(\color{RubineRed}P(A\|B)\\) is what we're trying to find: the probability of the coin being unfair given that we had a trial in which 21/30 turned up heads. 

\\(\color{BlueGreen}P(B\|A)\\) is the probability of the coin turning up heads 21/30 times given that the coin is weighted. In our study, the weighted coins were weighted such that heads would come up 75 percent of the time. If you know this, you can do the sampling simulation we did above, where we saw that out of 10,000 coins weighted 75-25 in favor of heads, 99.0 percent of them got a score as extreme as 21/30 heads or more. In Baysian terminology, this is called the likelihood, since it's the likelihood that under the conditions of our alternate hypothesis (the coin is weighted), we would see our outcome (31+ heads).  

\\(\color{purple}P(A)\\) is the probability of a coin being unfair to begin with. Like the weight just mentioned, we didn't know this value in the original question. Here we'll use it for the sake of illustration, and then talk about what happens when we don't. The percent of weighted coins we used to run the trials was 1,000/10,000, or 10%. This is known as the prior, since it's the prior probability of seeing the alternate hypothesis in general. In our case it's the prior probability that any given coin is weighted. 

The first set of terms in the denominator is equivelent to the numerator. 

\\(\color{orange}P(B\|not~A)\\) is the probability that the coin would turn up heads 21/30 times given that the coin is fair. This was the probability of 21/30 given complete randomness, akin to a p-value. We had this at 0.0237. 

\\(\color{orangered}P(not~A)\\) is the probability that the coin is not unfair, which in our case is .90, since 90% of the coins were fair. Again, we'll pretend for the sake of this example that we knew the weighted/fair ratio.

Plugging in the values, this is our result:

$$ \color{RubineRed}{0.7890} \color{black}= \frac{ \color{BlueGreen}{(.80)}\color{purple}{(.10)} } { \color{BlueGreen}{(.80)}\color{purple}{(.10)}~\color{black}{+}~\color{orange}{(.0237)}\color{orangered}{(.90)} } $$

## Wait, what? 

The probability of a desired outcome can be defined as:

$$ \frac{ \color{blue}{ways~for~desired~outcome~to~occur} } { \color{green}{all~possible~outcomes} } $$

In our simulation, the outcome is the coin being weighted, given the fact that we have 21+ heads. 

The outcome we're testing for is a weighted coin that turns up 21+/30 heads. How often does this happen? We know that in our experiment, a <span style="color:blue">weighted coin will turn up 21+ heads 80 percent of the time, but we also know that only 10 percent of coins are weighted</span>. So there is a \\(\color{blue}{(.80)}\color{blue}{(.10)}\\) probability of our coin being weighted and turning up 21+ heads. 

The denominator, all possible outcomes, is <span style="color:green">the probability that *any* coin would turn up 21+ heads</span>. So we add the probability of a weighted coin turning up 21+ heads to the probability of a non-weighted coin turning up 21+ heads: \\(\color{green}{(.80)}\color{green}{(.10)} + \color{green}{(.0237)}\color{green}{(.90)}\\)

The result, \\(\color{RubineRed}{(0.7890)}\\), is how often we'd be right if we determined a 21+ heads coin to be weighted. 

## So this probabilistic statistician walks into a bar...

Remember, in our original question, we didn't have the probability that any random coin was unfair (the percent of all coins that were weighted), and we also didn't have the degree to which a coin being weighted would determine the coin being heads (75-25 advantage). But now that we know how to use Bayes' theorem, we can instead plug in estimated values for those elements.

The key really is to know whether you're in the Galactic Coin Flipping Olympics or the Mos Eisley cantina. If the experiment were conducted in the former, you might have guessed the probability of the coin being rigged to be pretty low, perhaps .05, in which case the equation would be: 

$$ \color{RubineRed}{0.6398} \color{black}= \frac{ \color{BlueGreen}{(.80)}\color{purple}{(.05)} } { \color{BlueGreen}{(.80)}\color{purple}{(.05)}~\color{black}{+}~\color{orange}{(.0237)}\color{orangered}{(.95)} } $$

You would thus only consider there to be a 63.98 percent chance the coin is actually rigged when we assumr 5% of the coins to be rigged. And what happens when we change the weight of a rigged coin? If you thought that when the coin was rigged, the rigging would be less obvious, you might estimate that a rigged coin would cause heads to turn up 55 percent of the time--a much lower value than the .75 we have been using. In such a scenario, for the 500 coins that would be weighted (remember, we are also assuming .05 probability of finding a weighted coin), 97 of them would would have an outcome 21/30 heads or more extreme: 

```python
## define experiment with new parameters
study_2 = CoinTossTrials(n_flips=30, n_trials=10000, coins_weighted=.05, weight_heads=55)

## simulate experiment
weighted_distributions_2, fair_distributions_2 = study_2.population_trial_distributions()

## count how many weighted coin trials yielded 21+ heads
weighted_count = 0
for trial in weighted_distributions_2:
    if trial >= 21:
        weighted_count += 1
print '{} trials out of {} weighted flips, or {}'.format(weighted_count, 
                                                         study_2.n_trials*study_2.coins_weighted, 
                                                         float(weighted_count)/(study_2.n_trials*study_2.coins_weighted))
#output: 31 trials out of 500.0 weighted flips, or 0.062
```
The probability would thus be calculated as:

$$ \color{RubineRed}{0.1210} \color{black}= \frac{ \color{BlueGreen}{(.062)}\color{purple}{(.05)} } { \color{BlueGreen}{(.062)}\color{purple}{(.05)}~\color{black}{+}~\color{orange}{(.0237)}\color{orangered}{(.95)} } $$


So we see that when we decrease our guess as to the advantage lent by the weighted coin, the probability that a 21+/30 heads trial is actually weighted decreases as well, in this case from 63.98 when the weight was 75-25, to 12.10 percent now that the advantage has been reduced to 55-45. This should be somewhat intuitive. When the coin was weighted 75-25, 21+/30 heads was a very likely outcome (.80), whereas when the weight is 55-45, that outcome is only 6.2 percent likely. We should thus be less inclined to suggest the coin is rigged, and more inclined to explain the outcome away as a matter of chance. 

Let's work in the opposite direction now. If i were in the Mos Eisley cantina, I would certainly expect my many unscrupulous opponents to try to use weighted coins to give them an advantage. Let's say 35 percent. However, these swindlers are not foolish, and they would most likely be in it for the long haul. I'd assume that they would use a coin that gives them no more than a 55-45 advantage. If my opponent got 21+/30 heads in such a scenario, we get an equation of:

$$ \color{RubineRed}{0.5848} \color{black}= \frac{ \color{BlueGreen}{(.062)}\color{purple}{(.35)} } { \color{BlueGreen}{(.062)}\color{purple}{(.35)}~\color{black}{+}~\color{orange}{(.0237)}\color{orangered}{(.65)} } $$

Thus, if i'm on Mos Eisely and my opponent gets 21/30 heads, given my prior assumptions I would guess that there is a 58.48 percent chance that he is swindling me. I might want to wait to see a few more flips before drawing any conclusions.  

## P-values revisited

We now should have a pretty good idea that p-values alone don't tell us how confident to be that an outcome is the result of an effect. For that we need the prior probability of seeing the effect (how often we expected to see a rigged coin) and the likelihood that the outcome would occur given the effect (the weight the coin was rigged).

So what are p-values alone good for? Simply put, they tell us when the outcomes we're seeing are weird. It's obvious that 29/30 heads is a strange outcome. But how strange is 19? A p-value tells us how likely or unlikely an outcome we are seeing is, given that there is no effect. However, while the outcome before us might be unlikely, the probability of any effect being present may be even more unlikely, perhaps even close to zero. In such a case we should still be confident the null hypothesis is true. If there are no alternate hypotheses that are remotely likely, then there's a strong probability that the strange outcome we are seeing is simply a fluke.
