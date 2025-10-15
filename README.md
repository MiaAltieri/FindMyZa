## 🧑‍💻 Developer Notes

### 🚀 Getting Started

#### 1️⃣ Install prerequisites

Make sure you have:

* **Python** ≥ 3.10
* **Poetry** (dependency + environment manager)
* **Git**

##### 📦 Install Poetry

```bash
curl -sSL https://install.python-poetry.org | python3 -
```

Add Poetry to your PATH if needed.
Verify installation:

```bash
poetry --version
```

---

#### 2️⃣ Clone the repository

```bash
git clone https://github.com/<your-username>/FindMyZa.git
cd FindMyZa
```

---

#### 3️⃣ Create and activate the virtual environment

```bash
poetry install
poetry shell
```

This installs all dependencies and development tools (`black`, `ruff`, `isort`, `mypy`, `pytest`, etc.).

---

#### 4️⃣ Install pre-commit hooks

```bash
pre-commit install
```

---

### 🧪 Running Checks and Tests

| Purpose                  | Command                         | Description                            |
| ------------------------ | ------------------------------- | -------------------------------------- |
| **Lint + format**        | `poetry run ruff check .`       | Runs Ruff lint checks                  |
| **Auto-fix lint errors** | `poetry run ruff check . --fix` | Applies Ruff’s auto-fixes              |
| **Format code**          | `poetry run black .`            | Runs Black formatter                   |
| **Sort imports**         | `poetry run isort .`            | Sorts imports automatically            |
| **Type-check**           | `poetry run mypy src/`          | Validates type hints                   |
| **Run tests**            | `poetry run pytest`             | Runs all unit/integration tests        |
| **Run everything**       | `poetry run tox`                | Runs all test/lint/type checks via tox |

---

### 🧩 Before Committing Code

Always run:

```bash
poetry run pre-commit run --all-files
poetry run tox
```

These ensure:

* Code is formatted (Black + isort)
* Lint checks pass (Ruff)
* Type checks pass (Mypy)
* Tests pass (Pytest)
* Everything matches CI standards

---

### 🔄 Updating Dependencies

To add a new dependency:

```bash
poetry add <package-name>
```

For dev-only tools:

```bash
poetry add --group dev <package-name>
```

To update everything:

```bash
poetry update
```

---

### 🤝 Contributing Guidelines

* Write small, focused commits
* Add/modify tests for new features
* Keep imports sorted + code formatted
* Don’t commit directly to `main` — open a pull request
