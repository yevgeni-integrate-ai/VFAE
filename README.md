# Two birds with one stone

One of our core values at integrate.ai is to Love People. We mean this in every sense, but the one that is most pertinent to our business, is that we wish to ensure that whatever model we create not only does not affect people negatively, but instead affects them positively. 

We do so by setting our main objective with every client to increase the overall lifetime value between them and their customers (i.e. real people like you). One of the ways we achieve this, is by making sure that our models are not biased by human attributes that we believe should not affect our model predictions. This includes, but is not limited to, gender, race, and so on. 

As much as we Love People, we also love cake (did anybody say red velvet?). Therefore we always strive to have it as well as eat it. And Variational Fair Autoencoders (VFAE) allow us to do just that. 

Briefly, autoencoders are machine learning models which are designed to reproduce the input given to them. This is in contrast to typical models, in which the target variable is different from the input. What autoencoders allow us to do, is to transform a dataset into a new mathematical representation, in what we call latent space, and have a different representation of the original data. This might seem like a silly exercise, however autoencoders are very powerful. For example, if during training we inject noise into the input, while trying to predict the “clean” dataset, we will obtain a model that is very flexible to “real-world” (i.e. messy) data.

The variational piece describes the use of variational Bayes techniques to approximate the posterior, i.e. the probability distribution of the output, the model is trying to predict.

Most interesting, and novel, is the fair component. In VFAE we explicitly give information about the properties of the data we wish to avoid have influence our models. The latent version of a dataset generated by this model contains much of the valuable details of the original dataset, but without any influence of the sensitive information. We are then able to train models on this latent representation, and obtain good predictions that are not biased or influenced by sensitive properties of the people included in the dataset.

In our models we will use neural networks to transform the data through the encoding and decoding pieces.

