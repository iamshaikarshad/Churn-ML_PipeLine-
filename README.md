# ML_PipeLine

This repository implements a Django REST API that exposes machine learning models
as production-ready endpoints. It contains an API to register and manage ML
algorithms, serve single and batch predictions, and run A/B tests to compare
algorithms.

This README describes the architecture, the main components, how data flows
through the system, how to run and register models, and known issues and
recommendations.

**Repository structure (important parts)**

- `backend/server/` — Django project root.
	- `server/settings.py` — Django settings.
	- `server/urls.py`, `server/wsgi.py` — app wiring and registry initialization.
	- `apps/endpoints/` — main application exposing REST endpoints, models and
		views.
		- `models.py` — `Endpoint`, `MLAlgorithm`, `MLAlgorithmStatus`, `MLRequest`, `ABTest`.
		- `serializers.py` — DRF serializers for models and uploaded files.
		- `views.py` — API views and logic for prediction, batch predictions and AB testing.
		- `urls.py` — routes under `/api/v1/`.
	- `apps/ml/` — ML registry and model wrapper classes.
		- `registry.py` — `MLRegistry` maps DB `MLAlgorithm` ids -> algorithm objects.
		- `classifier/` — wrappers for specific trained models that implement a
			common `compute_prediction` method.

- `research/` — model artifact files (joblib files) used by the classifier wrappers.
- `requirements.txt` — Python dependencies.

**High-level purpose**

The project provides a way to:
- Register ML algorithms (stored as `MLAlgorithm` model rows).
- Load algorithm objects into an in-memory registry keyed by the DB id.
- Serve single predictions at `/api/v1/<endpoint_name>/predict`.
- Accept CSV uploads for batch predictions (`/api/v1/batch-predict`).
- Run and manage A/B tests comparing two algorithms and promote winners to
	production.

Architecture and data flow
--------------------------

1. Registry and algorithm registration

	 - `apps.ml.registry.MLRegistry` keeps an in-memory dict `endpoints` where
		 keys are `MLAlgorithm` database ids and values are algorithm objects (the
		 wrapper instances from `apps.ml.classifier`).
	 - When `add_algorithm` is called, the method ensures the `Endpoint` and
		 `MLAlgorithm` DB rows exist (creating them if necessary), optionally
		 creates an initial `MLAlgorithmStatus`, and stores the provided
		 algorithm object at `self.endpoints[db_id]`.

	 - The registry is expected to be created and populated in startup code
		 (for example, `server.wsgi` or an `AppConfig.ready()` method). Views that
		 need an algorithm reference `import registry` from `server.wsgi` and look
		 up the algorithm object with `registry.endpoints[mlalgorithm_id]`.

2. Serving single predictions (`PredictView`)

	 - Route: `/api/v1/<endpoint_name>/predict` (POST)
	 - Query parameters:
		 - `status` (optional): which MLAlgorithmStatus to use (default: `testing`).
		 - `version` (optional): algorithm `version` string to narrow selection.
	 - The view finds MLAlgorithm rows for `parent_endpoint__name == endpoint_name`
		 and matching active status (e.g., `testing`, `production`, or
		 `ab_testing`). If multiple match and status != `ab_testing`, the view
		 returns an error asking for an explicit `version`.
	 - For `ab_testing` status the view randomly chooses one of the two
		 algorithms (50/50) and calls `compute_prediction(request.data)` on the
		 algorithm object retrieved from `registry.endpoints[algorithm_id]`.
	 - The view logs/saves the incoming request and prediction to the `MLRequest`
		 model and returns the prediction JSON (including a `request_id`).

3. Batch predictions (`BatchPredictionViewSet`)

	 - Route: `/api/v1/batch-predict` (registered as a viewset endpoint)
	 - Accepts multipart form upload with `file` field (CSV). The view loads
		 the CSV into `pandas.DataFrame`, prepares a feature matrix `X` by
		 dropping hard-coded columns (project-specific), iterates rows, calls
		 `compute_prediction` per row, persists `MLRequest` objects for each
		 prediction and returns an array of results.

4. A/B test lifecycle

	 - `ABTest` model stores two `MLAlgorithm` references, `created_at`, and later
		 `ended_at` and `summary`.
	 - Creating an `ABTest` sets both algorithms' statuses to `ab_testing` and
		 sets `active=True` for those statuses, deactivating previous statuses.
	 - Stopping an AB test (`StopABTestView`) compares `MLRequest` rows created
		 within the test window and computes per-algorithm accuracy by comparing
		 `response` to `feedback`. The better algorithm is promoted to `production`
		 and the other set to `testing`. The ABTest row is updated with a summary.

Models (summary)
----------------

- `Endpoint`: name, owner, created_at.
- `MLAlgorithm`: name, description, code (string), version, owner,
	parent_endpoint (FK to Endpoint).
- `MLAlgorithmStatus`: status label (testing/staging/production/ab_testing),
	active bool, created_by, created_at, FK parent_mlalgorithm.
- `MLRequest`: input_data (string), full_response (string), response (string),
	feedback (string, optional), created_at, parent_mlalgorithm (FK).
- `ABTest`: title, created_by, created_at, ended_at, summary, and two FKs to
	`MLAlgorithm` rows.

Important implementation notes & gotchas
---------------------------------------

