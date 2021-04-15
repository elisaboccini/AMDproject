# AMDproject
I declare that this material, which I now submit for assessment, is entirely my own work and has not been taken from the work of others, save and to the extent that such work has been cited and acknowledged within the text of my work. I understand that plagiarism, collusion, and copying are grave and serious offences in the university and accept the penalties that would be imposed should I engage in plagiarism, collusion or copying. This assignment, or any part of it, has not been previously submitted by me or any other person for assessment on this or any other course of study.

# Market Basket Analysis on IMDb Dataset

## Introduction
Market Basket Analysis is an example of frequent itemsets mining, an attempt to discover patterns among the items within a dataset in an immediate and easily interpretable way. The most popular application of this technique is in the retail sector where it is performed with the aim of predicting future purchases of customers.
Through this kind of analysis on large datasets, association rules that would not be otherwise discovered by managers are highlighted. The downside is that the size of the datasets often represents a problem to face, since it can make very slow even the simplest computation.
Many different algorithms have been developed in order to perform Market Basket Analysis and therefore highlighting association rules among items. 
One of the most popular is the Apriori algorithm. However this algorithm may have problems in dealing with large datasets, due to the fact it performs two passes over the data, first creating a set of candidates and then selecting among them the frequent ones. 
For this reason, many alternatives and variants have been developed and also in this analysis, firstly the Apriori algorithm is applied to the filtered data, but then a Spark environment is created and the FPGrowth algorithm is performed in order to obtain frequent itemsets and association rules in a faster and less computationally expensive way.

## Dataset
This analysis has been conducted on the IMDb dataset, therefore it is not the typical Market Basket Analysis performed over transactions: in this case movies have been considered as baskets and actors as items. The aim of the work is that of highlighting potential associations among actors and in particular if there are any of them frequently occurring together within the cast of movies. 
The IMDb dataset contains 5 tab-separated-values tables:
1.	title.akas.tsv.gz
2.	title.basics.tsv.gz
3.	title.principals.tsv.gz
4.	title.ratings.tsv.gz
5.	name.basics.tsv.gz
These tables store many information about movies, TV shows, videogames, actors playing in them, directors and operative staff, ratings assigned to the movies and many other. 
All these information result in attributes that are redundant and in some case useless for the purpose of this analysis. For this reason, only the title.basics, title.principals and name.basics tables have been considered and even within these tables, not all the attributes have been considered. The title.principals is the junction storing the many-to-many relationship occurring between movies and actors, linking them using the identifiers, but it does not contain any information about the name of the actor or the title of the movie. For this reason the other two tables are required: the title.basics associates to each unique identifier the full title - other than many other information that have not been taken into consideration - while the name.basics table stores the full name of all the actors whose identifier appears in the title.principals table.

## Data organization and pre-processing
As anticipated before a Spark environment has been created in order to speed up computations.
To perform the analysis on a more manageable dataset, a new object called ‘explicit’ has been created. In particular this new spark dataframe is not only composed by selected attributes with respect to the full original tables, but it is also filtered with respect to the records in it. Indeed, for this analysis only actors are required, while the name.basics and the title.principals tables both contain records related to the crew working in a movie, so also the director’s name, the producers and other staff components.
The ‘explicit’ spark dataframe instead contains information related to:
- tconst: unique identifier for the title,
- primaryTitle: full title of the movie,
- nconst: unique identifier for the actor,
-primaryName: full name of the actor.

In the first analysis that has been performed, this new database has been considered as a whole. However, with the analysis of the outcome it has been clear that the result was highly influenced by the presence of episodes within the tconst and primaryTitle. Indeed for each episode a single record is stored in the dataset: this of course implies that there were actors found to be very frequent singularly taken, but also taken with other as itemsets. The actors resulting in the association rules, they were all part of the cast of a TV series, since as it is reasonable, for all or almost all the episodes of the series the cast remains the same.
For this reason, the analysis has been conducted a second time, by filtering the title.basics by selecting only the movies belonging to the categories movie, TVmovie, short, TVshort and only in a second step the merge is performed, keeping as before only the four attributes and creating a second spark dataframe called ‘explicit2’.
Then these two rearranged datasets have been restructured in such a way that the algorithm was able to perform its computations and therefore baskets containing items have been created, in particular it has been associated to each movie a list of actors playing in it.

