# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Initial release of Scotty Deploy Action
- Support for all scottyctl parameters
- Multi-line input support for services, custom domains, env vars, and middleware
- Create and destroy actions
- Comprehensive documentation and examples
- Example workflows for common use cases
- Example docker-compose files

### Features
- 🚀 Deploy docker-compose apps to Scotty
- 🔄 PR review app lifecycle (deploy/destroy)
- 🌐 Custom domain mapping
- 🔒 HTTP basic authentication
- ⚙️ Environment variable injection
- ⏱️ TTL with optional auto-destroy
- 📊 Multi-service deployments
- 🔧 Traefik middleware support

## [v1] - 2026-11-08

`v1` is a moving major-version tag; this entry lists what it picked up since 1.0.0.

### Changed
- Default `scottyctl-version` is now `main` (was `next`), so the action pulls `ghcr.io/factorial-io/scotty:main`. Set `scottyctl-version: next` to keep the old image.

### Fixed
- Use the correct scotty image and executable path
- Stream scotty command output in real time

## [1.0.0] - 2025-11-17

Initial release.
