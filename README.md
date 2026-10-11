# Employee Attendance & Analytics API

FastAPI + PyMongo implementation of the HROne Employee Attendance & Analytics API contract v2.0. The application code is in `app/main.py` as requested.

## Run

Use Python 3.11+ and MongoDB 6.0+. Set `MONGO_URI` and `MONGO_DB` in your shell before starting the app (real environment variables take precedence over values in a local `.env` file). Then run:

```bash
python -m venv .venv
# Linux/macOS: source .venv/bin/activate
# Windows PowerShell: .venv\Scripts\Activate.ps1
pip install -r requirements.txt
uvicorn app.main:app --port 8000
```

The app creates its indexes idempotently during startup. `GET /health` checks MongoDB connectivity. Interactive API documentation is available at `/docs`.

## Notes

- Calendar dates and shift times use the assignment's IST rules; API instants use epoch milliseconds and MongoDB stores BSON datetimes.
- Attendance analytics are built with MongoDB aggregation pipelines. The department trend fills its date range in MongoDB and computes its seven-row moving average with a window stage.
- Do not commit `.env`, credentials, virtual environments, or data dumps. The evaluator seeds its own database.
