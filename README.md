# 9dots — reconstructing motor protocols of the nine-dot problem

Code for the paper **"A computational processing method for motor protocols of solving the nine-dot problem"**
by T.P. Chernaya and V.F. Spiridonov (RANEPA, Moscow), published in the proceedings of the conference *Cognitive Science in Moscow: New Research* (2021), pp. 543–547.

**How to cite**

> Черная Т.П., Спиридонов В.Ф. Метод компьютерной обработки моторных протоколов решения задачи 9 точек // Когнитивная наука в Москве: новые исследования. Материалы конференции 2021 / Под ред. Е.В. Печенковой, М.В. Фаликман, А.Я. Койфман. М., 2021. С. 543–547.

> Chernaya T.P., Spiridonov V.F. A computational processing method for motor protocols of solving the nine-dot problem. In: E.V. Pechenkova, M.V. Falikman, A.Ya. Koifman (Eds.), *Cognitive Science in Moscow: New Research. Conference Proceedings 2021*. Moscow, 2021. P. 543–547.

The [nine-dot problem](https://en.wikipedia.org/wiki/Thinking_outside_the_box#Nine_dots_puzzle) (Maier, 1930) asks you to connect a 3×3 grid of dots with four straight lines without lifting the pen. Participants solved it on a touchscreen tablet, and the tablet logged the finger position at about 100 Hz. This repository turns those raw trajectories into a list of **straight line segments** for each attempt, such as "dots 0→4→8, then left the square". It then uses those segments to test hypotheses about how insight arises.

## Data

The experimental data comes from three studies run for [Spiridonov, Loginov, Ivanchei & Kurgansky (2019)](http://dx.doi.org/10.3389/fpsyg.2019.00002): n = 154 participants aged 16–28, 84 % female.

**The data is not included in this repository.** To run the notebooks you need the raw files. Each participant has one `.csv` file, laid out like this:

| Rows (0-based, after header) | Columns 1–4 | Meaning |
|---|---|---|
| 6–8, 11–13, 16–18 | left, upper, width, height | Bounding boxes of the 9 dots |
| 26, 27 | column 1 only | Age, gender |
| 29 → end | try_number, inner_index, x, y | Touch trajectory, one row per sample |

The code multiplies all x coordinates by 9 and all y coordinates by 8 to bring them into a common pixel scale. Every tolerance below is in these scaled units.

## How the algorithm works

For every attempt (`try`), the steps are:

1. **Skeleton.** Walk along the trajectory (every 5th sample) and record events in order: *dot `k` touched* (the finger is within `dot_radius + max_dot_error` of the dot's centre) or *left the square* (the finger is more than `outer_border_dist` outside the outermost dot centres). If two consecutive samples are more than `max_inline_gap` apart, the attempt is marked invalid.
2. **Lines.** Split the event sequence into segments. A new segment starts when
   - the next dot is not collinear with the current segment (the line bends), or
   - the trajectory leaves the square (the attempt either ends or must turn back).
3. **Geometry clean-up.** Segments that go out of the square are extended by `prolong_len`. Two consecutive out-of-square segments are joined at their intersection point if that point lies outside the square.

Each output line carries these fields: `dots_intersected`, `start`, `end`, `start_oob`, `end_oob` (*oob* = out of box).

The algorithm only produces geometrically *valid* lines. For example, three dots count as one line only if they really are collinear. Messy drawings are therefore snapped to the nearest valid interpretation.

### Parameters

| Name | Value | Meaning |
|---|---|---|
| `dot_radius` | 180 | Visible dot radius |
| `max_dot_error` | 90 | Extra tolerance around a dot |
| `outer_border_dist` | 500 | Distance beyond the outer dots that counts as "outside the square" |
| `max_inline_gap` | 1500 | Largest allowed jump between samples before an attempt counts as broken |
| `prolong_len` | 3000 | How far out-of-square segments are extended (for plotting) |

### Dot numbering

```
0  1  2
3  4  5
6  7  8
```

## Repository structure

```
9dots/
├── README.md
├── requirements.txt
├── data/                        # not published — put your copy of the data here (git-ignored)
│   ├── raw/
│   │   ├── exp_1/               # one .csv per participant
│   │   ├── exp_2/
│   │   └── exp_3/
│   └── processed/               # lines, solved and H1 tables, demographics
└── notebooks/
    ├── 1_reconstruction/
    │   ├── 1_algorithm_demo.ipynb
    │   └── 2_reconstruct_all.ipynb
    ├── 2_hypotheses/
    │   ├── H1_diagonal.ipynb
    │   ├── H2_accidental_out_of_box.ipynb
    │   ├── H3_non_dot_turn.ipynb
    │   ├── H4_line_exhaustion.ipynb
    │   └── H4_line_exhaustion_alternative.ipynb
    └── 3_extras/
        ├── extract_demographics.ipynb
        └── plot_raw_trajectories.ipynb
```

### 1 — Reconstruction

| Notebook | What it does |
|---|---|
| `1_algorithm_demo` | Develops the algorithm on one participant and plots each attempt (raw trajectory + reconstructed lines, as in Fig. 1 of the paper). **Start here** to see how the algorithm works. |
| `2_reconstruct_all` | Runs the algorithm on all participants of experiments 1–3 → a lines table per experiment |

### 2 — Hypotheses

The letters match the paper's *Results and discussion* section.

| Notebook | Hypothesis | Result in the paper |
|---|---|---|
| `H1_diagonal` | (a) Starting with or using a **diagonal** early predicts success. Four variants: first line / any line × first attempt / first 3 attempts | Not confirmed (p = .24–.87) |
| `H2_accidental_out_of_box` | (b) "Accidentally" **ending / starting** a line outside the square, followed by ≥ 3 attempts strictly inside it, predicts success | Not confirmed (end: p = .99, start: p = .06) |
| `H3_non_dot_turn` | (c) Making a **non-dot turn** (turning outside the square) in a failed attempt predicts success | **Confirmed** (χ² = 45.72, p = 1.36 × 10⁻¹¹) |
| `H4_line_exhaustion` | (d) After trying ≥ 5 of the 8 inside-the-square line types (with no out-of-square lines so far), the participant breaks out of the square within the next 5 attempts | Not confirmed (χ² = 1.59, p = .21) |
| `H4_line_exhaustion_alternative` | Variant of (d): all 8 line types in the first half of the attempts vs. breaking out in the second half | Not reported in the paper |

Every hypothesis is tested with Pearson's χ² test on a 2×2 contingency table (`scipy.stats.chi2_contingency`) at α = 0.05.

### 3 — Extras

| Notebook | What it does |
|---|---|
| `extract_demographics` | Extracts age and gender → `demo_data.csv` |
| `plot_raw_trajectories` | Plots raw trajectories only, with the start of each attempt highlighted in red |

### Pipeline

```
data/raw/exp_{1,2,3}/*.csv
   │
   ├── 1_reconstruction/2_reconstruct_all ──► exp_{1,2,3}_lines_data.csv ──┐
   │                                                                       ├──► H2, H3, H4
   │                                     exp_{1,2,3}_solved_data_clear.csv ┘
   │
   ├── 3_extras/extract_demographics ──► demo_data.csv
   └── (not in repo) ──────────────────► hypothesis_1_exp{1,2,3}_final.csv ──► H1
```

Intermediate files in `data/processed/`:

- **`exp_N_lines_data.csv`**: one row per line. Index columns are `participant_id`, `try_id`, `line_id`; the other columns are the line fields listed above.
- **`exp_N_solved_data_clear.csv`**: one row per participant. `solved` is 0 = not solved, 1 = solved, or 2 = invalid (excluded).
- **`hypothesis_1_exp{N}_final.csv`**: one row per participant, with `solved` and the flags `first_diag_1_try`, `any_diag_1_try`, `first_diag_3_tries` and `any_diag_3_tries`.

> **Note:** `2_reconstruct_all` builds the lines table but has no code that saves it (`exp_N_data.to_csv(...)`). The `solved` and H1 tables are built outside this repository.

## Running

The notebooks were written for **Python 3.7** and **pandas < 2.0**. They use `DataFrame.append`, which was removed in pandas 2.0.

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
jupyter notebook
```

1. Put the raw participant files in `data/raw/exp_1`, `exp_2` and `exp_3`.
2. The data paths in the notebooks are still the original absolute paths (`/Users/tatyanach/Desktop/Cognitive Lab/9 dots/...`). Change them to point at `data/` before running. They are in the second cell of each notebook:

   | Original path | Change to |
   |---|---|
   | `.../9 dots/9d_exp_N/*.csv` | `../../data/raw/exp_N/*.csv` |
   | `.../9 dots/hypotheses_data/...` | `../../data/processed/...` |

3. Run `1_reconstruction`, then `2_hypotheses`.

## References

- Kershaw T.C., Ohlsson S. (2004). Multiple causes of difficulty in insight: The case of the nine-dot problem. *JEP: LMC*, 30(1), 3–13. https://doi.org/10.1037/0278-7393.30.1.3
- Lung C.-t., Dominowski R.L. (1985). Effects of strategy instructions and practice on nine-dot problem solving. *JEP: LMC*, 11(4), 804–811. https://doi.org/10.1037/0278-7393.11.1-4.804
- Maier N.R.F. (1930). Reasoning in humans. I. On direction. *Journal of Comparative Psychology*, 10(2), 115–143. https://doi.org/10.1037/h0073232
- Ohlsson S. (1984). Restructuring revisited. *Scandinavian Journal of Psychology*, 25(2), 117–129. https://doi.org/10.1111/j.1467-9450.1984.tb01005.x
- Spiridonov V., Loginov N., Ivanchei I., Kurgansky A.V. (2019). The role of motor activity in insight problem solving (the case of the nine-dot problem). *Frontiers in Psychology*, 10. https://doi.org/10.3389/fpsyg.2019.00002

## Contact

Tatyana Chernaya — tanya.p.chernaya@gmail.com
