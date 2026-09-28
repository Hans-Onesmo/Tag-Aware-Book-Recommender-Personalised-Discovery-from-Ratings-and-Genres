# Tag-Aware Book Recommender: Personalised Discovery from Ratings and Genres

**Course:** DSA 4060 – Recommender Systems  
**Author:** _[Your Full Name]_ (Student ID: _[Your ID]_) – individual project  
**Repository:** _[GitHub repository URL]_

---

## 1. Project Description (A. Title and Problem Statement)

**Problem.** I will develop a book recommender that helps readers discover titles matching their preferred genres and rating history, while surfacing relevant, less-obvious books they have not yet rated.

**Who experiences the problem.** Readers on book-sharing platforms who face thousands of titles and usually fall back on bestseller lists or friends' suggestions.

**Why choosing books is difficult.**
- A book is an experience good: its quality cannot be judged before reading it, and reading takes hours.
- Taste is multi-dimensional (genre, author, tone, era), so one average rating says little about fit.
- Popularity dominates most lists, so niche books matching a reader's taste stay hidden.

**How recommendations help.** A ranked list built from a reader's own ratings and genre interests replaces browsing with a short, relevant, explained shortlist.

---

## 2. Intended Users and Recommendation Task (B)

| Element | Definition |
|---|---|
| **Users** | Readers with a history of book ratings (or who supply a few ratings and favourite genres) |
| **Items** | Books from the Goodbooks-10k catalogue (10,000 titles) |
| **Inputs** | Historical ratings (1–5); optionally a user-selected genre filter; for a new user, 5+ ratings entered in the interface |
| **Output** | A ranked list of Top-10 unread books, each with a short explanation (e.g. "Because you rated *X* 5 stars – shares tags: fantasy, adventure") |
| **Recommendation moment** | On demand, when a reader opens the app and asks "What should I read next?" |
| **Constraints** | Never recommend a book the user has already rated; recommend only books with a minimum number of ratings (to avoid unreliable items); respect an optional genre filter |

**User scenario.** A reader rates five books they enjoyed (three fantasy, two mystery) and selects "fantasy". The system returns ten unread fantasy-leaning books, ranked by predicted relevance, each with a one-line reason. The reader can adjust the genre filter or the "more popular ↔ more niche" slider and see the list update.

---

## 3. Dataset Selection and Feasibility (C)

**Dataset:** Goodbooks-10k  
**Source:** https://www.kaggle.com/datasets/zygmunt/goodbooks-10k (Kaggle) and https://github.com/zygmuntz/goodbooks-10k (GitHub)  
**Licence/usage:** _[Confirm the licence stated on the repository page before submitting.]_ Raw data is **not** committed to this repository; download instructions are given below. User IDs are anonymised integers, so there is no identifiable personal information.

### Inspected statistics (computed from the downloaded files)

| Measure | Value |
|---|---|
| Ratings (`ratings.csv`) | 5,976,479 |
| Users | 53,424 |
| Books | 10,000 |
| Matrix sparsity | 98.9% (about 1.1% of user–book cells are filled) |
| Ratings per user | min 19, median 111, max 200 |
| Ratings per book | min 8, median 248, max 22,806 |
| Rating distribution | 1★ 2.1%, 2★ 6.0%, 3★ 22.9%, 4★ 35.8%, 5★ 33.2% |
| Duplicate (user, book) pairs | 0 |
| Tag records (`book_tags.csv`) | 999,912 (about 100 tags per book, 34,252 distinct tags) |

### Files and important columns

| File | Key columns | Meaning |
|---|---|---|
| `ratings.csv` | `user_id`, `book_id`, `rating` | Explicit user–item interactions (1–5) |
| `books.csv` | `book_id`, `title`, `authors`, `original_publication_year`, `average_rating`, `ratings_count`, `language_code` | Item attributes |
| `tags.csv` | `tag_id`, `tag_name` | Tag vocabulary |
| `book_tags.csv` | `goodreads_book_id`, `tag_id`, `count` | How many users applied each tag to each book (content signal) |
| `to_read.csv` | `user_id`, `book_id` | Books users saved to read (optional implicit signal, not used initially) |
| `sample_book.xml` | – | Example of the raw Goodreads XML record for one book; documentation only, not used for modelling |

### Sample records

`ratings.csv`

| user_id | book_id | rating |
|---|---|---|
| 1 | 258 | 5 |
| 2 | 4081 | 4 |
| 2 | 260 | 5 |
| 2 | 9296 | 5 |
| 2 | 2318 | 3 |

`books.csv` (selected columns)

