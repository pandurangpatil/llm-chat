# Task 3.1: React Application Setup

## Objective
Initialize React 19 TypeScript frontend for LLM chat application.

## Requirements
Set up:
- Vite configuration for React 19 with TypeScript 5
- Material-UI 6 integration with custom theme
- React Router v6 with route configuration
- Project structure: src/components/, src/pages/, src/services/, src/hooks/, src/types/
- Environment configuration for API endpoints
- Zustand store setup for state management
- API client service with axios or fetch
- TypeScript types matching backend data models

## Project Structure
```
src/
├── components/          # Reusable UI components
├── pages/              # Route-level page components
├── services/           # API clients and external services
├── hooks/              # Custom React hooks
├── types/              # TypeScript type definitions
├── stores/             # Zustand state management
├── utils/              # Utility functions
├── styles/             # Global styles and theme
└── assets/             # Static assets (images, icons)
```

## Dependencies
- React 19, React DOM 19
- TypeScript 5, @types/react, @types/react-dom
- Vite, @vitejs/plugin-react
- Material-UI 6 (@mui/material, @mui/icons-material, @emotion/react, @emotion/styled)
- React Router v6
- Zustand for state management
- Axios for API calls
- React Query/TanStack Query for data fetching

## Vite Configuration
- TypeScript support with path aliases
- Environment variable handling
- Hot module replacement
- Build optimization for production
- Proxy configuration for development API calls

## Material-UI Theme
- Custom color palette matching brand
- Typography configuration
- Component style overrides
- Dark/light theme support preparation
- Responsive breakpoints

## Type Definitions
Match backend data models:
```typescript
interface User {
  id: string;
  username: string;
  displayName: string;
  settings: UserSettings;
}

interface Thread {
  id: string;
  title: string;
  createdAt: number;
  updatedAt: number;
  models: Record<string, ThreadModel>;
}

interface Message {
  id: string;
  role: 'user' | 'assistant';
  content: string;
  status: MessageStatus;
  createdAt: number;
}
```

## Deliverables
- Complete React application setup with Vite
- Material-UI integration with custom theme
- Router configuration with basic routes
- API client service setup
- Zustand store configuration
- TypeScript types for all data models
- Development environment ready for component development