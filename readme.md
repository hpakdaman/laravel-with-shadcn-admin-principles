# Laravel + shadcn Admin Principles

A compact reference for building scalable admin modules with
**Laravel**, **React 19**, **TypeScript**, **Inertia 2**, and
**shadcn/ui**.\
Includes a consistent architecture, strict typing, modular structure,
reusable UI components, and Wayfinder-based routing.

## Features

-   React 19 + TypeScript admin structure\
-   shadcn/ui component standards\
-   Zod schemas for type safety\
-   Modular page folders (components, hooks, stores)\
-   Wayfinder routing (no hardcoded URLs)\
-   Reusable dialogs, forms, tables, filters

## Module Structure

    resources/js/pages/admin/[module]/
      index.tsx
      columns.tsx
      schema.tsx
      [Module]FiltersConfig.tsx
      hooks/
      components/
      stores/

## Purpose

A blueprint for clean, maintainable, scalable admin interfaces.