| book_id | title | authors | year | average_rating | ratings_count |
|---|---|---|---|---|---|
| 1 | The Hunger Games | Suzanne Collins | 2008 | 4.34 | 4,780,653 |
| 2 | Harry Potter and the Sorcerer's Stone | J.K. Rowling, Mary GrandPré | 1997 | 4.44 | 4,602,479 |
| 3 | Twilight | Stephenie Meyer | 2005 | 3.57 | 3,866,839 |
| 4 | To Kill a Mockingbird | Harper Lee | 1960 | 4.25 | 3,198,671 |
| 5 | The Great Gatsby | F. Scott Fitzgerald | 1925 | 3.89 | 2,683,664 |

`book_tags.csv` joined with `tags.csv` (book 1)

| goodreads_book_id | tag_name | count |
|---|---|---|
| 1 | to-read | 167,697 |
| 1 | fantasy | 37,174 |
| 1 | favorites | 34,173 |
| 1 | currently-reading | 12,973 |
| 1 | young-adult | 12,716 |

### Why the data supports the methods

- **Collaborative filtering and matrix factorization:** about 6 million explicit ratings from 53k users.
- **Content-based filtering:** about 100 user-generated tags per book, plus authors and publication year.
- **Baseline:** `average_rating` and `ratings_count` allow a weighted popularity ranking.

### Known limitations

1. **Popularity skew.** Only the 10,000 most-rated books are included, so the catalogue is already mainstream.
2. **Noisy tags.** Many top tags are personal shelves (`to-read`, `favorites`, `owned`, `currently-reading`) and not genres. These will be filtered out using a stop-list and a genre whitelist.
3. **No book descriptions.** Text-based methods (TF-IDF on synopses) are not possible from this dataset alone.
4. **Missing metadata.** `language_code` is missing for 1,084 books, `isbn` for 700, `original_title` for 585, `original_publication_year` for 21.
5. **No cold-start users.** Every user has at least 19 ratings, so new-user behaviour must be simulated.
6. **No timestamps**, so temporal evaluation and preference drift cannot be studied directly.

### Data access

