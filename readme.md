# Admin Development Documentation

This repository includes three core documents that define the conventions and standards for building and maintaining the admin panel in this project.  
Each document targets a separate layer of the system to keep architecture clean, predictable, and scalable.

---

## 1. Admin Backend Development Guide
**File:** `docs/admin-backend-development-guide.md`  
Describes the backend architecture for admin modules, including:
- Resource structure (Controllers, Data Objects, Actions)
- Validation strategies
- Request/Response patterns
- Authorization rules
- Routing standards
- Data transformations and API conventions

This guide ensures all backend modules stay consistent and maintainable.

---

## 2. Admin Frontend Module Guide
**File:** `docs/admin-module-frontend-guide.md`  
Defines standards for React + Inertia + TypeScript admin pages:
- Folder structure for each module
- shadcn/ui component usage
- Zod schemas and strict typing rules
- Hooks & stores architecture
- Table, forms, filters, modal patterns
- Wayfinder routing conventions

This guide enforces a predictable and scalable frontend system.

---

## 3. Testing Guide
**File:** `docs/testing-guide.md`  
Provides rules and patterns for backend & API test coverage:
- Unit test conventions
- Feature test structure
- Database transaction rules
- Mocking external services
- Test data factories & seeds
- CI-friendly testing principles

Ensures all features are covered with consistent, reliable test practices.

---

## Purpose
Together, these three documents establish a unified development standard across backend, frontend, and testing layers—resulting in a clean, maintainable, and future-proof admin ecosystem.
