---
layout: post
title: Fantasy Baseball Projections in Python
subtitle: 2020 Shortened Season
gh-repo: jacobrichey/Fantasy-Baseball
gh-badge: [star]
tags: [Python, baseball]
---

*Note: Article is still being edited.*

Well, the 2020 MLB Season has finally arrived, roughly four months after the previously scheduled start date. In advance of Opening Day tonight, I'm going to share my fantasy baseball player rankings, and the methodology behind the system. Note these rankings are designed for a snake draft (no auction values) in an ESPN Head-to-Head Categories league with a standard scoring system. These rankings are for fantasy baseball, and do not reflect overall player value. The general approach is to ensemble nine reputable projection systems to produce a consensus player ranking. The sample skewness for player projections is used to add potential boom/bust notes for players. All links to projections can be found in Jupyter Notebook write up of this project, located in the GitHub repository linked above.

First, we'll need to import the necessary modules. 

```
import pandas as pd
import numpy as np
import seaborn as sns
import matplotlib.pyplot as plt
from scipy.stats import zscore, skew
```

We'll now begin the ardous process of reading in the various hitter projections. We will want to be able to merge projection systems and historical performance statisticsw based on player id's over player names, to ensure accuracy. However, many sites have their own id conventions, so we will need a method to convert between id mappings. The Chadwick Register will be used to provide these conversions, with the extensive Steamer Projections providing official player naming conventions. 

```
names = pd.concat([
    pd.read_csv("fb2020_data/steamer_batters_2020.csv").set_index('playerid').loc[:, 'Name'],
    pd.read_csv("fb2020_data/steamer_pitchers_2020.csv").set_index('playerid').loc[:, 'Name']])

# create conversion df with player name, fangraphs id, and mlbam id
# will use fangraphs id as index throughout
register = (pd.read_csv("fb2020_data/people.csv", dtype = 'str')
            .loc[:, ['key_mlbam', 'key_fangraphs']]
            .merge(names, how = 'right', left_on = 'key_fangraphs', right_on = 'playerid')
            .dropna()
            .query("key_fangraphs != '10171'")) # double Jose Ramirez entries, remove incorrect key
```

The ESPN Projections are a mess, but because the fantasy league is within an ESPN format, it's a necessary inclusion. We will also add ESPN positional eligibility later. Some player names need to be changed to sync with the register (ESPN does not publish player id's). Team names and injury designations are also removed, as they will show up in the draft software anyway. 

```
espn_batters = pd.read_csv("fb2020_data/espn_batters_2020.csv")

# cells of importance (Name, Position) located every third row over the Rank and Player columns
rank = espn_batters['RANK'].iloc[::3]
espn_names = (espn_batters['PLAYER'].iloc[1::3]
              .replace(['Pete Alonso', 'Gio Urshela', 'Nate Lowe'],
                       ['Peter Alonso', 'Giovanny Urshela', 'Nathaniel Lowe'])
              .str.replace('DTD|IL10|IL60', '', regex = True))
positions = espn_batters['PLAYER'].iloc[2::3].str.replace(
    ("Ari|Atl|Bal|Bos|ChC|CHW|Cin|Cle|Col|Det|FA|Hou|KC|"
     "LAA||LAD|Mia|Mil|Min|NYM|NYY|Oak|Phi|Pit|SD|Sea|SF|"
     "StL|TB|Tex||Tor|Wsh|ChW"), '', regex = True)
hab = espn_batters['H/AB'].iloc[0:300].str.split("/", expand = True)

# combine isolated Rank, Name, and Position with player stats
espn_batters = (pd.DataFrame({
    'ESPN_Rank': rank.reset_index(drop = True),
    'Name': espn_names.reset_index(drop = True),
    'Position': positions.reset_index(drop = True),
    'AB': hab[1],
    'H': hab[0],
    'R': espn_batters['R'].iloc[0:300],
    'HR': espn_batters['HR'].iloc[0:300],
    'RBI': espn_batters['RBI'].iloc[0:300],
    'SB': espn_batters['SB'].iloc[0:300],
    'AVG': espn_batters['AVG'].iloc[0:300]})
                .merge(register[['key_fangraphs', 'Name']], on = 'Name') # add fangraphs id, set as index
                .assign(System = "ESPN")
                .rename(columns = {'key_fangraphs': 'playerid'})
                .set_index('playerid')
                .query("R != '--'") # remove row of TBD projections for Yasiel Puig (unsigned free agent)
                .drop_duplicates()
                .dropna()
                .astype({'ESPN_Rank': 'int32', 'AB': 'int32', 'H': 'int32', 'R': 'int32',
                        'HR': 'int32', 'RBI': 'int32', 'SB': 'int32', 'AVG': 'float64'}))
```

