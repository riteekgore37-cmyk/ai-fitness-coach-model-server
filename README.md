# AI Fitness Coach — Model Server

A Flask microservice that serves the machine learning models behind the
AI Fitness Coach platform's personalization features: workout plan
generation and nutrition/meal planning based on a user's profile.

This service is intentionally decoupled from the main backend API — it
only knows how to take structured input and return a plan, so the
models can be retrained, swapped, or scaled independently of the rest
of the system.

---

## Tech Stack

- **Framework:** Flask
- **ML/Data:** scikit-learn, pandas, NumPy
- **Serving:** Gunicorn (production WSGI server)
- **Deployment:** Docker, configured for Hugging Face Spaces

---

## What It Does

### `POST /fitness` — Personalized Workout Plans

Given a user's training environment, experience level, goal, gender,
age, prior feedback, current working weight, and available equipment,
returns a **30-day workout plan**.

Under the hood this combines two models:

1. **A trained plan classifier** picks a body-part split template based
   on the user's level, goal, and gender (e.g. which muscle groups to
   train on which days across a rotation).
2. **A KMeans clustering model**, fit on an exercise dataset grouped by
   level/goal/body-part, is used to select a specific exercise for each
   day — filtering out exercises that need unavailable equipment when
   the user is training at home.

On top of the model output, a set of **rule-based adjustment functions**
personalizes each exercise's working weight, rep count, and duration
using age, gender, experience level, and whether the user gave positive
or negative feedback on their last session — so two users with the same
cluster assignment still get different loads.

### `POST /nutrition` — Weekly Meal Plans

Given a daily calorie target, splits it across breakfast (30%), lunch
(40%), dinner (30%), and a snack, builds a nutrient profile for each
meal slot (calories, fat, protein, carbs, fiber, sodium, etc.), and uses
a trained model to match each slot against a real meals dataset —
returning a **7-day meal plan** built from actual recipes rather than
generic macros.

---

## Project Structure

```
ai-fitness-coach-model-server/
├── server.py                      # Flask app, route handlers
├── models/
│   ├── fitness_model.py           # Workout plan generation (KMeans + classifier + rules)
│   └── nutrition_model.py         # Meal plan generation
├── resources/
│   ├── models/                    # Trained model artifacts (.pkl)
│   └── datasets/                  # Supporting datasets (e.g. meals.json)
├── Dockerfile
├── requirements.txt
└── runtime.txt
```

---

## Getting Started

### Prerequisites
- Python 3.10+

### Setup

```bash
# Install dependencies
pip install -r requirements.txt

# Run the server (dev)
python server.py

# Run with Gunicorn (production-style)
gunicorn -b 0.0.0.0:7860 server:app
```

The service listens on port `7860` by default (configurable via the
`PORT` env variable) and exposes a health check at `/`.

### Docker

```bash
docker build -t ai-fitness-coach-model-server .
docker run -p 7860:7860 ai-fitness-coach-model-server
```

---

## Example Request

```bash
curl -X POST http://localhost:7860/fitness \
  -H "Content-Type: application/json" \
  -d '{
    "home_or_gym": 1,
    "level": "Beginner",
    "goal": "Lose Weight",
    "gender": "Male",
    "age": 25,
    "feedback": true,
    "old_weight": 20,
    "equipments": []
  }'
```

---

## Part of a Larger System

This model server is one of three services in the AI Fitness Coach project:

- [**Backend API**](../ai-fitness-coach-backend) — core application
  logic, auth, and data layer; calls this service to generate plans for users
- **Model Server** (this repo) — ML inference for workout and nutrition plans
- [**Admin Panel**](../ai-fitness-coach-admin-panel) — Next.js dashboard
  for managing platform content

The mobile app (Android) that surfaces these plans to end users was
built by a teammate and isn't part of this repo.
