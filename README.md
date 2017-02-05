# Understanding Customer Conversion <br> with Snowplow Web Event Tracking <br> <sub> Benjamin S. Knight, January 27th 2017 </sub>

### Project Overview
Here I apply machine learning techniques to Snowplow web event data to infer whether trial account holders will become paying customers based on their history of visiting the marketing site. By predicting which trial account holders have the greatest likelihood of adding a credit card and converting to paying customers, we can more efficiently deploy scarce Sales Department resources. 

[Snowplow](http://snowplowanalytics.com/) is a web event tracker capable of handling tens of millions of events per day. The Snowplow data contains far more detail than the [MSNBC.com Anonymous Web Data Set](https://archive.ics.uci.edu/ml/datasets/MSNBC.com+Anonymous+Web+Data) hosted by the University of California, Irvine’s Machine Learning Repository. At the same time, we do not have access to demographic data as was the case with the [Event Recommendation Engine Challenge](https://www.kaggle.com/c/event-recommendation-engine-challenge) hosted by [Kaggle](https://www.kaggle.com/). Given the origin of the data, there is no industry-standard benchmark for model performance. Rather, assessing baseline feasibility is a key objective of this project. 

### Problem Statement
To what extent can we infer a visitor’s likelihood of becoming a paying customer based upon that visitor’s activity history on the company marketing site? We are essentially confronted with a binary classification problem. Will the trial account in question add a credit card (cc_date_added IS NOT NULL ‘yes’/‘no’)? This labeling information is contained in the ‘cc’ column within the file ‘munged df.csv.’ 

There is no clear precedent for how effective a model our analysis may ultimately yield, and so a key component of the project is ultimately assessing feasibility of using visitor history on the marketing site to predict conversion. Regarding the most appropriate model, there is no way of knowing which family of algorithms will prove to be most effective in predicting customer conversion. With this in mind, we adopt an "all of the above" approach - applying all algorithms that are feasible given the nature of the classification problem and the structure of the data (see the section entitled 'Algorithms and Techniques' for additional details). That being said, certain families of algorithms are more promising than others (at least initially). Based on findings from [Wainer (2016)](https://arxiv.org/pdf/1606.00930v1.pdf), we predict that a Support Vector Machine (SVM) with a Radial Basis Function (RBF) kernel is most likely to yield the best fit. 

### Metrics
As we discuss later, the data is highly imbalanced (successful customer conversions average 6%). Thus, we are effectively searching a haystack for rare, but exceeedingly valuable needles. In more technical terms, we want to maximize recall as our first priority. Selecting the model that maximizes precision is a subsequent priority. To this end, our primary metric is the F2 score shown below.
<div align="center">
<img src="https://github.com/b-knight/Understanding-Customer-Conversion-with-Snowplow-Web-Event-Tracking/blob/master/Images/F2_Score_Equation.png" align="middle" width="453" height="113" />
</div>

The F2 score is derived from the [F1 score](https://en.wikipedia.org/wiki/F1_score) by setting the weight of the \beta parameter to 2, effectively increasing the penalty for false negatives. While the F2 score is the arbiter for ultimate model selection, we also use [precision-recall curves](http://scikit-learn.org/stable/auto_examples/model_selection/plot_precision_recall.html) to clarify model performance. We have opted for precision-recall curves as opposed to the more conventional [receiver operating characteristic](https://en.wikipedia.org/wiki/Receiver_operating_characteristic) (ROC) curve due to the highly imbalanced nature of the data [(Saito, 2016)](http://journals.plos.org/plosone/article?id=10.1371/journal.pone.0118432).

### Data Preprocessing 
The raw Snowplow data available is approximately 15 gigabytes spanning over 300 variables and tens of millions of events from November 2015 to January 2017. When we omit fields that are not in active use, are redundant, contain personal identifiable information (P.I.I.), or which cannot have any conceivable bearing on customer conversion, then we are left with 14.6 million events spread across 22 variables. 

<p align="center"><b>Table 1: Selected Snowplow Variables Prior to Preprocessing</b></p>
|<sub>Snowplow Variable Name</sub>   |<sub>Snowplow Variable Description</sub>                                         |
| ---------------------------------- |---------------------------------------------------------------------------------| 
| <sub>*event_id*</sub>              | <sub>The unique Snowplow event identifier</sub>                                 |
| <sub>*account_id*</sub>            | <sub>The account number if an account is associated with the domain userid</sub>|
| <sub>*reg_date*</sub>              | <sub>The date an account was registered </sub>                                  |
| <sub>*cc_date_added*</sub>    | <sub>The date a credit card was added                                                     |
| <sub>*collector_tstamp*</sub> | <sub>The timestamp (in UTC) when the Snowplow collector first recorded the event          |
| <sub>*domain_userid*</sub>    | <sub>This corresponds to a Snowplow cookie and will tend to correspond to a single internet device|
| <sub>*domain_sessionidx*</sub>     | <sub>The number of sessions to date that the domain userid has been tracked               |
| <sub>*domain_sessionid*</sub>      | <sub>The unique identifier for the Snowplow cookie/session                                |
| <sub>*event_name*</sub>            | <sub>The type of event recorded                                                           |
| <sub>*geo_country*</sub>           | <sub>The ISO 3166-1 code for the country that the visitor’s IP address is located         |
| <sub>*geo_region_name*</sub>       | <sub>The ISO-3166-2 code for country region that the visitor’s IP address is in           |
| <sub>*geo_city*</sub>              | <sub>The city the visitor’s IP address is in                                              |
| <sub>*page_url*</sub>              | <sub>The page URL                                                                         |
| <sub>*page_referrer*</sub>         | <sub>The URL of the referrer (previous page)                                              |
| <sub>*mkt_medium*</sub>            | <sub>The type of traffic source (e.g. ’cpc’, ’affiliate’, ’organic’, ’social’)            |
| <sub>*mkt_source*</sub>            | <sub>The company / website where the traffic came from (e.g. ’Google’, ’Facebook’)        |
| <sub>*se_category*</sub>           | <sub>The event type                                                                       |
| <sub>*se_action*</sub>             | <sub>The action performed / event name (e.g. ’add-to-basket’, ’play-video’)               |
| <sub>*br_name*</sub>               | <sub>The name of the visitor’s browser                                                    |
| <sub>*os_name*</sub>               | <sub>The name of the vistor’s operating system                                            |
| <sub>*os_timezone*</sub>           | <sub>The client’s operating system timezone                                               |
| <sub>*dvce_ismobile*</sub>         | <sub>Is the device mobile? (1 = ’yes’)                                                    |

I use the phrase 'variable' as opposed to 'feature', since this dataset will need to undergo substantial transformation before we can employ any supervised learning technique. Each row has an 'event_id' along with an 'event_name' and a ‘page url.’ The event id is the row’s unique identifier, the event name is the type of event, and the page url is the URL within the marketing site where the event took place.

The distillation of the raw data into a transformed feature set with labels is handled by the iPython notebook 'Notebook 1 - Data Munging.' In transforming the data, we will need to create features by creating combinations of event types and distinct URLs, and counting the number of occurrences while grouping on accounts. For instance, if ‘.../pay-ment plan.com’ is a frequent page url, then the number of page views on payment plan.com would be one feature, the number of page pings would be another, as would the number of web forms submitted, and so forth. Given that there are six distinct event types and dozens of URLs within the marketing site, then the feature space quickly expands to encompass hundreds of features. This feature space will only widen as we add additional variables to the mix including geo region, number of visitors per account, and so forth.
<p align="center"><b>Figure 1: Management of Original Categorical Variables into Features </b></p>
<div>
<div align="center">
<img src="https://github.com/b-knight/Understanding-Customer-Conversion-with-Snowplow-Web-Event-Tracking/blob/master/Images/Data_Transformation.png" align="middle" width="565" height="368" />
</div>
</div>


With the raw data transformed, our observations are no longer individual events but indivual accounts spanning the period November 2015 to January 2017. Our data set has 16,607 accounts and 581 features. 290 of these represent counts of various combinations of web events and URLs grouped by account. Next there are two aggregated features - the total number of
distinct cookies associated with the account, and the sum total of all Internet sessions linked to that account.There are also 151 features that represent counts of page view events linked to IP addresses within a certain country (e.g. a count of page views from China, a count of page views from France, and so forth). <br>

46 of the features represent counts of page views coming from a specific marketing medium ('mkt medium'). Recall that 'mkt medium' is the type of traffic. Examples include ‘partner link,' 'adroll,' or 'appstore.' The ‘mkt medium’ subset of features is followed by 86 features that correspond to Snowplow’s 'mkt source' field. 'mkt source' designates the company / website where the traffic came from. Examples from this subset of the feature space include counts of page views from Google.com ('mkt source google com') and Squarespace ('mkt source square'). There are two additional feature:'mobile pageviews' and 'non-mobile pageviews' that represent counts of page views that took place on mobile versus non mobile devices. I have also included an additional feature derived from these two - the share of page views that took place on a mobile device.<br>

With the aggregations completed, we then take the transformed data and drop all features that are uniformly zero for all observations. Finally, we scale the features using [robust scaling](http://scikit-learn.org/stable/modules/generated/sklearn.preprocessing.RobustScaler.html). 

It bears noting that 'br_name' (the name of the visitor’s browser), 'os_name' (the name of the vistor’s operating system), and 'os_timezone' (the client’s operating system timezone) were not included in the ultimate version of the transformed data. The transformed variables of 'br_name' and 'os_name' were used initially. However, their incorporation added +40 features to the already expansive feature space resulting in inferior performance and so were subsequently dropped.<br>

### Data Exploration
Exploring the transformed data, two features quickly become apparent. First, we can see that the data is highly imbalanced. Only approximately 6% of the labeled accounts show a succesful conversion to paying customer. 

<br>
<div align="center">
<p align="center"><b>Figure 2: Summary Statistics - Distribution of Labels (16,607 Observations)</b></p>
<img src="https://github.com/b-knight/Understanding-Customer-Conversion-with-Snowplow-Web-Event-Tracking/blob/master/Images/exploratory_analysis-labels.png" align="middle" width="600" height="225" />
</div>

The second feature of note is that in addition to our feature space being wide with over 500 features, the features themselves are fairly sparse as the histograms below make clear. This is to be expected. The Snowpow features are highly specific. Examples include counts of certain types of events localized within Bangladesh, or the number page views associated with a bit of on-line content that was only made briefly available. As a result, the majority of features are extremely sparse.       

<div align="center">
<p align="center"><b>Figure 3: Summary Statistics - Means and Standard Deviations of Sparse Feature Space (581 Features)</b></p>
<img src="https://github.com/b-knight/Understanding-Customer-Conversion-with-Snowplow-Web-Event-Tracking/blob/master/Images/exploratory_analysis-feature_means.png" align="middle" width="528" height="198" />
<img src="https://github.com/b-knight/Understanding-Customer-Conversion-with-Snowplow-Web-Event-Tracking/blob/master/Images/exploratory_analysis-feature_sds.png" align="middle" width="528" height="198" />
</div>

### Benchmark 
How do we know if our ultimate model is any good? To establish a baseline of model performance, I implement a [K-Nearest Neighbors](http://scikit-learn.org/stable/modules/generated/sklearn.neighbors.KNeighborsClassifier.html) model within the iPython notebook 'Notebook 3 - KNN (Baseline).' In the same manner as the subsequent model selection, I allocate 90% of the data for training (14,946 observations) and 10% for model testing (1,661 observations). I use the model's default setting of 5 neighbors. I run the resulting model on the test data using 100-fold cross validation. Averaging the 100 resultant F2 scores, we thus establish a benchmark model performance of F2 = 0.04.

### Algorithms and Techniques
We start our analysis with establishing a benchmark using K-Nearest Neighbors (KNN) before moving on to more sophisticated algorithms. The KNN classifer works by selecting the target observation's n closest neighbors. The target observation is then classifed as being a member of the same class as the majority class within the n-sample. We use Sci-Kit Learn's 
[KNeighborsClassifier](http://scikit-learn.org/stable/modules/generated/sklearn.neighbors.KNeighborsClassifier.html#sklearn.neighbors.KNeighborsClassifier) which defaults to n = 5. Regarding what qualifies as a 'neighbor,' the KNeighborsClassifier uses the [Minkowski distance](https://en.wikipedia.org/wiki/Minkowski_distance), which at its default setttings is effectively the [Euclidean distance](https://en.wikipedia.org/wiki/Euclidean_distance).

Like KNN, logistic regression is computationally inexpensive - a definite strength given the size of the data set (n = 16,607). In addition, logistic regression is uniquely well-suited to the binary nature of the outcome variable. Sci-Kit Learn's [LogisticRegression](http://scikit-learn.org/stable/modules/generated/sklearn.linear_model.LogisticRegression.html) classifier works through [maximum likelihood estimation](https://en.wikipedia.org/wiki/Maximum_likelihood_estimation). Through many iterations, the algorithm determines what sequence of weights will, when applied to our 581 features, maximize the likelihood that the pattern of successes and failures seen in the training set will emerge. An added strength of the classifier (of potential interest to future, more in-depth analysis) is that it generates the 'coef_' attribute - a vector of coefficients that can indicate which features are the most useful predictors of the outcome variable. 

The second and third algorithms selected are Support Vector Machines (SVM) - the first with a [Radial Basis Function](Radial basis function kernel) (RBF) kernel and the other using a linear function. SVM works by placing multiple hyperplane (support vectors) through the data. The set of hyperplanes that maximizes the distance between the classes is then selected, with the center of the space delimited by the hyperplanes becoming the threshold for classification. 

<div>
<div align="center">
<p align="center"><b>Figure 4: A System of Linear Support Vector Machines Finding the Optimal Separation Between Two Classes</b></p>
<p>Source:<a href="https://commons.wikimedia.org/wiki/File%3ASvm_max_sep_hyperplane_with_margin.png"> Wikipedia</a></p>
<img src="https://github.com/b-knight/Understanding-Customer-Conversion-with-Snowplow-Web-Event-Tracking/blob/master/Images/SVM_example_image.png" align="middle" width="400" height="431" />
</div>
</div>

For the SVM + RBF kernel model, we use Sci-Kit Learn's [SVM](http://scikit-learn.org/stable/modules/svm.html) functionality. The RBF kernel works by creating an additional dimension of information derived fom the squared Euclidean distance of one point vis-a-vis all of the other points. Models using SVM with RBF kernels tend to be perform well with binary data [(Wainer, 2016)](https://arxiv.org/pdf/1606.00930v1.pdf), but are far less expensive than random forest models. 

A strength of RBF kernels is that they can accomodate data that is not [linearly separable](https://en.wikipedia.org/wiki/Linear_separability). However, if our data is already linearly separable, then such an approach is not just expensive - it may lead to over-fitting (Webb, 2002, p.138). For this reason, we also use a linear SVM. Linear SVM assumes that the data is linearly separable, and as a result of this assumption, is relatively inexpensive. Linear SVM is also well-suited for the high dimensionality of our data set (581 features).

### Implementation
One of the edifying aspects of this project was the refinement of the success metric. Given that the task is at its core, a binary classification problem, the initial metric intended was the area under the curve (AUC) as delimited by the receiver operating characteristic [ROC](https://en.wikipedia.org/wiki/Receiver_operating_characteristic). The ROC AUC nicely captures the models' efficacy in terms of the rate of true postives versus false positives. However, the data is highly imbalanced, with only 6% of the customers sucessfully converting to paying accounts. Thus, the challange is less about maximizing the number of true positives relative to the number of false postives, but rather maximizing the share of customer conversions sucessfully capatured while at the same time ensuring the greatest possible precision.     

In more technical terms, recall - not precision - became the metric of highest priority. With this in mind, we discarded the ROC curve in favor of a precision-recall curve. A precision-recall curve plots the average precision (the Y-axis) over a given threshold of recall (the X-axis). From this, we could more meaningfully gauge the trade-off between maximizing recall versus precision. However, this new metric also proved to be inadequate. Speaking to the Sales Department prompted us to narrow the success metric even further. Capturing the majority of customer conversion events was the highest priority, and if a member of the Sales Department had to make due with a less accurate model, then so be it. Thus, seeing things from the end user perspective, we settled on the final success metric - the F2 score.   

With the success metric finalized, we then proceeeded to implement the algorithms detailed in the previous section. Originally, the models were implemented using a 80%:20% split between training and testing data. However, experimenting with the various models showed that increasing the size of the training data set significantly improved model performance. Ultimately, the models were run using a 90%:10% split between training (14,946 observations) and testing (1,661 observations). 

All models were first run using their default setting, while SVM + RBF, linear SVM, and logistic regression were run a second time after tuning the hyper-parameters with Bayesian optimization (see below). Mean F2 scores, recall, and precison were derived from 100-fold cross validation of the testing data. As a final note, it quickly became evident that the SVM + RBF kernel model was by far, the most computationally expensive. Including the tuning of hyper-parameters (see below), implementation of the SVM + RBF model took approximately 17 hours.

### Refinement
In theory, we should be able to improve upon the baseline models by tuning the models' hyper-parameters. Our primary hyper-parameters of interest are C and gamma for the SVM + RBF model, and just C for the linear SVM model. Recall that C is the penalty parameter - how much we penalize our model for incorrect classifications. A higher C value holds the promise of greater accuracy, but at the risk of overfitting. The selection of the the gamma hyper-parameter determines the variance of the distributions generated by the [RBF kernel](https://www.youtube.com/watch?v=3liCbRZPrZA), with a large gamma tending to lead to higher bias but lower variance. 

The C and gamma hyper-parameters can vary by several orders of magnitude, so finding the optimal configuration is no small task. Employing a [grid search](http://scikit-learn.org/stable/modules/generated/sklearn.model_selection.GridSearchCV.html) can be computationally expensive - almost prohibitatively expensive without parallel computing resources. Fortunately, we do not have to exhaustively scan the hyper-parameter space. Rather, we can use Bayesian optimization to find the optima within the hyper-parameter space using surprisingly few iterations.

Here I am indebted to Fernando Nogueira and his development of the [BayesianOptimization](https://github.com/fmfn/BayesianOptimization) for Python. By means of this package, we are able to scan the hyper-parameter space of the SVM + RBF kernel model for suitable values of C and gammma within the range 0.0001 to 1,000. We are able to scan for suitable values for C within the linear SVM model in similiar fashion. 

The below figure illustrates the second and third iterations of this process in a hypothetical unidimensional space - for instance, the hyper-parameter C. Thus, the horizontal axis represents the individual values of C while the horizontal axis represents the metric that we are trying to optimize - in this case, the F2 score. 

The true distribution of F1 scores is represented by the dashed line, but in reality is unknown. The dots represent derived F2 scores. The continuous line represents the inferred distribution of F2 score. The blue areas represent aa 95% confidence interval for the inferred distribution, or in other words, represent areas of potential information gain.  
<div>
<div align="center">
<p align="center"><b>Figure 5: An Acquisition Function Combing a Unidimensional Space for Two Iterations</b></p>
<p>Source:<a href="https://advancedoptimizationatharvard.wordpress.com/2014/04/28/bayesian-optimization-part-ii/"> Bayesian Optimization and Its Applications Part II, gauravbharaj, April 28, 2014</a></p>
<img src="https://github.com/b-knight/Understanding-Customer-Conversion-with-Snowplow-Web-Event-Tracking/blob/master/Images/bayesian_optimization.png" align="middle" width="591" height="387" />
</div>
</div>

The Bayesian optimizer resolves the perennial dilemma between exploration and optimization by use of an acquisition function, shown above in green. The red triangle denotes the global maximum of the acquisition function, with the subsequent iteration deriving the F2 score for that value of C. Note how the acquisition function derives high value from regions of relatively low information (the exploration impetus), yet achieves even greater values when in the vicinity of known maxima of the inferred distribution (the optimization impetus).

For the purposes of Bayesian optimization, we used 20-fold cross validation with a [custom scoring function](http://scikit-learn.org/stable/modules/generated/sklearn.metrics.make_scorer.html) to maximize the F2 scores. The optimization yielded values of C = 998 and gamma = 0.2 for the SVM + RBF model, C = 335 for the linear SVM model, and C = 585 for the logistic regression model.


### Results 
The optimal model in terms of F2 score, recall, but also precision was the linear SVM model with hyper-parameter tuning via Bayesian optimization. The linear SVM model was so successful that the non-optimized version was the second best performing model. The linear SVM model achieved a mean F2 score of 0.25 versus 0.04 for the benchmark KNN model. For recall, the linear SVM achieved a mean score of 0.33. In other words, the model sucessfully found a third of the valuable needles within our haystack. The model also achieved a mean precision of 0.14 - effectively tying with the hyper-parameter tuned logistic regression model. To put this in context, a sales representative engaged in blind guessing which acccounts would convert to paying customers would be hard pressed to be accurate more than 6% of the time (the rate of customer conversion).        

<div align="center">
<p align="center"><b>Table 2: Comparision of Performance Metrics Averaged from 100-Fold Cross Validation</b></p>
</div>
|              Model Used                                                 | F2 Score | Recall  | Precision |
| :---------------------------------------------------------------------- | :------: | :-----: | :-------: |
|<sub> K-Nearest Neighbors (Baseline)                                     |   0.04   |  0.04   | 0.04      |
|<sub> Logistic Regression                                                |   0.16   |  0.19   | 0.13      |
|<sub> Logistic Regression with Hyper-Parameter Tuning                    |   0.15   |  0.18   | 0.14      |
|<sub> Support Vector Machines with RBF Kernel                            |   0.00   |  0.00   | 0.00      |
|<sub> Support Vector Machines with RBF Kernel and Hyper-Parameter Tuning |   0.03   | 0.03    | 0.03      |
|<sub> Linear Support Vector Machines                                     |   0.16   | 0.20    | 0.13      |
|<sub> Linear Support Vector Machines with Hyper-Parameter Tuning         | **0.25** | **0.33**| **0.14**  |

It is striking how the AUC scores for the precision-recall curves imply a very different performance ranking than what the F2 scores report. Looking to the figures below, we can see that the linear SVM with hyper-parameter tuning actually has the lowest AUC of any of the curves (AUC = 0.10). These AUC scores are based upon average precision as opposed to recall. However, it is recall, not precision, that is our priority here. Nevertheless, it is worthing bearing in mind that more often than not, the price for greater recall is precision and vice versa.    

<div align="center">
<p align="center"><b>Figure 6: Precision-Recall Curves of All Three Algorithms with and without Hyper-Parameter Tuning </b></p>
<img src="https://github.com/b-knight/Understanding-Customer-Conversion-with-Snowplow-Web-Event-Tracking/blob/master/Images/SVM_with_RBF.png" width="432" height="360" />
<img src="https://github.com/b-knight/Understanding-Customer-Conversion-with-Snowplow-Web-Event-Tracking/blob/master/Images/Linear_SVM.png" width="432" height="360" />
</div>
<div align="center">
<img src="https://github.com/b-knight/Understanding-Customer-Conversion-with-Snowplow-Web-Event-Tracking/blob/master/Images/Logistic_Regression.png" width="432" height="360" />
</div>

### Conclusion 
This project sought to differentiate soon-to-be paying customers from non-paying account-holders based solely on their activity history on the marketing site. This is a tall order, and while the linear SVM did achieve a F2 score of 0.25 (a more than five-fold improvement vis-avis the KNN benchmark) in practical terms the model is not yet successful enough to be used as part of the sales process. Should the company see a drastic influx of new leads, then our linear SVM model can conceivably be used to prioritize sales outreach. However, in the mean time we should re-examine the model for areas of improvement. 

A major challange of this project is the high dimensionality of the data combined with the sparse feature space. Dimensionality reduction via [principal component analysis](https://en.wikipedia.org/wiki/Principal_component_analysis) (PCA) was attempted, but yielded inferior model performance and was discontinued. Nevertheless, dimensionality reduction could be used to open up an array of robut models such as random forests. A possible alternative to PCA is [linear discriminant analysis](https://en.wikipedia.org/wiki/Linear_discriminant_analysis) (LDA). Unlike PCA, LDA emphasizes discrimination between classes. Moreover, given the large number of observations available to us, PCA is generally less likely to perform well relative to LDA [(Martinez and Kak, 2001)](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.144.5303&rep=rep1&type=pdf).

LDA coupled with random forest modeling is one potential area to pursue. Another, more radical aproach would be to reconceive the problem not as one of supervised classification, but rather outlier detection. Given that the accounts of interest only make up 6% of the data set, there is an argument to be made that such observations are in fact - outliers. In pratical terms, we could leverage Sci-Kit Learn's [One Class SVM](http://scikit-learn.org/stable/modules/generated/sklearn.svm.OneClassSVM.html) funcationality. In this fashion, future extensions might take a factorial approach, varying between random forest versus One Class SVM and LDA-reduced data versus no dimensionality reduction. 

