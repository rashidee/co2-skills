# Component Patterns — Tailwind Design Tokens, Headless UI, TanStack Query, Zustand, React Hook Form

This reference describes the reusable component and state patterns for the spec.
Include this content in Sections 5, 9, 10, 12, 14, 16, and 17 of the generated specification.

All UI is built **utility-first with Tailwind**; interactive/accessible widgets use
**Headless UI** primitives and **Heroicons**. Every composed/conditional className goes
through the `cn()` helper, and components with visual variants use `class-variance-authority`.

---

## Design Tokens & `cn()`

The Tailwind theme is driven by CSS custom properties (semantic tokens) so light/dark mode
switches automatically. See `spec-template.md` Section 5 for `tailwind.config.js`,
`src/index.css`, and the `cn()` helper. The token → class mapping the spec must produce:

| Mockup CSS property | Tailwind token / class |
|---|---|
| Primary brand color (e.g., `#2271b1`) | `--color-primary` → `bg-primary`, `text-primary`, `ring-primary` |
| Primary hover (e.g., `#135e96`) | `--color-primary-dark` → `hover:bg-primary-dark` |
| Page background (e.g., `#f0f0f1`) | `--color-canvas` → `bg-canvas` |
| Surface/card background (e.g., `#ffffff`) | `--color-surface` → `bg-surface` |
| Error color (e.g., `#d63638`) | `--color-danger` → `bg-danger`, `text-danger` |
| Font family | `theme.fontFamily.sans` → `font-sans` |
| Border radius | `theme.borderRadius.DEFAULT` → `rounded` |

```typescript
// src/lib/utils/cn.ts
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

/** Merge Tailwind class names, resolving conflicts (last-wins) via tailwind-merge. */
export function cn(...inputs: ClassValue[]): string {
  return twMerge(clsx(inputs));
}
```

---

## Button (cva variants)

```tsx
// src/shared/components/Button.tsx
import { forwardRef, type ButtonHTMLAttributes } from 'react';
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '../../lib/utils/cn';
import { Spinner } from './Spinner';

const buttonVariants = cva(
  // base classes
  'inline-flex items-center justify-center gap-2 rounded font-semibold transition-colors ' +
    'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-primary focus-visible:ring-offset-2 ' +
    'disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        primary: 'bg-primary text-white hover:bg-primary-dark',
        outline:
          'border border-slate-300 bg-transparent text-slate-700 hover:bg-slate-100 ' +
          'dark:border-slate-600 dark:text-slate-200 dark:hover:bg-slate-800',
        ghost: 'bg-transparent text-slate-700 hover:bg-slate-100 dark:text-slate-200 dark:hover:bg-slate-800',
        danger: 'bg-danger text-white hover:opacity-90',
      },
      size: {
        sm: 'h-8 px-3 text-sm',
        md: 'h-10 px-4 text-sm',
        lg: 'h-11 px-6 text-base',
        icon: 'h-9 w-9',
      },
    },
    defaultVariants: { variant: 'primary', size: 'md' },
  },
);

export interface ButtonProps
  extends ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  /** Show a spinner and disable the button while an async action is in flight */
  loading?: boolean;
}

export const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, loading, disabled, children, ...props }, ref) => (
    <button
      ref={ref}
      className={cn(buttonVariants({ variant, size }), className)}
      disabled={disabled || loading}
      {...props}
    >
      {loading && <Spinner className="h-4 w-4" />}
      {children}
    </button>
  ),
);
Button.displayName = 'Button';
```

```tsx
// src/shared/components/Spinner.tsx
import { cn } from '../../lib/utils/cn';

export function Spinner({ className }: { className?: string }) {
  return (
    <svg
      className={cn('animate-spin text-current', className)}
      viewBox="0 0 24 24"
      fill="none"
      aria-hidden="true"
    >
      <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4" />
      <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8v4a4 4 0 00-4 4H4z" />
    </svg>
  );
}
```

```tsx
// src/shared/components/Card.tsx
import { type HTMLAttributes } from 'react';
import { cn } from '../../lib/utils/cn';

export function Card({ className, ...props }: HTMLAttributes<HTMLDivElement>) {
  return (
    <div
      className={cn(
        'rounded border border-slate-200 bg-surface shadow-sm dark:border-slate-700',
        className,
      )}
      {...props}
    />
  );
}
```

---

## TanStack Query Setup

```typescript
// src/lib/queryClient.ts
import { QueryClient } from '@tanstack/react-query';
import { useNotificationStore } from '../store/notification.store';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Data is considered fresh for 30 seconds — prevents redundant refetches
      staleTime: 30_000,
      // Unused data is garbage collected after 5 minutes
      gcTime: 5 * 60_000,
      // Retry failed queries up to 2 times (not for 4xx errors)
      retry: (failureCount, error) => {
        if (error instanceof Error && 'status' in error) {
          const status = (error as { status: number }).status;
          if (status >= 400 && status < 500) return false;
        }
        return failureCount < 2;
      },
      refetchOnWindowFocus: true,
    },
    mutations: {
      onError: (error) => {
        const message =
          error instanceof Error ? error.message : 'An unexpected error occurred';
        useNotificationStore.getState().push('error', message);
      },
    },
  },
});
```

