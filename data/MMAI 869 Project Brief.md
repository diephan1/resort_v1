![Image result for smith school of business logos][image1]

**Team Project Brief**

MMAI 869 (Machine Learning and AI)

Dr. Stephen Thomas

*Updated: April 15, 2026*

# Overview

The Team Project is a machine learning competition. The main objectives of this project are for teams to:

1. Push ML models to the limit: Explore techniques in data cleaning, feature engineering, feature selection, data augmentation, ML algorithms, hyperparameter tuning, ensembles, and more.  
2. Cross-dataset application: Understand the impact of building a model on one dataset (e.g., train) and applying it to another (e.g., test).  
3. Collaborative project management: Practice dividing a complex project among team members within a tight timeline.

# The Project

*Steve’s Luxury Resort* is a premium, all-inclusive resort chain with properties across three major regions. The resort has recently noticed that a significant portion of guests are not returning for future stays or are canceling their loyalty memberships shortly after their visit.

Your task is to build a classification model to predict which guests are likely to churn (not return / cancel loyalty membership) based on their booking information and on-site spending behavior. Understanding churn patterns will help the resort target at-risk guests with retention campaigns.

## The Competition

Teams will be provided with a training dataset of past guests:

| Feature | Description |
| :---- | :---- |
| GuestID | Unique identifier for each guest. |
| BookingDate | Date the reservation was made (YYYY-MM-DD format) |
| PromoCode | Promotional code used at booking: PromoA, PromoB, or missing if none |
| Region | The guest's home region: Americas, Europe, or AsiaPacific. |
| AllInclusive | Whether the guest purchased the premium all-inclusive package (True/False). Guests on this package typically don't charge amenities separately. |
| Room | Room assignment in format Wing/Floor/View. Wings are A-G, floors are numbered, and view is either P (Pool) or S (Sea). |
| PackageType | The experience package selected: Relaxation (most popular), Adventure, or Wellness. |
| Age | Guest's age in years. |
| VIP | Whether the guest has VIP loyalty status (True/False). |
| RoomService | Amount billed to room service, in dollars. |
| Dining | Amount spent at resort restaurants and bars, in dollars. |
| Retail | Amount spent at resort shops and boutiques, in dollars. |
| Spa | Amount spent on spa and wellness services, in dollars. |
| Entertainment | Amount spent on entertainment (shows, excursions, activities), in dollars. |
| LoyaltyPoints | Guest's loyalty program points balance |
| SurveyScore | Post-stay satisfaction survey score (1-5) |
| DaysSinceEmail | Days since guest last opened a marketing email |
| BookingChannel | How the reservation was made |
| AgeGroup | Categorical age bracket |
| ReferralSource | How the guest heard about the resort |
| Churned | Target. True if the guest churned (cancelled membership or did not return), False if they remained loyal. |

Teams will use the training dataset to build ML models that can predict whether a guest churned. Teams will also be provided with a competition test dataset containing features of additional guests. Teams will use their models to predict whether each of the guests in the test set churned. Teams will save the models’ predictions to a CSV file, and submit it to the competition website:

[Kaggle: Steve's Luxury Resorts](https://www.kaggle.com/t/3411646fd31446aabfaa15aa0bbdfbcb)

The competition website will automatically score their predictions (as it knows the correct answers) using the F1 metric and will add the team’s score to the competition leaderboard.

Teams shall:

* Try many different techniques for preprocessing, cleaning, feature engineering, ML algorithms, hyperparameter tuning, and all the other topics we learn in this course.  
* Try at least one non-tree-based algorithm, such as KNN, Logistic Regression, NNs, or SVM.   
* Try at least one tree-based algorithm, such as Decision Trees, Random Forests, LGBM, XGBoost, or CatBoost.  
* Compete\! Do whatever you can to move up the rankings. Try things that should work, and try things that shouldn’t work. Experiment, iterate, and go for it.  
* Feel free to do as much data exploration/EDA as necessary to understand the dataset. For example, you can:  
  * Use plots and graphs to tell a story and discover correlations/trends/outliers  
  * Use association rule learning to uncover patterns and trends  
  * Use cluster analysis to build clusters, and describe the clusters and build personas. 

# Deliverables and Rubric

Teams will be responsible for a 12-minute live presentation that outlines the story of their competition journey. An accompanying report is not necessary. 

### Presentation Requirements

* **Final Leaderboard Accuracy Score**: Include the F1 score of your best model.   
  * *Note*: You will not be graded on the actual performance/ranking in the competition (although I wish you the very best of luck).  
* **Distinguish Scores**: When discussing your various models, clearly differentiate between a model’s cross-validation score (CV score), which you estimated yourself, and its leaderboard score (LB score), which Kaggle calculates on the competition test data. This will allow me to tell how good your CV process was at estimating LB scores.

### Grading Criteria

1. **Data Cleaning, Preprocessing, and Feature Engineering (10%)**:  
   * Describe the cleaning, preprocessing, and feature engineering steps you tried  
   * Summarize which steps worked and which didn’t (i.e., how each affected the CV scores).  
2. **Machine Learning Algorithms (10%)**:

   * List the ML algorithms you tried, including at least one non-tree-based algorithm.  
   * Evaluate the performance of each algorithm using cross-validation.  
   * Compare each model’s CV score with its LB score.  
3. **Hyperparameter Tuning (10%)**:

   * For each algorithm, report the CV score with the default hyperparameter values.  
   * Then, describe the hyperparameter tuning procedures/algorithms you tried.  
   * Present the final/best hyperparameters along with their CV scores.  
4. **Innovative Approaches and Insights (10%)**:

   * Highlight any additional techniques or approaches you tried.  
   * Share any particular insights or “Eureka\!” moments.  
5. **Detailed Model Description (10%)**:

   * Provide a detailed description of your best model/submission.  
   * Quantify the model’s performance using confusion matrices and associated metrics.  
   * Describe the drivers (i.e., feature importances) of your model’s performance.  
   * Show at least three training instances (i.e., feature values) that your model predicted correctly and three that it predicted incorrectly. Draw insights from these examples.  
6. **Next Steps (10%)**:

   * Describe what you would try if you had more time/budget. Be as specific as possible.  
   * Specify what you would need (in terms of data, compute power, algorithms, etc.) to improve the model’s performance.  
7. **Lessons Learned (10%)**:

   * Include concise and helpful lessons learned during the project.  
8. **Clarity of Presentation (30%)**:

   * Assess the overall clarity and understandability of the presentation, including slide design and oral delivery.  
   * Evaluate the ability to answer questions during the Q\&A portion of the presentation.

## Presentation Tips

* This is a short presentation. Don’t linger on unimportant stuff. Focus on the juicy bits.  
  * Don’t include an agenda slide. This presentation is not long enough to need one, and spending time on an agenda is not worth the time.  
  * Don’t spend time on team member introductions. (“*Hi everyone, I’m Steve, and this Bill, and over there is Mary, and there’s Hector, and then we have Mona, and finally my dog Roofus. We’re part of Team Toronto and we have been working on this project together.”)* It takes too long and is not worth the time. (In the past, teams have spent 1-2 minutes introducing themselves. That’s almost 10% of the entire presentation spent on fluff\!)  
  * Don’t spend any time on the title slide – just get started. (In the past, teams have spent 1-3 minutes with the title slide showing, talking about “meta” topics, like “*you know, we really had a great time in this project, and I’m happy to be here, and in fact, my father used to work at a pharmacy, but then he moved into retail, but I still love the movies, you know, and my teammates, uh, my teammates and I are excited to share our results, and I wanted to thank Uncle Steve for letting us use his code, and I’m kinda nervous right now which is why I’m talking a lot hahaha. Can you see my screen?*”) The clock is ticking, and everyone has limited patience. Everyone wants you to get started \- so just get started.  
* If necessary, create an Appendix with additional information:  
  * Cleaning steps  
  * Package details  
  * Modelling details (algorithm choice, hyperparameter values, etc.)  
  * Code snippets  
  * Etc.  
* Be creative and have fun\!  
  * Pictures are better than words  
  * Graphs are better than words  
  * Charts are better than words  
  * Tables are better than words  
* The target audience for this presentation is your average MMA student: a tech-savvy manager who wants to learn what you learned.

# Language and Platform

Teams may use any programming language and IDE/platform/tool.

I recommend using the Python programming language (using standard packages like pandas and scikit-learn) on the Jupyter Notebook platform. Kaggle Notebooks or Google Colab will be perfect for this project. Other IDE options include PyCharm and Spyder.

For tips on learning Python and Jupyter, please see the “Programming Languages and Tools” section of the course portal.

See the course portal for the link to the competition website.

# FAQ

*Can we use your example Python Notebooks in your GitHub repository?*

Absolutely\! Yes. Please use them as a launching off point.

*Is there a Subject Matter Expert (SME) to whom we can ask questions about the data?*

No, but Uncle Steve and the TA are here to help.

*My code has an error. What should I do?*

First, you should understand the error. Read the whole thing. What is it telling you? The error message will often lead you directly to the answer if you read it carefully.

If the error message isn’t clear, or you don’t know how to solve it, you should Google the error. Google is by far your best friend. You probably aren’t the first person to have this error. You can also try Claude or other GenAI tools. They can be very helpful\!

If you can’t figure it out by Googling, you should consult your teammates. Teams that learn together stay together\! 

If you still have the error, you should read your code carefully. You know what they say: 3 hours of debugging can prevent 3 minutes of reading your code\! (Or something like that. It’s a joke.)

Next, you should ask the TA via email. When you ask the TA, please include the following: 

* What exactly is the error message?  
* What have you tried so far to fix your code?  
* What kind of data is in the data frames/variables involved (if any)?  
* What have you Googled? What documentation have you read?  
* What will you try if you can’t get this to work? (What is Plan B?)

The more information you give the TA, the higher the probability that the TA can help you.

Finally, email Uncle Steve and/or bring your issue to the next office hours with Uncle Steve if all else fails.

*How can I improve the performance of my model?*

* Feature engineering is your friend  
  * In particular, there are a lot of categorical features with many unique values. What can we do? (Hint: check out the package [category\_encoders](https://contrib.scikit-learn.org/category_encoders/) or use an algorithm with built-in categorical support like Histogram-based Gradient Boosting, LGBM, or CatBoost.)  
* Boosting is your friend  
  * XGBoost, LightGBM, CatBoost  
*  The data is probably imbalanced, so you can do things like:  
  * add weights  
  * downsample  
  * upsample  
* Hyperparameter tuning is your best friend  
  * GridSearchCV, RandomizedSearchCV, Optuna  
  * But don’t overfit\! Cross-validation is your friend.

*In the real world, what’s the point in spending an extra three months to build a model that is only 0.05% better?*

I recognize that ML competitions are not always realistic examples of ML in business settings. However, I believe there is tremendous learning value when you try to push something to its absolute limits. I hope that the competitive nature of this project will motivate teams to work hard and try many new techniques quickly, thereby allowing them to learn what works and what doesn’t.

[image1]: <data:image/png;base64,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>