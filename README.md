# GitHub Issue Predictor

Will this GitHub issue get fixed? A machine learning project on real issues from the [pandas](https://github.com/pandas-dev/pandas) repository.

## What this project does

This project predicts whether a GitHub issue will get fixed or not. It looks at an issue 7 days after it was opened and predicts whether it will eventually be closed as completed.

**Who would use it:**

- **Maintainers**, so they can see which issues are likely to get fixed and which are at risk of being ignored, and decide what to look at first.
- **People who report bugs**, so they know if their issue has a low chance and can add more details, like steps to reproduce, to improve it.

## How "fixed" is defined, and why day 7

**Fixed vs not fixed:** an issue counts as **fixed** if GitHub shows it was closed as *completed*. Everything else counts as **not fixed**: closed as *not planned*, closed as a *duplicate*, or still open.

**Why day 7:** choosing when to predict was a trade-off.

- On **day 1 or 2**, there isn't enough information yet to predict well.
- By **day 30**, there's more information, but the issue may already have been ignored, and the prediction comes too late to help.
- **Day 7** is a sweet spot: there's some early activity to learn from, and there's still time to follow up. Day 14 would also be reasonable, and comparing the two is a possible next step.

**Why issues closed before day 7 are removed:** the model is meant for issues that are still open on day 7, so there's no point teaching it with issues that were already closed. It would also cause **leakage**: issues closed in the first week often have the answer in their early comments (like *"Fixed in PR #1234"*), and the model would learn a shortcut that real open issues never have.

## The data

**Source:** issues from the pandas GitHub repository, collected through the GitHub API.

**Rules used to build the dataset:**

1. **Only real issues.** GitHub's issues list also includes pull requests, so those were removed.
2. **Only issues at least 6 months old.** A newer issue that's still open might simply not have had time to get fixed yet. Labelling it "not fixed" would be wrong.
3. **Only issues still open on day 7**, as explained above.
4. **Only first-week information.** Comments were collected only from the first 7 days of each issue. Anything later counts as the future.

**Dataset size:**

| Stage | Issues |
|---|---|
| Collected (May 2024 to Apr 2026, PRs removed) | 1,920 |
| After the day-7 rule | **1,212** |
| Fixed | 874 (72%) |
| Not fixed | 338 (28%) |

The data is **imbalanced**: a model that always says "fixed" is already right 72% of the time. That's why the results use AUC instead of accuracy.

**A note on data quality:** pandas maintainers often close issues as *completed* even when nothing was fixed (for example, junk issues or duplicates closed within minutes). So "fixed" here really means "closed and resolved in some way." The day-7 rule removes most of those quick closes.

## Features

All features use only information available on **day 7**.

| Idea | Features |
|---|---|
| How detailed is the description? | Body length, has a code block, has an error message (traceback), mentions a version number |
| Who opened it? | Opened by a pandas team member (MEMBER) or not |
| First-week activity | Number of comments in week 1, did a team member reply, hours until the first team member reply |
| What kind of issue? | Title prefix: BUG, ENH, DOC, NONE or OTHER (rare prefixes grouped) |

If no team member replied in the first week, the reply time was set to **168 hours** (the full week). Setting it to 0 would have told the model "replied instantly," which is the opposite of the truth.

**Not used, because of leakage:**

- **Total comment count:** it includes comments from after day 7, sometimes like *"Fixed in PR #1234."*
- **Labels:** the API returns labels as they are today. Some may have been added after day 7, for example when the issue was fixed.

**What surprised me:**

- **Documentation issues had the highest fix rate**, at **94%**, not bugs as I expected. Feature requests (ENH) had the lowest, at 56%. Bugs were in the middle, at 74%.
- **A team member replying in the first week made almost no difference** (73% fixed with a reply vs 72% without). A reply doesn't mean a fix: maintainers also reply to ask for details, point out duplicates, or say it isn't a bug.
- **Issues opened by team members were fixed *less* often** (64% vs 74%). Team members often open issues as long-term notes, reminders or discussions, which stay open longer than a regular bug report.

The lesson for me: features that sound logical still need to be checked against the data.

## Results

**Why not accuracy:** the data is imbalanced (72% fixed). A lazy model that always says "fixed" already gets **72% accuracy** without learning anything, and every real model also landed at about 70–72%. Accuracy couldn't tell them apart, so models were compared with **AUC**: how often the model ranks a "not fixed" issue as riskier than a "fixed" one. 0.5 means random guessing.

**One test split vs cross-validation:**

| Model | Single test split AUC | Cross-validation AUC | Spread |
|---|---|---|---|
| Lazy (always "fixed") | 0.50 | — | — |
| Logistic regression | 0.66 | **0.62** | 0.04 |
| Decision tree (depth 3) | 0.67 | 0.59 | 0.01 |
| Random forest | 0.68 | 0.60 | 0.03 |
| Boosting | 0.67 | 0.58 | 0.05 |

The single test split (243 issues) turned out to be **lucky**: every model scored lower with 5-fold cross-validation, and the ranking changed. On the single split, random forest looked best. With cross-validation, **logistic regression** was best on average, with a small spread. The fancier models did not help.

**What the results mean:** an AUC of about **0.62** is better than random, but weak. Whether an issue gets fixed seems to depend mostly on things these features can't see, such as how busy the maintainers are, how hard the fix is, and project priorities.

**Model choice:** I chose **logistic regression**. It had the best average AUC (0.62), its spread (0.04) is small enough to trust, and it's the simplest and cheapest model. The fancier tree models gave no real improvement, so there was no reason to use them.

**Final test-set result:** I chose a cutoff of **0.30**, favoring recall because a missed at-risk issue can be ignored for months, while a wrong flag only costs a quick look. The cutoff was chosen with cross-validated scores on the training data only. On the 243 hidden test issues, the model flagged 91 (37%) as at risk, with **precision 0.44** and **recall 0.59** (it caught 40 of the 68 issues that were not fixed), and an **AUC of 0.67**. This test split scored higher than cross-validation (0.62), so I treat **0.62 as the more realistic estimate**.

## What I'd try next

- **Predict on day 14 instead of day 7**, and compare the scores. More time means more information, but the prediction comes later, so it's a trade-off worth measuring.
- **Measure how busy the maintainers are.** For example, count how many issues were opened in the same week. In a busy week, each issue may get less attention. This has to use only information from before day 7, to avoid leakage.
- **Use labels as they were on day 7.** Labels like "Bug" or "Enhancement" could help, but the API only shows today's labels. GitHub keeps a history of when each label was added, so it's possible to rebuild the labels as they were on day 7.

## Related work

Predicting which bugs get fixed is a known research problem. For example, Microsoft Research studied it on Windows bugs in [Characterizing and Predicting Which Bugs Get Fixed](https://www.microsoft.com/en-us/research/?p=158889) (2010). This project looks at the same question on open-source GitHub issues, with a strict day-7 rule to avoid leakage.

## How to run

1. Create a virtual environment and install the packages:
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   pip install pandas scikit-learn requests python-dotenv tqdm jupyter
   ```
2. Create a GitHub fine-grained personal access token (public repositories, read-only) and put it in a `.env` file:
   ```
   GITHUB_TOKEN=your_token_here
   ```
   `.env` is in `.gitignore`, so it is never committed.
3. Open `01_collect.ipynb` in VS Code (or Jupyter) and run the cells in order.