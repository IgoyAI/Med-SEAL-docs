# Contributing

Thank you for your interest in contributing to Med-SEAL Suite! This guide covers the development workflow and conventions.

## Getting Started

1. **Fork** the repository on GitHub
2. **Clone** your fork locally
3. Create a **feature branch** from `main`
4. Make your changes
5. **Test** locally with Docker Compose
6. Submit a **pull request**

## Development Setup

```bash
# Clone
git clone https://github.com/YOUR_USERNAME/Med-SEAL-Suite.git
cd Med-SEAL-Suite

# Start the full stack
docker compose up -d

# For app development, install dependencies
cd apps/patient-portal-native && npm install
cd apps/ai-service && npm install
cd apps/ai-frontend && npm install
```

## Project Structure

```
Med-SEAL-Suite/
├── apps/
│   ├── ai-service/          # AI backend (Node/TS)
│   ├── ai-frontend/         # Admin dashboard (Vite + React)
│   ├── patient-portal/      # Web portal (Next.js)
│   └── patient-portal-native/  # Mobile app (Expo)
├── medplum/                 # Medplum config
├── openemr/                 # OpenEMR customisations
├── orthanc/                 # Orthanc config
├── ohif/                    # OHIF Viewer config
├── scripts/                 # Utilities and data scripts
├── docs/                    # Internal design docs
└── docker-compose.yml       # Stack deployment
```

## Code Style

- **TypeScript**  - strict mode, no `any` types where avoidable
- **React**  - functional components with hooks
- **Naming**  - camelCase for variables/functions, PascalCase for components/types
- **Formatting**  - use Prettier defaults (or project `.prettierrc` if present)

## Commit Messages

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
feat(patient-portal): add medication adherence chart
fix(ai-service): handle empty FHIR bundle response
docs(readme): update access point table
chore(docker): bump medplum to latest
```

## Pull Request Guidelines

1. **One feature per PR**  - keep changes focused
2. **Description**  - explain what and why
3. **Test**  - verify the full stack runs with `docker compose up -d`
4. **Screenshots**  - include for UI changes
5. **Review**  - request review from a maintainer

## Documentation

Documentation lives in the `Med-SEAL-docs` repository, built with Sphinx + RTD theme. To contribute to docs:

```bash
cd Med-SEAL-docs/docs
pip install -r requirements.txt
make html
# Open _build/html/index.html
```

## Reporting Issues

- Use **GitHub Issues** with clear titles
- Include: steps to reproduce, expected behaviour, actual behaviour
- Tag with appropriate labels (`bug`, `enhancement`, `documentation`)

## License

This project is licensed under the Eclipse Public License 2.0  - see `LICENSE` for details.
