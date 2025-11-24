# Admin Module Frontend Development Guide

## Architecture Overview

React 19 + TypeScript + Inertia 2 + shadcn/ui components for admin interfaces.
This guide defines the standards for creating clean, maintainable, and type-safe admin pages.

## Directory Structure

Follow this structure for all admin modules:

```
resources/js/pages/admin/[module]/
├── index.tsx              // Main listing page (orchestrator)
├── columns.tsx            // Table column definitions
├── schema.tsx             // Zod schemas & TypeScript interfaces
├── UsersFiltersConfig.tsx // Filter & Sort configuration
├── hooks/
│   ├── use[Module]Actions.ts  // CUD operations (Create, Update, Delete)
│   └── use[Module]Filters.ts  // Filter logic extraction
├── components/
│   ├── [Module]Actions.tsx    // Row-level actions dropdown
│   ├── [Module]DetailModal.tsx// Detail view modal
│   ├── ConfirmDialog.tsx      // Reusable confirmation dialog
│   ├── StatsCards.tsx         // Dashboard statistics
│   └── form/                  // Split form components
│       ├── [Module]GeneralInfo.tsx
│       ├── [Module]Security.tsx
│       └── [Module]Roles.tsx
└── stores/
    └── [module]Store.ts       // Zustand state for UI (modals, selections)
```

## Core Principles

1.  **Strict Type Safety**: No `any`. Use Zod schemas mirrored from Backend Resources.
2.  **Component Splitting**: Break down large forms and complex UIs into smaller, focused components.
3.  **Logic Extraction**: Move complex logic (filtering, API calls) into custom hooks.
4.  **UI/UX Standards**: Use `shadcn/ui`, custom confirmation dialogs (no native alerts), and consistent loading states.

## shadcn/ui Components

### Installing Components

Use the shadcn CLI to add components to the project:

```bash
# Example: Add a switch component
npx shadcn@latest add switch

# Add multiple components
npx shadcn@latest add button card input label textarea select

# Common components used in this CMS:
npx shadcn@latest add button card input label textarea select switch
npx shadcn@latest add table dialog dropdown-menu avatar badge
npx shadcn@latest add pagination toast form checkbox
```

## Wayfinder Routing & Navigation

### Wayfinder Setup

Wayfinder should already be installed and configured. Generate routes after adding new backend routes:

```bash
php artisan wayfinder:generate
```

### Common Error: `route is not defined`

When using Inertia.js with React/TypeScript, the `route()` helper function is **not available** in the frontend by default.

**ALWAYS use Wayfinder routes instead of `route()` function or hardcoded URLs:**

```typescript
// ❌ WRONG - This will cause "route is not defined" error
<Link href={route('admin.blog.categories.index')}>
    <Button>Categories</Button>
</Link>

// ❌ ALSO WRONG - Hardcoded URLs are brittle and break easily
<Link href="/admin/blog/categories">
    <Button>Categories</Button>
</Link>

// ✅ CORRECT - Use Wayfinder routes
import categoriesRoutes from '@/routes/admin/blog/categories';

<Link href={categoriesRoutes.index().url}>
    <Button>Categories</Button>
</Link>
```

### Available Route Methods

1. **Wayfinder routes** (HIGHLY RECOMMENDED):

```typescript
import categoriesRoutes from '@/routes/admin/blog/categories';
import postsRoutes from '@/routes/admin/blog/posts';

// Basic usage
<Link href={categoriesRoutes.index().url}>Categories</Link>

// With parameters
<Link href={categoriesRoutes.edit(categoryId).url}>Edit Category</Link>

// With query parameters
<Link href={categoriesRoutes.index().url({ query: { search: 'example' } })}>
    Search Categories
</Link>
```

## 1. Type Safety & Schemas (`schema.tsx`)

Define strict Zod schemas that match your Backend API Resources.

```tsx
import { z } from 'zod';

// Match the Backend Resource exactly
export const userSchema = z.object({
    id: z.number(),
    name: z.string(),
    email: z.string(),
    roles: z.array(z.string()), // e.g., ['admin', 'editor']
    created_at: z.string(),
    // ... other fields
});

export type User = z.infer<typeof userSchema>;

// Form Schema for Validation
export const userFormSchema = z.object({
    name: z.string().min(1, 'Name is required'),
    email: z.string().email(),
    roles: z.array(z.string()).min(1, 'Select at least one role'),
});

export type UserFormData = z.infer<typeof userFormSchema>;

// Page Props Interface
export interface UsersPageProps {
    users: {
        data: User[];
        meta: any;
    };
    filters: Record<string, any>;
    stats: Record<string, any>;
}

## Layout Components

### AppHeader

The `AppHeader` component provides the top navigation bar for the module, including tabs for sub-sections.

```tsx
<AppHeader
    links={[
        {
            title: 'Users',
            href: 'dashboard/overview',
            isActive: true,
            disabled: false,
        },
        {
            title: 'Roles',
            href: 'admin/users/roles',
            isActive: false,
            disabled: true,
        },
    ]}
/>
```

### PageHeader

The `PageHeader` component displays the page title, description, breadcrumbs, and primary actions.

```tsx
<PageHeader
    title="Users Management"
    description="Manage user accounts, roles, and permissions"
    breadcrumb={[
        { label: 'Admin', href: '/admin' },
        { label: 'Users' },
    ]}
    actions={
        <Link href={create.url()}>
            <Button className="flex items-center gap-2">
                <Plus className="h-4 w-4" />
                Add User
            </Button>
        </Link>
    }
