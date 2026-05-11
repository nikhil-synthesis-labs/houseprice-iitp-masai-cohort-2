# Pre-planted Issues for the House Price Predictor anchor repo

This file documents the 6 bugs deliberately planted in this codebase, to be filed as GitHub Issues when the instructor publishes the repo before class.

**Issue #1 is fixed LIVE during class.** Issues #2-#6 are reserved for the assignment — students pick one and submit a PR.

For each issue below: copy the title and body into a new GitHub Issue and add the listed labels. Use the issue numbers below as your filing order so they line up.

---

## Issue #1 — Typo: "Hosue Price Predictor" in app title

**Labels:** `good-first-issue`, `documentation`

**Body:**

The main page title in `app.py` reads "Hosue Price Predictor" — should be "House".

Steps to reproduce:
1. Run `streamlit run app.py`
2. Open the app in a browser
3. The big page title shows the typo

Expected: "House Price Predictor"

Where to look: `app.py`, the `st.title(...)` call near the top of the UI section.

This is a one-character fix, ideal for a first-ever pull request.

---

## Issue #2 — Predicted price displays as raw float, not Indian Rupee format

**Labels:** `bug`, `good-first-issue`

**Body:**

The "Predicted Price" metric currently shows numbers like `12849532.4321`. We want Indian Rupee formatting — `₹1,28,49,532` with rupee symbol and Indian-grouping commas (lakh/crore style: 1,28,49,532, not 12,849,532).

Steps to reproduce:
1. Run the app
2. Move any slider
3. The "Predicted Price" metric shows a raw decimal number

Expected: `₹` prefix + Indian-grouped integer part. Example: `₹1,28,49,532`.

Hints:
- The standard library's `locale` module supports Indian formatting via `locale.setlocale(locale.LC_MONETARY, 'en_IN')` but this depends on system locale being installed.
- A simpler self-contained approach: implement an `indian_format(n: int) -> str` helper that groups the rightmost 3 digits, then groups by 2s.

Where to look: `app.py`, the `st.metric(...)` call.

---

## Issue #3 — Nonsensical "Price per sqft: inf" when "Square footage" slider is 0

**Labels:** `bug`

**Body:**

Setting the Square footage slider to 0 produces a nonsensical metric: the app displays `Price per sqft: inf`.

Steps to reproduce:
1. Run the app
2. Drag the Square footage slider all the way to the left (value 0)
3. The "Price per sqft" line below the metric shows `inf`

What's happening: the line `price_per_sqft = predicted_price / sqft` divides by zero. Because `predicted_price` is a `numpy.float64` (returned from `model.predict()[0]`), the division returns IEEE-754 `inf` rather than raising an exception (NumPy is IEEE-754-compliant; pure Python's `float / 0` would raise `ZeroDivisionError`). A `RuntimeWarning` is logged but doesn't stop the app.

Either way, the displayed value is meaningless to the user.

