# Match Predictor Application

This monorepo contains an application for predicting the results of upcoming Premier League fixtures, built on a microservice architecture. Users can join leagues, make predictions and collect points for them. Users can't see each other's predictions until the first match of the round kicks off, which is also the deadline for submitting predictions.

For each fixture in the upcoming round, several ML models suggest the exact score. When making a prediction, a user can also see the most popular predictions of other users, as well as the most popular predictions among users with similar prediction habits. Besides the ML-based and user-based suggestions, there is also an AI assistant that can explain those suggestions, suggest its own predictions and answer questions about upcoming matches, team form and so on.

## Documentation

- [Architecture](docs/architecture.md) - components, data, main flows and the decisions behind them
- [Development plan](docs/development-plan.md) - phases, tasks and progress
