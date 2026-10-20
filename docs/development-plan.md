# Development plan

> This document is subject to change. Checkboxes are ticked as the work progresses. Phases 6 and 7 are only a rough outline for now and will be detailed when their time comes - they are marked as _(TO BE DETAILED)_. Parts marked _(TO BE DECIDED)_ are waiting for a decision whether they are even going to be made or their technical details are not yet known.

## Overview

Development plan is roughly divided into phases that should go approximately in this order:

0. Data collection (`data-collector`)
1. Machine learning models
2. `predictor-service` with automated processes
3. Organizing the data layer
4. `assistant-service` - the AI agent/assistant
5. `spring-backend-service` for authentication, CRUD operations, the points system, the recommender system, etc.
6. `ui` - a SPA in a TypeScript framework
7. Finalization and deployment

### Thesis core

The application supports the research part of the thesis(ML vs AI predictions) and it is a working, interactive product that people can actually use but the focus is on the research itself. If time for development runs out, these three things have to be finished no matter what:

1. Comparison of ML models - classical and more modern ones, for predicting both the outcome and the exact scores of upcoming matches, measured on the same metrics.
2. Live evaluation - predictions are saved before each match is played and after the match they are checked against the real result. This shows how well the models do on matches they have never seen.
3. AI assistant with an evaluation - the assistant explains predictions, looks up data and makes its own predictions. To check if its answers are correct, it is tested on questions where the correct answer is already in the database (e.g. "How many goals did Arsenal score in their last 5 matches?"). Its predictions are compared with the ML models.

### Principles

- Bookmaker odds are not used as features. They can only be kept for evaluation.
- Validation is always chronological, never a random split. Features for a match can only use data that was available before it so that data leakage doesn't happen.
- The AI assistant can make its own predictions but they are kept separate from the ML predictions. ML predictions come from `predictor-service` and `assistant-service` gets them through a tool.
- Start simple with results, form and match statistics and then add injuries, lineups and player data one by one to see how much they actually help.
- Each database schema has only one service writing to it - `app` is written only by `spring-backend-service` and `football` only by `data-collector`. The only exception is `predictions.prediction_log`, where both `predictor-service` and `assistant-service` add their predictions - each one only with its own `source` and neither can change or delete rows.

---

## Preparing for extension

Scaling and distributed deployment are not the main part of this plan, but the code should be written so that they can be added later without rewriting everything. If you want to know more about that, see [architecture.md](architecture.md), section 10.

---

## Phase 0: Data collection (`data-collector`)

### Sources

