I declare that this material, which I now submit for assessment, is entirely my own work and has not been taken from the work of others, save and to the extent that such work has been cited and acknowledged within the text of my work. I understand that plagiarism, collusion, and copying are grave and serious offences in the university and accept the penalties that would be imposed should I engage in plagiarism, collusion or copying. This assignment, or any part of it, has not been previously submitted by me or any other person for assessment on this or any other course of study.

# Market Basket Analysis on IMDb Dataset


## Introduction

Market basket analysis is an example of frequent itemsets mining, an attempt to discover patterns among items within a dataset in an immediate and easily interpretable way. The most popular application of this technique is in the retail sector, where market basket analysis are performed with the aim of developing marketing strategies by looking at customers’ purchases and in particular at associations among items that seems to be frequently purchased together.
Through this kind of analysis on large datasets, association rules that would not be otherwise discovered, are highlighted. However the size of the data, even if potentially useful in order to get many different insights, often represents a problem to deal with, making difficult and time consuming even the simplest computations.
Many different algorithms have been developed in order to perform market basket analysis. One of the most popular is the Apriori algorithm. This algorithm is easy to implement and interpret, but it may have problems in dealing with large datasets and for this reason many alternatives and variants have been developed. 
In this project, after a Spark environment is created, the FP-Growth algorithm is performed in order to obtain frequent itemsets and association rules in a faster and less computationally expensive way.
This report will present how the analysis has been conducted. In the first section the dataset is presented, then the pre-processing made over the data is explained. Afterwards the algorithm and the hyperparameters to be set are outlined and finally the results of the analysis are shown with a particular attention over the differences in the output determined by the tuning of the hyperparameters. 


## Dataset

This market basket analysis has been conducted on the IMDb dataset, therefore it is not the classical use case where transactions are analyzed in order to discover associations among purchased items; in this case movies have been considered as baskets and actors as items. 
The aim of the work is that of highlighting potential associations among actors and understanding if there are any of them frequently occurring together within the cast of a movie. 
The IMDb dataset contains 5 tab-separated-values tables:
1.	title.akas.tsv.gz
2.	title.basics.tsv.gz
3.	title.principals.tsv.gz
4.	title.ratings.tsv.gz
5.	name.basics.tsv.gz

These tables store many information about movies, shows, TV series, videos, actors playing in them, directors and operative staff, ratings assigned to the movies and many others. 
All these information result in attributes that are redundant and in some case useless for the purpose of this analysis. For this reason, only the title.basics, title.principals and name.basics tables have been considered and even within these tables, not all the attributes have been taken into account. The title.principals is the junction storing the many-to-many relationship occurring between movies and actors, linking them using identifiers, but not containing information about the name of the actor or the title of the movie. For this reason the other two tables are required: the title.basics is used to associate to each unique identifier for the movie its full title, while the name.basics table provides information on the full name for the actors appearing in the title.principals table.


## Data organization and pre-processing

After getting the datasets of interest through kaggle API, the Spark environment is created in order to distribute data and computations among Java machines, ultimately speeding up the analysis and ensuring scalability.
In order to convert the Spark dataframes into temporary views available only for the Spark Session in place, the createOrReplaceTempView() method is needed. In this way, it does not persist to memory unless you cache the dataset that underpins the view.
Firstly, duplicates are dropped from the datasets and counting only unique records, over 9 million actors and 6 million movies come up. 
To perform the analysis on a more manageable dataset, a new Spark dataframe, called ‘data’, is created. It is filtered both from the attributes perspective, since it only stores the ‘primaryTitle’, the ‘tconst’, the ‘nconst’ and the ‘primaryName’, but also from the records perspective. Indeed, for this analysis only actors and movies are required, while the above-mentioned tables contain identifiers and names related to the crew working in a movie - so also the director, the producers and other staff components - and identifiers and titles of videos, shows and TV series that have been filtered out.
The new object ‘data’ is a Spark dataframe having columns:
- ‘tconst’: unique identifier for the movie,
- ‘primaryTitle’: full title of the movie,
- ‘nconst’: unique identifier for the actor,
- ‘primaryName’: full name of the actor.