/>
```

## 2. Component Architecture

### Main Page (`index.tsx`)

The main page should act as an orchestrator, pulling in data, hooks, and components.

```tsx
export default function UsersDataTable() {
    const { props } = usePage<UsersPageProps>();
    const { users, stats } = props;
    
    // UI State (Zustand)
    const { isModalOpen, openModal } = useUserStore();
    
    // Filter Logic (Custom Hook)
    const { filtersConfig, handleFiltersChange } = useUserFilters({ ... });

    return (
        <AuthenticatedLayout>
            <PageHeader title="Users" actions={<Button onClick={openModal}>Add User</Button>} />
            
            <StatsCards stats={stats} />
            
            <Card>
                <FilterBar config={filtersConfig} onChange={handleFiltersChange} />
                <DataTable columns={columns} data={users.data} />
            </Card>

            <UserDetailModal open={isModalOpen} />
        </AuthenticatedLayout>
    );
}
```

### Data Table Columns (`columns.tsx`)

Use strict types for columns. Create reusable badge components for status/roles.

```tsx
export const columns: ColumnDef<User>[] = [
    {
        accessorKey: 'roles',
        header: 'Roles',
        cell: ({ row }) => (
            <div className="flex gap-1">
                {row.original.roles.map(role => <RoleBadge role={role} />)}
            </div>
        ),
    },
    // ...
];
```

### Forms (`components/form/*.tsx`)

Split large forms. Use `react-hook-form` or Inertia's form helper passed down as props.

**Important:** We always share a form component between Edit and Create pages to ensure consistency and reduce code duplication.

```tsx
// components/form/UserGeneralInfo.tsx
export function UserGeneralInfo({ data, setData, errors }: FormProps) {
    return (
        <div className="space-y-4">
            <Label>Name</Label>
            <Input 
                value={data.name} 
                onChange={e => setData('name', e.target.value)} 
            />
            {errors.name && <span className="text-red-500">{errors.name}</span>}
        </div>
    );
}
```

### Modal-Based Interactions

Detail modal pattern:

```tsx
// components/DetailModal.tsx
interface DetailModalProps {
    item: ModuleData | null;
    isOpen: boolean;
    onClose: () => void;
}

export function DetailModal({ item, isOpen, onClose }: DetailModalProps) {
    if (!item) return null;

    return (
        <Dialog open={isOpen} onOpenChange={onClose}>
            <DialogContent className="max-w-2xl">
                <DialogHeader>
                    <DialogTitle>{item.name}</DialogTitle>
                </DialogHeader>
                <div className="space-y-4">{/* Detail content */}</div>
            </DialogContent>
        </Dialog>
    );
}
```

## 3. Custom Hooks

### Actions Hook (`hooks/use[Module]Actions.ts`)

Centralize API calls and error handling.

```tsx
export const useUserActions = () => {
    const createUser = (data: UserFormData, options?: ActionOptions) => {
        router.post(route('users.store'), data, {
            onSuccess: () => {
                ToastManager.success('User created');
                options?.onSuccess?.();
            },
            onError: (errors) => {
                ToastManager.handleInertiaError(errors);
            }
        });
    };
    return { createUser };
};
```

### Filters Hook (`hooks/use[Module]Filters.ts`)

Encapsulate URL parameter manipulation.

```tsx
export function useUserFilters({ initialFilters }) {
    const handleFiltersChange = (newFilters: any) => {
        // Logic to update URL search params
        router.get(url, newFilters, { preserveState: true });
    };
    return { handleFiltersChange };
}
```

## 4. UI/UX Best Practices

*   **Confirmation Dialogs**: NEVER use `window.confirm`. Use a custom `<ConfirmDialog />` component built with `shadcn/ui` `AlertDialog`.
    ```tsx
    <ConfirmDialog 
        open={isOpen} 
        title="Delete User?" 
        onConfirm={handleDelete} 
    />
    ```
*   **Loading States**: Always show loading spinners in buttons during async actions.
    ```tsx
    // ✅ CORRECT - Show loading state
    <Button disabled={processing}>
        {processing ? 'Creating...' : 'Create Item'}
    </Button>

    // ❌ WRONG - No loading feedback
    <Button type="submit">Create Item</Button>
    ```
*   **Toasts**: Use `ToastManager` for consistent success/error notifications.
*   **Empty States**: Handle empty arrays (e.g., no roles) gracefully with text like "No roles assigned".

## 5. General Best Practices

### Component Organization

- Keep components small and focused
- Use TypeScript for type safety
- Separate data fetching from UI logic
- Use consistent naming conventions

### State Management

- Use Zustand for UI state (modals, loading states)
- Use URL params for persistent state (filters, sorting)
- Keep server state and UI state separate

### Performance

- Use React.memo for expensive components 
- Use debounced search inputs
- Lazy load non-critical components
 
### Error Handling

- Show user-friendly error messages (using `ToastManager`)
- Implement retry mechanisms

## Checklist for New Pages

1.  [ ] Define strict `schema.tsx` matching Backend Resource.
2.  [ ] Create `columns.tsx` with typed columns.
3.  [ ] Setup `use[Module]Store` for UI state.
4.  [ ] Create `use[Module]Filters` for filter logic.
5.  [ ] Create `use[Module]Actions` for CUD operations.
6.  [ ] Build `index.tsx` using the Orchestrator pattern.
7.  [ ] Split forms into sub-components in `components/form/`.
8.  [ ] Use `ConfirmDialog` for destructive actions.
9.  [ ] Ensure all `any` types are removed.