## Algorithms and implementation
The Apriori algorithm, as previously said is one of the most popular techniques used to find association rules among itemsets. 
It relies on the finding that a frequent itemset always contains frequent subsets and equivalently only frequent itemsets having size k can be subsets of a frequent itemset having size k+1.
The algorithm computes the count of the items taken singularly at the first step. Then among those found to be frequent, it computes the counts of their possible combinations in order to find frequent pairs and so on, up to when no frequent itemset is found. 
Of course the result will depend on the definition of frequent itemset: a support threshold has to be set in order to define which itemsets are actually frequent. Moreover, the step of the algorithm in which candidates sets are created is really time consuming, especially with large datasets, since the number of possible combinations tends to explode with the increase in the size of the dataset.
Moreover this solution doesn’t work well with distributed computing: each worker would find its set of candidates and frequent itemsets depending on how the data are distributed and relatively to the subset of data it has received.
For this reason, a solution that provides an alternative for large datasets is the Frequent Pattern-Growth algorithm.
This technique consists of building a suffix-tree structure starting from the frequent items, singularly taken and extracting frequent itemsets from it, therefore avoiding the candidates selection step, that is the most expensive one in the Apriori algorithm. In Spark the Parallel Frequent Pattern Growth algorithm is performed in order to distribute among the workers the building of the tree, furtherly speeding up the process.

## Market Basket Analysis
In this section the code will be explained in depth, in order to show the reasoning behind the analysis. 
After getting the datasets of interest though kaggle API, the Spark environment has been created and therefore the following computations has been split among the Java machines.
In order to convert the Spark Dataframes into temporary views available only for the Spark Session put in place and lazily evaluated with Spark SQL, the createOrReplaceTempView() method has been used. In this way, it does not persist to memory unless you cache the dataset that underpins the view.
Firstly, duplicates have been dropped from the datasets and from an initial count including all the unique records, over 9 million actors and 6 million movies came up. 
In order to merge and filter the datasets, the “data1”  spark dataframe has been created, including records for all the movies, TV shows, videos and the corresponding actors playing in them. This dataframe shows the many-to-many relationship occurring between titles and actors and stores over 14 million unique couples title-actor.
As said before the distinct titles stored in the IMDb title.basics table are over 6 million and the most frequent actor, singularly taken, appears in 5000 of them. Therefore the minimum support to be taken as parameter for the algorithm can not be, as suggested in “Mining of Massive Datasets” by Leskovec, Rajaraman, Ullman, the 1% of the data, since it would result in zero frequent actors.
A little tuning has been performed for this parameter, even if not a greedy search due to the size of the data. In order to obtain a significant result, including not too many association rules the best value is 0,0005: actors or itemsets of actors appearing in at least 3000 movies are considered frequent.
In order to use the FP-Growth algorithm, it is necessary, not only to choose an appropriate minSupport value, but also to structure the data as if they were baskets. Movies have been grouped by their tconst value and for each unique row a list of actors playing in that movie has been associated.
Then the baskets have been split in training and test set with a proportion of respectively 90% and 10%, in order to make sort of predictions over the cast of movies using the transform() method.
Finally, the algorithm has been fed with the data structured as baskets and the minSupport parameter previously chosen


## Results
The difference in output between with episodes and without episodes
-	Frequent itemsets in general, containing one actor, two actors, three…
-	Association Rules
The tuning of the minSupport
-	How changes the result
-	Conclusion of the most significant outputs: no episodes and 0.0001 minSupport -> frequent itemset of 4 is 
-	In apriori 0.0003 -> 1 association rule