| Source                                                                                                | What it gives us                                                   | Notes                                                           |
| ----------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ | --------------------------------------------------------------- |
| [football-data.co.uk](https://www.football-data.co.uk/englandm.php)                                   | Historical EPL seasons in CSV (results, match statistics, odds)    | Used for training. Odds stay in the dataset only for evaluation |
| [API-Football](https://www.api-football.com/) ([docs](https://www.api-football.com/documentation-v3)) | Fixtures in all competitions, injuries, lineups, player statistics | The free plan has 100 requests per day                          |

### Tasks

- [ ] Download historical EPL seasons into `predictor-service/data/raw/`
- [ ] Register an API-Football key and check `coverage` for the Premier League (injuries, lineups, statistics_players)
- [ ] Fetch results and fixtures for the current season
- [ ] Fetch fixtures for each club in all competitions so rest days and fixture congestion can be calculated
- [ ] Fetch injuries, lineups and player statistics, every record with `fetched_at`
- [ ] Match club names and IDs between different sources
- [ ] Decide on the structure of `data/raw` and `data/processed` and the file format (CSV or Parquet)
- [ ] Put the `data-collector` code in a separate module `predictor-service/data_collector/` with clear functions

**Phase output:** raw data that can be reproduced and a script that can fetch it again.

---

## Phase 1: Machine learning models

### 1.1 Jupyter notebook: W/D/L classification

- [ ] Baseline models - most frequent outcome and Elo
- [ ] Logistic regression, random forest, gradient boosting and an MLP with team embeddings
- [ ] Walk-forward validation
- [ ] Metrics - accuracy, log-loss, Brier score, calibration and results per class
- [ ] Comparison with probabilities from the odds

### 1.2 Features and dimensionality reduction

- [ ] Features - rolling form, home/away split, Elo, match statistics (shots, xG), rest days, number of matches in the last 14 days, missing key players
- [ ] Feature importance (permutation importance / SHAP) and correlations
- [ ] Dimensionality reduction - feature selection and PCA and comparison
- [ ] Model with injuries and lineups vs. model without them and model that only knows pre-match data vs. model that also knows the confirmed lineup

### 1.3 Exact score prediction

- [ ] Poisson regression for goals of each team (Dixon-Coles as reference)
- [ ] Gradient boosting with a Poisson objective
- [ ] Matrix of score probabilities, then get W/D/L from it and compare with the results from 1.1
- [ ] Metrics for scores (log-loss over scores, RPS)

### 1.4 Deep learning models

With only a few thousand matches, gradient boosting usually beats neural nets on tabular data. DL models are still worth testing and if they don't win that is a valid result too.

- [ ] MLP with team embeddings - each team gets a learned vector, the other inputs are the same features as in 1.2. Two outputs: W/D/L probabilities and expected goals for each team
- [ ] Sequence model (LSTM or a small Transformer) - instead of hand-made form features, it gets the raw sequence of each team's last N matches (goals, shots, xG, home/away) and learns the form by itself
- [ ] Research question: are hand-made features (1.2) better than what the sequence model learns from raw match history?
- [ ] Dropout and early stopping because of the small dataset and the same walk-forward validation, metrics and calibration check as the other models

### 1.5 More modern ML approaches _(TO BE DECIDED)_

- [ ] Go through the literature and current approaches
- [ ] Candidates - TabPFN, Bayesian / Dixon-Coles model, conformal prediction to show uncertainty
- [ ] Compare them with the models from 1.1, 1.3 and 1.4 on the same metrics

### 1.6 Evaluation module

- [ ] One module with the walk-forward backtest and metrics, used both in the notebook and in live evaluation
- [ ] The source of predictions can be swapped (ML, AI, odds, majority of users, hybrid) and the metrics stay the same

### 1.7 Unit tests for ML (pytest)

- [ ] Tests for feature functions on a small DataFrame made by hand
- [ ] Data leakage test - features for a match on date D only use data from before D
- [ ] Input and output validation (probabilities are in [0, 1] and add up to 1)
- [ ] Smoke test of training on a small dataset with a fixed seed

**Phase output:** a notebook with the model comparison and the evaluation module, which the evaluation chapter of the thesis will be based on.

---

## Phase 2: `predictor-service` and live predictions

> **Note:** predictions should start being saved as soon as there is a first working model. The season is already going and every missed round is a round of unseen matches that can't be recovered for live evaluation.

### 2.1 Pipeline

- [ ] Pipeline: fetch --> features --> (re)training --> evaluation --> save the model with metadata (version, date, metrics)
- [ ] Automatic refresh after a round is played (triggered by the round being finished)

### 2.2 Saving predictions (`prediction_log`)

- [ ] Append-only table `prediction_log` in Postgres (schema `predictions`) with `match_id`, `source`, `model_version`, `snapshot_time` and W/D/L probabilities
- [ ] `source` says who made the prediction - `ml` for `predictor-service` and later `ai` / `ai-hybrid` for `assistant-service` (phase 4), which writes to the same table
- [ ] Written before the match starts and never changed after that
- [ ] Two snapshots per match - an early one during the week and a final one after the lineup is confirmed
- [ ] Cron / scheduler that writes predictions for the next round

### 2.3 FastAPI and Docker

- [ ] `/health`, predictions for a round, prediction for one match and explanation (SHAP) for a match
- [ ] `Dockerfile` for `predictor-service`
- [ ] `docker-compose.yml` (`predictor-service`, Postgres, volumes for data and models)
- [ ] OpenAPI / Swagger documentation

### 2.4 Keeping saved predictions safe

The saved predictions can't be made again, so losing them or missing a round means losing part of the live evaluation.

- [ ] Run the scheduler and Postgres on an always-on server from the start of this phase, not only on a laptop (a free-tier VM or a hosted Postgres like Neon or Supabase) _(TO BE DECIDED)_
- [ ] Backup of the `predictions` schema after each round (`pg_dump`), stored outside that server
- [ ] Restore a backup at least once to check that it works

### 2.5 Integration tests

- [ ] FastAPI endpoints (TestClient)
- [ ] Whole pipeline on a small sample of data
- [ ] Smoke test in Docker

**Phase output:** `predictor-service` running in a container and already saving predictions for live evaluation.

---

## Phase 3: Data layer

This becomes necessary once there is a second service that needs the data - in this case: `assistant-service` and `spring-backend-service`.

- [ ] Move `data-collector` into its own folder in the repo (`data-collector/`) with its own `Dockerfile`
- [ ] `football` schema in Postgres (matches, results, injuries, lineups, statistics) with `fetched_at`
- [ ] `data-collector` writes to `football` and `predictor-service` reads from it instead of from CSVs
- [ ] Features stay in `predictor-service` because training and inference both use them
- [ ] Scheduler that runs the job after a round is played
- [ ] Idempotency and tests for `data-collector`

**Phase output:** `data-collector` running as a separate job that writes to the `football` schema, which the other services can read from.

---

## Phase 4: `assistant-service`

The assistant is an AI agent: an LLM that decides which tools to call, gets the data from them and then answers the user. It explains predictions, looks up data and makes its own predictions. ML predictions still come only from `predictor-service` and the assistant's own predictions are saved separately so the two can be compared.

> **Note:** this phase comes before the backend so the research part (ML vs AI predictions) is finished first. Until phase 5 the service is called directly (Swagger or a script) and tools that need user data are added in phase 5.

### 4.1 What the assistant does

Answering the user in chat:

| Task                      | Example question                                            | Tools it uses                                 |
| ------------------------- | ----------------------------------------------------------- | --------------------------------------------- |
| Explain the ML prediction | "Why does the model think Arsenal will win?"                | `get_ml_prediction`, `get_ml_explanation`     |
| Look up facts             | "How many goals did Arsenal score in their last 5 matches?" | `get_team_form`, `get_match`, `get_standings` |
| Match preview             | "What do you think about Liverpool - Chelsea?"              | form, injuries, head-to-head, ML prediction   |
| Round overview            | "Which matches this round are the most uncertain?"          | `get_fixtures`, `get_ml_prediction`           |
| Its own prediction        | "What is your prediction for the derby?"                    | data tools, then its own reasoning            |
| How good the models are   | "How accurate has the model been this season?"              | `get_model_performance`                       |
| Personal stats (phase 5)  | "Where am I losing the most points?"                        | `get_my_stats`, `get_crowd_picks`             |

In the background (no user involved):

- Before each round, the assistant makes its own predictions for every match (see 4.5)

### 4.2 LLM models _(TO BE DECIDED)_

| Type                | Candidates                           | Why                                                        |
| ------------------- | ------------------------------------ | ---------------------------------------------------------- |
| Hosted, strong      | Claude Sonnet, GPT, Gemini Pro       | Best tool calling and reasoning, the reference for quality |
| Hosted, small/cheap | Claude Haiku, Gemini Flash, GPT mini | Cheap and fast, good enough for most chat questions        |
| Open model          | Qwen, Llama, Mistral                 | Free, can run locally, shows how far an open model gets    |

- [ ] Pick 2-3 models (at least one hosted and one open model) considering current prices and free tiers when choosing
- [ ] The same evaluation (4.8) is run on all of them, so the result is a table of accuracy, cost and response time per model

### 4.3 Foundation

- [ ] Stateless FastAPI service - it gets the conversation history, the question and (from phase 5) a `user_id` that is already authenticated
- [ ] LLM provider behind an interface, implemented with the providers' SDKs or a library that supports many providers _(TO BE DECIDED)_
- [ ] Agent loop: the LLM asks for a tool --> the service calls it --> the result goes back to the LLM --> repeat until there is an answer
- [ ] Limits: maximum number of tool calls per question, timeout for each tool and the whole answer
- [ ] First version without tools - just put the ML output for the round in the prompt, then move to tool calling
- [ ] System prompt kept in the repo with a version number, so every answer and prediction knows which prompt produced it

### 4.4 Tools

About ten tools, each one small and with a clear description so the LLM knows when to use it, can be subject to change:

| Tool                                  | What it returns                                         | Served by                      |
| ------------------------------------- | ------------------------------------------------------- | ------------------------------ |
| `get_fixtures(round or team)`         | upcoming matches and kick-off times                     | `football` schema              |
| `get_match(match_id)`                 | result and statistics of a played match                 | `football` schema              |
| `get_team_form(team, last_n)`         | last N results, goals scored and conceded               | `football` schema              |
| `get_head_to_head(team_a, team_b)`    | previous matches between two teams                      | `football` schema              |
| `get_standings()`                     | current league table                                    | `football` schema              |
| `get_injuries(team)`                  | injured and suspended players, with `fetched_at`        | `football` schema              |
| `get_lineup(match_id)`                | confirmed lineup (about one hour before kick-off)       | `football` schema              |
| `get_ml_prediction(match_id)`         | W/D/L probabilities and the most likely scores          | `predictor-service`            |
| `get_ml_explanation(match_id)`        | the most important features for that prediction (SHAP)  | `predictor-service`            |
| `get_model_performance()`             | how accurate the ML models have been in live evaluation | `predictor-service`            |
| `get_crowd_picks(match_id)` (phase 5) | the most common user predictions                        | `spring-backend-service` (M2M) |
| `get_my_stats()` (phase 5)            | points and accuracy of the calling user only            | `spring-backend-service` (M2M) |

- [ ] Tools for `football` (read-only database user)
- [ ] Tools for `predictor-service` (HTTP client with a timeout)
- [ ] Unit tests for each tool on a small test database

### 4.5 The assistant's own predictions

- [ ] Scheduled job that runs before each round, at the same times as the ML snapshots (early and after lineups)
- [ ] Two variants, so the research can show whether the AI adds anything to the ML:
    - **AI alone** - the assistant only gets the data from the tools, without the ML prediction
    - **AI + ML (hybrid)** - the assistant also sees the ML prediction and its explanation and can change it
- [ ] Structured output in the same format as the ML predictions - W/D/L probabilities, the most likely score and a short reasoning
- [ ] Low temperature and a fixed prompt version, so the results can be repeated
- [ ] `assistant-service` saves its predictions directly in `prediction_log` with its own `source` (e.g. `ai`, `ai-hybrid`) and the LLM model name and prompt version in `model_version`

### 4.6 How the assistant answers

- [ ] Short answers, with the numbers that came from the tools
- [ ] It always says where something comes from - "the model predicts…", "the data shows…" or "my own prediction is…"
- [ ] If a tool returns no data, it says that there is no data instead of guessing
- [ ] For injuries and lineups it says how fresh the data is (from `fetched_at`)
- [ ] Refuses questions that are out of scope (other leagues, betting advice)
- [ ] Answers in the language of the question
- [ ] Response contains the answer and the list of tools that were called (used for evaluation and debugging)

### 4.7 Explainability _(TO BE DECIDED)_

- [ ] Turn the model's SHAP values into sentences
- [ ] Measure how well the explanations match the features that are actually the most important and how many claims are not backed by the data

### 4.8 Assistant evaluation

- [ ] 50-100 test questions whose correct answers are in the database (e.g. "How many goals did Arsenal score in their last 5 matches?"), kept in the repo as tests
- [ ] Automatic check whether the right tool was called and whether the numbers are correct
- [ ] Quality of explanations - LLM-as-judge, plus checking a sample by hand
- [ ] Run the whole evaluation on every model from 4.2
- [ ] Compare prediction sources on the saved predictions - ML, AI alone, AI + ML, odds and (from phase 5) majority of users. AI predictions only count for matches after the LLM's knowledge cutoff, so it can't already know the result.

### 4.9 Docker and tests

- [ ] `Dockerfile` for `assistant-service` and update `docker-compose.yml`
- [ ] Integration tests `assistant-service` <-> `predictor-service` and `assistant-service` <-> `football`, with a fake LLM that returns fixed tool calls (so tests don't cost anything)

**Phase output:** `assistant-service` running in a container, making its own predictions every round, with an evaluation report and a comparison of LLM models.

---

## Phase 5: `spring-backend-service`

### 5.1 Scoring system

- [ ] Decide on the rules (outcome, exact score, goal difference, extra bonuses) and write them in [`docs/spring-backend-service/`](spring-backend-service/)

### 5.2 DB schema and repositories

- [ ] `app` schema - users, leagues (including the global one), memberships, invitations, rounds, matches, predictions, points, chat history
- [ ] Migrations
- [ ] JPA repositories and tests (`@DataJpaTest`)

### 5.3 Services and unit tests

- [ ] Users, leagues and invitations
- [ ] Predictions (locked before the match starts)
- [ ] Scoring after a round is finished
- [ ] Most common predictions for each round
- [ ] Client for `predictor-service` (ML suggestion for each match) with a timeout and fallback
- [ ] Chat - forwards the question and history to `assistant-service` and saves the conversation
- [ ] M2M endpoints for the assistant's user tools (`get_crowd_picks`, `get_my_stats`)
- [ ] Syncing fixtures and results (`round`, `match`) from `data-collector` through M2M endpoints

### 5.4 REST controllers and tests

- [ ] Endpoints for everything from 5.3
- [ ] Tests (`@WebMvcTest`)
- [ ] OpenAPI / Swagger documentation

### 5.5 Authentication and authorization

- [ ] Two different providers (still need to pick which, e.g. Google OAuth2)
- [ ] JWT - issuing and validating tokens
- [ ] `SecurityFilterChain` configuration and roles
- [ ] M2M tokens for communication between services (`data-collector`, `predictor-service`, `assistant-service` --> `spring-backend-service`)
- [ ] Rate limiting per user on the chat endpoint
- [ ] Security tests

### 5.6 Docker

- [ ] `Dockerfile` for `spring-backend-service`
- [ ] Add to `docker-compose.yml` (`spring-backend-service`, Postgres, `predictor-service`, `assistant-service`, `data-collector`)

### 5.7 Integration tests

- [ ] `spring-backend-service` <-> PostgreSQL
- [ ] `spring-backend-service` <-> `predictor-service`
- [ ] `spring-backend-service` <-> `assistant-service` <-> `predictor-service`
- [ ] Flow: round --> predictions --> scoring

**Phase output:** `spring-backend-service` running in a container that talks to `predictor-service` and `assistant-service` and handles users, leagues, predictions, scoring and chat.

---

## Phase 6: `ui` _(TO BE DETAILED)_

- [ ] Confirm Vite + React + TypeScript
- [ ] Basic screens - login, leagues, round predictions (with the most common predictions, the ML suggestion and the assistant's prediction), leaderboard
- [ ] Chat screen with the assistant
- [ ] Write a more detailed plan when phase 5 is close to being done

---

## Phase 7: Finalization and deployment _(TO BE DETAILED)_

- [ ] Finish `docker-compose.yml` for the whole system
- [ ] E2E tests
- [ ] Deployment of the database, services and `ui`

---

## What gets cut if there isn't enough time

1. Recommender system
2. LLM for extracting features from text (injury news, press conferences)
3. Scaling and distributed deployment
