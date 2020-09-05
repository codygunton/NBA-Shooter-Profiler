# NBA Shooter Profiler

We provide a tool for game planning by analyzing an opposing player's shot profile.
For instance, for a given player X, we might like to know:
  - From what distances does X tend to shoot the ball?
  - For how long does X tend to hold the ball before shooting?
  - How does X's behavior depend on the shot clock; does their decision making change under stress?
  
Our tool segments X's shots into shot types by clustering, then gives a multi-slice visualization of their shot profile. It also provides a statistical tests of the question:
  "Does X's shot profile differ significantly in wins v.s. in loses?"

We will use the NBA shot logs data available [here](https://www.kaggle.com/dansbecker/nba-shot-logs), which covers around the first 75% of the 2014-2015 season, and which is available locally at [shot_logs.csv](./data/shot_logs.csv). We provide several files documenting our tool.

 - [eda.md](eda.md) A basic overview of the data.
 - [cleaning.md](cleaning.md) A file demonstrating our cleaning of the shot logs data (TODO: Say one or two more sentences.)
 - [nba-shooter-profiler.ipynb](nba-shooter-profiler.ipynb) A Google Colab notebook performing the cluster analysis. The notebook can also be run interactively:
 [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1rNeJoEB4d4X9axz7dgKi6ebDLF1oumHN?usp=sharing).
 - Example analyses:
   - [...](examples/...)
   - [...](examples/...)
   - [...](examples/...)