Download the files from Kaggle (https://www.kaggle.com/datasets/zygmunt/goodbooks-10k) into a local `data/` folder, or fetch the five CSV files directly from GitHub:

```bash
mkdir -p data && cd data
for f in books.csv ratings.csv tags.csv book_tags.csv to_read.csv; do
  curl -L -O https://raw.githubusercontent.com/zygmuntz/goodbooks-10k/master/$f
done
```

`sample_book.xml` is included in the Kaggle download and is not needed to run the code.

Only one dataset is used, so no cross-dataset record matching is needed.

---

## 4. Proposed Recommendation Approaches (D)

### Baseline: Weighted popularity

- **Why:** a standard reference point every personalised method should beat.
- **Data:** `average_rating`, `ratings_count` (or aggregates computed from the training split).
- **Role:** ranks unread books by a Bayesian-weighted rating, so a book with few ratings does not outrank a widely loved one. Also serves as a fallback for brand-new users.
- **Limitation:** not personalised, and reinforces popularity bias.

### Approach 1: Content-based filtering (tags, authors, year)

- **Why:** handles items with few ratings and gives natural explanations ("shares tags: fantasy, adventure").
- **Data:** filtered tag vectors (TF-IDF weighted by tag `count`), authors, publication era.
- **Role:** builds a user profile from the tag vectors of books they rated highly and ranks unread books by cosine similarity.
- **Limitation:** over-specialisation, since it keeps recommending near-copies of what the user already read.

### Approach 2: Item-based collaborative filtering

- **Why:** with 10,000 items and dense per-item rating counts, item–item similarity is stable and computationally practical.
- **Data:** the user–item rating matrix (mean-centred).
- **Role:** recommends books similar, by rating patterns, to books the user rated highly.
- **Limitation:** sparsity (98.9%) makes similarity noisy for books with few co-raters, and it cannot handle brand-new items.

### Approach 3: Matrix factorization (SVD / ALS)

- **Why:** learns latent taste factors that capture patterns beyond explicit tags; typically the most accurate on rating data. This meets the course requirement for a matrix factorization approach.
- **Data:** the same rating matrix.
- **Role:** predicts a rating for every unread book; the top predictions become the recommendation list.
- **Limitation:** latent factors are hard to explain, and the model needs retraining for new users or items.

### Stretch goal: Hybrid

A weighted combination of the content-based and matrix factorization scores, with the weight tuned on validation data. This directly addresses the weaknesses of each part (explainability and cold-start from content; accuracy from factorization).

---

## 5. Evaluation Plan (E)

### Data splitting

1. Keep users with at least 19 ratings (all users qualify).
2. **Per-user split:** for every user, hold out 20% of their ratings as the **test set**; use a further 10% of the remaining ratings as a **validation set** for tuning. The rest is training data.
3. A book is treated as **relevant** if the user rated it **4 or higher**.
4. All models, including the baseline, are evaluated on the same test users and the same held-out items, with a fixed random seed.

Because every model is trained only on the training split, the test ratings act as "unseen" items for each user, which is what makes ranking metrics meaningful.

### Metrics

| Metric | What it assesses | Why it fits this data |
|---|---|---|
| **Precision@10 / Recall@10** | How many of the top-10 suggestions are relevant, and how many of the user's relevant held-out books are found | The explicit ratings define relevance (≥ 4) |
| **NDCG@10** | Whether relevant books are ranked near the top | The output is a ranked list, and graded ratings (4 vs 5) allow gain weighting |
| **RMSE / MAE** | Accuracy of predicted ratings (item-based CF and matrix factorization only) | Ratings are numeric 1–5 |
| **Catalogue coverage** | Share of the 10,000 books that appear in anyone's Top-10 | Detects popularity-only behaviour |
| **Intra-list diversity** | Average tag dissimilarity within a Top-10 list | Detects repetitive recommendations, using the tag vectors |

Metrics are averaged over test users, and results for all approaches and the baseline are reported in one comparison table. Response time per recommendation request will also be recorded for the interface.

**Simulated cold start.** To test new-user behaviour, I will keep only 5 ratings for a random sample of users and compare all methods on their held-out ratings.

---

## 6. Challenges and Responsible Design (F)

| Challenge | Initial response |
|---|---|
| **New users with no history** | Ask for 5+ ratings or favourite genres in the interface; fall back to weighted popularity within the chosen genre. Evaluated through the simulated cold-start test above. |
| **Sparse interactions (98.9% sparsity)** | Regularised matrix factorization, similarity shrinkage in item-based CF, and a minimum co-rating threshold. |
| **Popularity bias** | Report catalogue coverage; apply a popularity penalty or re-ranking, and give the user a "more niche" control. |
| **Noisy tags** | Remove personal-shelf tags and keep only genre-like tags above a frequency threshold. |
| **New items with no ratings** | Content-based scores (tags, author) are available even without ratings. |
| **Repetitive lists** | Limit to a maximum of 2 books per author in a Top-10, and monitor intra-list diversity. |
| **Privacy and data use** | The dataset has only anonymised user IDs; no raw data is uploaded to GitHub, and no personal data is collected by the interface. |

**User understanding and control.** Each recommendation shows a short explanation (the shared tags or the similar book it came from), and the user can filter by genre and move a popularity ↔ niche slider to change the results.

---

## 7. Implementation Plan (G)

**Tools:** Python, Pandas, NumPy, Scikit-learn, SciPy, Surprise or `implicit` (matrix factorization), Matplotlib/Seaborn, Jupyter Notebook, Streamlit (interface), Git/GitHub.

**Responsibility:** This is an individual project. I will complete and understand every component: data preparation, models, evaluation, interface, and documentation.

| Week | Tasks | Expected output |
|---|---|---|
| 1–2 | Select problem, inspect dataset, submit proposal, create repository | This README, data inspection notebook (`01_data_inspection.ipynb`) |
| 3–4 | Clean data and tags, create splits, build weighted popularity baseline and initial content-based prototype | `02_preparation.ipynb`, `03_baseline_content.ipynb`, first Precision@10 results |
| 5–6 | Build item-based CF, run preliminary evaluation against baseline | `04_item_cf.ipynb`, comparison table v1 |
| 7 | Mid-semester examination | – |
| 8–10 | Matrix factorization, hyperparameter tuning, hybrid model, cold-start and diversity experiments | `05_matrix_factorization.ipynb`, `06_hybrid.ipynb`, comparison table v2 |
| 11–12 | Streamlit interface with explanations and controls, address limitations, finalise evaluation and documentation | `app/`, final results, updated README |
| 13 | Present and demonstrate the system | Demo and slides |
| 14 | End-semester examination | – |

---

## 8. Repository Structure

```
.
├── README.md
├── requirements.txt
├── data/            # not committed; see download instructions
├── notebooks/
├── src/
└── app/
```

## 9. References

- Goodbooks-10k dataset: https://www.kaggle.com/datasets/zygmunt/goodbooks-10k
- Goodbooks-10k GitHub repository: https://github.com/zygmuntz/goodbooks-10k
