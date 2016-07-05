---
title: "Intuitive A/B Testing with Bayes"
subtitle:
layout: post
date: 2016-07-05 09:48
image: /assets/images/markdown.jpg
headerImage: false
tag:
- A/B testing
- experiment design
- bayes

blog: true
author: davidberger
description: 
---
## A/B testing isn't as easy as it seems

A/B testing is common practice in product design. It’s the comparison of version (A), to another version (B). Figuring out whether A or B is superior seems like it should pretty straightforward: if we wanted to measure conversion rates across different versions of a web page, we could split users randomly into 2 groups and serve them 2 different web designs. Whichever version leads to more conversions could be deemed superior. It’s that easy, right? Nope. 

The complicating factor is luck. It’s entirely possible that the version which had the highest conversion rate is actually inferior to the other, and only by sheer luck did it produce better results in our samples. So how can we know for sure whether A or B is the more effective layout?

## Conducting the experiment
Let’s use a hypothetical situation. Imagine we run a racing broom e-commerce site to supplement our store on Diagon Alley.  We are considering switching the color scheme of the site: One version uses Slytherin colors of green and silver, and one uses Gryffindor colors of red and gold. Otherwise, both versions are exactly the same. 

![markdown image](https://67.media.tumblr.com/070bbf5e92c5ccb3470bae7abe759afc/tumblr_nn0ulgpQEN1sfmnojo1_500.jpg)

We run an experiment where we deploy the Gryffindor version of the site to 300 users and the Slytherin version to 300 users. The Gryffindor version led to 105 conversions, and the Slytherin version to 90. 

| Version        ||| Views  || Conversions || Conversion Rate|
| :-------- |||:----------:||:--------:||:--------:|
| Slytherin    ||| 300|| 90|| .30 |
| Gryffindor      ||| 300 ||105|| .35 |

But, before we green light version Gryffindor, it would be nice to get an idea how confident we really should be about it’s superiority. It’s possible that version Gryffindor is indeed superior, but it’s also possible that over time we’d see both versions perform exactly the same, and the extra 15 conversions Gryffindor colors made were just luck. It’s also possible that version Slytherin is actually superior, but Gryffindor was lucky, or Slytherin was unlucky, or some combination of the two. If we’re going to act, we should act smart, and to do this we need to estimate the probabilities of each of these possibilities. 

## Beta distributions are awesome

[Previously](https://dberger1989.github.io/p-values-and-bayes/), we simulated coin flip outcomes by writing python functions that generated random numbers corresponding to the weight we set for heads and tails. Another way to simulate this generative model is by using the beta distribution. 

The beta distribution is a distribution of probabilities shaped by parameters alpha and beta.
It’s used to determine the true rate after observing \\(\alpha\\)(alpha) successes and \\(\beta\\)(beta) failures. Given the alpha and beta, we get a distribution of the potential true conversion rates:

```python
## Define parameters
n_trials = 300
alpha = 90
beta = alpha-n_trials

## Plot a Probability Density Function of the beta distribution
x = np.linspace(0,1,1000)
beta_gryff = stats.beta(alpha, beta).pdf(x)

plt.plot(x,bet)
plt.title("PDF of Gryffindor Beta Distribution")
plt.xlabel("Conversion Rate")
plt.ylabel("Density")
plt.show()

```

![markdown image](/assets/images/post_images/ab_testing/Gryff_beta_dist.svg)

This chart shows us that the true conversion rate is most likely 3.5, but that it’s also possible, albeit less likely, that the true conversion rate was higher or lower. The beta distribution gives us those probabilities. Harkening back to our coin flipping experiments, if we had a coin that showed up 21/30 heads, it’s possible that the coin is weighted .70 to show heads, but it’s also entirely possible that the coin isn’t weighted at all.

If we use fewer trials, and keep the ratio the same, we get a different shape to our plot:

![markdown image](/assets/images/post_images/ab_testing/Gryff_beta_dist_weaker.svg)

This is because with fewer previous occurrences guiding the beta distribution, there is more room for the possibility of other true conversion rates, since at lower numbers of observations, luck has a greater role in the outcome. 

## Sampling from a beta distribution (extremeley useful!)
You can use the beta distribution in Bayes theorem to arrive at the probability that Gryffindor colors indeed perform better than Slytherin’s in the long run, but the math is kind of hairy: 
{\rm Pr}(p_B > p_A) = \sum_{i=0}^{\alpha_B-1}{\frac{B(\alpha_A+i, \beta_B + \beta_A)}{(\beta_B+i)B(1+i,\beta_B)B(\alpha_A,\beta_A)}}

The details of this equation can be found [here](http://www.evanmiller.org/bayesian-ab-testing.html). 

We can avoid using this equation and keep things within the bounds of intuition by sampling. With numpy, we can randomly sample from a beta distribution:

```python
## Draw 1000 samples from the Gryffindor’s version's beta distribution
n_trials = 1000
gryff_beta_samples = np.random.beta(105, 195, size=n_trials) 
```
And here is the distribution of that sampling:

![markdown image](/assets/images/post_images/ab_testing/gryffindor_beta_samples.svg)

This histogram shows us the distribution of 1000 random samples taken from the Gryffindor version's beta distribution.

This histogram is interesting because we can clearly see that given 105 conversions out of 300, it’s still entirely possible that the true conversion rate is .30 (Slytherin’s conversion rate) or even lower, and the Gryffindor version just happened to have a lucky stretch of conversions. We can quantify the probability of Gryffindor’s true rate being equal or worse than Slytherin’s result rate by counting how many of the samples drawn from Gryffindor’s beta distribution were .30 or lower: 

```python 
slyth_alpha = .30
count = 0
for samp in gryff_beta_samples:
    if samp <= slyth_alpha:
        count += 1
print float(count)/n_trials
#output: 0.032
```

So given the Gryffindor version’s results, it’s true rate would only match or be worse than the .30 put up by Slytherin’s version approximately 3.2 percent of the time. 

## Further accounting for luck

But just as it’s possible Gryffindor’s results were lucky,  it’s also possible Slytherin’s results were unlucky. Perhaps the Slytherin version’s true conversion rate over the long run will prove to be even higher than the 105/300 rate of Gryffindor. Furthermore, perhaps *both* the Slytherin version’s true rate is actually higher (but was unlucky), and Gryffindor’s true rate is actually lower (but got lucky). We can determine the probability of this by comparing the distributions to one another and seeing how often either of them wins out: 

``` python
gryff_superior_count = 0
for i, slyth_samp in enumerate(slytherin_beta_dist_samples):
    gryff_samp = gryffindor_beta_dist_samples[i]
    if gryff_samp > slyth_samp:
        gryff_superior_count += 1
print float(gryff_superior_count)/n_samples
#output: .912
```
So when 1000 potential conversion rates are chosen randomly from each version’s beta distribution, 91.2 percent of the time, Gryffindor won out, and we can thus put the probability that Gryffindor is the superior version at 91.2 percent. 

## Quantifying the magnitude of difference

Knowing how likely it is that one version is superior to another is important in A/B testing, but it's also to know how how much better we can expect the version to be if we’re right and how much worse it might be if we’re wrong.  If we’d like to know how the magnitude of the Gryffindor version’s superiority over Slytherin, we can simply plot the differences seen between the samples in a Cumulative Distribution Function:

![markdown image](/assets/images/post_images/ab_testing/beta_sample_differences.svg)

Let’s do a quick sanity check: the median is approximately 5 percent, so we’d on average expect there to be a 5 point increase in conversion rate from Slytherin to Gryffindor. This plays out in our original samples: 105/300 Gryffindor conversions is a rate of .35,  and Slytherin’s 90/300 makes for .30, and the difference between them is 5 percent points. 

Furthermore, from this plot of the cumulative distributions, we can see that approximately 10 percent of the time, Slytherin will actually have a higher true conversion rate. We saw this ourselves when we saw that when pulling samples from both distributions randomly, about 10 percent of them had Slytherin with the higher conversion rate, although as we can see from the plot, Slytherin never beats Gryffindor by more than 5 points. 

90 percent of the time, Gryffindor’s version will beat Slytherin’s, and 50 percent of the time, it will have a conversion rate 5 points higher.

## Updating with a Prior

The last great thing about beta distributions is that they make it extremely easy for us to update our prior assumptions. For example, if over the next few days we continue our experiment and get another 18 out of 50 conversions for Gryffindor and another 16 out of 50 conversions for version Slytherin, we can simply update the beta distributions by adding to the alpha and beta parameters:

Since these new results were pretty similar to one another, there is now more overlap between the potential true conversion rates, and we’d likely be less certain that version Gryffindor is indeed better. To quantify this uncertainty, we would simply draw samples from both distributions  as we did above but using the updated beta distributions.

Let’s now imagine we go with the Gryffindor version of the site, and over the course of the next 6 months we only send users that version. Over that span, we generate 3200 views and 1100 conversions. After more testing, we decide to go with a new version of the site, version Hufflepuff. Our testing on Hufflepuff yielded 135/300 conversions. What should we expect our new conversion rate to be, after implementing the Hufflepuff version of the site?

If we simply take Hufflepuff’s rate, we get `135/300=.45`. But this ignores our previous experience in conversion rates, where our rate was `1100/3200=.34`. It’s entirely possible that the .45 is the true rate of the new version, but, it’s also possible that luck was involved. How can we predict what the new true rate will be while also taking into consideration the prior norm? 

Adding our prior experience/expectations is equivalent to using the Bayesian Prior mentioned in our [P-Vales/Bayes article](https://dberger1989.github.io/p-values-and-bayes/). To do this, we combine our prior rate of conversion to the current Hufflepuff rate by adding the prior alpha and betas together with the new ones in the beta distribution:

```python
prior_alpha = 1100
prior_beta = 2200

huff_alpha = 135
huff_beta = 165

beta_prior=beta(prior_alpha, prior_beta).pdf(x)
beta_huff=beta(huff_alpha, huff_beta).pdf(x)
beta_huff_with_prior=beta(prior_alpha+huff_alpha, prior_beta+huff_beta).pdf(x)
```

![markdown image](/assets/images/post_images/ab_testing/beta_dist_huff_prior.svg)


Adding the prior makes our expectations more pessimistic than if we had just considered the new Hufflepuff version alone. But perhaps we feel very strongly that these new Hufflepuff colors are awesome and much better than the old Gryffindor ones. We can strengthen the influence of the Hufflepuff version by downplaying the influence of the prior. The fewer prior observations we add to the Hufflepuff version’s observations in the beta distribution, the less influence the prior will have. Let’s only keep 1/3 of the prior expectations, but still maintain the same rate:

```python 
beta_huff_with_weak_prior=beta(prior_alpha/3 + huff_alpha,  prior_beta/3 +huff_beta).pdf(x)
```

![markdown image](/assets/images/post_images/ab_testing/beta_dist_huff_weak_prior.svg)

Following the same logic, if we continued to use the Hufflepuff version, the number of observations we accumulate will increase, and the prior will be less and less influential in predicting the true conversion rate:

``` python 
beta_strong_huff_with_prior=beta(prior_alpha+ huff_alpha*10, prior_beta+huff_beta*10 ).pdf(x)
```
![markdown image](/assets/images/post_images/ab_testing/beta_dist_large_huff_prior.svg)


It’s important to note that since we are increasing the number of observations, our range would also get smaller, because with more observations comes a greater degree of certainty. 

So there you have it. When running an A/B test, beta distributions and using a Bayesian Prior are great tools to use. As in our coin-flipping simulations, sampling from the distributions we are examining is an extremely intuitive way to get error rates. 











