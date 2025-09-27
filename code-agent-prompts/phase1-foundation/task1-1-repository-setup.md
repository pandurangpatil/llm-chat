# Task 1.1: Repository Setup and Project Scaffolding

## Objective
Create a Node.js 22 TypeScript backend project structure for an LLM chat application.

## Requirements
Set up the initial project with:
- Package.json with Node.js 22 and TypeScript 5 dependencies
- TypeScript configuration for strict mode and ES2022 target
- ESLint and Prettier configurations
- Basic Express server setup with modular architecture
- Environment configuration using dotenv
- Docker setup with multi-stage build for production
- GitHub Actions CI/CD workflow skeleton
- .gitignore for Node.js projects
- Project structure: src/, src/services/, src/routes/, src/middleware/, src/utils/, src/types/

## Dependencies to Include
- express, cors, helmet, compression
- typescript, @types/node, @types/express
- eslint, prettier, @typescript-eslint/parser
- dotenv, nodemon, concurrently
- jest, supertest, @types/jest

## Configuration Requirements
- TypeScript: strict mode, ES2022 target, outDir: dist
- ESLint: TypeScript rules, no-unused-vars, consistent-return
- Prettier: 2-space indentation, single quotes, trailing commas
- Docker: Multi-stage build (build -> production)
- GitHub Actions: Basic workflow with lint, test, build stages

## Deliverables
- Complete project structure
- Working Express server with basic health endpoint
- All configuration files properly set up
- Docker container that builds and runs successfully
- GitHub Actions workflow file ready for CI/CD