Expected: when sqft is 0, either don't display "Price per sqft" at all, OR show "—" / "N/A" / "(invalid)", OR raise the slider's `min_value` to a sensible floor (100 or 400 — see also Issue #6 for training-range alignment).

This is a real-world ML deployment trap: production users send inputs you didn't anticipate. Defensive checks belong in the prediction layer.

Where to look: `app.py`, after the `predict_price(...)` call.

---

## Issue #4 — README is missing key sections

**Labels:** `documentation`

**Body:**

The current `README.md` has only the title and a one-line description. Anyone landing on this repo (a recruiter, a teammate, a future you) cannot:
- Set the project up
- Run the app
- Find the live demo
- Understand how to contribute

Add the following sections, in order:

1. **Live demo** — link to the Streamlit Community Cloud URL (replace with your fork's URL after deploying)
2. **Setup with `uv`** — three-line install: install uv, `uv sync`, `uv run streamlit run app.py`
3. **Project structure** — directory tree explaining what's in `app.py`, `model/`, `data/`
4. **How to contribute** — fork → branch → fix → push → PR → reference an issue

Use clean Markdown headers (`##`). Aim for ~80-150 lines total.

Where to look: `README.md`.

---

## Issue #5 — `scikit-learn` pin is too narrow; can break deployment

**Labels:** `dependency`, `bug`

**Body:**

> **Note for instructors:** This template's `pyproject.toml` currently ships with `scikit-learn>=1.4` (a sensible floor) plus a `[tool.poetry] package-mode = false` block (required for Streamlit Cloud deploys — see `Session_13_1_Anchor_Repo_Setup.md`). To restore the teaching bug for a fresh cohort, narrow the pin (e.g. back to `scikit-learn==1.0.2` or `scikit-learn==1.4.*`) and re-commit. The Issue #5 narrative below assumes the bug is in place; if you've left the modern floor pin, repurpose this issue as "audit the pin: is `>=1.4` too loose? should we lock it via `uv.lock`?" instead.

Our `pyproject.toml` pins `scikit-learn` to a single old or narrow version. Two real risks follow from this:

1. **Build / install failure on newer Python.** Old sklearn versions have no prebuilt wheels for Python 3.13 and require a source build with `setuptools.build_meta:__legacy__`, which has been removed. Streamlit Cloud / a fresh Colab kernel will fail with `PEP517 build of a dependency failed`.
2. **`InconsistentVersionWarning` at runtime.** If the install succeeds but the runtime sklearn differs from the version that originally trained `model.joblib`, `joblib.load()` warns. Predictions still work but the log is noisy and the model is "drifting."

Steps to reproduce:
1. `uv sync` on a clean machine OR push to Streamlit Cloud
2. Observe either (a) a `PEP517 build of a dependency failed` error, OR (b) a noisy `InconsistentVersionWarning` when the app loads

Expected: pin a current scikit-learn (`>=1.4` works as of 2026) AND re-train the model so the saved version matches the runtime version.

Fix:

1. Update the `scikit-learn` line in `pyproject.toml`. Suggestion: `"scikit-learn>=1.4"`.
2. Run `uv lock` to regenerate the lock file (pinning a deterministic exact version).
3. Run `uv run python model/train.py` to re-train and re-save the model with the current sklearn.
4. Verify the app starts cleanly with no `PEP517` failure AND no `InconsistentVersionWarning`.

This is a real-world MLOps trap: dependency pins drift, model artifacts drift, and the gap between them silently breaks production. The fix is conceptually simple — re-pin, re-train, re-commit — but you only learn the discipline by hitting the bug once.

Where to look: `pyproject.toml`, then `model/train.py`.

---

## Issue #6 — Predictions extrapolate poorly outside the training range

**Labels:** `ml-bug`, `bug`

**Body:**

The training data has features in these ranges (see `synthesize_houses()` in `model/train.py`):
- `sqft`: [400, 4500]
- `bedrooms`: [1, 6]
- `age`: [0, 79]
- `location_score`: [1.0, 9.5]

The Streamlit sliders allow:
- `sqft`: [0, 5000]
- `bedrooms`: [0, 8]
- `age`: [0, 100]
- `location_score`: [0.0, 10.0]

When users move sliders OUTSIDE the training range, the `LinearRegression` model extrapolates linearly with no awareness of physical constraints. Predictions for sqft = 50 fall well below ₹5L (the implicit minimum in the training distribution); predictions for sqft = 5000 with high location_score can drift outside the upper end of any comparable's price.

This is a foundational ML pitfall: linear regression has no built-in awareness of the training-data domain — the model will happily extrapolate any input you give it, even nonsensically.

Steps to reproduce:
1. Run the app
2. Set sqft=50, bedrooms=1, age=70, location_score=1.0
3. Look at the predicted price — far below the lowest price in the "Sample comparable houses" table at the bottom of the app
4. Now set sqft=400 (the training-range minimum) with the same other inputs and compare

Expected: predictions outside the training range should either be (a) clipped, (b) accompanied by a user-facing warning, OR (c) the model retrained on a wider synthetic dataset to cover the full slider range.

Fix options (any ONE is acceptable for the assignment; deeper options score higher):

- **Option A — Clip inputs in `app.py`** (cheapest):
  ```python
  TRAINING_RANGES = {
      "sqft": (400, 4500),
      "bedrooms": (1, 6),
      "age": (0, 79),
      "location_score": (1.0, 9.5),
  }
  # before calling predict_price:
  sqft_clamped = max(TRAINING_RANGES["sqft"][0], min(TRAINING_RANGES["sqft"][1], sqft))
  if sqft_clamped != sqft:
      st.warning(f"sqft={sqft} is outside the training range [400, 4500]; using {sqft_clamped} for prediction.")
  ```
- **Option B — Restrict the slider ranges to match training** (medium): edit each `st.slider()` call so `min_value` / `max_value` align with the training-data ranges. Cheap, but less educational about defensive programming.
- **Option C — Retrain on a wider dataset** (deepest): edit `synthesize_houses()` in `model/train.py` to use wider ranges (e.g., `sqft = RNG.integers(0, 5000, ...)` etc.), re-train, re-save `model.joblib`. Then no clipping needed at inference time.

This is the most ML-substantive issue. Recommended for someone who's done at least one regression project before.

Where to look: `app.py` (input handling), `model/train.py` (training data ranges), or both.

---

## After-class maintenance

When students fix issues #2-#6 and PRs are merged, the issue pool depletes. Before each new cohort, the instructor should:

- Either revert the merge commits (recreates the bugs cleanly)
- Or rotate to a fresh app variant — e.g., a "Movie Rating Predictor" with a parallel set of bugs.

Track this in `Asset/Session_13_1_Anchor_Repo_Setup.md` (the instructor pre-class checklist).
