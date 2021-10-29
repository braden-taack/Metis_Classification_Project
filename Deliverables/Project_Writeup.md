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
1. Remove obvious duplicate columns, hour-of-the-day data, and gap data  
  - Obvious duplicate columns included the driver number, which is a unique number to each driver. A Driver column was already incorporated so these were not necessary.  
  - I was not interested in incorporating hour-of-the-day data because its likelihood to have a useful impact on the model was questionable. In each race, there would be the same hour-of-the-day for each finish position.  
  - Gap data was not included either because it was already captured in the respective Time columns. 
2. Drop columns based on VIF (variance inflation factor)
  - 'Starting_grid_Time','Qualifying_Q3', and 'Qualifying_Pos' had huge VIF scores so were dropped. 
3. Use a LASSO model to identify more features to be removed. 
  - Despite some Time features with coefficients -> 0 from the LASSO model, all features were initially for alternate feature engineering.
4. Attempt to normalize all time data when grouped by year and track
  - The goal of this was to normalize the time data so it would more easily be compared from track to track.  
  - While this resulted in a similar model score, it was actually *worse* than the raw linear regression model.
5. Create dummy variables for Driver and Car
  - Using the base Linear Regression model from model selection, Driver and Car (essentially team) were independently assigned dummy variables. 
  - Using Driver dummy variables with an other category including drivers with less than 25 total races made the R^2 worse.
  - Using Car (aka team) dummy variables with an other category including cars with less than 50 total races also made the R^2 worse.
  - Neither categorical feature was included. 
7. Add new interaction features
  - 'P3_P2_delta', 'P2_P1_delta', 'Q2_Q1_delta' replaced the respective qualifying and practice lap times with the differences between laps. The goal of this was to judge based on improved performance from one practice to the next rather than just time. This marginally improved the R^2 so it was kept. 
  - 'SG_Pos_Q_Laps' multiplied the starting grid position by the number of qualifying laps
  - 'FL_avg_p_laps' multiplied the the fastest lap average speed by the total number of practice laps
  - 'P3_Pos_P3_Laps', 'P2_Pos_P2_Laps', 'P1_Pos_P1_Laps' multiplied the number of practice laps by the practice position (ranking). These were not kept. 
  - The goal of each of these interaction terms was to explore the saying: "Practice makes perfect". Theoretically, with more practice, the better the driver would perform during the actual race. These interaction terms did prove useful and increased R^2.

*Model Selection*  
4 models were tested for this project using cross validation methods via sklearn: Linear Regression, Lasso, Ridge regression, and 2nd degree polynomial. For every cross validation test, the overall data set was initially broken up into 80% train and 20% test data. The training data was then used to cross validate using the kfolds method with 5 folds and scoring based on R^2.  
  
The results all had similar R2 values with the exception of the polynomial model which did worse. I selected the basic linear regression model based on the results in order to retain the features and maintain simplicity. 

#### Tools

- [Formula 1 website](https://www.formula1.com/en/results.html/2021/races.html) for data source using Selenium and BeautifulSoup
- Pandas for data ingestion and basic exploration
- Numpy and Pandas for data manipulation
- datetime for cleaning lap time data
- sklearn for model preprocessing, cross validation, model selection, and final model training and testing
- Matplotlib and Seaborn for plotting

#### Conclusions  
  
My final model consisted of a Linear Regression model with the following features and coefficients. The R^2 was 0.684 and the MAE was 2.187.
  
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