A similar process is repeated for Number Fire and Razzball Hitter Projections. Full code can be found through the GitHub link at the top of the article.

FanGraphs publishes several projection systems in the same format, so we can write one function to read the rest.

```
# remaining projection systems are similar
def read_batters(path, system):
    return (pd.read_csv(path)
            .set_index("playerid")
            .loc[:, ["ADP", "PA", "AB", "H", "R", "HR", "RBI", "SB", "AVG"]]
            .assign(System = system))

atc_batters = read_batters("fb2020_data/atc_batters_2020.csv", "ATC")
fgdc_batters = read_batters("fb2020_data/fgdepthchart_batters_2020.csv", "FanGraphs Depth Charts")
steamer_batters = read_batters("fb2020_data/steamer_batters_2020.csv", "Steamer")
thebat_batters = read_batters("fb2020_data/thebat_batters_2020.csv", "The Bat")
thebatx_batters = read_batters("fb2020_data/thebatx_batters_2020.csv", "The Bat X")
zips_batters = read_batters("fb2020_data/zips_batters_2020.csv", "ZiPS")
```

Now, we'll concat the various hitter projections into one dataframe, called `all_batters`. Note for a couple systems, there was not a AB or H projection given (only a raw average). Either the mean AB of other systems for the same player or the total PA number is used to estimate the projection. We'll keep only players with an average draft pick under 999 and projected to receive more than 50 PA, eliminating players with little-to-no chance of being drafted, The statistic Hits Above Average (HAA) is also calculated and used in lieu of AVG, as a .300 hitter with 250 PA is far more impressive than a .300 hitter with 10 PA. Finally, the aggregate Rating stat is quite simple: just a zscore of the statistics relevant to the ESPN Fantasy scoring system (R, HR, RBI, SB, HAA). 

```
all_batters = (pd.concat([atc_batters, fgdc_batters, steamer_batters, 
                          thebat_batters, thebatx_batters, zips_batters,
                          espn_batters[['AB', 'H', 'R', 'HR', 'RBI', 'SB', 'AVG', 'System']],
                          nf_batters[['PA', 'R', 'HR', 'RBI', 'SB', 'AVG', 'System']],
                          razzball_batters[['PA', 'AB', 'H', 'R', 'HR', 'RBI', 'SB', 'AVG', 'System']]])
               # if no AB or H projection available, use mean of other projections for same player or use PA to estimate ABs
               .assign(AB = lambda df: df.AB.fillna(df.groupby('playerid')['AB'].transform('mean')).fillna(df.PA*0.92).astype(int),
                       H = lambda df: df.H.fillna(df.groupby('playerid')['H'].transform('mean')).fillna(df.AB*df.AVG).astype(int))
               .query('(ADP < 999 | ADP.isnull()) & (PA > 50 | PA.isnull())', engine = 'python')
               # use Hits Above Average value for player rating
               .assign(HAA = lambda df: df.H - df.AB * df.H.sum()/df.AB.sum(),
                       Rating = lambda df: zscore(df.R) + zscore(df.HR) + zscore(df.RBI) + 
                                           zscore(df.SB) + zscore(df.HAA, nan_policy = 'omit'))
               .sort_values("Rating", ascending = False)
               # drop Mancini, Desmond, Zimmerman, Cain, Smith (players who have opted out)
               .drop(['15149', '6885', '4220', '9077', '8048']))
```