### Standard Query Hook Pattern

```typescript
// Example: src/features/heroSection/hooks/useHeroSections.ts
import { useQuery } from '@tanstack/react-query';
import { heroSectionApi } from '../api/heroSectionApi';
import type { HeroSectionListParams } from '../heroSection.types';

export const heroSectionKeys = {
  all: ['heroSections'] as const,
  lists: () => [...heroSectionKeys.all, 'list'] as const,
  list: (params: HeroSectionListParams) => [...heroSectionKeys.lists(), params] as const,
  details: () => [...heroSectionKeys.all, 'detail'] as const,
  detail: (id: string) => [...heroSectionKeys.details(), id] as const,
};

export function useHeroSections(params: HeroSectionListParams) {
  return useQuery({
    queryKey: heroSectionKeys.list(params),
    queryFn: () => heroSectionApi.list(params),
    placeholderData: (previousData) => previousData, // avoid empty flash on page change
  });
}

export function useHeroSection(id: string) {
  return useQuery({
    queryKey: heroSectionKeys.detail(id),
    queryFn: () => heroSectionApi.getById(id),
    enabled: !!id,
  });
}
```

### Standard Mutation Hook Pattern

```typescript
// Example: src/features/heroSection/hooks/useHeroSectionMutations.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { heroSectionApi } from '../api/heroSectionApi';
import { heroSectionKeys } from './useHeroSections';
import { useNotificationStore } from '../../../store/notification.store';
import type { HeroSectionCreate, HeroSectionUpdate } from '../heroSection.types';

export function useCreateHeroSection() {
  const queryClient = useQueryClient();
  const push = useNotificationStore((s) => s.push);

  return useMutation({
    mutationFn: (data: HeroSectionCreate) => heroSectionApi.create(data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: heroSectionKeys.lists() });
      push('success', 'Hero section created successfully.');
    },
  });
}

export function useUpdateHeroSection() {
  const queryClient = useQueryClient();
  const push = useNotificationStore((s) => s.push);

  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: HeroSectionUpdate }) =>
      heroSectionApi.update(id, data),
    onSuccess: (_, { id }) => {
      queryClient.invalidateQueries({ queryKey: heroSectionKeys.detail(id) });
      queryClient.invalidateQueries({ queryKey: heroSectionKeys.lists() });
      push('success', 'Hero section updated successfully.');
    },
  });
}

export function useDeleteHeroSection() {
  const queryClient = useQueryClient();
  const push = useNotificationStore((s) => s.push);

  return useMutation({
    mutationFn: (id: string) => heroSectionApi.delete(id),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: heroSectionKeys.lists() });
      push('success', 'Hero section deleted.');
    },
  });
}
```

---

## Zustand Stores

### Notification Store

```typescript
// src/store/notification.store.ts
import { create } from 'zustand';

type NotificationType = 'success' | 'error' | 'warning' | 'info';

export interface Notification {
  id: string;
  type: NotificationType;
  message: string;
}

interface NotificationState {
  notifications: Notification[];
  push: (type: NotificationType, message: string) => void;
  dismiss: (id: string) => void;
}

export const useNotificationStore = create<NotificationState>((set) => ({
  notifications: [],

  push: (type, message) => {
    const id = crypto.randomUUID();
    set((state) => ({ notifications: [...state.notifications, { id, type, message }] }));
    // Auto-dismiss after 5 seconds
    setTimeout(() => {
      set((state) => ({
        notifications: state.notifications.filter((n) => n.id !== id),
      }));
    }, 5_000);
  },

  dismiss: (id) =>
    set((state) => ({ notifications: state.notifications.filter((n) => n.id !== id) })),
}));
```

### UI Store (sidebar + dark mode)

```typescript
// src/store/ui.store.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface UiState {
  sidebarOpen: boolean;
  themeMode: 'light' | 'dark';
  toggleSidebar: () => void;
  toggleThemeMode: () => void;
}

export const useUiStore = create<UiState>()(
  persist(
    (set) => ({
      sidebarOpen: true,
      themeMode: 'light',
      toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
      toggleThemeMode: () =>
        set((state) => ({ themeMode: state.themeMode === 'light' ? 'dark' : 'light' })),
    }),
    { name: 'ui-preferences' }, // Persisted to localStorage (safe — no tokens here)
  ),
);
```

> The `themeMode` value drives the Tailwind `dark` class on `document.documentElement`. The
> sync effect lives in `App.tsx` (see `spec-template.md` Section 5.5).

---

