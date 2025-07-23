---
id: security/input-validation
title: Input Validation and Sanitization
description: Validate and sanitize all user inputs to prevent security vulnerabilities
tags:
  - security
  - validation
  - sanitization
  - xss
  - injection
languages:
  - typescript
  - javascript
  - python
  - java
frameworks: []
trigger: always
globs: []
source: contexture
---

# Input Validation and Sanitization

Always validate and sanitize user inputs to prevent security vulnerabilities.

## Validation Rules

1. **Never trust user input** - validate everything
2. **Use allow-lists** rather than deny-lists
3. **Validate on both client and server** side
4. **Sanitize before storage** and display

## TypeScript/JavaScript Example

```typescript
import validator from 'validator'
import DOMPurify from 'dompurify'

function validateEmail(email: string): boolean {
  return validator.isEmail(email) && email.length <= 254
}

function sanitizeHtml(html: string): string {
  return DOMPurify.sanitize(html)
}

function validateAndSanitizeUser(input: any) {
  const user = {
    email: typeof input.email === 'string' ? input.email.trim() : '',
    name: typeof input.name === 'string' ? input.name.trim() : '',
    bio: typeof input.bio === 'string' ? sanitizeHtml(input.bio) : ''
  }

  if (!validateEmail(user.email)) {
    throw new Error('Invalid email format')
  }

  if (user.name.length < 1 || user.name.length > 100) {
    throw new Error('Name must be 1-100 characters')
  }

  return user
}
```

## SQL Injection Prevention

```typescript
// Good - parameterized query
const user = await db.query(
  'SELECT * FROM users WHERE email = $1',
  [email]
)

// Bad - string concatenation
const user = await db.query(
  `SELECT * FROM users WHERE email = '${email}'`
)
```

## Common Validation Libraries

- **Zod** (TypeScript-first)
- **Joi** (JavaScript)
- **Yup** (JavaScript)
- **class-validator** (TypeScript decorators)