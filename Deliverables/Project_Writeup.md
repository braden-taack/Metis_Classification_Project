Braden Taack
### Chess.com Classification Project
#### October 29, 2021
---

#### Abstract
  
The goal of this project is to use exploratory data analysis and classification methods to build a model to predict a win/loss from player statistics. The data was obtained from the [Chess.com API](https://www.chess.com/news/view/published-data-api). After initial data cleaning and model selection, I used feature engineering and cross validation analysis via GridSearchCV to select the best model using accuracy as the main metric. My final model had an accuracy of 0.702 and an F1 score of 0.72.


#### Design

This project originates from my growing interest in chess after picking up some of the beginner lessons from Chess.com. To learn more about the game and for a fun classification project, I sought out to create a classification model to predict whether a game would result in a win or loss for a player given both player's statistics. 

#### Data

The data was obtained from the [Chess.com API](https://www.chess.com/news/view/published-data-api). The full dataset consisted of 3.3 million games from the archives of 342 Chess.com streamers. Streamers were selected for their normally distributed ratings and because they were expected to play frequently. However, the dataset was too large to load into pandas and efficiently work with my machine. So for the purpose of getting a model built and tuned in a time crunch, I took a subset of this data for modeling purposes.  
  
The subset of data was gerated by randomly selecting 3 players from 10 rating bins: (0,750),(750,1000),(1000,1250),(1250,1500),(1500,1750),(1750,2000),(2000,2250),(2250,2500), (2500,2750),(2750,3100). A random sample of 1000 games from those players is queried from the db for a total of about 30000 games as the usable dataset. There were many new usernames in this dataset, ~13000, as the streamers were not gauranteed to play against other streamers. A pipeline was constructed to gather the player stats from the Chess.com API for each new player. 
  
The initial features for each player were: 'rating', 'game', 'last_rating_x', 'last_date_x', 'best_rating_x', 'best_date_x', 'wins_x', 'draw_x', 'loss_x', 'tactic_rating_low', 'tactic_rating_low_date', 'tactic_rating_high', 'tactic_rating_high_date', 'puzzle_best_attempts', 'puzzle_best_score', 'puzzle_daily_attempts', 'puzzle_daily_score', 'lesson_rating_low', 'lessons_rating_low_date', 'lesson_rating_high', 'lesson_rating_high_date',

#### Algorithms
  
*API Requests*  
  
1. Progress Bar  
    - Some cells took hours to run, so added in function for updating progress bar
3. Streamers
    - Called API page to get full list of Chess.com streamers
    - save to local mongoDB collection 'players'
3. Archives
    - Games could not be accessed via a single call, instead in YYYY/MM archives
    - Gather all available archive links for list of streamers
    - save to local mongoDB collection 'archives'
5. Games
    - Query 'archives' collection
    - Aggregate all archive links into 1 list
    - Request all game data for full archive list
    - Save to local mongoDB collection 'games'
7. Stats
    - Find set of all usernames in subset of games (derived from data section)
    - Request player stats for all usernames
    - Save to local mongoDB collection 'stats' 
  
*Data Cleanup* 
  
Data cleaning begun with missing data. There were frequently missing data for the tactics, lessons, and puzzle related features because not all players attempted those challenges. Some players wanted to jump right into matches, or matches of a certain type. There were also less experienced players who had not attempted certain game types yet and therefore had no rating or wins/loss record. For features involving ratings, the Chess.com default of 1200 was assigned to missing values. For dates, wins, losses, tactics scores, lessons taken, and puzzle scores, 0 was assigned to missing values. Any remaining nulls were dropped. 

There were also several features with date-based information. The API had formatted all date data in Epoch time. All Epoch times were reformatted to be datetime objects of YYYY/MM/DD HH/MM/SS format.  
  
*Feature Engineering*  
  
1. Dropped columns time_per_move, username, and move accuracy. 
    - time_per_move had very few observations
    - username as a categorical variable would have generated about 13,000 new feature columns after creating dummy variables. Since dataset was ~30,000 rows, this was left out. 
    - move accuracy would have been a cool metric to work with, but could not be used for predictability since it would only be known after the game occurs. 
3. Make dummy variables for columns 'rated','rules','game'.
    - Ideally, these would have helped the model find interactions among players and game types or time classes. 
    - These were not included as they did not perform well with the logistic regression model.
5. Player Win Percentage
    - Wins, Loss, and Draw were included in original features
    - Win/ (Win+Loss+Draw) was calculated for all players
    - Win percentage out performed the individual columns so they were dropped. 
    - This was intended to help standardize the metrics for the win loss record among all players. 
7. Days Since Best Rating
    - Calculated the difference (in days) of the date of the last rating and date of best rating
    - Since this was not a multiplicative feature, used to replace the individual features
    - Used to give an indication of how close the player is to their best performance level
  
*Model Selection*  
  
Initially, there was a significant issue with being able to handle a class imbalance on the draw class. The draw class made up only about ~6% of the modeling dataset. On initial modeling attempts with 5 different models (KNN, LOG, RF, DT, and XGBoost), the average accuracy for draw prediction was only 0.03%.  
  
Multiple attempts to correct the class imbalance were run with KNN, RF, and LOG REG models. Oversampling via the RandomOverSampler and SMOTE algorythms from imblearn were used. Changing the class weights was also attempted. While there was a minor increase in correctly predicted draws, there was a decline in overall accuracy and F1 score. Specifcally, the RF model and KNN model were overfitting with oversampling, and underfitting (underperforming) with class weight adjustments. As expected, the logistic model had little to no ability to predict the draw class at all. In order to simplify my problem, I decided to use just win and loss as classes for now to make the problem binary. I also made this choice to use a logistic regression model for easily interpretable results and to ensure my model would be scalable to larger datasets in the future.  
  
Before settling on logistic reggression, I also tested 4 other models using only rating as the 2 features: KNN, RF, DT, and XGBoost. I tested these by writing a function to automatically perform a grid search with 5 cross validation folds given a model, param_grid, and X_train and y_train data. The funciton would output the best parameters, score, score with a validation set, F1 score with the validation set, and a seaborn heatmap of the confusion matrix on the validation data.  
  Model|Param_Grid Hyperparameter|Accuracy|F1 Score
  -----|-------------------------|--------|--------
  KNN|n_neighbors|0.69|0.69
  RF|n_estimators|0.66|0.66
  DT|n_estimators|0.65|0.65
  XGB(no grid search)||0.58|0.65
  LOGREG|C|0.70|0.70
  
From the results, I settled on logistic regression and spent some additional time tuning hyperparamters with the grid search method. I tested this time with a wider range of C's, different solvers, new features, and differnt cost metrics. 
  
#### Tools

- [Chess.com API](https://www.chess.com/news/view/published-data-api) as data source 
- MongoDB for local storage 
- Pandas for data ingestion and basic exploration
- Numpy and Pandas for data manipulation
- datetime for formatting time-based data
- sklearn for model preprocessing, cross validation, model selection, and final model training and testing
- Seaborn for data visualization
- imblearn for class imbalances

#### Conclusions  
  
My final model consisted of a Logisitic Regression model with the following features and coefficients. The accuracy was 0.703 and the F1 score was 0.73.
  
  Intercept|-9.764
  ---------|------  
  
  Feature|coef
  -------|----
  Fastest_laps_Pos| 0.308
  Fastest_laps_Lap| 0.003
  SG_Pos_Q_Laps|0.005
  FL_avg_p_laps|-0.0003
  Fastest_laps_Avg_Speed| 0.039
  Pit_stop_summary_Stops|0.74
  Pit_stop_summary_Total|-0.001
  Starting_grid_Pos|0.272
  Qualifying_Laps|0.058
  Practice_3_Pos|0.077
  Practice_3_Laps|0.075
  Practice_2_Pos|0.063
  Practice_2_Laps|0.083
  Practice_1_Pos|0.08
  Practice_1_Laps|0.069
  P3_P2_delta| 0.006
  P2_P1_delta|0.016
  Q2_Q1_delta|0.01