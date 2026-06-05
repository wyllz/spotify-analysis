# The Impact Of Explicitness On A Song's Popularity

**By Samuel Yu and William Zhang** · DSC 80 Final Project, UC San Diego

Some songs rack up hundreds of millions of streams while others, often just as polished, go nowhere. Is a track's popularity written into the music itself — its energy, its danceability, its mood — or does it come down to who released it? This project digs into a catalog of 114,000 Spotify tracks to find out, and ends by building a model that predicts how popular a track will be.

## Introduction

The dataset contains audio and metadata features for **114,000 tracks** spanning 114 genres, collected from the Spotify Web API, paired with a second table of artist information (follower counts and artist-level popularity). Each row is one track. After cleaning and removing tracks that appear under more than one genre label, we work with **89,740 unique tracks**.

The single question driving the project is: **what makes a track popular on Spotify, and can we predict a track's popularity from information available when it is released?** This matters to anyone who works with music — artists deciding what to release, labels deciding what to promote, and curators deciding what to feature — because understanding the drivers of popularity is the first step to anticipating it.

The columns most relevant to that question are:

- **`popularity`** (renamed `track_popularity`): Spotify's 0–100 popularity score for the track — our response variable.
- **`explicit`**: whether the track contains explicit content.
- **`danceability`, `energy`, `valence`, `acousticness`, `instrumentalness`, `speechiness`, `liveness`, `loudness`, `tempo`**: audio features Spotify computes from the recording (most on a 0–1 scale).
- **`duration_ms`** (converted to `duration_min`): the track's length.
- **`release_date`** (parsed to `release_year`): when the track came out.
- **`track_genre`**: the genre Spotify assigns the track.
- **`followers`** and **`popularity`** from the artist table (renamed `artist_popularity`): the releasing artist's follower count and overall popularity.

## Data Cleaning and Exploratory Data Analysis

### Data Cleaning

Our cleaning steps followed the way the data is generated:

- **Dropped the unused index column** and removed the handful of rows missing a track id, artist, album, or track name (one row).
- **Renamed `popularity` to `track_popularity`** to distinguish it from the artist-level popularity in the other table.
- **Derived new columns**: `duration_min` from `duration_ms`, and `release_year` from the front of `release_date` (whose format varies between `YYYY`, `YYYY-MM`, and `YYYY-MM-DD`).
- **Imputed missing `tempo`** (about 19% of tracks) with the median tempo of the track's genre — see the Assessment of Missingness section for why this is reasonable.
- **Parsed the artist genre tags** (stored as strings like `"['indie rock', 'alternative']"`) into real lists, and **joined the artist table** onto the tracks on artist name, keeping the primary artist for each track.
- **Kept one row per track.** Because a track can be labeled under several genres, we deduplicated on track id (keeping the first-listed genre) so that no track is counted twice.

The head of the cleaned data (a few relevant columns):

| track_name                 | track_genre   |   track_popularity | explicit   |   danceability |   energy |   artist_popularity |   followers |
|:---------------------------|:--------------|-------------------:|:-----------|---------------:|---------:|--------------------:|------------:|
| Comedy                     | acoustic      |                 73 | False      |          0.676 |   0.461  |                  66 |      852637 |
| Ghost - Acoustic           | acoustic      |                 55 | False      |          0.42  |   0.166  |                  53 |       11874 |
| To Begin Again             | acoustic      |                 57 | False      |          0.438 |   0.359  |                  68 |      722496 |
| Can't Help Falling In Love | acoustic      |                 71 | False      |          0.266 |   0.0596 |                  71 |      438860 |
| Hold On                    | acoustic      |                 82 | False      |          0.618 |   0.443  |                  70 |       99345 |

### Univariate Analysis

<iframe src="assets/popularity_distribution.html" width="100%" height="500" frameborder="0"></iframe>

Track popularity is right-skewed: a large share of tracks score low (many at or near zero), and relatively few reach the upper end of the 0–100 scale. Predicting popularity is therefore hard — most tracks are clustered near the bottom, and the popular ones are the exception.

### Bivariate Analysis

<iframe src="assets/explicit_vs_popularity.html" width="100%" height="500" frameborder="0"></iframe>

