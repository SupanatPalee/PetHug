# 🔌 API Standardization Guide - PetHug

## 📋 Table of Contents
1. [Introduction](#introduction)
2. [API Design Principles](#api-design-principles)
3. [Base URL Structure](#base-url-structure)
4. [Request/Response Standards](#requestresponse-standards)
5. [Authentication](#authentication)
6. [Error Handling](#error-handling)
7. [Rate Limiting](#rate-limiting)
8. [Security](#security)
9. [Examples](#examples)

## Introduction

This guide defines the standards for all API endpoints in the PetHug platform. Following these standards ensures consistency, maintainability, and ease of use across the entire application.

## API Design Principles

### RESTful Standards
- Use proper HTTP methods
```
GET     - Retrieve data
POST    - Create new resource
PUT     - Update entire resource
PATCH   - Partial update
DELETE  - Remove resource
```

### Naming Conventions
- Use kebab-case for URLs
- Use camelCase for parameters
- Use PascalCase for data models
```
✅ Good:
/api/pet-profiles
?sortBy=createdAt

❌ Bad:
/api/petProfiles
?sort_by=created_at
```

## Base URL Structure

### Environments
```
Production: https://api.pethug.com
Development: https://dev-api.pethug.com
```

### API Version
```
/api/v1/[resource]
```

### Core Endpoints
```
/api/v1/
├── /pets
│   ├── GET    /           # List pets
│   ├── POST   /           # Create pet
│   ├── GET    /:id        # Get pet details
│   ├── PUT    /:id        # Update pet
│   └── DELETE /:id        # Delete pet
│
├── /users
│   ├── GET    /me         # Get current user
│   ├── PUT    /me         # Update profile
│   └── GET    /:id        # Get user profile
│
├── /chats
│   ├── GET    /           # List chats
│   ├── POST   /           # Create chat
│   ├── GET    /:id        # Get messages
│   └── POST   /:id/messages # Send message
│
└── /contracts
    ├── POST   /           # Create contract
    ├── GET    /:id        # Get contract
    └── POST   /:id/sign   # Sign contract
```

## Request/Response Standards

### Request Format

#### Query Parameters
```typescript
interface QueryParams {
  // Pagination
  page?: number;      // Default: 1
  limit?: number;     // Default: 10

  // Sorting
  sortBy?: string;    // Field to sort by
  order?: 'asc' | 'desc'; // Sort order

  // Filtering
  type?: 'DOG' | 'CAT' | 'RABBIT' | 'BIRD';
  gender?: 'MALE' | 'FEMALE';
  province?: string;
}
```

#### Request Body
```typescript
// Example: Create Pet
interface CreatePetRequest {
  name: string;
  type: 'DOG' | 'CAT' | 'RABBIT' | 'BIRD';
  breed: string;
  age: number;
  gender: 'MALE' | 'FEMALE';
  description?: string;
  images: string[];
  vaccines?: string[];
  personality?: string[];
}
```

### Response Format

#### Success Response
```typescript
interface ApiResponse<T> {
  data: T;
  meta?: {
    total: number;
    page: number;
    limit: number;
    hasMore: boolean;
  };
}

// Example
{
  "data": {
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "name": "Buddy",
    "type": "DOG",
    "age": 2
  },
  "meta": {
    "total": 100,
    "page": 1,
    "limit": 10,
    "hasMore": true
  }
}
```

#### Error Response
```typescript
interface ApiError {
  error: {
    code: string;
    message: string;
    details?: Record<string, any>;
  };
}

// Example
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": {
      "name": "Name is required"
    }
  }
}
```

## Authentication

### Headers
```typescript
{
  "Authorization": "Bearer <jwt_token>",
  "Content-Type": "application/json"
}
```

### JWT Structure
```typescript
interface JWTPayload {
  sub: string;       // User ID
  email: string;     // User email
  role: string[];    // User roles
  iat: number;       // Issued at
  exp: number;       // Expiration time
}
```

## Error Handling

### Standard Error Codes
```typescript
enum ErrorCode {
  // 4xx Client Errors
  BAD_REQUEST = 'BAD_REQUEST',           // 400
  UNAUTHORIZED = 'UNAUTHORIZED',          // 401
  FORBIDDEN = 'FORBIDDEN',               // 403
  NOT_FOUND = 'NOT_FOUND',              // 404
  VALIDATION_ERROR = 'VALIDATION_ERROR', // 422
  
  // 5xx Server Errors
  INTERNAL_ERROR = 'INTERNAL_ERROR',     // 500
  SERVICE_UNAVAILABLE = 'SERVICE_UNAVAILABLE' // 503
}
```

### Error Response Example
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": {
      "age": "Age must be a positive number"
    }
  }
}
```

## Rate Limiting

### Headers
```typescript
{
  "X-RateLimit-Limit": "100",
  "X-RateLimit-Remaining": "95",
  "X-RateLimit-Reset": "1625097600"
}
```

### Limits
```typescript
const rateLimits = {
  public: {
    window: '15m',
    max: 100
  },
  authenticated: {
    window: '15m',
    max: 1000
  }
};
```

## Security

### CORS Configuration
```typescript
const corsConfig = {
  origin: [
    'https://pethug.com',
    'https://dev.pethug.com'
  ],
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  maxAge: 86400
};
```

### Security Headers
```typescript
const securityHeaders = {
  'Strict-Transport-Security': 'max-age=31536000; includeSubDomains',
  'X-Frame-Options': 'DENY',
  'X-Content-Type-Options': 'nosniff',
  'X-XSS-Protection': '1; mode=block',
  'Content-Security-Policy': "default-src 'self'"
};
```

## Examples

### Implementation Example (Next.js API Route)

```typescript
// app/api/pets/route.ts
import { NextResponse } from 'next/server';
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';

export async function GET(request: Request) {
  try {
    // Get query parameters
    const { searchParams } = new URL(request.url);
    const page = parseInt(searchParams.get('page') || '1');
    const limit = parseInt(searchParams.get('limit') || '10');
    const type = searchParams.get('type');

    // Initialize Supabase client
    const supabase = createRouteHandlerClient({ cookies() {} });

    // Build query
    let query = supabase
      .from('pets')
      .select('*', { count: 'exact' })
      .order('created_at', { ascending: false });

    // Apply filters
    if (type) {
      query = query.eq('type', type);
    }

    // Apply pagination
    const from = (page - 1) * limit;
    const to = from + limit - 1;
    query = query.range(from, to);

    // Execute query
    const { data, error, count } = await query;

    if (error) throw error;

    // Return formatted response
    return NextResponse.json({
      data,
      meta: {
        total: count,
        page,
        limit,
        hasMore: count > (page * limit)
      }
    });

  } catch (error) {
    return NextResponse.json({
      error: {
        code: 'INTERNAL_ERROR',
        message: 'An unexpected error occurred',
        details: error.message
      }
    }, { status: 500 });
  }
}
```

### Testing Example

```typescript
// __tests__/api/pets.test.ts
import { describe, it, expect } from 'vitest';

describe('GET /api/pets', () => {
  it('should return paginated list of pets', async () => {
    const response = await fetch('/api/pets?page=1&limit=10');
    const json = await response.json();

    expect(response.status).toBe(200);
    expect(json.data).toBeInstanceOf(Array);
    expect(json.meta).toMatchObject({
      total: expect.any(Number),
      page: 1,
      limit: 10,
      hasMore: expect.any(Boolean)
    });
  });

  it('should handle invalid parameters', async () => {
    const response = await fetch('/api/pets?page=invalid');
    const json = await response.json();

    expect(response.status).toBe(400);
    expect(json.error).toMatchObject({
      code: 'BAD_REQUEST',
      message: expect.any(String)
    });
  });
});
```
