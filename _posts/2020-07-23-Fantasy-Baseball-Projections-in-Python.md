---
layout: post
title: Fantasy Baseball Projections in Python
subtitle: 2020 Shortened Season
gh-repo: jacobrichey/Fantasy-Baseball
gh-badge: [star]
tags: [Python, baseball]
---

Well, the 2020 MLB Season has finally arrived, roughly four months after the previously scheduled start date. In advance of Opening Day tonight, I'm going to share my fantasy baseball player rankings, and the methodology behind the system. Note these rankings are designed for a snake draft (no auction values) in an ESPN Head-to-Head Categories league with a standard scoring system. These rankings are for fantasy baseball, and do not reflect overall player value. The general approach is to ensemble nine reputable projection systems to produce a consensus player ranking. The sample skewness for player projections is used to add potential boom/bust notes for players. All links to projections can be found in Jupyter Notebook write up of this project, located in the GitHub repository linked above.

## Method

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

We'll take a similar download approach for pitchers. Difference here is we're looking for new set of statistics; the ESPN standard scoring gives weight to K, W, ERA, WHIP, SV. My league happens to use SVHD as the fifth metric, so although most projections don't account for holds, we'll denote this column as SVHD regardless, and understand the projections here aren't perfect. We'll use the same transformation we performed for batters missing H or PA entries, fill in missing SVHD values with the mean of other projection systems for that player or 0 if there are no other projections. We'll also determine the position of pitchers (starter or reliever) based on the number of games started and innings pitched. This classification will then be used to rate starters and relievers differently, with less weight given to the naturally low ERA and WHIP of relievers.

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

