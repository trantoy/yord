# Contributing to Yord

Thanks for your interest in contributing.

## Getting Started

1. Fork the repository
2. Create a feature branch from `main`
3. Make your changes
4. Submit a pull request

## Development Setup

Prerequisites:

- [Rust](https://rustup.rs/) (stable)
- [Node.js](https://nodejs.org/) (LTS)
- [pnpm](https://pnpm.io/)
- Tauri 2 system dependencies — see [Tauri prerequisites](https://v2.tauri.app/start/prerequisites/)

```sh
git clone https://github.com/YOUR_FORK/yord.git
cd yord
pnpm install
cargo build
```

## Code Style

- **Rust**: follow `cargo fmt` and `cargo clippy`
- **TypeScript/Svelte**: follow project ESLint and Prettier configs
- Write clear commit messages — imperative mood, concise subject line

## Pull Requests

- Keep PRs focused on a single change
- Reference related issues if applicable
- Ensure CI passes before requesting review
- Add or update tests for new functionality

## Reporting Issues

Open an issue with:

- What you expected to happen
- What actually happened
- Steps to reproduce
- OS and version info

## License

By contributing, you agree that your contributions will be licensed under both the MIT and Apache 2.0 licenses (see [LICENSE-MIT](LICENSE-MIT) and [LICENSE-APACHE](LICENSE-APACHE)).
