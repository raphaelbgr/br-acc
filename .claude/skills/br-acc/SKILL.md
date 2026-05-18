```markdown
# br-acc Development Patterns

> Auto-generated skill from repository analysis

## Overview
This skill provides guidance on contributing to the `br-acc` Python codebase. It covers coding conventions, commit patterns, file organization, and testing practices derived from repository analysis. The repository does not use a framework and follows a set of clear, conventional patterns for maintainable development.

## Coding Conventions

### File Naming
- Use **camelCase** for file names.
  - Example: `accountManager.py`, `userProfile.py`

### Import Style
- Use **relative imports** within the package.
  - Example:
    ```python
    from .utils import calculateBalance
    from .models import Account
    ```

### Export Style
- Use **named exports** (explicitly listing exported classes/functions).
  - Example:
    ```python
    __all__ = ['Account', 'calculateBalance']
    ```

### Commit Patterns
- Follow **Conventional Commits** with prefixes like `fix` and `docs`.
- Commit messages are concise (average 66 characters).
  - Example:
    ```
    fix: correct balance calculation in accountManager
    docs: update usage instructions in README
    ```

## Workflows

### Code Contribution
**Trigger:** When adding or updating code
**Command:** `/contribute`

1. Create a new branch for your feature or fix.
2. Write code following camelCase file naming and relative imports.
3. Use named exports for modules.
4. Write or update tests in files matching `*.test.*`.
5. Commit using conventional commit messages (e.g., `fix: ...`, `docs: ...`).
6. Open a pull request for review.

### Testing
**Trigger:** Before merging or submitting code
**Command:** `/test`

1. Locate or create test files using the `*.test.*` pattern.
2. Run tests using your preferred Python test runner (e.g., `pytest`, `unittest`).
3. Ensure all tests pass before submitting your code.

## Testing Patterns

- Test files are named using the pattern `*.test.*` (e.g., `accountManager.test.py`).
- The specific test framework is not enforced; use standard Python testing tools.
- Place tests alongside the modules they test or in a dedicated test directory.

**Example:**
```python
# accountManager.test.py

from .accountManager import calculateBalance

def test_calculateBalance():
    assert calculateBalance([100, -50]) == 50
```

## Commands
| Command      | Purpose                                 |
|--------------|-----------------------------------------|
| /contribute  | Start a new code contribution workflow  |
| /test        | Run tests before submitting code        |
```