Now, let's see how this works. For that we would need a dataset to play with, and one which ideally includes sensitive information. Obviously, we can't use any of our clients' data, because that would put their customers' personal information at risk. So instead, we will use a publicly available dataset. Specifically, the Stanford Open Policing Project - Ohio ([link](https://www.kaggle.com/stanford-open-policing/stanford-open-policing-project-ohio)), which aggregates anonymous records of police arrest, both pedestrian and traffic, in the state of Ohio. This dataset includes sensitive information, such as the race of the people involved, which is something that we definitely do not want to influence our model's predictions. We will use whether the person was arrested as the target variable.

Let us stress that we would never actually build a model to do this, because although we wholly believe in automation, this kind of sensitive analysis should be performed by a human (at least for the foreseeable future).

Given the sensitivity of the task, we choose to optimize for precision in correctly classifying those who should be arrested. This would come at the expense of missing people who do, but we are OK with that. 

This dataset is highly imbalanced, with only about 0.6% of cases being arrests. Furthermore, it does not include many features (26 overall, with 16 being either of no use or redundant). Another challenge is that once the data is processed, which will be described below, the number of columns significantly expands due to categorical features being transformed into dummy variables. This results in a very sparse dataframe.

The first thing to do is to clean up the features. The following table lists all the features in the raw dataset, whether the feature will be kept, and reasons for doing so:

Feature | Keeping (Yes/No) | Reason
--- | :---: | ---
id | No | Irrelevant information
state | No | All are in Ohio
stop_date | Yes | The month of year might be an interesting feature (i.e. there might be some seasonal effect)
stop_time | Yes | The time of day is interesting because, for example, stops at night are more likely to result in arrests, as compared with stops during the day.
location_raw | Yes | There may be a dependence of location on odds of being arrested.
county_name | No | This feature is very similar to ‘locations_raw’
county_fips | No | This feature is very similar to ‘locations_raw’
fine_grained_location | No | This feature is very similar to ‘locations_raw’, but is too fine grained (containing 69204 unique values).
police_department | No | Only contains null values
driver_gender | Yes | An important feature
driver_age_raw | No | Only contains null values
driver_age | No | Only contains null values
driver_race_raw | No | Contains the same information as ‘driver_race’, but with some values unknown
driver_race | Yes | Will act as the sensitive feature we attempt to remove using VFAE
violations_raw | No | This is an extended version of ‘violations’, with too much detail
violations | Yes | An important feature, containing a distilled version of ‘violations_raw’
search_conducted | Yes | An important feature
search_type_raw | No | Contains mostly null values
search_type | No | Contains mostly null values
contraband_found | Yes | An important feature
stop_outcome | No | Very closely correlated with ‘is_arrested’
is_arrested | Yes | Our target variable
lat | No | Is essentially a very fine grained version of ‘location_raw’
lon | No | Is essentially a very fine grained version of ‘location_raw’
officer_id | No | There are too many unique values.
drug_related_stop | Yes | An important feature

### Let's clean the data

We take the time of day and convert it to an hour integer. 
We take the date and convert it to a month integer. 

To avoid including too many locations in the ‘location_raw’ feature, thereby significantly increasing the sparseness of the final dataframe, we keep only the most frequent 20 locations, which account for nearly 90% of all records. The remaining values are set to null, and will be imputed as described further down.

The violations column is an interesting one. It includes lists of strings, each denoting the violations (allegedly) performed by the person. For model simplicity we only want to focus on one, and we want to make sure it is the one that is most severe. Overall there are 12 unique violations included in the dataset, ordered here with #1 being the worst (which is a subjective definition):

1. dui
2. speeding
3. stop sign/light
4. license
5. cell phone
6. paperwork
7. registration/plates
8. safe movement
9. seat belt    
10. equipment
11. lights
12. truck
13. other
14. other (non-mapped)

We then replace the violations column with an integer column representing the worst violation attributed to the person. 

We take the remaining columns, which all contain categorical data, and replace null values with values taken from a probability distribution of the other values in the column. 

Finally, we then create dummy versions of every column.

The result is a data frame with 82 columns, which is now ready to be analyzed using a machine learning algorithm.

The classification model we will use will be Random Forest. 

Once we perform a pass on the dataset, we get the following results

[[[Precision and recall of the model]]]

However, it is upon a deeper look that we see an issue. Consider the distribution of the races of the people in the dataset

[[[Table which lists the distribution of race in the raw dataset, compared with the model’s prediction of arrests versus non-arrests]]]

We can see that after modelling, the fraction of African Americans and Hispanics increases, whereas the fraction of White and Asian decreases. We get a similar result if we look at the discrimination parameter proposed by Zemel et al. (2013)

[[[Discrimination values]]]

Clearly this is problematic, and this is where VFAE step it.

By transforming the dataset using a VFAE, which involves injecting information regarding the race of the drivers into the model, we obtain a dataset which contains most of the useful information, but removes the dependence on race. Let us see what happens when we retrain the model. 

[[[Precision and recall of the model]]]

We see that we still obtain comparable performance as before, however, this time the model does not pick up the drivers races, and hence does not result in bias with respect to race.

[[[Table which lists the distribution of race in the raw dataset, compared with the model’s prediction of arrests versus non-arrests]]]

[[[Discrimination values]]]

These sorts of techniques are very exciting for us at integrate.ai. Because they allow us to train great models, while making sure we are not accounting for information we don't need, nor want to.

There are, of course, limitations to these models. For one, we need to explicitly specify what features we consider sensitive. This may not always be obvious. Also, given that we transform the dataset into a latent space, which is essentially abstract, we will have difficulty explaining the reasons for our model results. For example, in the above analysis although we can be confident that race is not a factor in predicting whether a person should be arrested, we would have trouble saying what is an important factor. This is where techniques such as FairML, for example, can help.

Loving people means doing the best we can to ensure every person affected by our model is treated fairly. Doing so is a complicated task, especially when dealing with complicated non-linear models, but one that we proudly undertake. Using VFAE is just one of the tools we will employ to achieve this goal. We are investing a lot of effort to ensure our infrastructure is secure, to reduce the chance of data breaches, we are creating partnerships with leaders in the field such as Richard Zemel of the University of Toronto and Yoshua Bengio of the Université de Montréal, and we are making deliberate choices about the projects we take on from clients.
