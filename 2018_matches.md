2018 FRC Match Analysis
================
Greg Marra

A notebook to get FRC Match data from The Blue Alliance API.

Distribution Of Match Scores
============================

The 2018 FRC game is somewhat unique, in that there is a maximum number of ownership points that can exist in a match, and teams need to tip the accumulation of those points towards their alliance. Early scoring is worth much more in that it can accumulate longer, while a box scored in a Switch or Scale during the last second of the match can only be worth a single point.

Here we look at the results from 7106 matches played so far this year.

``` r
matches %>%
  gather(alliances.red.score, alliances.blue.score, key = "alliance", value = "score") %>%
  filter(score >= 0) %>%
  ggplot(aes(parse_factor(comp_level, levels = comp_levels, ordered = TRUE), score)) +
  geom_boxplot() +
  labs(
    title = "Score distribution across 2018 FRC matches",
    subtitle = "Scores are higher in elims, andwith a smaller range in finals",
    x = "Competition Level",
    y = "Score"
  )
```

![](2018_matches_files/figure-markdown_github/match_score_distribution-1.png)

``` r
matches %>%
  filter(alliances.red.score >= 0) %>%
  filter(alliances.blue.score >= 0) %>%
  mutate(
    win_margin = abs(alliances.red.score - alliances.blue.score)
  ) %>%
  ggplot(aes(parse_factor(comp_level, levels = comp_levels, ordered = TRUE), win_margin)) +
  geom_boxplot() +
  labs(
    title = "Win margins across 2018 FRC matches",
    subtitle = "Matches are closer in finals, as expected",
    x = "Competition Level",
    y = "Win Margin"
  )
```

![](2018_matches_files/figure-markdown_github/win_margins-1.png)

Ownership Deltas
----------------

How big are the swings between ownership?

``` r
matches_ownership <- matches %>%
  filter(alliances.red.score >= 0) %>%
  filter(alliances.blue.score >= 0) %>%
  filter(winning_alliance != "") %>%
  mutate(
    red_switch_ownership_pts = score_breakdown.red.autoSwitchOwnershipSec * 2 + score_breakdown.red.teleopSwitchOwnershipSec,
    blue_switch_ownership_pts = score_breakdown.blue.autoSwitchOwnershipSec * 2 + score_breakdown.blue.teleopSwitchOwnershipSec,
    red_scale_ownership_pts = score_breakdown.red.autoScaleOwnershipSec * 2 + score_breakdown.red.teleopScaleOwnershipSec,
    blue_scale_ownership_pts = score_breakdown.blue.autoScaleOwnershipSec * 2 + score_breakdown.blue.teleopScaleOwnershipSec,
    switch_ownership_delta = red_switch_ownership_pts - blue_switch_ownership_pts,
    scale_ownership_delta = red_scale_ownership_pts - blue_scale_ownership_pts,
    endgame_delta = score_breakdown.red.endgamePoints - score_breakdown.blue.endgamePoints,
    vault_delta = score_breakdown.red.vaultPoints - score_breakdown.blue.vaultPoints,
    foul_delta = score_breakdown.red.foulPoints - score_breakdown.blue.foulPoints,
    auto_delta = score_breakdown.red.autoPoints - score_breakdown.blue.autoPoints,
    score_delta = alliances.red.score - alliances.blue.score)

matches_ownership %>%
  ggplot(aes(switch_ownership_delta, 
             scale_ownership_delta, 
             color = winning_alliance,
             shape = parse_factor(comp_level, levels = comp_levels, ordered = TRUE)),
         alpha = 0.8) +
  geom_point() + 
  geom_abline(slope = -1) +
  labs(
    title = "Ownership Points",
    x = "Switch Ownership Delta (Red - Blue)",
    y = "Scale Ownership Detla (Red - Blue)",
    color = "Winner",
    shape = "Comp Level"
  ) + 
  scale_colour_manual(values = c(red = "red", blue = "blue"))
```

![](2018_matches_files/figure-markdown_github/ownership_deltas-1.png)

``` r
matches_ownership %>%
  gather(switch_ownership_delta, scale_ownership_delta, key = "ownership_object", value = "ownership_delta") %>%
  mutate(ownership_object = factor(ownership_object)) %>%
  mutate(ownership_object = fct_recode(ownership_object,
    "Switch" = "switch_ownership_delta",
    "Scale"  = "scale_ownership_delta"
  )) %>%
  ggplot(aes(abs(ownership_delta), color = ownership_object)) +
  geom_freqpoly(binwidth = 6) +
  labs(
    title = "Ownership Points",
    x = "Ownership Delta",
    y = "Match Count",
    color = "Scoring Object"
  )
```

![](2018_matches_files/figure-markdown_github/ownership_deltas-2.png)

Lose the Scale Win the Match
----------------------------