The decision of considering only records related to movies and shorts, but not to the other categories, is taken after a first analysis where the dataset was filtered only by columns, leaving the number of records unchanged. However, once obtained the output, it was clear that the result was highly influenced by the presence of episodes within the tconst and primaryTitle. Indeed, the dataset stores for each episode a single record: this of course implies that the most frequent actors or sets of actors were all playing in TV series and therefore they were associated to titles recorded as ‘episode#xyz’, even without specifying the series’ name. All the resulting association rules were referred to actors playing in episodes, since, as it is reasonable, the cast of a TV series is likely to stay the same for all or almost all the episodes.
For this reason the analysis is started with the creation of the Spark dataframe ‘data’, storing 3686467 associations movie-actor. The check for the presence of null values, stored in the dataset as ‘/N’, highlights that no missing values are stored.
Then, the dataframe is restructured again in order to feed the FP-Growth algorithm: baskets are created and each unique concatenation of the ‘tconst’ value with the correspondent primaryTitle is associated to a list of actors playing in the movie. 
After this rearrangement of the data in baskets, the FP-Growth() method is applied. This technique provides a strong improvement over the Apriori algorithm, avoiding the explicit creation of the candidate sets, that is usually the most expensive task in the algorithm. 


## Apriori

The Apriori algorithm, relies on the monotonicity principle stating that a frequent itemset of size k is composed by subsets that also must be frequent. 
The algorithm computes the count of the items taken singularly at the first step. Then, a filtering procedure is put in place and only those items found to be frequent according to a predefined threshold are selected for the construction step. For the frequent items, all the possible pairs are constructed and again the filtering procedure is applied so that only frequent pairs will pass to the later step. The algorithm goes on with this filtering-construction process, increasing the cardinality of the itemsets, up to when no frequent itemset is found anymore.
The Apriori algorithm may have problems in scaling up with large datasets. This depends also on the minimum support threshold chosen and therefore when an itemset has to be considered frequent, but in general it requires to hold in main memory many candidates to be counted in a number of steps that exponentially increases with the size of the dataset. 
For this reason alternative solutions have been designed, some of them are exact algorithms as the SON algorithm, while others may lead to the presence of false positives and/or false negatives in the result, as the sampling basket method or the Toivonen algorithm.


## FP-Growth 

The Frequent Pattern-Growth algorithm is an alternative solution, scaling well with the size of the data and its implementation on a distributed environment further reduces the costs related to the memory usage and the runtime. It consists of building a tree structure starting from the frequent items, singularly taken, and extracting frequent itemsets from the tree, maintaining the association between the itemsets in the structure itself, while avoiding the candidates selection step.
The first step is the same as in the Apriori algorithm: the database is scanned to find the occurrences of the single items. Then, the FP-tree is started by the root, set as null, and baskets are investigated one by one: for the first basket the items are counted, ordered and placed each one in a different node, from the root to the leaves, from the most frequent to the less one. Then the same is done for the second basket and so on. For each item already occurring in another basket, a common path to the root is built and the counter is increased.
Once the tree has been built, it has to be mined: starting from the lowest nodes, a conditional tree is grown considering only those items exceeding the minimum support threshold, as resulting from the first step’s count. For each item respecting this constraint, the possible paths to the root are taken as conditional pattern base. Then, for each of them another FP-tree is built, still discarding patterns not appearing more than the established threshold and therefore finally generating the frequent patterns.


## Algorithm and its implementation

The FP-Growth() method applied in pyspark is actually the implementation of the Parallel Frequent Pattern Growth algorithm, as described in Li et al., ‘PFP: Parallel FP-Growth for Query Recommendation’, where tree building tasks are distributed among the workers, so that they execute independent mining tasks. 
Pyspark provides the FPGrowth() implementation in the ml.fpm library. The method takes as input the data on which to compute frequencies and association rules, the minimum support threshold, the minimum confidence and the number of partitions. 
As previously mentioned, tuning hyperparameters can lead to different outputs and therefore selecting the right value is a crucial point of the analysis. There is not a universal way to identify the “right” hyperparameters’ value, but it strongly depends on the specific case and on the kind of analysis to be conducted. For example, even if in “Mining of Massive Datasets” by Leskovec, Rajaraman, Ullman, a support threshold equal to the 1% of the number of baskets is suggested as considerably high in order to obtain a reasonable number of association rules, in the IMDb case it would be even too high. It would be unlikely to find actors having played in more than 974 movies and indeed, if considering only movies and short, not the whole categories stored in the dataset (‘tvSeries’, ‘tvMiniSeries’, ‘tvEpisode’, ‘tvSpecial’, ‘video’, ‘videogame’ excluded), when running the algorithm, no output is retrieved. 
For this reason after some tuning, the value for the minimum support is set to 0.0003, considering frequent those actors playing in at least 292 movies. 
On the other hand the minimum confidence hyperparameter, affecting the output of the association rules, has been risen with respect to what suggested by the above mentioned authors. It has been set to 0.6, instead of 0.5, in order to filter out those associations that are less likely to be found.
The number of partitions is used to distribute the work and if not set, it takes the number of partitions of the input dataset.
As will be explained in the next paragraph of the report, the analysis through the FP-growth algorithm outputs frequent items and itemsets, association rules, their confidence and their lift, through the methods freqItemset(), associationRules() and transform().


