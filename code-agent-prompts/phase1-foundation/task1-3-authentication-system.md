# Task 1.3: Authentication System

## Objective
Build JWT-based authentication system for a Node.js/Express backend.

## Requirements
Implement:
- User authentication service with bcrypt password hashing
- JWT token generation and validation middleware
- httpOnly cookie configuration with secure flags
- Login and logout endpoints
- Session management with token refresh
- User creation endpoint (admin-only, no self-registration)
- Authentication middleware for protected routes
- Rate limiting for authentication endpoints
- Input validation using Zod for authentication requests

## Authentication Flow
1. User submits login credentials
2. Server validates credentials against database
3. JWT token generated and sent as httpOnly cookie
4. Subsequent requests validated via middleware
5. Token refresh mechanism for long sessions
6. Logout clears cookies and invalidates tokens

## Security Requirements
- bcrypt for password hashing (min 12 rounds)
- JWT with HS256 algorithm and rotating secrets
- httpOnly cookies with secure, sameSite flags
- Rate limiting: 5 attempts per 15 minutes per IP
- Input validation and sanitization
- CSRF protection considerations

## API Endpoints
- POST /api/login - User authentication
- POST /api/logout - Session termination
- POST /api/admin/users - Create new user (admin only)
- GET /api/auth/verify - Token validation

## Deliverables
- Authentication service with password hashing
- JWT middleware for route protection
- Login/logout endpoints with proper error handling
- Rate limiting middleware
- Input validation schemas
- Session management utilities
- Admin user creation functionality