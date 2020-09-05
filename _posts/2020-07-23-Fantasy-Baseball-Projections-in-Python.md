---
layout: post
title: Fantasy Baseball Projections in Python
subtitle: 2020 Shortened Season
gh-repo: jacobrichey/Fantasy-Baseball
gh-badge: [star]
tags: [Python, baseball]
---

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


