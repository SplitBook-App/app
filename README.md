# Splitbook

A free app that keeps all your fitness personal records in one place.

Splitbook collects completed workouts from Google Health (Pixel Watch and Fitbit) and shows your all-time bests for every workout type you do. It never mixes numbers from different workouts in a way that suggests they came from one workout.

> **Status:** planning and verification. The app has not been built yet. See the [spec](docs/SPEC.md) for the full design.

## What it does

- **One tab per workout type** you've actually recorded: running, cycling, pickleball, swimming and so on. Types you don't want to see can be hidden without losing their data.
- **Whole-workout records** such as longest distance, longest duration and fastest average pace.
- **Best efforts on clean splits** (0–5K, 0–10K and so on), each with the heart rate from that same split.
- **Most efficient efforts**, scored by distance per heartbeat adjusted for steadiness. The formula is public in the spec, and suggestions are welcome.
- **Heart rate zones**: Google Health's defaults, custom zones, or 5 zones calculated from max HR.
- **Time windows**: all-time, this year and last 90 days.
- **Progress graphs**, yearly and lifetime totals, and achievements.
- **GPS glitch flags**, with the choice to exclude a bad workout.
- **Push notifications** when you set a new record.
- Metric and imperial units, with split sets for each.

## Tech stack

| Part | Technology |
|---|---|
| App | React Native with Expo (Android first; iOS and web later) |
| Backend | Python + FastAPI on Google Cloud Run ([SplitBook-App/backend](https://github.com/SplitBook-App/backend)) |
| Data source | Google Health API |
| Database | Neon Postgres, with raw workout streams in Google Cloud Storage |
| Notifications | Expo push notifications |

The app never holds secrets. Google sign-in tokens and all health data are handled by the backend.

## Repository layout

```
app/
├── docs/
│   └── SPEC.md          # Full product spec: every decision and formula
├── .env.example         # Template for local environment variables (no real values)
├── CONTRIBUTING.md      # How to work on this repo
└── README.md
```

The Expo project will be added in a later issue.

## Getting started

Setup instructions will be added once the Expo project is created. You'll need:

- Node.js (LTS) and npm
- An Android phone or emulator
- A Google account with a Pixel Watch or Fitbit, for real data

To prepare your local environment, copy `.env.example` to `.env` and fill in the values you're given. Never commit `.env`.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Ideas for better formulas go in [Discussions](https://github.com/SplitBook-App/app/discussions).

## License

To be decided.