## React Hook Form + Zod Patterns

### Zod Schema Example

```typescript
// Example: src/features/heroSection/heroSection.schema.ts
import { z } from 'zod';

/**
 * Validation schema for hero section create/update form.
 * Rules derived from PRD.md NFRs and constraints.
 */
export const heroSectionSchema = z
  .object({
    headline: z
      .string()
      .min(1, 'Headline is required')
      .max(100, 'Headline must be at most 100 characters'),
    subheadline: z
      .string()
      .min(1, 'Subheadline is required')
      .max(200, 'Subheadline must be at most 200 characters'),
    ctaText: z
      .string()
      .min(1, 'CTA text is required')
      .max(50, 'CTA text must be at most 50 characters'),
    ctaUrl: z.string().url('Must be a valid URL').min(1, 'CTA URL is required'),
    effectiveDate: z.string().min(1, 'Effective date is required'),
    expirationDate: z.string().min(1, 'Expiration date is required'),
    // image is optional in schema — uploaded separately via multipart
  })
  .refine((data) => new Date(data.expirationDate) > new Date(data.effectiveDate), {
    message: 'Expiration date must be after effective date',
    path: ['expirationDate'],
  });

export type HeroSectionFormValues = z.infer<typeof heroSectionSchema>;
```

### Form Component Pattern

```tsx
// Example: src/features/heroSection/components/HeroSectionForm.tsx
import { useState } from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { Button } from '../../../shared/components/Button';
import { TextFieldController } from '../../../shared/form/TextFieldController';
import { TextAreaController } from '../../../shared/form/TextAreaController';
import { DatePickerController } from '../../../shared/form/DatePickerController';
import { ImageUpload } from '../../../shared/components/ImageUpload';
import { heroSectionSchema, type HeroSectionFormValues } from '../heroSection.schema';

interface HeroSectionFormProps {
  defaultValues?: Partial<HeroSectionFormValues>;
  onSubmit: (values: HeroSectionFormValues, imageFile?: File) => void;
  isSubmitting: boolean;
  onCancel: () => void;
}

export function HeroSectionForm({
  defaultValues,
  onSubmit,
  isSubmitting,
  onCancel,
}: HeroSectionFormProps) {
  const { control, handleSubmit } = useForm<HeroSectionFormValues>({
    resolver: zodResolver(heroSectionSchema),
    defaultValues: {
      headline: '',
      subheadline: '',
      ctaText: '',
      ctaUrl: '',
      effectiveDate: '',
      expirationDate: '',
      ...defaultValues,
    },
  });

  const [imageFile, setImageFile] = useState<File | undefined>();

  const handleFormSubmit = (values: HeroSectionFormValues) => {
    onSubmit(values, imageFile);
  };

  return (
    <form onSubmit={handleSubmit(handleFormSubmit)} noValidate className="space-y-6">
      <ImageUpload
        label="Hero Image (1600x500 px)"
        onFileChange={setImageFile}
        aspectRatio="1600/500"
        helperText="Image must be exactly 1600×500 pixels"
      />
      <TextFieldController
        name="headline"
        control={control}
        label="Headline"
        required
        maxLength={100}
        helperText="Max 100 characters"
      />
      <TextAreaController
        name="subheadline"
        control={control}
        label="Subheadline"
        required
        rows={2}
        maxLength={200}
      />
      <TextFieldController name="ctaText" control={control} label="CTA Button Text" required maxLength={50} />
      <TextFieldController
        name="ctaUrl"
        control={control}
        label="CTA Button URL"
        required
        placeholder="https://..."
      />
      <div className="grid grid-cols-1 gap-4 sm:grid-cols-2">
        <DatePickerController name="effectiveDate" control={control} label="Effective Date" required />
        <DatePickerController name="expirationDate" control={control} label="Expiration Date" required />
      </div>

      <div className="flex justify-end gap-3">
        <Button type="button" variant="outline" onClick={onCancel}>
          Cancel
        </Button>
        <Button type="submit" loading={isSubmitting}>
          {defaultValues ? 'Update' : 'Create'}
        </Button>
      </div>
    </form>
  );
}
```

### RHF Controller Wrappers

