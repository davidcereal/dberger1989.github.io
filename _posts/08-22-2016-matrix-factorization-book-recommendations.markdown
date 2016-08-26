---
title: "Interactive Book Recommendations Using Matrix Factorization "
subtitle:
layout: post
date: 2016-08-22 22:48
image: /assets/images/markdown.jpg
headerImage: false
tag:
- SVD
- Matrix Factorization 
- Machine Learning
- Recommendation Engines
blog: true
author: davidberger
description:    
---
1. [BookBrew](#bookbrew)
2. [Defining the Product](#defining-the-product)
3. [Web Scraping for Data](#web-scraping-for-data)
4. [Latent Feature Extraction](#latent-feature-extraction)
5. [Implementing the SVD](#implementing-the-svd)
6. [Penalizing Popularity](#penalizing-popularity)
7. [KNN Using Book Tags](#knn-using-book-tags)
8. [Tuning Singular Values using KNN](#tuning-singular-values-using-knn)
9. [Updating Results Through User Feedback](#updating-results-through-user-feedback)
10. [Recommendation Systems are Awesome](#recommendation-systems-are-awesome)

## BookBrew

Over the past few months I’ve worked to create an interactive book recommendation web-app, [BookBrew](http://bookbrew.io). To get quality and real-time book recommendations, I used a hybrid recommendation approach combining matrix factorization and content-based similarity ranking. In this post, I’ll speak about the methodology and algorithms as well as some of the interesting choices I faced along the way. 

<img src ="/assets/images/post_images/book_rec_post/bookbrew_screenshot.jpg" style="width:560px"/>

## Defining the Product

In setting out to build a book recommender, I had some specific goals in mind. I love books, but I spend hours toiling to find my next read. I would scour “Top 50” lists and Goodreads, trying to find a genre that spoke to the mood I was in, and also seemed to be recommended by a source with similar taste. I found myself thinking things like “If only I could combine that great George Saunders book with that great David Sedaris but also add some magic..” And that became the product goal: to give people the ability to combine books and features and create the perfect book recipes for finding their next book.  

Defining this goal was important, because it shifted the focus of the problem from being just about delivering quality recommendations to delivering quality recommendations that were also similar to the input. If a user submitted The Great Gatsby and Pride and Prejudice as input, I didn’t simply want to return results that readers of those books would like too, but rather books they’d love that also share characteristics with both of those inputs.

## Web Scraping for Data

In all, I scraped features and ratings for over 500,000 users and 100,000 books. At this point, the problem looks a little bit like the [Netflix Prize](https://en.wikipedia.org/wiki/Netflix_Prize) competition, in that we have users and ratings and we want to fill in missing values. In our case, the missing values we are trying to fill are all the books not read by the enduser submitting his/her book recipe. However, there is one extremely relevant difference between the web-app and the Netflix competition, and that’s time. The appeal of creating a recipe-based recommendation engine is that a user can see results of his/her input right away, and change the recipe if it’s not working for them. I needed whichever algorithms I implemented to deliver its results in real-time, almost instantaneously. 

While the necessity of real-time recommendations discounted the vast, if not full majority of the Netflix submissions, there was a huge wealth of knowledge to be mined from them, particularly with regards to the use of matrix factorization using Singular Value Decomposition (SVD) as a way of filling in missing ratings.

## Latent Feature Extraction

SVD is a form of matrix factorization that breaks down a matrix into 3 smaller matrices. It’s often used in Principle Component Analysis (PCA) as a means of discovering the latent features in a dataset. 

PCA and SVD both determine the orthogonal dimensions of greatest variance and deliver eigenvectors (PCA can use SVD for this, or the eigenvectors from a covariance matrix). Consider this chart plotting user ratings for Harry Potter and Lord of the Rings:

<img src ="/assets/images/post_images/book_rec_post/lotr_vs_hp_scatter.svg" style="width:560px"/>


Implicitly, each user is described in terms of both his/her like of Harry Potter and LOTR. But this is actually pretty redundant. Since these 2 books are so strongly correlated, it makes better sense to simply describe them both together, and if we wanted to put a label on that generality, we could call it “fantasy.” Thus, each user can be described not as a function of how much they like or dislike Harry Potter and LOTR, but how much they like fantasy. We can quantify this generality with the use of eigenvectors, which draw lines through the orthogonal dimensions and capture the greatest variance:

<img src ="/assets/images/post_images/book_rec_post/lotr_vs_hp_eigens.svg" style="width:560px"/>


If we describe our users in terms of their relation to the magenta line, we shift from a 2 dimensional representation of users and their specific books to a 1 dimensional representation of users and  their love of fantasy:


<img src ="/assets/images/post_images/book_rec_post/fantasy_fandom_scale.svg" style="width:560px"/>



We lose the variation in the dimension represented by the cayenne line, the second eigenvector, but that information is really just noise. Because we were able to map the data in fewer dimensions, PCA, SVD, and other forms of dimensionality reduction are popular in data compression, and because we were able to portray it in terms of latent features (in this case fantasy fandom), these tools are invaluable for feature extraction. 

## Implementing the SVD 

SVD allows us to extract both latent book features and latent user relationships to recreate the rating process. Let’s imagine our matrix is comprised of 3 users and 3 books:

|      ||| Sherlock Holmes  || Harry Potter || Lord of the Rings|
| :-------- |||:----------:||:--------:||:--------:|
| David   ||| 4|| 0|| 0 |
| Ben      ||| 0 ||5|| 4 |
| Joanna    ||| 0 ||4|| 5 |

From this chart, it seems that people who like Harry Potter are more likely to also like Lord of the Rings, and vice versa, but they don’t appear to be too keen to pick up Sherlock Holmes, and when they do, they don’t rate it too highly.

If you wanted to describe what kind of book Lord of the Rings is, you might tell me that other people who like Harry Potter and chronicals of Narnia like it, and people who like Sherlock Holmes and Agatha Christie books don’t seem to. But a more generalized and concise way of describing this would be to say that it’s very correlated with books we might call ‘fantasy,’ and not very correlated with books we might call ‘mystery’. Similarly, in describing Ben, if his interests correlate highly with those of lovers of Lord of the Rings and Harry Potter, we can say that a large part of his fandom is related to fantasy. All these relationsips are in the ratings matrix.

Thus, to capture the rating process, we must decompose both books and users. Matrix factorization using SVD achieves this decomposition by factoring each row (user) into a linear combination of the other rows and each column (book) into a linear combination of the other columns. In doing so, SVD mimics the rating process described above. 

To get the SVD of a matrix, we have: 

$${A_{m\times n}=U_{m\times m}S_{m\times n}V^T_{m\times n}}$$

$$A$$ is an $$m\times n$$ matirx, $$U$$ is an $$m\times m$$ orthogonal matrix, $$V$$ is a $$n\times n$$ transpose of an orthogonal matrix, and $$S$$ is a $$m\times n$$ diagonal matrix whose diagonal entries are the singular values of $$A$$. Each singular value is the square root of an eigenvector for $$A^TA$$ 

Each column in $$V^T$$ is an eigenvector for $$A^TA$$ and corresponds to a book. If 2 columns have the same value in a given row, then those books correlate and are somehow similar in that specific orthogonal dimension. Using the analogy from above, this might mean that these books are both rated highly among users who rated books we’d call “mystery”.  In this regard, the SVD captures the latent features in the data and captures the type of each book. 
Similarly, each column in $$U$$ is an eigenvector for $$AA^T$$ and each row corresponds to a specific user row, so that if any 2 rows share the same column value, those users can be thought of as being similar with regards to that specific orthogonal dimension. Here, the SVD captures the latent relationships between users, and can “determine” what type of fan the user is, whether it be mystery lover, or a predominantly-romance-but-also-some-horror lover. 

We only keep top-$$k$$ singular values and chop off the rest. The theory is that in doing so, we create an approximation of the data that is based on the latent features and relationships and disregard the noise. We’ll talk more about tuning k later below. 

When we fold in a new user, we can use these 3 matrices to re-enact the rating process using the latent book features and user-relationships:

First we see that the best $$k$$-rank approximation of a matrix is found using:

$${Ak_{m\times n}=Uk_{m\times k}Sk_{k\times k}Vk^T_{k\times n}}$$

Since we’re adding in the enduser vector with their book recipe, we need to find a way of approximating a new row in Uk by folding in our new enduser vector. To get $$Uk$$, we can use the formula:

$${Uk=(Ak)(Vk^T)^{-1}(Sk^{-1}})$$

Thus, to get the approximation of our enduser vector, we can do:

$${Uku=(Au)(Vk^T)^{-1}(Sk^{-1}})$$

Where $$Uku$$ is the $$k$$-rank approximation of the enduser vector, and $$Au$$ is the submitted enduser vector. We now have our predicted enduser ratings for each book. 

## Penalizing Popularity

The SVD does a great job of predicting which books our end user would enjoy given his/her submitted book recipe. However, one downside is that books that are generally popular overall may rank higher than those that are less popular but a better fit for the user's recipe. One way around this dilemma is to give weights to the SVD results such that popular books are penalized proportionately. To implement this, I used a variation of TF-IDF, by using the predicted rating in place of document frequency, and penalized each score by how many users had read a given book compared to overall book reads (popularity):

$${adjusted\ score = rating\times log(\frac{read\ count}{total\ reads})}$$

This pushed the more unique books up in the list and delivered higher quality results.


## KNN Using Book Tags

The results delivered from our SVD have the merit of quality, but keeping our goal in mind, we also want to ensure our results to be similar to those submitted by the enduser. To achieve this, I used a K-Nearest Neighbors algorithm on the top section of the results, $$n$$, and used the tags associated with each book as features. Thus, if a series of books submitted had tags containing comedy, feminist, post-modern, and autobiography, those top-$$n$$ results from the SVD closet closely matching this breakdown would be returned first. 

An alternative worth considering is using k means clustering on the results and returning books in the same cluster as the input based on a combination of their return rank and distance from the input. Tags that were more frequently associated with a book by users were given a proportionately greater weight. 

This aspect of the recommendation process was another area that was a design choise and had tremendous impact on the returned results. If I made the number ($$n$$) of top results from the SVD delivered to the KNN model too high, the quality of my SVD results would be totally disregarded in favor of the similarity predicted by the KNN (and the mode would run too slow). If I made n too small, the results may be of high quality  but wouldn't be similar enough, and the user would lose confidence in the concept of custom results for custom recipes. 

Thus, $$n$$, the number of results to pass to KNN, became another high impact design choice. One method that worked particularly well was to create an expanding window for $$n$$. This meant that when a user sent in the recipe, $$n$$ results were passed to the KNN, but if the user pressed next to see more results, $$n$$ would expand. The expansion degree that worked for me was 1.2. Thus, books of high quality that didn't make it into the initial delivered results had a chance to be delivered in the next result set, but the result set was also expanded, making it that new SVD results were passed to the KNN and had a shot of being returned. 

<img src ="/assets/images/post_images/book_rec_post/collab_filtering_result.svg" style="width:560px"/>

The illustration above provides an example of the process: the sliding window starts at the top 5 books returned from the SVD, and then a K=3 Nearest Neighbors model is run, with the 3 closest neighbors returned in the final result set. If the user clicks for more results, the window expands to the top 5 results, and the KNN model is run on that sample. One of the new, blue books from the expanded window makes the cut, but so do 2 from the original window.

Another way to tackle this would have been to create a similarity-proportion cutoff, whereby the window is only expanded once the previous window had no more results within a certain threshold of similarity. A task for another day.

## Tuning Singular Values using KNN

In order to determine the appropriate number of singular values to keep and chop off, I decided to use the KNN model to add a level of supervision. Keeping in mind that one of the goals for the recommendations is that they also share similarities to the input, I tuned $$k$$ using nearest neighbors distance for the top $$$$` SVD results by performing a grid search where for each level of $$k$$, multiple logical book recipes were submitted and the average KNN distance for the top 100 results were recorded. The value for k that minimized the KNN distance was deemed best, and used in production. 

## Updating Results Through User Feedback

Finally, users have the option of refining their results trough up-voting and down-voting. To register the feedback and update the model, I simply added a positive rating to the enduser vector for an up-voted book and the opposite for a down-vote, and ran the model again to get the next and updated round of results.

## Recommendation Systems are Awesome

Building this recommendation engine was an incredible experience. In particular, getting more experience with matrix factorization and dimensionality reduction was very beneficial. On their own, each of the elements involved were challenging and instructive, but by far the hardest aspect of this project was making the algorithms work in a way that made the end-product work. Finding a means to deliver quality recommendations in real-time and in alignment with the recipe-focused product goal made the problem more challenging, but also extremely rewarding.