![Bellinger](https://github.com/jacobrichey/jacobrichey.github.io/blob/master/assets/img/Bellinger_distribution.png)

Now, let's look at Acuna (a medium upside player) and Ramirez (a low risk player). Note the graphs appear in that order. Here, we see two bimodal distributions: Acuna with some upside (one system thinks he will be an MVP level player), and Ramirez with some downside. 

![Acuna](https://github.com/jacobrichey/jacobrichey.github.io/blob/master/assets/img/Acuna_distribution.png)

![Ramirez](https://github.com/jacobrichey/jacobrichey.github.io/blob/master/assets/img/Ramirez_distribution.png)

We'll import in some stats from 2019, specifically context neutral run value and xwOBA, and add them into the final sheet. See the GitHub repository for full code and necessary data. We'll also merge in player positions from ESPN, as well as their average draft pick. Finally, the summary stats will be re-computed for accuracy. And there we have it, our final sheet! 

```
final_sheet = (all_players.groupby('playerid')
               .mean()
               .sort_values("Rating", ascending=False)
               .assign(Notes = notes)
               .merge(names, how = 'left', on = 'playerid')
               .merge(cnrv19, how = 'left', on = 'playerid')
               .merge(savant_batters, how = 'left', on = 'playerid')
               .merge(savant_pitchers, how = 'left', on = 'playerid')
               .merge(pd.concat([espn_batters[['Position']], espn_pitchers[['Position']]]), 
                                how = 'left', on = 'playerid')
               .drop_duplicates()
               .reset_index()
               .assign(Rank = lambda df: df.index + 1,
                       AVG = lambda df: round(df.H_x / df.AB, 3),
                       ERA = lambda df: round(df.ER * 9 / df.IP, 2),
                       WHIP = lambda df: round((df.BB + df.H_y) / df.IP, 2),
                       K9 = lambda df: round(df.K * 9 / df.IP, 2),
                       xwOBA19 = lambda df: np.where(df.IP.isnull() | (df.IP < 5), df.xwOBA19_x, df.xwOBA19_y))
               .round({'ADP': 1, 'Rating': 2, 'PA': 0, 'R': 0, 'HR': 0, 'RBI': 0, 'AVG': 3,
                       'SB': 0, 'K': 0, 'IP': 0, 'W': 0})
               .rename(columns = {'K9': 'K/9'})
               .fillna("")
               .loc[:, ['Rank', 'ADP', 'Name', 'Position', 'Rating', 'Notes', 'xwOBA19', 'cnrv19',  
                        'PA', 'R', 'HR', 'RBI', 'SB', 'AVG', 
                        'K/9', 'IP', 'K', 'W', 'ERA', 'WHIP', 'SVHD', 'playerid']])
```

## Rankings

The top 300 players from the projection system are given below.

| Rank | ADP   | Name                  | Position   | Rating          | Notes           |
| ---- | ----- | --------------------- | ---------- | --------------- | --------------- |
| 1    | 1.7   | Ronald Acuna Jr.      | OF         | 11.38           | Med Upside      |
| 2    | 1.6   | Christian Yelich      | OF         | 11.32           | Stable          |
| 3    | 6.1   | Gerrit Cole           | SP         | 10.24           | Med Upside      |
| 4    | 6     | Mike Trout            | OF         | 10.15           | Stable          |
| 5    | 4     | Cody Bellinger        | OF, 1B     | 9.90            | Stable          |
| 6    | 4.9   | Mookie Betts          | OF         | 9.50            | Stable          |
| 7    | 7.8   | Trea Turner           | SS         | 9.39            | Low Upside      |
| 8    | 7.3   | Francisco Lindor      | SS         | 9.24            | Stable          |
| 9    | 15.3  | Justin Verlander      | SP         | 9.05            | Med Upside      |
| 10   | 8.8   | Jacob deGrom          | SP         | 8.61            | Low Upside      |
| 11   | 13.5  | Nolan Arenado         | 3B         | 8.55            | Stable          |
| 12   | 15.1  | Max Scherzer          | SP         | 8.49            | Stable          |
| 13   | 13    | Jose Ramirez          | 3B         | 8.45            | Low Risk        |
| 14   | 11    | Juan Soto             | OF         | 8.35            | Low Upside      |
| 15   | 12    | Trevor Story          | SS         | 8.34            | Low Risk        |
| 16   | 23.2  | J.D. Martinez         | OF, DH     | 7.78            | Low Risk        |
| 17   | 23.6  | Rafael Devers         | 3B         | 7.74            | Low Risk        |
| 18   | 17.2  | Alex Bregman          | 3B, SS     | 7.40            | Stable          |
| 19   | 39.7  | Jose Altuve           | 2B         | 7.22            | Stable          |
| 20   | 19.6  | Bryce Harper          | OF         | 7.01            | Stable          |
| 21   | 27.3  | Starling Marte        | OF         | 6.75            | Stable          |
| 22   | 14.9  | Walker Buehler        | SP         | 6.72            | Low Upside      |
| 23   | 33.8  | Javier Baez           | SS         | 6.62            | Stable          |
| 24   | 25.4  | Shane Bieber          | SP         | 6.60            | Stable          |
| 25   | 32.4  | Ozzie Albies          | 2B         | 6.57            | Stable          |
| 26   | 28.3  | Stephen Strasburg     | SP         | 6.50            | Stable          |
| 27   | 17.3  | Fernando Tatis Jr.    | SS         | 6.31            | Med Risk        |
| 28   | 47.1  | George Springer       | OF         | 6.28            | Stable          |
| 29   | 48.9  | Charlie Morton        | SP         | 6.23            | Stable          |
| 30   | 22.9  | Jack Flaherty         | SP         | 6.16            | Low Upside      |
| 31   | 21.1  | Mike Clevinger        | SP         | 6.13            | Low Risk        |
| 32   | 29.3  | Gleyber Torres        | SS, 2B     | 6.13            | Low Upside      |
| 33   | 34.5  | Clayton Kershaw       | SP         | 6.12            | Stable          |
| 34   | 40.7  | Xander Bogaerts       | SS         | 6.04            | Stable          |
| 35   | 26.1  | Freddie Freeman       | 1B         | 6.02            | Low Upside      |
| 36   | 46.1  | Blake Snell           | SP         | 5.87            | Low Risk        |
| 37   | 31    | Adalberto Mondesi     | SS         | 5.86            | Stable          |
| 38   | 66    | Manny Machado         | 3B, SS     | 5.82            | Med Risk        |
| 39   | 25.6  | Anthony Rendon        | 3B         | 5.76            | Low Upside      |
| 40   | 39.2  | Ketel Marte           | OF, 2B     | 5.74            | Stable          |
| 41   | 99.9  | Marcell Ozuna         | OF         | 5.64            | Low Risk        |
| 42   | 64.9  | Charlie Blackmon      | OF         | 5.40            | Stable          |
| 43   | 60.6  | Nelson Cruz           | DH         | 5.40            | Low Risk        |
| 44   | 33.3  | Peter Alonso          | 1B         | 5.40            | Stable          |
| 45   | 75.3  | Anthony Rizzo         | 1B         | 5.31            | Stable          |
| 46   | 37.2  | Keston Hiura          | 2B         | 5.27            | Stable          |
| 47   | 47.1  | Bo Bichette           | SS         | 5.25            | Stable          |
| 48   | 67.3  | Giancarlo Stanton     | OF         | 5.20            | Low Upside      |
| 49   | 57.8  | Yordan Alvarez        | DH         | 5.16            | Stable          |
| 50   | 44.5  | Josh Hader            | RP         | 5.15            | Med Risk        |
| 51   | 63.2  | Whit Merrifield       | 2B, OF     | 5.14            | Stable          |
| 52   | 95.1  | Tim Anderson          | SS         | 5.13            | Low Risk        |
| 53   | 60.1  | Kris Bryant           | 3B, OF     | 5.09            | Low Upside      |
| 54   | 61.4  | Zack Greinke          | SP         | 5.05            | Med Upside      |
| 55   | 37.5  | Austin Meadows        | OF, DH     | 5.04            | Stable          |
| 56   | 47.3  | Lucas Giolito         | SP         | 5.02            | Low Upside      |
| 57   | 78.3  | Paul Goldschmidt      | 1B         | 4.96            | Stable          |
| 58   | 67.3  | Victor Robles         | OF         | 4.90            | Stable          |
| 59   | 47.5  | Patrick Corbin        | SP         | 4.89            | Stable          |
| 60   | 100.4 | Eddie Rosario         | OF         | 4.82            | Stable          |
| 61   | 74.8  | Jose Abreu            | 1B, DH     | 4.78            | Stable          |
| 62   | 59.9  | Eloy Jimenez          | OF         | 4.77            | Low Upside      |
| 63   | 96.3  | Marcus Semien         | SS         | 4.76            | Low Risk        |
| 64   | 71    | Yoan Moncada          | 3B         | 4.64            | Low Upside      |
| 65   | 41.6  | Luis Castillo         | SP         | 4.60            | Stable          |
| 66   | 75.9  | Eugenio Suarez        | 3B         | 4.59            | Low Upside      |
| 67   | 62.3  | Aaron Judge           | OF         | 4.58            | Low Risk        |
| 68   | 52.7  | Chris Paddack         | SP         | 4.50            | Stable          |
| 69   | 77.8  | Nicholas Castellanos  | OF         | 4.46            | Med Risk        |
| 70   | 70.1  | Tyler Glasnow         | SP         | 4.45            | Stable          |
| 71   | 40.3  | Jonathan Villar       | 2B, SS     | 4.37            | Low Upside      |
| 72   | 94.4  | Tommy Pham            | OF, DH     | 4.34            | Low Upside      |
| 73   | 81.6  | Trevor Bauer          | SP         | 4.32            | Stable          |
| 74   | 67.1  | Aaron Nola            | SP         | 4.31            | Stable          |
| 75   | 129.9 | Carlos Carrasco       | SP, RP     | 4.24            | Stable          |
| 76   | 53    | Yu Darvish            | SP         | 4.23            | Stable          |
| 77   | 98.3  | Josh Bell             | 1B         | 4.22            | Med Upside      |
| 78   | 95.9  | James Paxton          | SP         | 4.20            | Stable          |
| 79   | 96.7  | Mike Moustakas        | 3B, 2B     | 4.18            | Low Upside      |
| 80   | 60.7  | Kirby Yates           | RP         | 4.16            | Med Risk        |
| 81   | 87    | Josh Donaldson        | 3B         | 4.12            | Stable          |
| 82   | 93.1  | Jeff McNeil           | OF, 2B, 3B | 4.10            | Stable          |
| 83   | 162.6 | Jorge Polanco         | SS         | 4.05            | Low Risk        |
| 84   | 120.4 | Andrew Benintendi     | OF         | 4.01            | Stable          |
| 85   | 56.9  | Vladimir Guerrero Jr. | 3B, DH     | 4.01            | Low Risk        |
| 86   | 66.4  | Roberto Osuna         | RP         | 4.01            | Med Risk        |
| 87   | 110.7 | Lance Lynn            | SP         | 3.99            | Stable          |
| 88   | 75.5  | Liam Hendriks         | RP         | 3.92            | Med Risk        |
| 89   | 66.1  | Jose Berrios          | SP         | 3.91            | Low Upside      |
| 90   | 74    | DJ LeMahieu           | 2B, 1B, 3B | 3.71            | Stable          |
| 91   | 133   | Michael Brantley      | OF, DH     | 3.69            | Stable          |
| 92   | 137.6 | Amed Rosario          | SS         | 3.68            | Stable          |
| 93   | 104.9 | Carlos Correa         | SS         | 3.68            | Low Risk        |
| 94   | 107.7 | Edwin Diaz            | RP         | 3.65            | Med Risk        |
| 95   | 93.7  | Matt Chapman          | 3B         | 3.62            | Low Upside      |
| 96   | 45.1  | J.T. Realmuto         | C          | 3.57            | Low Risk        |
| 97   | 83.1  | Brandon Woodruff      | SP         | 3.51            | Stable          |
| 98   | 101.6 | Jorge Soler           | OF, DH     | 3.51            | Low Risk        |
| 99   | 152.5 | Justin Turner         | 3B         | 3.47            | Low Risk        |
| 100  | 121.4 | Kyle Schwarber        | OF         | 3.45            | Low Upside      |
| 101  | 160.1 | Yuli Gurriel          | 1B, 3B     | 3.42            | Stable          |
| 102  | 76.7  | Aroldis Chapman       | RP         | 3.40            | Med Risk        |
| 103  | 95.5  | Corey Kluber          | SP         | 3.38            | Low Upside      |
| 104  | 49.2  | Matt Olson            | 1B         | 3.32            | Low Risk        |
| 105  | 84    | Kenley Jansen         | RP         | 3.28            | Low Risk        |
| 106  | 123.2 | Rhys Hoskins          | 1B         | 3.26            | Stable          |
| 107  | 75.6  | Max Muncy             | 2B, 1B, 3B | 3.24            | Stable          |
| 108  | 94.8  | Sonny Gray            | SP         | 3.24            | Stable          |
| 109  | 142.1 | Carlos Santana        | 1B, DH     | 3.24            | Stable          |
| 110  | 128   | Michael Conforto      | OF         | 3.23            | Stable          |
| 111  | 146.1 | Max Kepler            | OF         | 3.21            | Low Risk        |
| 112  | 205.9 | Ryan Braun            | OF         | 3.20            | Stable          |
| 113  | 83.1  | Ramon Laureano        | OF         | 3.19            | Low Upside      |
| 114  | 148.8 | Elvis Andrus          | SS         | 3.18            | Stable          |
| 115  | 83.8  | Taylor Rogers         | RP         | 3.17            | Med Risk        |
| 116  | 117.3 | Oscar Mercado         | OF         | 3.15            | Low Upside      |
| 117  | 190.1 | Paul DeJong           | SS         | 3.14            | Med Risk        |
| 118  | 106.4 | Franmil Reyes         | OF, DH     | 3.12            | Stable          |
| 119  | 203.1 | Jean Segura           | SS         | 3.09            | Med Upside      |
| 120  | 92.5  | Frankie Montas        | SP         | 3.08            | Med Upside      |
| 121  | 117.9 | Mike Soroka           | SP         | 3.08            | Stable          |
| 122  | 94.2  | Brad Hand             | RP         | 3.06            | Med Risk        |
| 123  | 151.3 | Matthew Boyd          | SP         | 3.01            | Low Risk        |
| 124  | 122.9 | Nick Anderson         | RP         | 2.98            | Low Risk        |
| 125  | 149.6 | Hyun-Jin Ryu          | SP         | 2.95            | Stable          |
| 126  | 202.4 | Adam Eaton            | OF         | 2.93            | Stable          |
| 127  | 101.2 | Ken Giles             | RP         | 2.92            | Low Risk        |
| 128  | 140.3 | Max Fried             | SP         | 2.91            | Low Risk        |
| 129  | 143.7 | David Dahl            | OF         | 2.91            | Stable          |
| 130  | 130.5 | Eduardo Escobar       | 3B, 2B     | 2.88            | Stable          |
| 131  | 130.8 | Shohei Ohtani         | DH, SP     | 2.86            | Low Risk        |
| 132  | 139.7 | Corey Seager          | SS         | 2.85            | Low Upside      |
| 133  | 212   | Andrew McCutchen      | OF         | 2.84            | Stable          |
| 134  | 153.6 | Byron Buxton          | OF         | 2.81            | Low Risk        |
| 135  | 198.9 | Avisail Garcia        | OF, DH     | 2.77            | Low Upside      |
| 136  | 95.8  | Joey Gallo            | OF         | 2.76            | Stable          |
| 137  | 171.9 | Mike Minor            | SP         | 2.76            | Med Upside      |
| 138  | 195.7 | Andrew Heaney         | SP         | 2.76            | Stable          |
| 139  | 163.3 | Lance McCullers Jr.   | SP         | 2.70            | Med Risk        |
| 140  | 169.9 | Eduardo Rodriguez     | SP         | 2.66            | Low Upside      |
| 141  | 150.7 | Kyle Hendricks        | SP         | 2.59            | Stable          |
| 142  | 97.7  | Jesus Luzardo         | RP         | 2.55            | Stable          |
| 143  | 159.6 | Zack Wheeler          | SP         | 2.55            | Stable          |
| 144  | 249.8 | Starlin Castro        | 2B, 3B     | 2.53            | Low Upside      |
| 145  | 114.6 | Raisel Iglesias       | RP         | 2.52            | Med Risk        |
| 146  | 197.6 | German Marquez        | SP         | 2.49            | Stable          |
| 147  | 174.6 | Jake Odorizzi         | SP         | 2.49            | Low Upside      |
| 148  | 139.8 | Danny Santana         | OF, 1B     | 2.45            | Low Upside      |
| 149  | 132.4 | Brandon Workman       | RP         | 2.45            | Med Risk        |
| 150  | 146.1 | Lourdes Gurriel Jr.   | OF         | 2.45            | Low Risk        |
| 151  | 126.1 | Zac Gallen            | SP         | 2.43            | Low Upside      |
| 152  | 145.9 | Kenta Maeda           | SP, RP     | 2.37            | Stable          |
| 153  | 190.2 | Bryan Reynolds        | OF         | 2.29            | Low Risk        |
| 154  | 165.2 | Robbie Ray            | SP         | 2.29            | Stable          |
| 155  | 129.6 | Dinelson Lamet        | SP         | 2.27            | Stable          |
| 156  | 200.4 | Alex Verdugo          | OF         | 2.26            | Stable          |
| 157  | 257.8 | Howie Kendrick        | 1B, 2B     | 2.26            | Low Risk        |
| 158  | 235.3 | Didi Gregorius        | SS         | 2.14            | Stable          |
| 159  | 125.3 | Julio Urias           | RP, SP     | 2.11            | Med Risk        |
| 160  | 172.7 | Gavin Lux             | 2B         | 2.10            | Stable          |
| 161  | 169.5 | Khris Davis           | DH         | 2.09            | Stable          |
| 162  | 163   | Edwin Encarnacion     | 1B, DH     | 2.05            | Stable          |
| 163  | 213.4 | Dylan Bundy           | SP         | 2.02            | Stable          |
| 164  | 232.1 | Masahiro Tanaka       | SP         | 2.00            | Stable          |
| 165  | 121.8 | Miguel Sano           | 3B         | 1.98            | Stable          |
| 166  | 140.7 | Madison Bumgarner     | SP         | 1.94            | Low Risk        |
| 167  | 146.6 | Jose Leclerc          | RP         | 1.93            | Med Risk        |
| 168  | 251.3 | Luis Arraez           | 2B, OF     | 1.92            | Stable          |
| 169  | 211.6 | Joe Musgrove          | SP         | 1.91            | Low Risk        |
| 170  | 283.9 | Joey Votto            | 1B         | 1.89            | Low Upside      |
| 171  | 173.8 | Mallex Smith          | OF         | 1.88            | Stable          |
| 172  | 265.4 | Cesar Hernandez       | 2B         | 1.84            | Low Upside      |
| 173  | 121.9 | Hector Neris          | RP         | 1.83            | Low Risk        |
| 174  | 203.7 | C.J. Cron             | 1B         | 1.79            | Stable          |
| 175  | 185.6 | Rich Hill             | SP         | 1.78            | Stable          |
| 176  | 208.6 | Marcus Stroman        | SP         | 1.77            | Low Upside      |
| 177  | 244.8 | Daniel Murphy         | 1B         | 1.77            | Low Risk        |
| 178  | 224.3 | Kolten Wong           | 2B         | 1.72            | Stable          |
| 179  | 214.5 | A.J. Puk              | RP         | 1.70            | Low Risk        |
| 180  | 228.4 | Rougned Odor          | 2B         | 1.69            | Low Risk        |
| 181  | 196.7 | Nick Senzel           | OF         | 1.68            | Stable          |
| 182  | 148.8 | Archie Bradley        | RP         | 1.67            | Med Risk        |
| 183  | 128   | Craig Kimbrel         | RP         | 1.67            | Low Risk        |
| 184  | 183.1 | Sean Manaea           | SP         | 1.65            | Low Upside      |
| 185  | 177.8 | Keone Kela            | RP         | 1.65            | Med Risk        |
| 186  | 175.6 | Joe Jimenez           | RP         | 1.62            | Low Risk        |
| 187  | 183.9 | Kyle Tucker           | OF         | 1.62            | High Upside     |
| 188  | 130.4 | Tommy Edman           | 3B, 2B     | 1.60            | Stable          |
| 189  | 195.3 | Kevin Newman          | SS, 2B     | 1.58            | Stable          |
| 190  | 132   | Alex Colome           | RP         | 1.56            | Med Risk        |
| 191  | 206.1 | Justin Upton          | OF         | 1.53            | Stable          |
| 192  | 205.1 | Brandon Lowe          | 2B         | 1.50            | Low Risk        |
| 193  | 224.1 | Jose Urquidy          | SP         | 1.45            | Stable          |
| 194  | 378.6 | Jonathan Schoop       | 2B         | 1.43            | Stable          |
| 195  | 178.6 | Sean Doolittle        | RP         | 1.42            | Low Risk        |
| 196  | 203.5 | Giovanny Gallegos     | RP         | 1.40            | Low Risk        |
| 197  | 132.6 | Hansel Robles         | RP         | 1.34            | Med Risk        |
| 198  | 161.6 | J.D. Davis            | OF, 3B     | 1.32            | Stable          |
| 199  | 282.7 | Yonny Chirinos        | SP, RP     | 1.32            | Low Upside      |
| 200  | 255.3 | David Peralta         | OF         | 1.29            | Low Upside      |
| 201  | 230.4 | Ryan Yarbrough        | RP, SP     | 1.27            | Stable          |
| 202  | 270.3 | Jon Gray              | SP         | 1.22            | Low Risk        |
| 203  | 196   | Christian Walker      | 1B         | 1.22            | Stable          |
| 204  | 256   | Eric Hosmer           | 1B         | 1.20            | Stable          |
| 205  | 248.8 | Dansby Swanson        | SS         | 1.20            | Low Upside      |
| 206  | 266.1 | Shogo Akiyama         | OF         | 1.19            | Low Upside      |
| 207  | 601   | Ian Miller            | OF         | 1.17            | Not Enough Data |
| 208  | 171.1 | Ryan McMahon          | 2B, 3B     | 1.17            | Low Risk        |
| 209  | 87.9  | Gary Sanchez          | C          | 1.16            | Low Upside      |
| 210  | 214.8 | Joc Pederson          | OF, 1B     | 1.14            | Low Risk        |
| 211  | 236.2 | Joshua James          | RP         | 1.11            | Low Risk        |
| 212  | 170.7 | Scott Kingery         | OF, 3B     | 1.11            | Stable          |
| 213  | 163.5 | Carlos Martinez       | RP         | 1.09            | Stable          |
| 214  | 275.9 | Randal Grichuk        | OF         | 1.09            | Low Risk        |
| 215  | 285.1 | Dallas Keuchel        | SP         | 1.09            | Stable          |
| 216  | 247.8 | Caleb Smith           | SP         | 1.09            | Stable          |
| 217  | 528.8 | Marco Gonzales        | SP         | 1.08            | Stable          |
| 218  | 175.1 | Will Smith            | RP         | 1.06            | Low Risk        |
| 219  | 291.8 | Emilio Pagan          | RP         | 1.02            | Stable          |
| 220  | 191   | Hunter Dozier         | 3B, OF     | 1.00            | Stable          |
| 221  | 325.7 | Ryan Pressly          | RP         | 0.99            | Stable          |
| 222  | 201   | Luke Weaver           | SP         | 0.99            | Low Upside      |
| 223  | 227.3 | Brian Anderson        | 3B, OF     | 0.98            | Stable          |
| 224  | 223   | Ross Stripling        | RP, SP     | 0.96            | Low Upside      |
| 225  | 272.3 | Miles Mikolas         | SP         | 0.95            | Low Upside      |
| 226  | 125   | Cavan Biggio          | 2B         | 0.90            | Stable          |
| 227  | 257   | Joey Lucchesi         | SP         | 0.83            | Stable          |
| 228  | 299.2 | A.J. Pollock          | OF         | 0.83            | Stable          |
| 229  | 339.5 | Diego Castillo        | RP, SP     | 0.81            | Stable          |
| 230  | 228.2 | Shin-Soo Choo         | OF, DH     | 0.80            | Low Risk        |
| 231  | 211.8 | Mike Foltynewicz      | SP         | 0.80            | Stable          |
| 232  | 256.3 | Nomar Mazara          | OF         | 0.79            | Low Risk        |
| 233  | 237.8 | Wil Myers             | OF         | 0.78            | Stable          |
| 234  | 180   | Ian Kennedy           | RP         | 0.77            | High Risk       |
| 235  | 243.6 | Giovanny Urshela      | 3B         | 0.75            | Stable          |
| 236  | 428.4 | Willy Adames          | SS         | 0.75            | Low Risk        |
| 237  | 518.1 | Andrelton Simmons     | SS         | 0.75            | Low Risk        |
| 238  | 246.6 | Dustin May            | RP         | 0.75            | Low Upside      |
| 239  | 476.2 | Kwang-hyun Kim        | RP, SP     | 0.75            | Stable          |
| 240  | 292.2 | Renato Nunez          | 1B, DH     | 0.74            | Stable          |
| 241  | 453.1 | Rick Porcello         | SP         | 0.73            | Stable          |
| 242  | 241.8 | Alex Wood             | SP         | 0.72            | Low Upside      |
| 243  | 306   | Corey Dickerson       | OF         | 0.68            | Stable          |
| 244  | 600.8 | Adolis Garcia         | OF         | 0.66            | Not Enough Data |
| 245  | 291.8 | Gregory Polanco       | OF         | 0.65            | Stable          |
| 246  | 185.8 | Willie Calhoun        | OF         | 0.65            | Low Risk        |
| 247  | 395   | Kevin Gausman         | SP, RP     | 0.64            | Low Upside      |
| 248  | 558.9 | Adam Frazier          | 2B         | 0.64            | Stable          |
| 249  | 179.5 | Mark Melancon         | RP         | 0.63            | Low Risk        |
| 250  | 276.9 | Hunter Renfroe        | OF         | 0.61            | Stable          |
| 251  | 519.5 | Matt Barnes           | RP         | 0.58            | Stable          |
| 252  | 588.8 | Seth Beer             | 1B         | 0.55            | Not Enough Data |
| 253  | 533.4 | Hanser Alberto        | 2B, 3B     | 0.53            | Stable          |
| 254  | 238.3 | Mark Canha            | OF         | 0.52            | Low Risk        |
| 255  | 319.7 | Tony Watson           | RP         | 0.52            | Low Risk        |
| 256  | 276.8 | Austin Hays           | OF         | 0.52            | Low Risk        |
| 257  | 599.7 | Abraham Toro          | 3B, DH     | 0.51            | Not Enough Data |
| 258  | 110.3 | Yasmani Grandal       | C, 1B      | 0.51            | Stable          |
| 259  | 575.9 | Cedric Mullins II     | OF         | 0.51            | Not Enough Data |
| 260  | 288.3 | Nick Solak            | DH         | 0.50            | Low Upside      |
| 261  | 230.2 | Adrian Houser         | SP, RP     | 0.50            | Low Risk        |
| 262  | 116.2 | Willson Contreras     | C          | 0.49            | Stable          |
| 263  | 280.2 | Miguel Andujar        | DH         | 0.44            | Low Risk        |
| 264  | 264.8 | Seth Lugo             | RP         | 0.43            | Low Risk        |
| 265  | 472.7 | Anthony Santander     | OF         | 0.40            | Low Upside      |
| 266  | 403.4 | Dellin Betances       | RP         | 0.39            | Stable          |
| 267  | 555.3 | Nick Ahmed            | SS         | 0.39            | Stable          |
| 268  | 600.7 | Odubel Herrera        | OF         | 0.38            | Not Enough Data |
| 269  | 325.4 | J.A. Happ             | SP         | 0.38            | Low Risk        |
| 270  | 317.1 | Mychal Givens         | RP         | 0.38            | Stable          |
| 271  | 599   | Colin Poche           | RP         | 0.33            | Stable          |
| 272  | 566.9 | Chad Green            | RP, SP     | 0.33            | Stable          |
| 273  | 512.7 | Pablo Lopez           | SP         | 0.33            | Low Risk        |
| 274  | 277   | Austin Riley          | OF         | 0.32            | Med Risk        |
| 275  | 239.3 | Aaron Civale          | SP         | 0.32            | Low Upside      |
| 276  | 237.4 | Mitch Keller          | SP         | 0.29            | Low Risk        |
| 277  | 353.8 | Teoscar Hernandez     | OF         | 0.29            | Stable          |
| 278  | 575.4 | Jackie Bradley Jr.    | OF         | 0.28            | Stable          |
| 279  | 457.1 | Chris Bassitt         | SP         | 0.28            | Stable          |
| 280  | 249.9 | Anthony DeSclafani    | SP         | 0.26            | Low Risk        |
| 281  | 449   | Kyle Seager           | 3B         | 0.26            | Stable          |
| 282  | 407.8 | Eric Thames           | 1B         | 0.25            | Stable          |
| 283  | 589   | Jakob Junis           | SP         | 0.23            | Low Risk        |
| 284  | 202.1 | Yasiel Puig           | OF         | 0.23            | Low Upside      |
| 285  | 327.3 | Griffin Canning       | SP         | 0.21            | Low Upside      |
| 286  | 347.9 | Josh Lindblom         | SP         | 0.20            | Low Risk        |
| 287  | 524   | Jurickson Profar      | 2B         | 0.20            | Stable          |
| 288  | 328.9 | Trent Grisham         | OF         | 0.15            | Stable          |
| 289  | 566   | Jason Heyward         | OF         | 0.15            | Stable          |
| 290  | 600.2 | Jeter Downs           | SS         | 0.14            | Not Enough Data |
| 291  | 281.8 | Niko Goodrum          | SS, 2B, OF | 0.12            | Low Risk        |
| 292  | 445.2 | Kole Calhoun          | OF         | 0.12            | Low Upside      |
| 293  | 540   | Austin Adams          | RP         | 0.11            | Stable          |
| 294  | 256.5 | Ian Happ              | OF         | 0.11            | Stable          |
| 295  | 187.3 | Luke Voit             | 1B, DH     | 0.10            | Stable          |
| 296  | 600.3 | Adam Jones            | OF         | 0.09            | Not Enough Data |
| 297  | 570.7 | Ty Buttrey            | RP         | 0.06            | Stable          |
| 298  | 396.5 | Brendan McKay         | SP         | 0.05            | Med Risk        |
| 299  | 320.9 | Cole Hamels           | SP         | 0.03            | Low Risk        |
| 300  | 502.7 | Kevin Kiermaier       | OF         | 0.03            | Low Upside      |