```tsx
// src/shared/form/TextFieldController.tsx
import { Controller, type Control, type FieldPath, type FieldValues } from 'react-hook-form';
import { cn } from '../../lib/utils/cn';

interface TextFieldControllerProps<T extends FieldValues> {
  name: FieldPath<T>;
  control: Control<T>;
  label: string;
  type?: 'text' | 'email' | 'number' | 'url' | 'password';
  placeholder?: string;
  required?: boolean;
  maxLength?: number;
  helperText?: string;
}

export function TextFieldController<T extends FieldValues>({
  name,
  control,
  label,
  type = 'text',
  placeholder,
  required,
  maxLength,
  helperText,
}: TextFieldControllerProps<T>) {
  return (
    <Controller
      name={name}
      control={control}
      render={({ field, fieldState }) => (
        <div className="space-y-1">
          <label htmlFor={name} className="block text-sm font-medium text-slate-700 dark:text-slate-200">
            {label}
            {required && <span className="ml-0.5 text-danger">*</span>}
          </label>
          <input
            {...field}
            id={name}
            type={type}
            placeholder={placeholder}
            maxLength={maxLength}
            aria-invalid={!!fieldState.error}
            className={cn(
              'block w-full rounded border bg-surface px-3 py-2 text-sm text-slate-900 shadow-sm',
              'focus:border-primary focus:ring-1 focus:ring-primary dark:text-slate-100',
              fieldState.error ? 'border-danger' : 'border-slate-300 dark:border-slate-600',
            )}
          />
          {(fieldState.error?.message ?? helperText) && (
            <p className={cn('text-xs', fieldState.error ? 'text-danger' : 'text-slate-500')}>
              {fieldState.error?.message ?? helperText}
            </p>
          )}
        </div>
      )}
    />
  );
}
```

```tsx
// src/shared/form/TextAreaController.tsx
import { Controller, type Control, type FieldPath, type FieldValues } from 'react-hook-form';
import { cn } from '../../lib/utils/cn';

interface TextAreaControllerProps<T extends FieldValues> {
  name: FieldPath<T>;
  control: Control<T>;
  label: string;
  rows?: number;
  required?: boolean;
  maxLength?: number;
  helperText?: string;
}

export function TextAreaController<T extends FieldValues>({
  name,
  control,
  label,
  rows = 3,
  required,
  maxLength,
  helperText,
}: TextAreaControllerProps<T>) {
  return (
    <Controller
      name={name}
      control={control}
      render={({ field, fieldState }) => (
        <div className="space-y-1">
          <label htmlFor={name} className="block text-sm font-medium text-slate-700 dark:text-slate-200">
            {label}
            {required && <span className="ml-0.5 text-danger">*</span>}
          </label>
          <textarea
            {...field}
            id={name}
            rows={rows}
            maxLength={maxLength}
            aria-invalid={!!fieldState.error}
            className={cn(
              'block w-full rounded border bg-surface px-3 py-2 text-sm text-slate-900 shadow-sm',
              'focus:border-primary focus:ring-1 focus:ring-primary dark:text-slate-100',
              fieldState.error ? 'border-danger' : 'border-slate-300 dark:border-slate-600',
            )}
          />
          {(fieldState.error?.message ?? helperText) && (
            <p className={cn('text-xs', fieldState.error ? 'text-danger' : 'text-slate-500')}>
              {fieldState.error?.message ?? helperText}
            </p>
          )}
        </div>
      )}
    />
  );
}
```

```tsx
// src/shared/form/SelectController.tsx — Headless UI Listbox
import { Controller, type Control, type FieldPath, type FieldValues } from 'react-hook-form';
import { Listbox, ListboxButton, ListboxOption, ListboxOptions } from '@headlessui/react';
import { CheckIcon, ChevronUpDownIcon } from '@heroicons/react/20/solid';
import { cn } from '../../lib/utils/cn';

export interface SelectOption {
  value: string | number;
  label: string;
}

interface SelectControllerProps<T extends FieldValues> {
  name: FieldPath<T>;
  control: Control<T>;
  label: string;
  options: SelectOption[];
  required?: boolean;
  helperText?: string;
  placeholder?: string;
}

export function SelectController<T extends FieldValues>({
  name,
  control,
  label,
  options,
  required,
  helperText,
  placeholder = 'Select…',
}: SelectControllerProps<T>) {
  return (
    <Controller
      name={name}
      control={control}
      render={({ field, fieldState }) => {
        const selected = options.find((o) => o.value === field.value);
        return (
          <div className="space-y-1">
            <label className="block text-sm font-medium text-slate-700 dark:text-slate-200">
              {label}
              {required && <span className="ml-0.5 text-danger">*</span>}
            </label>
            <Listbox value={field.value} onChange={field.onChange}>
              <div className="relative">
                <ListboxButton
                  className={cn(
                    'relative w-full cursor-default rounded border bg-surface py-2 pl-3 pr-10 text-left text-sm shadow-sm',
                    'focus:border-primary focus:ring-1 focus:ring-primary',
                    fieldState.error ? 'border-danger' : 'border-slate-300 dark:border-slate-600',
                  )}
                >
                  <span className={cn('block truncate', !selected && 'text-slate-400')}>
                    {selected?.label ?? placeholder}
                  </span>
                  <ChevronUpDownIcon className="pointer-events-none absolute right-2 top-2.5 h-5 w-5 text-slate-400" />
                </ListboxButton>
                <ListboxOptions
                  anchor="bottom"
                  className="z-50 mt-1 max-h-60 w-[var(--button-width)] overflow-auto rounded border border-slate-200 bg-surface py-1 text-sm shadow-lg focus:outline-none dark:border-slate-700"
                >
                  {options.map((opt) => (
                    <ListboxOption
                      key={opt.value}
                      value={opt.value}
                      className="group flex cursor-default items-center gap-2 px-3 py-2 data-[focus]:bg-primary data-[focus]:text-white"
                    >
                      <CheckIcon className="invisible h-4 w-4 group-data-[selected]:visible" />
                      <span className="truncate">{opt.label}</span>
                    </ListboxOption>
                  ))}
                </ListboxOptions>
              </div>
            </Listbox>
            {(fieldState.error?.message ?? helperText) && (
              <p className={cn('text-xs', fieldState.error ? 'text-danger' : 'text-slate-500')}>
                {fieldState.error?.message ?? helperText}
              </p>
            )}
          </div>
        );
      }}
    />
  );
}
```

