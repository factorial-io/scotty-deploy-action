# Contributing to Scotty Deploy Action

Thank you for your interest in contributing! This document provides guidelines for contributing to the project.

## Reporting Issues

If you encounter a bug or have a feature request:

1. Check if the issue already exists in [GitHub Issues](https://github.com/factorial-io/scotty-deploy-action/issues)
2. If not, create a new issue with:
   - Clear title and description
   - Steps to reproduce (for bugs)
   - Expected vs actual behavior
   - Your environment (GitHub Actions runner, scottyctl version, etc.)
   - Relevant workflow snippets

## Submitting Pull Requests

1. **Fork the repository** and create a branch from `main`
2. **Make your changes**:
   - Follow existing code style
   - Update documentation if needed
   - Add examples if adding new features
3. **Test your changes**:
   - Test with a real Scotty server if possible
   - Verify all examples still work
4. **Commit your changes**:
   - Use clear commit messages
   - Reference issues (e.g., "Fixes #123")
5. **Submit the pull request**:
   - Describe what changed and why
   - Link related issues

## Development Setup

### Prerequisites

- Access to a Scotty server for testing
- Docker installed locally
- GitHub account

### Local Testing

Since this is a GitHub Action, local testing is limited. The best approach:

1. Create a test repository
2. Use your fork/branch in a workflow:
   ```yaml
   uses: your-username/scotty-deploy-action@your-branch
   ```
3. Test all scenarios:
   - Create application
   - Destroy application
   - Multiple services
   - Custom domains
   - Environment variables
   - Error handling

### Testing Checklist

Before submitting a PR, verify:

- [ ] Action creates apps successfully
- [ ] Action destroys apps successfully
- [ ] Multiple services work correctly
- [ ] URLs are extracted properly
- [ ] Error messages are helpful
- [ ] Documentation is updated
- [ ] Examples are still valid

## Code Style

- Use clear, descriptive variable names
- Add comments for complex logic
- Keep bash scripts readable with proper indentation
- Follow existing patterns in action.yml

## Documentation

When adding features:

- Update README.md with new inputs/outputs
- Add usage examples
- Update CHANGELOG.md
- Create example workflows if applicable

## Questions?

Open an issue or reach out to the maintainers.

## License

By contributing, you agree that your contributions will be licensed under the MIT License.
