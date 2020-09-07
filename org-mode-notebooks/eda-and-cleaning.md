
# Table of Contents

1.  [Imports](#orgf12e02e)
2.  [EDA](#org42725ec)
    1.  [How many games did each team play?](#orga463df8)
    2.  [Correlations](#orgdec3001)
    3.  [Simple profile: dribbles before shots](#org78c97b8)
3.  [Cleaning](#orgb98bf08)
        1.  [Restrict vars and inspect](#org82c68c5)
        2.  [Missing shot clock values](#org981e40b)
        3.  [Clip touch times](#org666cb14)
    1.  [Output data frame](#org55607c1)
    2.  [Summary of cleaning](#orgc2306b6)



<a id="orgf12e02e"></a>

# Imports

    import numpy as np
    import seaborn as sns
    import pandas as pd
    import matplotlib.pyplot as plt
    import re

Set up seaborn

    sns.set(context="notebook",
      rc={"font.size":12, "font.family":"sans-serif", 
          "font.serif":"P052", # note: kills the original list font.serif
          "axes.titlesize":"large", "axes.titlepad":20, 
          "axes.titleweight":"bold",
          "axes.labelsize":"medium", "axes.labelpad":10, 
          "axes.labelweight":"light",  # lighter is not actually an option
          "xtick.labelsize":"small", "ytick.labelsize":"small", 
    #      "axes.facecolor":"white", # vertical axis padding is werid with this
          "legend.fontsize": "medium",
          "xtick.major.width":3, "ytick.major.width":3,
          "legend.loc":"best",
          "figure.dpi":200, "figure.titlesize":"large", 
          "figure.titleweight":"bold"
      }
    )
    
    context: "notebook"

    df = pd.read_csv("../data/shot_logs.csv")
    df


<a id="org42725ec"></a>

# EDA


<a id="orga463df8"></a>

## How many games did each team play?

The GAME<sub>ID</sub> is formatted in one of two ways, depending ont the value of LOCATION:

-   AWAY @ HOME if the game was a home game for the player (i.e., LOCATION=H)
-   HOME vs. AWAY if the game was an away game for the player (i.e., LOCATION=A)

Therefore, the team of the shooter in question is always listed first.

We find of the distinct values GAME<sub>ID</sub> and record the corresponding matchup.

    # The away team takes a shot in every game, 
    # so we can throw out half of the entries.
    ddf = df[df.LOCATION=="A"]
    ddf = ddf.assign(AWAY_TEAM=lambda ddf: ddf.MATCHUP.apply(lambda s: s[-9:-6]))
    ddf = ddf.assign(HOME_TEAM=lambda ddf: ddf.MATCHUP.apply(lambda s: s[-3:]))
    ddf = ddf[["GAME_ID", "AWAY_TEAM", "HOME_TEAM"]].drop_duplicates()
    print(ddf)
    print(f"Compare with df.GAME_ID.nunique() = {df.GAME_ID.nunique()}.")

Many games are missing. There are 82\*30/2 = 1230 games in the regular season.

Aggregate by team

    # count games played by a given team with abbreviation ab
    team_abbrevs = df.MATCHUP.apply(lambda s: s[-3:]).unique()
    num_games = np.vectorize(lambda ab: ddf[(ddf.HOME_TEAM == ab) | (ddf.AWAY_TEAM == ab)].shape[0])
    pd.DataFrame({"games_played": num_games(team_abbrevs)}, index=team_abbrevs).T

Find the dates of the games played.

    rgx = re.compile("^(.*\d{1})") # capture date part of the values of MATCHUP.
    dates = df.MATCHUP.apply(lambda s: re.match(rgx, s).group(1)).drop_duplicates()
    dates = pd.to_datetime(dates).to_frame() # sort and produce frame.
    dates.sort_values(by="MATCHUP").reset_index(drop=True)


<a id="orgdec3001"></a>

## Correlations

We show the correlations between all of the revalant (dropping numerical ID's) variables across the entire dataset.

    drop_vars = ['GAME_ID', 'MATCHUP', 'LOCATION', 'W', 
                 'FINAL_MARGIN', 'CLOSEST_DEFENDER', 
                 'CLOSEST_DEFENDER_PLAYER_ID', 
                 'player_name', 'player_id']
    ddf = df.drop(drop_vars, axis=1)
    
    _ = sns.heatmap(ddf.corr(), cmap="YlGnBu")

We see a number of expected correlations. For example: 

-   There is a weak correlation between shot distance and closest defender distance. This is because challenging a player at the basket requires closer coverage than covering a player shooting from farther out.
-   We see a strong correlation between dribbles and touch time. This is because, in  This is to be expected, since, typically, longer touch times require more dribbles from the player, and, conversely, a larger number of dribbles tends to indicate a longer touch time.

For players near the basket (at least), it can happen that DRIBBLES and TOUCH<sub>TIME</sub>
are less strongly correlated. E.g., Anthony Davis in losses, Tyson Chandler in wins.

    def plot_player_heatmap(pname):
        ddf = df[(df.player_name==pname)]
        ddf =ddf.drop(drop_vars, axis=1)
        g = sns.heatmap(ddf.corr(), cmap="YlGnBu")
        _ = g.set_title(f"{pname.title()} Correlations Heatmap")
        return g
    
    pname = "stephen curry"
    _ = plot_player_heatmap(pname)

    pname = "deandre jordan"
    _ = plot_player_heatmap(pname)


<a id="org78c97b8"></a>

## Simple profile: dribbles before shots

A very simple characteristic of a player's shot profile is their tendency to take dribbles. For instance, some players tend to create shots for themselves by moving around with the ball, while others tend to move with the ball and shoot shortly after catching a pass. 

We compare James Harden, a player who frequently handles the ball for a long time before shooting, with Joe Ingles, who is more of a catch-and-shoot player. 

    # Note: there is an error in the dataset: "jon ingles" is really "joe ingles"
    # We will fix that here and again, alter, when we prepare the data for clustering.
    
    def jon_to_joe(name):
        if name == "jon ingles":
            return "joe ingles"
        else:
            return name
    df.player_name = df.player_name.apply(jon_to_joe)
    
    players = ["james harden", "joe ingles"]

    def compare_dribbles(players, norm_hist = False):
        fig, ax = plt.subplots()
    
        list(map(lambda s: 
                 sns.distplot(df[df.player_name==s].DRIBBLES, 
                              bins=range(31), kde=False, 
                              label=s.title(), norm_hist=norm_hist), 
                 players))
    
        plt.xlabel('Dribbles')
        if norm_hist:
            plt.ylabel('%')
            title_string = "Percentage of shots taken after a given number of dribbles"
        else:
            plt.ylabel('Shots')
            title_string = "Number of shots taken after a given number of dribbles"
        plt.legend()
    
        plt.title(title_string)
    compare_dribbles(players, norm_hist=False)

    compare_dribbles(players, norm_hist=True)


<a id="orgb98bf08"></a>

# Cleaning


<a id="org82c68c5"></a>

### RUN Restrict vars and inspect

We will try understand how a shooter's shot selection, described in terms of a restricted set of variables, is a good predictor of a win or loss for the team. We expect that this can only be the case for players that are one of the most important players on their team.

    keep_vars = ["GAME_ID", "W", "SHOT_CLOCK", 
                 "GAME_CLOCK", "DRIBBLES", "TOUCH_TIME", 
                 "SHOT_DIST", "player_name"]
    cluster_vars = ["SHOT_CLOCK", 
                    "DRIBBLES", 
                    "TOUCH_TIME", 
                    "SHOT_DIST"]
    df = df[keep_vars]

We check to see which columns contain a missing value.

    print(df.isna().any())

There are missing values of SHOT<sub>CLOCK</sub>, and not of any other variable. These come both from the clock being turned off at the time of the shot (this is resolvable by using the game clock at the time) and from the values simply being missing (I have often seen shot clocks malfunction while watching a game).

How prevalent is this problem?

    df_na = df[df.SHOT_CLOCK.isna()] # frame of rows from df with missing SHOT_CLOCK value
    num_na = df_na.shape[0]
    print(f"There are {num_na} rows out of 128069", 
          f"({np.round(100*num_na/df.shape[0], 1)}%)", 
          "with a missing value of SHOT_CLOCK.")

Here is a basic description of the variables we intend to use for clustering:

    print(df[cluster_vars].describe())

Everything seems to be in order except for some missing SHOT<sub>CLOCK</sub> values and the existence of some negative values of TOUCH<sub>TIME</sub>. How many such values exist?

    df_neg = df[df.TOUCH_TIME<0]
    print("The number of negative negative TOUCH_TIME",
          f"values is {df_neg.shape[0]}.")
    df_neg.nunique()

We see that the problem of negative touch times is prevalent: it occurs in 249/904 games, in a variety of SHOT<sub>CLOCK</sub>/GAME<sub>CLOCK</sub> settings. However, it only occurs for two values of DRIBBLES:

    df_neg.DRIBBLES.value_counts()


<a id="org981e40b"></a>

### Missing shot clock values

1.  RUN Replace GAME<sub>CLOCK</sub> column

    What does GAME<sub>CLOCK</sub> look like when SHOT<sub>CLOCK</sub> is nan? Values of the variable GAME<sub>CLOCK</sub> are times in the form mm:ss. We convert those times to seconds.
    
        def mmss_to_sec(s):  # is a series who entires are strings of times of the form "mm:ss"
            ddf = s.str.split(":", expand=True)
            ddf = ddf.astype(int)
            return ddf[0]*60 + ddf[1]
        df = df.assign(GAME_CLOCK=mmss_to_sec(df.GAME_CLOCK))
        df_na = df_na.assign(GAME_CLOCK=mmss_to_sec(df_na.GAME_CLOCK))
    
    I treat those missing values with GAME<sub>CLOCK</sub> <= 24 as uncontroversial with regard to how they should be imputed. In such cases, the shot clock is turned off. I'll fill those by using the game clock value. The remaining values are controversial:
    
        contro = df_na[df_na.GAME_CLOCK>24]
        print(f"There are {contro.shape[0]} controversial rows.")
        contro.nunique()

2.  Analysis of controversial shots

    How many controversial shots are there in each game?
    
        vcs = pd.DataFrame(contro.GAME_ID.value_counts())
        vcs = vcs.rename(columns={"GAME_ID":"# missing"})
        vcs.index.name = "GAME_ID"
        print(vcs)
    
    In the worst case, Game 21400339, almost all of the shots have a missing SHOT<sub>CLOCK</sub> value:
    
        bad_ID = 21400339
        num_total = df[df.GAME_ID==bad_ID].shape[0]
        num_missing = contro[contro.GAME_ID==bad_ID].shape[0]
        print(f"In game {bad_ID}, {num_missing} shots", 
              f"out of {num_total} have a missing value",
              "of SHOT_CLOCK.")
    
    What about a game with only a few controversial rows?
    
        contro[contro.GAME_ID == 21400741]
    
    Among these, we see that some have short touch time, or few dribbles, or short distance. This suggests that some were shot before the shot clock reset.
    
        num_contro = contro.shape[0]
        num_short = contro[(contro.TOUCH_TIME<4)].shape[0]
        print(f"{num_short} out of {num_contro} outstanding",
              f"shots have a touch time of <4 seconds.")
    
    We conclude ~75% of the controversial values have a short touch time, suggesting a missed shot clock reset.

3.  Less conservative policy:

    -   Drop those games with > <span class="underline">10? 20?</span> contraversial rows. For the rest: 
        -   If GAME<sub>CLOCK</sub> =< 24: fill SHOT<sub>CLOCK</sub> using GAME<sub>CLOCK</sub>;
        -   If DRIBBLES < <span class="underline">5?</span>: assume the clock didn't reset, 
            and set SHOT<sub>CLOCK</sub> = 24 - TOUCH<sub>TIME</sub>.
    
    Note: the last part of this is a bad polciy in more recent seasons, when the shot clock would only reset to 14 after an offensive rebound.

4.  Most conservative policy

    -   Drop all 35 games with a controversial value.
    
    The benefit of this policy is that it would allows us to conduct a game-by-game analysis for each player. We don't do this in our shot profiling application, so we opt not to use this policy since it would drop a large number of useful datapoints:
    
        num_shots = sum(df.GAME_ID.isin(contro.GAME_ID.unique()))
        print(f"This policy would drop {num_shots} shots.")

5.  RUN Our policy:

    We will follow this policy for resolving missing SHOT<sub>CLOCK</sub> values:
    
    -   Drop controversial rows.
    -   For the remaining rows, GAME<sub>CLOCK</sub> =< 24. For these, fill SHOT<sub>CLOCK</sub> using GAME<sub>CLOCK</sub>.
    
        df = df.drop(contro.index)
        GC_vals = df[df.SHOT_CLOCK.isna()].GAME_CLOCK.unique()
        print("For the remaining missing values," 
              f"the game clock is <= {GC_vals.max()}")
    
    We now fill the remaining missing values with 0:
    
        new_SHOT_CLOCK = df.SHOT_CLOCK.fillna(df.GAME_CLOCK)
        df = df.assign(SHOT_CLOCK = new_SHOT_CLOCK)
    
    We check each column to verify that there are no missing values:
    
        print(df.isna().any())


<a id="org666cb14"></a>

### RUN Clip touch times

    print(f"There are {sum(df.TOUCH_TIME==0)} shots with TOUCH_TIME==0.")

We saw before that all but one of the shots with negative touch time was preceded by zero dribbles (like a catch-and-shoot or tip-in situation). Since TOUCH<sub>TIME</sub>==0 occurs naturally in the dataset, it is natural to clip negative touch times from below to 0 whenever those shots were preceded by zero dribbles. This leaves only one outstanding row:

    df[(df.TOUCH_TIME<0)&(df.DRIBBLES>0)].iloc[0]

The one shot with two dribbles looks like a touch time of 1 second&#x2013;the shot clock was at 23. Hence we can reasonably set all of these touch times to zero, or possibly set the exceptional shot's touch time to 1 second. I'll just set them all to 0:

    masked_TOUCH_TIME = df.TOUCH_TIME.mask(df.TOUCH_TIME<0, 0)
    df = df.assign(TOUCH_TIME=masked_TOUCH_TIME)


<a id="org55607c1"></a>

## Output data frame

    print(f"Exporting a dataframe of shape {df.shape} to csv.")
    df.to_csv("../data/shot_logs_prepped.csv", index=False)


<a id="orgc2306b6"></a>

## Summary of cleaning

-   We kept the variables:
    ["GAME<sub>ID</sub>", "W", "SHOT<sub>CLOCK</sub>", 
     "GAME<sub>CLOCK</sub>", "DRIBBLES", "TOUCH<sub>TIME</sub>", 
     "SHOT<sub>DIST</sub>", "player<sub>name</sub>"]

-   We converted GAME<sub>CLOCK</sub> from strings to ints (unit: seconds).

-   We filled handled missing SHOT<sub>CLOCK</sub> values by:
    -   Replacing SHOT<sub>CLOCK</sub> by GAME<sub>CLOCK</sub> whenever GAME<sub>CLOCK</sub><=24.
    -   Dropping the remaining rows (2013/128069; 1.6% of the data) .

-   We clipped negative touch times to 0.