```tsx
// src/shared/form/SwitchController.tsx — Headless UI Switch
import { Controller, type Control, type FieldPath, type FieldValues } from 'react-hook-form';
import { Switch } from '@headlessui/react';
import { cn } from '../../lib/utils/cn';

interface SwitchControllerProps<T extends FieldValues> {
  name: FieldPath<T>;
  control: Control<T>;
  label: string;
}

export function SwitchController<T extends FieldValues>({
  name,
  control,
  label,
}: SwitchControllerProps<T>) {
  return (
    <Controller
      name={name}
      control={control}
      render={({ field }) => (
        <div className="flex items-center justify-between">
          <span className="text-sm font-medium text-slate-700 dark:text-slate-200">{label}</span>
          <Switch
            checked={!!field.value}
            onChange={field.onChange}
            className={cn(
              'group relative inline-flex h-6 w-11 items-center rounded-full transition-colors',
              'bg-slate-300 data-[checked]:bg-primary dark:bg-slate-600',
            )}
          >
            <span className="inline-block h-4 w-4 translate-x-1 rounded-full bg-white transition-transform group-data-[checked]:translate-x-6" />
          </Switch>
        </div>
      )}
    />
  );
}
```

> `DatePickerController` (react-day-picker inside a Headless UI `Popover`),
> `ComboboxController` (Headless UI `Combobox`), and `RichTextController` (Tiptap) follow the
> same `Controller`-wrapped pattern. Generate them when `DatePickers`, autocomplete fields, or
> `RichText` are required.

---

## Notification Provider (Headless UI Transition toasts)

```tsx
// src/shared/components/NotificationProvider.tsx
import { Transition } from '@headlessui/react';
import {
  CheckCircleIcon,
  ExclamationTriangleIcon,
  InformationCircleIcon,
  XCircleIcon,
  XMarkIcon,
} from '@heroicons/react/24/outline';
import { cn } from '../../lib/utils/cn';
import { useNotificationStore, type Notification } from '../../store/notification.store';

const TOAST_STYLES: Record<Notification['type'], { icon: typeof CheckCircleIcon; classes: string }> = {
  success: { icon: CheckCircleIcon, classes: 'border-success/40 text-success' },
  error: { icon: XCircleIcon, classes: 'border-danger/40 text-danger' },
  warning: { icon: ExclamationTriangleIcon, classes: 'border-warning/40 text-warning' },
  info: { icon: InformationCircleIcon, classes: 'border-primary/40 text-primary' },
};

export function NotificationProvider() {
  const notifications = useNotificationStore((s) => s.notifications);
  const dismiss = useNotificationStore((s) => s.dismiss);

  return (
    <div className="pointer-events-none fixed bottom-6 right-6 z-[9999] flex w-full max-w-sm flex-col gap-2">
      {notifications.map((n) => {
        const { icon: Icon, classes } = TOAST_STYLES[n.type];
        return (
          <Transition
            key={n.id}
            appear
            show
            enter="transition duration-200 ease-out"
            enterFrom="translate-y-2 opacity-0"
            enterTo="translate-y-0 opacity-100"
            leave="transition duration-150 ease-in"
            leaveFrom="opacity-100"
            leaveTo="opacity-0"
          >
            <div
              role="alert"
              className={cn(
                'pointer-events-auto flex items-start gap-3 rounded border bg-surface p-3 shadow-lg',
                classes,
              )}
            >
              <Icon className="mt-0.5 h-5 w-5 shrink-0" />
              <p className="flex-1 text-sm text-slate-800 dark:text-slate-100">{n.message}</p>
              <button
                type="button"
                onClick={() => dismiss(n.id)}
                className="text-slate-400 hover:text-slate-600 dark:hover:text-slate-200"
                aria-label="Dismiss"
              >
                <XMarkIcon className="h-4 w-4" />
              </button>
            </div>
          </Transition>
        );
      })}
    </div>
  );
}
```

---

## Status Badge (cva variants)

