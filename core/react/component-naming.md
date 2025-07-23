---
id: react/component-naming
title: React Component Naming Conventions
description: Consistent naming patterns for React components and files
tags:
  - react
  - naming
  - conventions
  - components
languages:
  - typescript
  - javascript
frameworks:
  - react
  - next.js
trigger: always
globs: []
source: contexture
---

# React Component Naming Conventions

Establish consistent naming patterns for React components and their files.

## File Naming

- **Components**: Use PascalCase for component files
  - `UserProfile.tsx` (not `userProfile.tsx` or `user-profile.tsx`)
  - `NavigationBar.tsx`
  - `ProductCard.tsx`

- **Hooks**: Use camelCase starting with "use"
  - `useUserData.ts`
  - `useLocalStorage.ts`

- **Utilities**: Use camelCase
  - `formatDate.ts`
  - `apiHelpers.ts`

## Component Naming

```typescript
// Good
export function UserProfile() {
  return <div>Profile</div>
}

// Good - matches file name
export default function UserProfile() {
  return <div>Profile</div>
}

// Avoid - inconsistent with file name
export default function Profile() { // file is UserProfile.tsx
  return <div>Profile</div>
}
```

## Directory Structure

```
src/
  components/
    UserProfile/
      UserProfile.tsx
      UserProfile.test.tsx
      UserProfile.module.css
      index.ts  // re-export
  hooks/
    useUserData.ts
  utils/
    formatDate.ts
```