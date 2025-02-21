# WhileSmart commit message guidelines

This document outlines the commit message conventions for WhileSmart and projects that choose to adopt these guidelines. Following these conventions ensures a consistent and understandable commit history, facilitating collaboration, automating releases, and improving overall code maintainability.

## Commit message format

Commit messages should adhere to the following structure:

```
<type>(<scope>): <subject>

<body (optional)>

<footer (optional)>
```

### Type

The `<type>` indicates the kind of change being committed.  Use one of the following types:

*   **feat:** A new feature.
*   **fix:** A bug fix.
*   **docs:** Documentation changes.
*   **style:** Changes that do not affect the meaning of the code (e.g., formatting, whitespace).
*   **refactor:** A code change that neither adds nor removes functionality but improves the code's structure.
*   **test:** Adding or modifying tests.
*   **chore:** Maintenance tasks, dependency updates, etc.
*   **build:** Changes related to the build system.
*   **ci:** Changes related to continuous integration.
*   **enh:** Enhancement.  A general improvement or addition to existing functionality.
*   **enhance:** More descriptive form of `enh`.  Use if needed for clarity.
*   **tweak:** A small adjustment or modification.
*   **imp:** Improvement. Similar to `enh` but can be more specific.
*   **improve:** More descriptive form of `imp`.  Use if needed for clarity.

### Scope (Optional)

The `<scope>` is optional and provides context about which part of the codebase the change affects.  Enclose the scope in parentheses.  Examples:

*   `(auth)`: Changes related to authentication.
*   `(ui)`: Changes related to the user interface.
*   `(database)`: Changes related to the database.

### Subject

The `<subject>` is a brief, concise description of the change.  Use sentence case (start with a capital letter, generally lowercase the rest) and keep it under 72 characters.  Avoid ending the subject with a period.  Examples:

*   Implement user login
*   Fix bug in data validation
*   Update documentation for API endpoints

### Body (Optional)

The `<body (optional)>` provides a more detailed explanation of the change.  Use complete sentences and paragraphs.  Explain the *why* of the change, not just the *what*.

### Footer (Optional)

The `<footer (optional)>` can be used for things like:

*   Referencing related issues (e.g., `Closes #123`).
*   Breaking change descriptions (if applicable).

## Examples

*   **feat(user-authentication):** Implement user registration
*   **fix(data-processing):** Fix bug in CSV export
*   **docs:** Update API documentation
*   **chore(dependencies):** Upgrade dependencies

## Why these guidelines?

Following these guidelines helps us:

*   **Create a consistent commit history:** Makes it easier to understand the project's evolution.
*   **Automate releases:** Tools can use commit messages to generate changelogs automatically.
*   **Improve collaboration:** Clear commit messages help team members understand the context of changes.

## Further reading

*   [Conventional Commits](https://www.conventionalcommits.org/v1.0.0/) (This project's guidelines are based conventional commits with some extensions.)