We'll take a similar download approach for pitchers. Difference here is we're looking for new set of statistics; the ESPN standard scoring gives weight to K, W, ERA, WHIP, SV. My league happens to use SVHD as the fifth metric, so although most projections don't account for holds, we'll denote this column as SVHD regardless, and understand the projections here aren't perfect. We'll use the same transformation we performed for batters missing H or PA entries, fillin in missing SVHD values with the mean of other projection systems for that player or 0 if there are no other projections. We'll also determine the position of pitchers (starter or reliever) based on the number of games started and innings pitched. This classification will then be used to rate starters and relievers differently, with less weight given to the naturally low ERA and WHIP of relievers.

```
all_pitchers = (pd.concat([atc_pitchers, fgdc_pitchers, steamer_pitchers, 
                           thebat_pitchers, zips_pitchers,
                           espn_pitchers[["IP", "H", "ER", "BB", "SO", "W", "ERA", "WHIP", "SV", "System"]],
                           nf_pitchers[["IP", "H", "ER", "BB", "SO", "W", "ERA", "WHIP", "SV", "System"]],
                           razzball_pitchers[["GS", "G", "IP", "H", "ER", "BB", "SO", "W", "ERA", "WHIP", "SV", "System"]]])
                .rename(columns = {"SO": "K", "SV": "SVHD"})
                .query('(ADP < 999 | ADP.isnull()) & (IP > 15)', engine = 'python')
                .assign(SVHD = lambda df: df.SVHD.fillna(df.groupby('playerid')['SVHD'].transform('mean')).fillna(0),
                        Position = lambda df: np.where((df.GS.isnull() & (df.IP > 30)) | (df.GS / df.G >= 0.5), 
                                                       'SP', 'RP'),
                        Rating = lambda df: np.where(df.Position == "SP", 
                                                     zscore(df.K) + zscore(df.W) + zscore(df.SVHD) + zscore(-df.ERA) + zscore(-df.WHIP),
                                                     zscore(df.K) + zscore(df.W) + zscore(df.SVHD) + 0.5*zscore(-df.ERA) + 0.5*zscore(-df.WHIP)))
                .sort_values("Rating", ascending = False)
                # drop Sale, Syndergaard, Severino, Price, Taillon, Archer, Vazquez, Leake, Ross, McHugh, Smith
                .drop(['10603', '11762', '15890', '3184', '11674', '6345', '12076', '10130', '12972', '7531']))
```

Now, we'll add in the "bread and butter" of this projection system: the automated notes. Using the skew function from `scipy.stats`, we'll auto-assign players upside/risk grades, based on the distribution of their projections.

```
all_players = all_batters.merge(all_pitchers, how = "outer", on = ['playerid', 'ADP', 'System', 'Rating'])

# using scipy.stats.skew, compute skewness of projection systems for individual players -> classify type of player accordingly
proj_skew = all_players['Rating'].groupby('playerid').agg(skew, nan_policy = 'omit')

conditions = [
    (proj_skew < -2),
    (proj_skew < -1.25),
    (proj_skew < -0.5),
    (proj_skew < 0),
    (proj_skew == 0),
    (proj_skew > 2),
    (proj_skew > 1.25),
    (proj_skew > 0.5),
    (proj_skew > 0)
]

values = ['High Risk', 'Med Risk', 'Low Risk', 'Stable', 'Not Enough Data', 
          'High Upside', 'Med Upside', 'Low Upside', 'Stable']

notes = pd.Series(np.select(conditions, values), index = proj_skew.index)
```

Let's look at three top fantasy players for example. First, consider Cody Bellinger's stable projection. His player rating across systems, shown below, is unimodal and clustered around 10. We feel fairly comfortable concluding Bellinger will likely perform close to this mark. The blue line gives the average player rating, with the red line giving the median.

![Bellinger](https://github.com/jacobrichey/jacobrichey.github.io/blob/master/assets/img/Bellinger_distribution)

Now, let's look at Acuna (a medium upside player) and Ramirez (a low risk player). Here, we see two bimodal distributions: Acuna with some upside (one system thinks he will be an MVP level player), and Ramirez with some downside. 

![Acuna](https://github.com/jacobrichey/jacobrichey.github.io/blob/master/assets/img/Acuna_distribution)

![Ramirez](https://github.com/jacobrichey/jacobrichey.github.io/blob/master/assets/img/Ramirez_distribution)