```tsx
// src/shared/components/StatusBadge.tsx
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '../../lib/utils/cn';

const badgeVariants = cva(
  'inline-flex items-center rounded-full px-2.5 py-0.5 text-xs font-medium',
  {
    variants: {
      variant: {
        neutral: 'bg-slate-100 text-slate-700 dark:bg-slate-700 dark:text-slate-200',
        primary: 'bg-primary/10 text-primary',
        success: 'bg-success/10 text-success',
        warning: 'bg-warning/10 text-warning',
        danger: 'bg-danger/10 text-danger',
      },
    },
    defaultVariants: { variant: 'neutral' },
  },
);

export type StatusVariant = NonNullable<VariantProps<typeof badgeVariants>['variant']>;

interface StatusConfig {
  label: string;
  variant: StatusVariant;
}

interface StatusBadgeProps {
  status: string;
  /** Map of status value → display config. Provide this per-module. */
  statusMap: Record<string, StatusConfig>;
  className?: string;
}

export function StatusBadge({ status, statusMap, className }: StatusBadgeProps) {
  const config = statusMap[status] ?? { label: status, variant: 'neutral' as const };
  return <span className={cn(badgeVariants({ variant: config.variant }), className)}>{config.label}</span>;
}

// Example usage in a module:
// const HERO_STATUS_MAP = {
//   DRAFT:   { label: 'Draft',   variant: 'neutral' },
//   ACTIVE:  { label: 'Active',  variant: 'success' },
//   EXPIRED: { label: 'Expired', variant: 'danger'  },
// };
// <StatusBadge status={row.status} statusMap={HERO_STATUS_MAP} />
```

---

## Confirm Dialog (Headless UI Dialog + imperative hook)

```tsx
// src/shared/components/ConfirmDialog.tsx
import { Dialog, DialogBackdrop, DialogPanel, DialogTitle } from '@headlessui/react';
import { create } from 'zustand';
import { Button } from './Button';

interface ConfirmState {
  open: boolean;
  title: string;
  message: string;
  confirmLabel: string;
  cancelLabel: string;
  onConfirm: (() => void) | null;
  show: (options: Omit<ConfirmState, 'open' | 'show' | 'hide'>) => void;
  hide: () => void;
}

const useConfirmStore = create<ConfirmState>((set) => ({
  open: false,
  title: '',
  message: '',
  confirmLabel: 'Confirm',
  cancelLabel: 'Cancel',
  onConfirm: null,
  show: (options) => set({ open: true, ...options }),
  hide: () => set({ open: false, onConfirm: null }),
}));

/** Hook to imperatively trigger a confirmation dialog. Resolves true when confirmed. */
export function useConfirm() {
  const show = useConfirmStore((s) => s.show);

  return (options: {
    title?: string;
    message: string;
    confirmLabel?: string;
    cancelLabel?: string;
  }): Promise<boolean> =>
    new Promise((resolve) => {
      show({
        title: options.title ?? 'Confirm Action',
        message: options.message,
        confirmLabel: options.confirmLabel ?? 'Confirm',
        cancelLabel: options.cancelLabel ?? 'Cancel',
        onConfirm: () => resolve(true),
      });
    });
}

/** Render this once at the app root to display the dialog. */
export function ConfirmDialog() {
  const { open, title, message, confirmLabel, cancelLabel, onConfirm, hide } = useConfirmStore();

  const handleConfirm = () => {
    onConfirm?.();
    hide();
  };

  return (
    <Dialog open={open} onClose={hide} className="relative z-[1000]">
      <DialogBackdrop
        transition
        className="fixed inset-0 bg-black/40 transition-opacity duration-200 data-[closed]:opacity-0"
      />
      <div className="fixed inset-0 flex items-center justify-center p-4">
        <DialogPanel
          transition
          className="w-full max-w-sm rounded-lg bg-surface p-6 shadow-xl transition duration-200 data-[closed]:scale-95 data-[closed]:opacity-0"
        >
          <DialogTitle className="text-lg font-semibold text-slate-900 dark:text-slate-100">
            {title}
          </DialogTitle>
          <p className="mt-2 text-sm text-slate-600 dark:text-slate-300">{message}</p>
          <div className="mt-6 flex justify-end gap-3">
            <Button variant="outline" onClick={hide}>
              {cancelLabel}
            </Button>
            <Button variant="danger" onClick={handleConfirm} autoFocus>
              {confirmLabel}
            </Button>
          </div>
        </DialogPanel>
      </div>
    </Dialog>
  );
}
```

---

## API Function Pattern

