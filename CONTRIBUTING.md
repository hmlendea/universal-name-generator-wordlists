# Contributing to Universal Name Generator — Wordlists

This document covers the guidelines and processes for contributing to this project, including how to report issues, suggest enhancements, submit code changes, and follow project standards.

## 📑 Table of Contents

- [Contributing](#-contributing)
  - [Reporting Issues](#reporting-issues)
  - [Suggesting Enhancements](#suggesting-enhancements)
  - [Code Contributions](#code-contributions)
    - [Prerequisites](#prerequisites)
    - [Development Setup](#development-setup)
    - [Making Changes](#making-changes)
    - [Pull Request Guidelines](#pull-request-guidelines)
  - [Code Style](#code-style)
  - [Testing](#testing)
  - [Documentation](#documentation)
- [Security](#-security)
- [License](#-license)

## 🤝 Contributing

### Reporting Issues

- Search existing issues first.
- Use the issue templates if available.
- Provide clear reproduction steps.
- Include environment details.

### Suggesting Enhancements

- Check the [ROADMAP](./ROADMAP.md) and the existing discussions.
- Explain the use case and expected conduct.
- Consider the implementation complexity.

### Code Contributions

#### Prerequisites

- Git
- Text editor for `.lst` and `.xml` files

#### Development Setup

```bash
# Clone the repository
git clone https://github.com/hmlendea/universal-name-generator-wordlists.git
cd universal-name-generator-wordlists

# No dependencies to install; this is an assets-only repository
```

#### Making Changes

1. Fork the repository.
2. Create a feature branch: `git checkout -b feature/your-feature-name`
3. Make your changes.
4. Validate XML schemas: `xmllint --schema GenerationSchemas.xml wordlists/**/*.lst` (if applicable)
5. Commit with clear and descriptive messages.
6. Push to your fork.
7. Open a Pull Request.

#### Pull Request Guidelines

- Target the `master` branch.
- Keep PRs focused and atomic.
- Revise documentation if applicable.
- Include tests for any new functionality.
- Ensure the CI checks pass.

### Code Style

Follow the project's coding standards:
- One entry per line in `.lst` files
- UTF-8 encoding
- No trailing whitespace
- Consistent directory naming (lowercase, hyphen-separated)
- Run `git diff --check` before committing to detect whitespace issues.

### Testing

This is an assets-only repository; there are no automated tests. Validate contributions by:
- Checking `.lst` files for formatting consistency
- Verifying XML schema compliance where applicable
- Ensuring no duplicate entries within a single wordlist

### Documentation

- Revise relevant docs for changes.
- Follow the documentation style guide.
- Preview changes locally if possible.

## 🔒 Security

Report security vulnerabilities per the [Security Policy](./SECURITY.md).

## 📄 License

By contributing, you agree that your contributions will be licensed under the GNU General Public License v3.0.