- Registry initialization: The views import `registry` from `server.wsgi` and
	expect it to contain algorithm objects mapped to DB ids. If `registry` is
	not populated at runtime (or populated before migrations/DB exists) lookups
	will fail with KeyError or similar.

- Data serialization when saving to DB: Some views save Python objects (dicts,
	DataFrames) directly to `CharField` columns (e.g., `full_response` or
	`input_data`) without converting them to JSON strings. This can cause errors
	or unexpected DB values. Convert to JSON via `json.dumps(...)` before
	saving, and for DataFrames use `df.to_json()`.

- `fillna` usage in classifier wrappers: In `apps/ml/classifier/*` the code
	calls `input_data.fillna(self.values_fill_missing)` but does not assign the
	returned DataFrame back. `fillna` returns a new DataFrame unless
	`inplace=True` is passed. This means missing-value filling is ineffective
	as written. Fix by using `input_data = input_data.fillna(self.values_fill_missing)`
	or `input_data.fillna(self.values_fill_missing, inplace=True)`.

- Missing imports and exception classes: `APIException` is used in
	`views.py` but not imported; add `from rest_framework.exceptions import APIException`.

- Division by zero risk in AB test stop: When calculating accuracy, if there
	are zero `MLRequest` rows for an algorithm in the AB test window, division
	by zero will occur. Add guards and handle the 'no samples' case.

- Hard-coded dataset columns in batch prediction: `BatchPredictionViewSet`
	drops a long list of specific columns from the incoming CSV. This is
	tightly coupled to a particular dataset and brittle. Consider adding a
	configurable mapping or column selector.

How to run (development)
------------------------

1. Create and activate a Python virtual environment (Windows PowerShell example):

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

2. Apply migrations and create a superuser (optional):

```powershell
cd backend\server
python manage.py migrate
python manage.py createsuperuser
```

3. Run the development server:

```powershell
python manage.py runserver
```

4. API base: `http://127.0.0.1:8000/api/v1/` — use the DRF browsable API or
	 send JSON requests with tools like `curl` or Postman.

How to register model objects into the registry
-----------------------------------------------

You must create algorithm wrapper instances (e.g., `RandomForestClassifier()`)
and register them with `MLRegistry.add_algorithm(...)`. A common pattern is to
do that at startup in `server/wsgi.py` or in an `AppConfig.ready()` method.

Example snippet (conceptual, put in `server/wsgi.py` or AppConfig.ready):

```python
from apps.ml.registry import MLRegistry
from apps.ml.classifier.random_forest import RandomForestClassifier

registry = MLRegistry()
rf = RandomForestClassifier()
# The parameters below must match desired DB values; if the MLAlgorithm row
# exists use the same name/version/owner so the DB lookup finds the row.
registry.add_algorithm(
		endpoint_name="classifier",
		algorithm_object=rf,
		algorithm_name="random_forest",
		algorithm_status="production",
		algorithm_version="1.0.2",
		owner="your_name",
		algorithm_description="Random forest model",
		algorithm_code="random_forest.py"
)
```

Notes:
- `add_algorithm` will create `Endpoint` and `MLAlgorithm` rows when they
	do not exist and create an initial `MLAlgorithmStatus` if the algorithm row
	was just created.
- If a DB row exists and you want to map an already-present DB id to an
	algorithm object, ensure that the combination of `name`, `description`,
	`version`, and `owner` matches the existing DB row so `get_or_create` finds
	it.

API usage examples
------------------

- Single prediction (POST JSON) to endpoint `classifier`:

```bash
curl -X POST "http://127.0.0.1:8000/api/v1/classifier/predict" \
	-H "Content-Type: application/json" \
	-d '{"feature_1": 1, "feature_2": 0, ... }'
```

- To request only `production` models add `?status=production`.
- To call a specific `version` add `?version=1.0.2`.

Known issues and recommended fixes
----------------------------------

1. Add missing import for `APIException` in `apps/endpoints/views.py`:

```python
from rest_framework.exceptions import APIException
```

2. Ensure `fillna` results are assigned in classifier wrappers:

```python
input_data = input_data.fillna(self.values_fill_missing)
```

3. Before saving `full_response` or `input_data` to DB, serialize them to
	 strings. Example:

```python
import json
MLRequest.objects.create(
		input_data=json.dumps(request.data),
		full_response=json.dumps(prediction),
		response=prediction.get('label',''),
		parent_mlalgorithm=alg
)
```

4. Guard against division by zero when computing AB test accuracy.

5. Replace hard-coded artifact paths `"../../research/"` with a configurable
	 project setting (e.g., `settings.MODEL_ARTIFACTS_PATH`) and resolve absolute
	 paths at runtime.

Next steps
--------------------------

- Add unit tests for `PredictView`, `MLRegistry`, and `BatchPredictionViewSet`.
- Improve batch prediction performance by using vectorized predictions
	(sending the full DataFrame to the model instead of iterating row-by-row).
- Add logging and structured error handling to improve observability.
- Move model artifact paths and other environment-specific settings to
	`server/settings.py` and/or environment variables.


---

Files referenced in this README: use backticks, for example `apps/endpoints/views.py`,
`apps/ml/registry.py`, and `research/*.joblib`.

License: see `LICENSE` in the repository root.