```typescript
// Example: src/features/heroSection/api/heroSectionApi.ts
import { axiosInstance } from '../../../lib/api/axiosInstance';
import type {
  HeroSection,
  HeroSectionCreate,
  HeroSectionUpdate,
  HeroSectionListParams,
  PaginatedResponse,
} from '../heroSection.types';

export const heroSectionApi = {
  /** Fetch paginated list of hero sections. */
  list: async (params: HeroSectionListParams): Promise<PaginatedResponse<HeroSection>> => {
    const { data } = await axiosInstance.get<PaginatedResponse<HeroSection>>('/hero-sections', {
      params,
    });
    return data;
  },

  /** Fetch a single hero section by ID. */
  getById: async (id: string): Promise<HeroSection> => {
    const { data } = await axiosInstance.get<HeroSection>(`/hero-sections/${id}`);
    return data;
  },

  /** Create a new hero section with an optional image upload (multipart). */
  create: async (payload: HeroSectionCreate, imageFile?: File): Promise<HeroSection> => {
    if (imageFile) {
      const formData = new FormData();
      formData.append('data', JSON.stringify(payload));
      formData.append('image', imageFile);
      const { data } = await axiosInstance.post<HeroSection>('/hero-sections', formData, {
        headers: { 'Content-Type': 'multipart/form-data' },
      });
      return data;
    }
    const { data } = await axiosInstance.post<HeroSection>('/hero-sections', payload);
    return data;
  },

  /** Update an existing hero section. */
  update: async (id: string, payload: HeroSectionUpdate, imageFile?: File): Promise<HeroSection> => {
    if (imageFile) {
      const formData = new FormData();
      formData.append('data', JSON.stringify(payload));
      formData.append('image', imageFile);
      const { data } = await axiosInstance.put<HeroSection>(`/hero-sections/${id}`, formData, {
        headers: { 'Content-Type': 'multipart/form-data' },
      });
      return data;
    }
    const { data } = await axiosInstance.put<HeroSection>(`/hero-sections/${id}`, payload);
    return data;
  },

  /** Delete a hero section by ID. */
  delete: async (id: string): Promise<void> => {
    await axiosInstance.delete(`/hero-sections/${id}`);
  },

  /** Serve image by ID (returns a URL for use in <img src="...">). */
  imageUrl: (id: string): string =>
    `${import.meta.env.VITE_API_BASE_URL}/hero-sections/${id}/image`,
};
```

---

## TypeScript Types Pattern

```typescript
// Example: src/features/heroSection/heroSection.types.ts

/** Enum derived from PRD.md constraint for hero section status */
export type HeroSectionStatus = 'DRAFT' | 'ACTIVE' | 'EXPIRED';

/** Full hero section entity returned by the API */
export interface HeroSection {
  id: string;
  headline: string;
  subheadline: string;
  ctaText: string;
  ctaUrl: string;
  effectiveDate: string;   // ISO date string (YYYY-MM-DD)
  expirationDate: string;  // ISO date string
  status: HeroSectionStatus;
  createdAt: string;       // ISO datetime string
  updatedAt: string;
}

/** Payload for creating a hero section (image uploaded separately) */
export interface HeroSectionCreate {
  headline: string;
  subheadline: string;
  ctaText: string;
  ctaUrl: string;
  effectiveDate: string;
  expirationDate: string;
}

/** Payload for updating a hero section */
export type HeroSectionUpdate = Partial<HeroSectionCreate>;

/** Query parameters for the list endpoint */
export interface HeroSectionListParams {
  page?: number;
  size?: number;
  status?: HeroSectionStatus | '';
  effectiveDateFrom?: string;
  effectiveDateTo?: string;
}

/** Generic paginated response wrapper — matches the backend API contract */
export interface PaginatedResponse<T> {
  content: T[];
  page: number;
  size: number;
  totalElements: number;
  totalPages: number;
}
```

---

## Page Component Pattern (Tailwind table list page)