Explicit tracks tend to sit a little higher than non-explicit tracks: their median popularity and upper quartile are both shifted up. The gap is modest but visible, which is what motivates the formal hypothesis test below.

### Interesting Aggregates

Grouping by genre shows just how much popularity depends on it. The ten most popular genres average well above the catalog as a whole, and several popular genres (like "sad" and "emo") are heavily explicit:

| track_genre   |   n_tracks |   avg_popularity |   pct_explicit |
|:--------------|-----------:|-----------------:|---------------:|
| pop-film      |       1000 |            59.28 |           0    |
| k-pop         |        999 |            56.95 |           0.05 |
| chill         |       1000 |            53.65 |           0.17 |
| sad           |       1000 |            52.38 |           0.45 |
| grunge        |       1000 |            49.59 |           0.07 |
| indian        |       1000 |            49.54 |           0.02 |
| anime         |       1000 |            48.77 |           0.06 |
| emo           |       1000 |            48.13 |           0.46 |

This is a useful warning for later: genre is tangled up with both popularity and explicitness, so genre is a likely confounder when we compare explicit and non-explicit tracks.

## Assessment of Missingness

### NMAR Analysis

The only column with substantial missingness is **`tempo`** (about 19% of tracks). We do **not** believe it is NMAR. Tempo is Spotify's algorithmic estimate of a track's beats-per-minute, and it comes back empty when the beat-detection algorithm cannot lock onto a steady pulse — which happens for music without a clear, regular beat (ambient, classical, spoken-word, and other free-form recordings). That property is reflected in columns we can observe: such tracks tend to have low energy, high acousticness, and belong to particular genres. Because the missingness is explained by other observed columns, it is **MAR**, not NMAR. The one extra piece of data that would model the mechanism directly is Spotify's internal beat-detection confidence score; with it, the missingness would be fully explained.

### Missingness Dependency

We ran permutation tests to see which columns the missingness of `tempo` depends on, using the absolute difference in means for quantitative columns and the total variation distance for categorical ones.

<iframe src="assets/energy_by_missingness.html" width="100%" height="500" frameborder="0"></iframe>

The missingness of `tempo` **depends on `energy`** (p ≈ 0.000): tracks with missing tempo average about 0.51 energy versus 0.67 for tracks with tempo present, and the observed difference falls far outside the entire permutation null distribution (shown above). It also depends strongly on `track_genre` (p ≈ 0.000). By contrast, it does **not** depend on `track_popularity` (p ≈ 0.71) — whether a track is popular tells us nothing about whether its tempo could be measured. This pattern — missingness driven by observed audio characteristics — is the signature of MAR.

## Hypothesis Testing

We tested whether explicit and non-explicit tracks differ in popularity.

- **Null hypothesis:** explicit and non-explicit tracks have the same mean popularity; any difference is due to chance.
- **Alternative hypothesis:** explicit and non-explicit tracks have different mean popularities.
- **Test statistic:** the absolute difference in mean popularity between the two groups (two-sided, since the alternative allows a difference in either direction).
- **Significance level:** 0.05.

A permutation test fits here because we are comparing a numerical outcome across two groups without assuming any particular distribution. Explicit tracks averaged a popularity of **36.9** versus **32.9** for non-explicit tracks — an observed difference of **4.0 points**. Across 10,000 shuffles, the difference never approached the observed value, giving a **p-value of ≈ 0.000**.

<iframe src="assets/hypothesis_null.html" width="100%" height="500" frameborder="0"></iframe>

We **reject the null hypothesis**: the data are highly inconsistent with explicit and non-explicit tracks being equally popular. Since this is observational, it is evidence of an *association*, not proof of causation — as the genre aggregate hinted, genres like hip-hop are both frequently explicit and popular, so genre is a plausible confounder.

## Framing a Prediction Problem

We predict a track's **popularity score** (`track_popularity`, 0–100) from its audio features, metadata, and artist information. This is a **regression** problem, continuing the popularity theme above.

We chose `track_popularity` as the response because it is the quantity the whole project investigates, and because Spotify reports it on a continuous 0–100 scale, so regression is natural (binarizing it would throw away information). We evaluate with **RMSE** (root mean squared error), reported alongside **R²**. RMSE is in the same units as the response — popularity points — so it is directly interpretable, and it penalizes large misses, which we care about. We prefer it to MAE for that reason, and to a classification metric like accuracy because the target is continuous; R² communicates how much of the variation in popularity the model explains.