## Results

The results obtained when setting the minimum support to 0.0003 will be firstly shown, since this value seems to represent a reasonable assumption, other than a good compromise between the number of generated rules and their interpretability. 
The output of the FP-Growth algorithm when considering a support threshold set to 0.0003, is a total of 41 frequent actors, each appearing in more than 292 movies, and just one frequent couple: Eddie Lyons and Lee Moran, having played together in 318 movies. The resulting association rule is that if Eddie Lyons appears in a movie, also Lee Moran will, with a confidence of  0.78; on the contrary, if Lee Moran is in the cast of a movie, the presence of Eddie Lyons has a confidence of 0.75. However, confidence of a rule is not meaningful per se: if a consequent is very frequent, the rule will have high confidence, no matter the antecedent. 
The lift helps the interpretation of the significance of a rule, since it considers the conditional probability of the consequent, given the antecedent. It can be seen as the rise in the probability of having the consequent actor in the cast, given the probability of having the antecedent actor. It is a positive value if the antecedent is for real predictive for the consequent one, it is equal to 0 when the two actors are totally independent, while it is negative if the presence of the antecedent actually anticipates the absence of the other one.
For the couple Eddie Lyons, Lee Moran, the lift indicates that the presence of the first actually rises the probability of the presence of the second one. 
In order to use the rules as if they were a prediction, the transform() method associates to each movie having Lee Moran in the cast, the presence of Eddie Lyons too and the other way around. This means that, if in the future there will be a cast with Eddie Lyons, the model will predict that also Lee Moran will be in the movie. 
As said before, the output of a market basket analysis depends on the definition of frequency: if the support threshold is lowered, many other actors will appear in the result and more association rules will be produced.
For example, when the minimum support hyperparameter is set to 0.0001 and therefore an actor is considered frequent if appearing in at least 97 movies, the algorithm ends up with many more frequent itemsets. In order to discriminate among the association rules in the output, only those having confidence 1 are selected and for this reason the output does not contain the couple obtained for the higher minimum support threshold case. After selecting the maximum value of confidence, we obtain three actors frequently occurring together: Curly Howard, Larry Fine and Moe Howard. The pair Curly and Moe Howard anticipates Larry Fine and the pair Curly Howard and Larry Fine anticipates Moe Howard, given the lifts for both the rules are positive.  In this case the largest frequent itemsets contain triplets of actors: there are 9 cases where three actors frequently occur together. Among those also Curly Howard, Larry Fine and Moe Howard, that are therefore the only frequent itemset of three actors having confidence equal to 1.


## Conclusions

This report has described a market basket analysis for an unconventional purpose: finding actors frequently occurring together within a movie. The FP-Growth algorithm used to mine the dataset is an alternative to the Apriori, trying to respond in a more efficient way to its drawbacks and, when performed in a distributed environment, it can offer a strong improvement over the performance, thanks to the more compact way of storing the itemsets in a tree structure and the less number of passes over the dataset. The findings from this analysis are affected by the choice of the minimum support threshold and therefore two alternative solutions are proposed: a stricter one, setting as threshold the 0.03% of the number of baskets, and a looser approach with a minimum support of 0.01%. The first one has been considered a better choice to obtain a clear and significant rule from a large dataset: the most frequent couple is composed by Eddie Lyons and Lee Moran, occurring together in 318 movies, equivalent to a support of 0.0326% of the baskets.