```tsx
// Example: src/features/heroSection/pages/HeroSectionListPage.tsx
import { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { PlusIcon, PencilSquareIcon, TrashIcon } from '@heroicons/react/24/outline';
import { Button } from '../../../shared/components/Button';
import { Card } from '../../../shared/components/Card';
import { PageHeader } from '../../../shared/components/PageHeader';
import { StatusBadge } from '../../../shared/components/StatusBadge';
import { LoadingOverlay } from '../../../shared/components/LoadingOverlay';
import { EmptyState } from '../../../shared/components/EmptyState';
import { Pagination } from '../../../shared/components/Pagination';
import { useConfirm } from '../../../shared/components/ConfirmDialog';
import { useHeroSections } from '../hooks/useHeroSections';
import { useDeleteHeroSection } from '../hooks/useHeroSectionMutations';
import type { HeroSectionListParams, HeroSectionStatus } from '../heroSection.types';

const HERO_STATUS_MAP = {
  DRAFT:   { label: 'Draft',   variant: 'neutral' as const },
  ACTIVE:  { label: 'Active',  variant: 'success' as const },
  EXPIRED: { label: 'Expired', variant: 'danger'  as const },
};

export function HeroSectionListPage() {
  const navigate = useNavigate();
  const confirm = useConfirm();

  const [params, setParams] = useState<HeroSectionListParams>({ page: 0, size: 10, status: '' });

  const { data, isLoading, isError } = useHeroSections(params);
  const deleteMutation = useDeleteHeroSection();

  const handleDelete = async (id: string, headline: string) => {
    const confirmed = await confirm({
      title: 'Delete Hero Section',
      message: `Are you sure you want to delete "${headline}"? This action cannot be undone.`,
      confirmLabel: 'Delete',
    });
    if (confirmed) deleteMutation.mutate(id);
  };

  if (isLoading) return <LoadingOverlay />;
  if (isError) return <p className="text-danger">Failed to load hero sections.</p>;

  return (
    <div>
      <PageHeader
        title="Hero Sections"
        breadcrumbs={[{ label: 'Dashboard', path: '/dashboard' }, { label: 'Hero Sections' }]}
        actions={
          <Button onClick={() => navigate('/hero-section/create')}>
            <PlusIcon className="h-4 w-4" />
            Add Hero Section
          </Button>
        }
      />

      {/* Filters */}
      <div className="mb-4 flex flex-wrap gap-3">
        <select
          value={params.status ?? ''}
          onChange={(e) =>
            setParams((p) => ({ ...p, status: e.target.value as HeroSectionStatus | '', page: 0 }))
          }
          className="rounded border border-slate-300 bg-surface px-3 py-2 text-sm focus:border-primary focus:ring-1 focus:ring-primary dark:border-slate-600"
        >
          <option value="">All Statuses</option>
          <option value="DRAFT">Draft</option>
          <option value="ACTIVE">Active</option>
          <option value="EXPIRED">Expired</option>
        </select>
      </div>

      <Card className="overflow-hidden">
        <div className="overflow-x-auto">
          <table className="min-w-full divide-y divide-slate-200 dark:divide-slate-700">
            <thead className="bg-canvas">
              <tr>
                <th className="px-4 py-3 text-left text-xs font-semibold uppercase tracking-wide text-slate-600 dark:text-slate-300">Headline</th>
                <th className="px-4 py-3 text-left text-xs font-semibold uppercase tracking-wide text-slate-600 dark:text-slate-300">Effective Date</th>
                <th className="px-4 py-3 text-left text-xs font-semibold uppercase tracking-wide text-slate-600 dark:text-slate-300">Expiration Date</th>
                <th className="px-4 py-3 text-left text-xs font-semibold uppercase tracking-wide text-slate-600 dark:text-slate-300">Status</th>
                <th className="px-4 py-3 text-right text-xs font-semibold uppercase tracking-wide text-slate-600 dark:text-slate-300">Actions</th>
              </tr>
            </thead>
            <tbody className="divide-y divide-slate-100 dark:divide-slate-800">
              {data?.content.length === 0 ? (
                <tr>
                  <td colSpan={5}>
                    <EmptyState message="No hero sections found." />
                  </td>
                </tr>
              ) : (
                data?.content.map((row) => (
                  <tr key={row.id} className="hover:bg-canvas/60">
                    <td className="px-4 py-3 text-sm text-slate-900 dark:text-slate-100">{row.headline}</td>
                    <td className="px-4 py-3 text-sm text-slate-600 dark:text-slate-300">{row.effectiveDate}</td>
                    <td className="px-4 py-3 text-sm text-slate-600 dark:text-slate-300">{row.expirationDate}</td>
                    <td className="px-4 py-3">
                      <StatusBadge status={row.status} statusMap={HERO_STATUS_MAP} />
                    </td>
                    <td className="px-4 py-3 text-right">
                      <div className="flex justify-end gap-1">
                        <Button
                          variant="ghost"
                          size="icon"
                          aria-label="Edit"
                          onClick={() => navigate(`/hero-section/${row.id}/edit`)}
                        >
                          <PencilSquareIcon className="h-4 w-4" />
                        </Button>
                        <Button
                          variant="ghost"
                          size="icon"
                          aria-label="Delete"
                          className="text-danger"
                          onClick={() => handleDelete(row.id, row.headline)}
                        >
                          <TrashIcon className="h-4 w-4" />
                        </Button>
                      </div>
                    </td>
                  </tr>
                ))
              )}
            </tbody>
          </table>
        </div>
        <Pagination
          page={params.page ?? 0}
          size={params.size ?? 10}
          totalElements={data?.totalElements ?? 0}
          onPageChange={(page) => setParams((p) => ({ ...p, page }))}
          onSizeChange={(size) => setParams((p) => ({ ...p, size, page: 0 }))}
        />
      </Card>
    </div>
  );
}
```

> For grids needing column sorting, multi-column filtering, or row selection, replace the
> raw `<table>` with the `DataTable` component built on **TanStack Table** (`@tanstack/react-table`).
> See `routing-patterns.md` for the `DataTable` and `Pagination` shared-component implementations.