Following the assignment's guidance to pick at least five musically distinct genres, we focus the model on eight: **pop, rock, hip-hop, classical, jazz, country, edm, and metal**. They are recognizable and span very different musical and popularity profiles, which makes the model and the later fairness check meaningful (4,863 tracks, split 75/25 into training and test sets).

Every feature we use is known at the **time of prediction** — the moment a track is released. That includes its audio features, genre, explicit flag, duration, release year, and the releasing artist's follower count and artist-level popularity (which reflect the artist's standing from earlier work). We do not use anything that would only be known after release, since the track's own future popularity is exactly what we are predicting.

## Baseline Model

Our baseline is a **linear regression** on **two quantitative audio features**, `danceability` and `energy` (both already 0–1, so no encoding is needed — zero ordinal and zero nominal features). All steps live in a single scikit-learn `Pipeline`, evaluated on the held-out test set.

Its test **RMSE is ≈ 29.5** and its **R² is ≈ 0.11**, meaning the two audio features explain only about 11% of the variation in popularity — only modestly better than predicting the average popularity for every track. That is expected for a baseline, and it sets a clear bar to beat.

## Final Model

The final model is a **random forest regressor** trained on a much richer feature set: all ten audio features, `release_year`, `explicit`, one-hot-encoded `track_genre`/`key`/`mode`/`time_signature`, and the two artist features (`followers` and `artist_popularity`, median-imputed for the ~1% of tracks with no artist match). On top of the baseline's columns we engineered **two new features**:

- **`n_artists`** — the number of collaborating artists on the track. Collaborations and features tend to broaden a track's reach.
- **`artist_genre_breadth`** — how many genre tags the artist carries, a proxy for how established and wide-reaching the artist is.

Both are computed per track from data we already have, so they introduce no leakage. We chose a random forest because the relationship between these features and popularity is nonlinear and interaction-heavy, which the linear baseline could not capture. Using `GridSearchCV` (3-fold) we tuned **`max_depth`** (the main control on complexity) over {None, 15, 25} and **`min_samples_leaf`** (a regularizer) over {1, 5}, holding the number of trees at 200; cross-validation selected a maximum depth of 15.

On the same held-out tracks, the final model's **test RMSE drops to ≈ 20.4** (from ≈ 29.5) and its **R² rises to ≈ 0.58** (from ≈ 0.11) — so the improvement comes from the model and features, not an easier test set.

<iframe src="assets/feature_importances.html" width="100%" height="500" frameborder="0"></iframe>

The feature-importance chart explains why: the **artist features** (`artist_popularity` and `followers`) are among the strongest predictors, alongside audio characteristics like `energy` and `acousticness` and the track's `release_year`. A track's popularity depends heavily on *who* released it and how recent it is — exactly the signal the audio-only baseline was missing.

## Fairness Analysis

Finally, we asked whether the model is **less accurate for explicit tracks than for non-explicit tracks**, on the held-out test set.

- **Group X:** explicit tracks. **Group Y:** non-explicit tracks.
- **Evaluation metric:** RMSE within each group (a regression metric — classification metrics like precision do not apply).
- **Null hypothesis:** the model is fair — its RMSE is the same for both groups, and any difference is chance.
- **Alternative hypothesis:** the model is unfair — its RMSE differs between the groups.
- **Test statistic:** the absolute difference in group RMSE. **Significance level:** 0.05.

The model's RMSE was **26.2 for explicit tracks** versus **19.6 for non-explicit tracks** — a gap of about **6.6 popularity points** — with a permutation **p-value of ≈ 0.001**.

<iframe src="assets/fairness_null.html" width="100%" height="500" frameborder="0"></iframe>

We **reject the null hypothesis**: the model is **less accurate for explicit tracks**. A likely explanation is that explicit tracks are a small minority of the training data (~9%) and are concentrated in high-variance genres like hip-hop, so the model has both less data and a harder target for this group. Its predictions should be treated with more caution for explicit tracks — a useful caveat, and a pointer toward collecting more explicit examples or modeling that segment separately. As with any permutation test, this identifies a performance disparity, not its ultimate cause.
