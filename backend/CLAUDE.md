# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working in the `backend/` directory.

## Commands

```bash
python -m venv venv && source venv/bin/activate   # create and activate venv
pip install -r requirements.txt                    # install dependencies
uvicorn app.main:app --reload                      # start dev server
pytest                                             # run all tests
pytest tests/test_foo.py::test_bar                 # run a single test
```

## Folder Structure

```
app/
  routes/      # FastAPI routers, one per resource
  services/    # business logic
  models/      # data models
  schemas/     # Pydantic request/response models
tests/         # mirrors app/ structure
```

## Naming Conventions

Use `snake_case` for modules, functions, and variables; `PascalCase` for classes and Pydantic models.

---

## Style

Use type hints on all function signatures. Follow PEP 8.

---

## Structure

Organize routes as FastAPI `APIRouter` instances, one per resource. Keep business logic in `services/`, not in route handlers — route functions should validate input (via Pydantic), call a service, and return a response. Use Pydantic `BaseModel` for all request/response schemas in `schemas/`.

---

## Error Handling

Raise `HTTPException` for expected errors in route handlers. For domain errors bubbling up from services, catch them in the route and translate to `HTTPException` with an appropriate status code. Use FastAPI's exception handler hooks (`@app.exception_handler`) for global error shapes.

---

## Testing

Use `pytest` with `httpx.AsyncClient` and FastAPI's `TestClient` for route tests. Mirror the `app/` structure in `tests/`. Test service functions directly with mocked dependencies.