``` r
matches_lose_scale_win_match <- matches_ownership %>%
  filter(((scale_ownership_delta < 0) & (winning_alliance == "red")) |
        ((scale_ownership_delta > 0) & (winning_alliance == "blue")))
```

What explains the 11.3284548% of matches where the team loses the Scale but still wins the Match (805 out of 7106 matches)?

We can look at the "scale deficit" a team had to overcome, and see what percentage of the scale deficit was overcome by deltas in other scoring objectives. We can normalize these deltas by the size of the Scale deficit that had to be overcome vs how large these deltas were. For example, if the Red Alliance Endgame scored more 60 points, and the Scale deficit they had to overcome was 45 points, their Endgame overcame 133% of the deficit.

Looking at the summary statistics for matches where Red lost the Scale but won the match, we can see that the median of the percent of the deficit overcome for the Endgame is about 67%, and the median for Switch ownership is about 122%, while the vault points are only 25%. It's hard to perfectly evaluate the vault points, because they also contribute to the powerups. I haven't totally vetted that I'm looking at the score breakdowns correctly here, so I'm not sure if this is all fully capturing everything. However, this suggests that if you're going to lose the Scale, you better win the Switch and nail your Endgame!

``` r
## Thought to explore here:
## In matches where the alliance loses the scale but manages to win, what fraction of the score_delta is explained by climb_delta, vault_delta, etc? What mechanic are teams using to lose the scale but win the match?

matches_lose_scale_win_match %>%
  ggplot(aes(scale_ownership_delta, score_delta, color = winning_alliance)) +
  geom_point() +
  scale_colour_manual(values = c(red = "red", blue = "blue"))
```

![](2018_matches_files/figure-markdown_github/lose_the_scale_win_the_match-1.png)

``` r
matches_ownership %>%
  filter(((scale_ownership_delta < 0) & (winning_alliance == "red")) |
         ((scale_ownership_delta > 0) & (winning_alliance == "blue"))) %>%
  filter(winning_alliance == "red") %>%
  filter(foul_delta < score_delta) %>% # remove matches won on fouls
  mutate(
    switch_ownership_delta_pct_margin = switch_ownership_delta / -scale_ownership_delta,
    endgame_delta_pct_margin = endgame_delta / -scale_ownership_delta,
    vault_delta_pct_margin = vault_delta / -scale_ownership_delta,
    foul_delta_pct_margin = foul_delta / -scale_ownership_delta,
    auto_delta_pct_margin = auto_delta / -scale_ownership_delta
  ) %>%
  select(contains("delta")) %>%
  skim()
```

    ## Skim summary statistics
    ##  n obs: 316 
    ##  n variables: 12 
    ## 
    ## Variable type: integer 
    ##       variable missing complete   n  mean    sd  p0    p25 median   p75
    ##     auto_delta       0      316 316  6.91 11.76 -42   0         6 15   
    ##  endgame_delta       0      316 316 22.22 25.03 -60   3.75     25 35   
    ##     foul_delta       0      316 316  5.44 33.38 -80 -10         0 20   
    ##    score_delta       0      316 316 68.28 58.11   1  26        52 95.25
    ##    vault_delta       0      316 316 13.88 15.29 -30   5        15 25   
    ##  p100     hist
    ##    40 ▁▁▁▃▇▆▃▁
    ##    85 ▁▁▁▇▇▂▅▁
    ##   205 ▁▃▇▂▁▁▁▁
    ##   319 ▇▅▃▂▁▁▁▁
    ##    45 ▁▁▃▆▇▇▃▂
    ## 
    ## Variable type: numeric 
    ##                           variable missing complete   n    mean    sd   p0
    ##              auto_delta_pct_margin       0      316 316   0.042  2.21  -25
    ##           endgame_delta_pct_margin       0      316 316   1.45   7.04  -50
    ##              foul_delta_pct_margin       0      316 316   0.71   5.1   -30
    ##              scale_ownership_delta       0      316 316 -51.41  37    -148
    ##             switch_ownership_delta       0      316 316  71.47  49.49  -37
    ##  switch_ownership_delta_pct_margin       0      316 316   4.51  15.8   -37
    ##             vault_delta_pct_margin       0      316 316   0.62   2.45  -10
    ##      p25 median    p75 p100     hist
    ##    0       0.12   0.38    5 ▁▁▁▁▁▁▇▁
    ##    0.028   0.38   1      75 ▁▁▁▇▁▁▁▁
    ##   -0.15    0      0.46   55 ▁▁▇▁▁▁▁▁
    ##  -82.25  -44    -19      -1 ▁▂▃▃▃▅▆▇
    ##   28      76.5  111     158 ▁▅▆▅▅▇▆▅
    ##    0.67    1.25   2.62  132 ▁▇▁▁▁▁▁▁
    ##    0.053   0.27   0.6    25 ▁▁▇▁▁▁▁▁
