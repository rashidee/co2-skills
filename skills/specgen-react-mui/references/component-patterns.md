# Component Patterns — MUI Theme, TanStack Query, Zustand, React Hook Form

This reference describes the reusable component and state patterns for the spec.
Include this content in Sections 5, 9, 10, 12, and 14 of the generated specification.

---

## MUI Theme Configuration

### Theme Setup

```typescript
// src/theme/palette.ts
import type { PaletteOptions } from '@mui/material/styles';

/**
 * Design tokens extracted from the application mockup.
 * Replace hex values with those found in MOCKUP.html CSS / inline styles.
 */
export const palette: PaletteOptions = {
  primary: {
    main: '{{PRIMARY_COLOR}}',       // e.g., '#2271b1' from mockup
    dark: '{{PRIMARY_DARK_COLOR}}',  // e.g., '#135e96' from mockup (hover state)
    contrastText: '#ffffff',
  },
  secondary: {
    main: '{{SECONDARY_COLOR}}',
    contrastText: '#ffffff',
  },
  error: {
    main: '{{ERROR_COLOR}}',         // e.g., '#d63638' from mockup
  },
  warning: {
    main: '{{WARNING_COLOR}}',       // e.g., '#dba617' from mockup
  },
  success: {
    main: '{{SUCCESS_COLOR}}',       // e.g., '#00a32a' from mockup
  },
  background: {
    default: '{{PAGE_BACKGROUND}}',  // e.g., '#f0f0f1' from mockup
    paper: '{{SURFACE_BACKGROUND}}', // e.g., '#ffffff' from mockup
  },
  text: {
    primary: '{{TEXT_PRIMARY}}',     // e.g., '#1e1e1e' from mockup
    secondary: '{{TEXT_SECONDARY}}', // e.g., '#646970' from mockup
  },
  divider: '{{BORDER_COLOR}}',       // e.g., '#c3c4c7' from mockup
};
```

```typescript
// src/theme/typography.ts
import type { TypographyOptions } from '@mui/material/styles/createTypography';

/**
 * Typography tokens extracted from the application mockup.
 */
export const typography: TypographyOptions = {
  // Extract font family from mockup CSS font-family declaration
  fontFamily: '{{FONT_FAMILY}}',  // e.g., '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, ...'
  h1: { fontSize: '2rem', fontWeight: 700 },
  h2: { fontSize: '1.75rem', fontWeight: 700 },
  h3: { fontSize: '1.5rem', fontWeight: 600 },
  h4: { fontSize: '1.25rem', fontWeight: 600 },
  h5: { fontSize: '1.125rem', fontWeight: 600 },
  h6: { fontSize: '1rem', fontWeight: 600 },
  body1: { fontSize: '1rem' },
  body2: { fontSize: '0.875rem' },
  caption: { fontSize: '0.75rem', color: '{{TEXT_SECONDARY}}' },
};
```

```typescript
// src/theme/index.ts
import { createTheme } from '@mui/material/styles';
import { palette } from './palette';
import { typography } from './typography';

export const lightTheme = createTheme({
  palette: { mode: 'light', ...palette },
  typography,
  shape: {
    // Extract border-radius from mockup
    borderRadius: {{BORDER_RADIUS_BASE}}, // e.g., 4
  },
  components: {
    MuiButton: {
      styleOverrides: {
        root: {
          borderRadius: {{BORDER_RADIUS_BUTTON}}, // e.g., 4 for buttons
          textTransform: 'none',
          fontWeight: 600,
        },
      },
      defaultProps: {
        disableElevation: true,
      },
    },
    MuiCard: {
      styleOverrides: {
        root: {
          borderRadius: {{BORDER_RADIUS_CARD}}, // e.g., 8 for cards
          boxShadow: '0 1px 3px rgba(0,0,0,0.1)',
        },
      },
    },
    MuiTextField: {
      defaultProps: {
        size: 'small',
        variant: 'outlined',
      },
    },
    MuiPaper: {
      styleOverrides: {
        root: {
          borderRadius: {{BORDER_RADIUS_CARD}},
        },
      },
    },
    MuiTableCell: {
      styleOverrides: {
        head: {
          fontWeight: 600,
          backgroundColor: '{{PAGE_BACKGROUND}}',
        },
      },
    },
  },
});

export const darkTheme = createTheme({
  palette: { mode: 'dark' },
  typography,
  shape: { borderRadius: {{BORDER_RADIUS_BASE}} },
});
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
  const { push } = useNotificationStore();

  return useMutation({
    mutationFn: (data: HeroSectionCreate) => heroSectionApi.create(data),
    onSuccess: () => {
      // Invalidate the list query so it refetches with the new item
      queryClient.invalidateQueries({ queryKey: heroSectionKeys.lists() });
      push('success', 'Hero section created successfully.');
    },
  });
}

export function useUpdateHeroSection() {
  const queryClient = useQueryClient();
  const { push } = useNotificationStore();

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
  const { push } = useNotificationStore();

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

### UI Store

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
    ctaUrl: z
      .string()
      .url('Must be a valid URL')
      .min(1, 'CTA URL is required'),
    effectiveDate: z.string().min(1, 'Effective date is required'),
    expirationDate: z.string().min(1, 'Expiration date is required'),
    // image is optional in schema — uploaded separately via multipart
  })
  .refine(
    (data) => new Date(data.expirationDate) > new Date(data.effectiveDate),
    {
      message: 'Expiration date must be after effective date',
      path: ['expirationDate'],
    },
  );

export type HeroSectionFormValues = z.infer<typeof heroSectionSchema>;
```

### Form Component Pattern

```tsx
// Example: src/features/heroSection/components/HeroSectionForm.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { Button, Stack, Box } from '@mui/material';
import { TextFieldController } from '../../../shared/form/TextFieldController';
import { DatePickerController } from '../../../shared/form/DatePickerController';
import { ImageUpload } from '../../../shared/components/ImageUpload';
import { heroSectionSchema, type HeroSectionFormValues } from '../heroSection.schema';
import type { HeroSection } from '../heroSection.types';

interface HeroSectionFormProps {
  defaultValues?: Partial<HeroSectionFormValues>;
  onSubmit: (values: HeroSectionFormValues, imageFile?: File) => void;
  isSubmitting: boolean;
  /** If provided, "Cancel" button navigates back */
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
    <Box component="form" onSubmit={handleSubmit(handleFormSubmit)} noValidate>
      <Stack spacing={3}>
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
          inputProps={{ maxLength: 100 }}
          helperText="Max 100 characters"
        />
        <TextFieldController
          name="subheadline"
          control={control}
          label="Subheadline"
          required
          multiline
          rows={2}
          inputProps={{ maxLength: 200 }}
        />
        <TextFieldController
          name="ctaText"
          control={control}
          label="CTA Button Text"
          required
          inputProps={{ maxLength: 50 }}
        />
        <TextFieldController
          name="ctaUrl"
          control={control}
          label="CTA Button URL"
          required
          placeholder="https://..."
        />
        <Stack direction="row" spacing={2}>
          <DatePickerController
            name="effectiveDate"
            control={control}
            label="Effective Date"
            required
          />
          <DatePickerController
            name="expirationDate"
            control={control}
            label="Expiration Date"
            required
          />
        </Stack>

        <Stack direction="row" spacing={2} justifyContent="flex-end">
          <Button variant="outlined" onClick={onCancel}>
            Cancel
          </Button>
          <Button type="submit" variant="contained" loading={isSubmitting}>
            {defaultValues ? 'Update' : 'Create'}
          </Button>
        </Stack>
      </Stack>
    </Box>
  );
}
```

### RHF Controller Wrappers

```tsx
// src/shared/form/TextFieldController.tsx
import { Controller, type Control, type FieldPath, type FieldValues } from 'react-hook-form';
import { TextField, type TextFieldProps } from '@mui/material';

type TextFieldControllerProps<T extends FieldValues> = Omit<
  TextFieldProps,
  'name' | 'value' | 'onChange' | 'error' | 'helperText'
> & {
  name: FieldPath<T>;
  control: Control<T>;
};

export function TextFieldController<T extends FieldValues>({
  name,
  control,
  ...textFieldProps
}: TextFieldControllerProps<T>) {
  return (
    <Controller
      name={name}
      control={control}
      render={({ field, fieldState }) => (
        <TextField
          {...field}
          {...textFieldProps}
          error={!!fieldState.error}
          helperText={fieldState.error?.message ?? textFieldProps.helperText}
          fullWidth
        />
      )}
    />
  );
}
```

```tsx
// src/shared/form/SelectController.tsx
import { Controller, type Control, type FieldPath, type FieldValues } from 'react-hook-form';
import {
  FormControl,
  FormHelperText,
  InputLabel,
  MenuItem,
  Select,
  type SelectProps,
} from '@mui/material';

export interface SelectOption {
  value: string | number;
  label: string;
}

type SelectControllerProps<T extends FieldValues> = Omit<
  SelectProps,
  'name' | 'value' | 'onChange' | 'error'
> & {
  name: FieldPath<T>;
  control: Control<T>;
  label: string;
  options: SelectOption[];
  helperText?: string;
};

export function SelectController<T extends FieldValues>({
  name,
  control,
  label,
  options,
  helperText,
  ...selectProps
}: SelectControllerProps<T>) {
  return (
    <Controller
      name={name}
      control={control}
      render={({ field, fieldState }) => (
        <FormControl fullWidth error={!!fieldState.error} size="small">
          <InputLabel>{label}</InputLabel>
          <Select {...field} {...selectProps} label={label}>
            {options.map((opt) => (
              <MenuItem key={opt.value} value={opt.value}>
                {opt.label}
              </MenuItem>
            ))}
          </Select>
          <FormHelperText>
            {fieldState.error?.message ?? helperText}
          </FormHelperText>
        </FormControl>
      )}
    />
  );
}
```

---

## Notification Provider

```tsx
// src/shared/components/NotificationProvider.tsx
import { Alert, Snackbar, Stack } from '@mui/material';
import { useNotificationStore } from '../../store/notification.store';

export function NotificationProvider() {
  const { notifications, dismiss } = useNotificationStore();

  return (
    <Stack
      spacing={1}
      sx={{ position: 'fixed', bottom: 24, right: 24, zIndex: 9999, maxWidth: 400 }}
    >
      {notifications.map((n) => (
        <Snackbar
          key={n.id}
          open
          anchorOrigin={{ vertical: 'bottom', horizontal: 'right' }}
        >
          <Alert
            severity={n.type}
            onClose={() => dismiss(n.id)}
            variant="filled"
            elevation={6}
          >
            {n.message}
          </Alert>
        </Snackbar>
      ))}
    </Stack>
  );
}
```

---

## Status Chip Component

```tsx
// src/shared/components/StatusChip.tsx
import { Chip, type ChipProps } from '@mui/material';

type StatusColor = 'default' | 'primary' | 'secondary' | 'error' | 'info' | 'success' | 'warning';

interface StatusConfig {
  label: string;
  color: StatusColor;
}

interface StatusChipProps extends Omit<ChipProps, 'color' | 'label'> {
  status: string;
  /** Map of status value → display config. Provide this per-module. */
  statusMap: Record<string, StatusConfig>;
}

export function StatusChip({ status, statusMap, ...chipProps }: StatusChipProps) {
  const config = statusMap[status] ?? { label: status, color: 'default' as StatusColor };

  return (
    <Chip
      label={config.label}
      color={config.color}
      size="small"
      {...chipProps}
    />
  );
}

// Example usage in a module:
// const HERO_STATUS_MAP = {
//   DRAFT:   { label: 'Draft',   color: 'default' },
//   ACTIVE:  { label: 'Active',  color: 'success' },
//   EXPIRED: { label: 'Expired', color: 'error'   },
// };
// <StatusChip status={row.status} statusMap={HERO_STATUS_MAP} />
```

---

## Confirm Dialog Hook

```tsx
// src/shared/components/ConfirmDialog.tsx
import {
  Button,
  Dialog,
  DialogActions,
  DialogContent,
  DialogContentText,
  DialogTitle,
} from '@mui/material';
import { create } from 'zustand';

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

/** Hook to imperatively trigger a confirmation dialog */
export function useConfirm() {
  const { show } = useConfirmStore();

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

/** Render this once at the app root to display the dialog */
export function ConfirmDialog() {
  const { open, title, message, confirmLabel, cancelLabel, onConfirm, hide } =
    useConfirmStore();

  const handleConfirm = () => {
    onConfirm?.();
    hide();
  };

  return (
    <Dialog open={open} onClose={hide} maxWidth="xs" fullWidth>
      <DialogTitle>{title}</DialogTitle>
      <DialogContent>
        <DialogContentText>{message}</DialogContentText>
      </DialogContent>
      <DialogActions>
        <Button onClick={hide} color="inherit">
          {cancelLabel}
        </Button>
        <Button onClick={handleConfirm} color="error" variant="contained" autoFocus>
          {confirmLabel}
        </Button>
      </DialogActions>
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
  /**
   * Fetch paginated list of hero sections.
   */
  list: async (params: HeroSectionListParams): Promise<PaginatedResponse<HeroSection>> => {
    const { data } = await axiosInstance.get<PaginatedResponse<HeroSection>>(
      '/hero-sections',
      { params },
    );
    return data;
  },

  /**
   * Fetch a single hero section by ID.
   */
  getById: async (id: string): Promise<HeroSection> => {
    const { data } = await axiosInstance.get<HeroSection>(`/hero-sections/${id}`);
    return data;
  },

  /**
   * Create a new hero section with an optional image upload.
   * Uses multipart/form-data when an image file is provided.
   */
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

  /**
   * Update an existing hero section.
   */
  update: async (
    id: string,
    payload: HeroSectionUpdate,
    imageFile?: File,
  ): Promise<HeroSection> => {
    if (imageFile) {
      const formData = new FormData();
      formData.append('data', JSON.stringify(payload));
      formData.append('image', imageFile);
      const { data } = await axiosInstance.put<HeroSection>(
        `/hero-sections/${id}`,
        formData,
        { headers: { 'Content-Type': 'multipart/form-data' } },
      );
      return data;
    }
    const { data } = await axiosInstance.put<HeroSection>(`/hero-sections/${id}`, payload);
    return data;
  },

  /**
   * Delete a hero section by ID.
   */
  delete: async (id: string): Promise<void> => {
    await axiosInstance.delete(`/hero-sections/${id}`);
  },

  /**
   * Serve image by ID (returns a URL for use in <img src="..."> via Blob URL or direct endpoint).
   */
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

## Page Component Pattern

```tsx
// Example: src/features/heroSection/pages/HeroSectionListPage.tsx
import { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import {
  Box,
  Button,
  Card,
  Chip,
  IconButton,
  MenuItem,
  Select,
  Stack,
  Table,
  TableBody,
  TableCell,
  TableContainer,
  TableHead,
  TablePagination,
  TableRow,
  Tooltip,
} from '@mui/material';
import AddIcon from '@mui/icons-material/Add';
import EditIcon from '@mui/icons-material/Edit';
import DeleteIcon from '@mui/icons-material/Delete';
import { PageHeader } from '../../../shared/components/PageHeader';
import { StatusChip } from '../../../shared/components/StatusChip';
import { LoadingOverlay } from '../../../shared/components/LoadingOverlay';
import { EmptyState } from '../../../shared/components/EmptyState';
import { useConfirm } from '../../../shared/components/ConfirmDialog';
import { useHeroSections } from '../hooks/useHeroSections';
import { useDeleteHeroSection } from '../hooks/useHeroSectionMutations';
import type { HeroSectionListParams, HeroSectionStatus } from '../heroSection.types';

const HERO_STATUS_MAP = {
  DRAFT:   { label: 'Draft',   color: 'default' as const },
  ACTIVE:  { label: 'Active',  color: 'success' as const },
  EXPIRED: { label: 'Expired', color: 'error'   as const },
};

export function HeroSectionListPage() {
  const navigate = useNavigate();
  const confirm = useConfirm();

  const [params, setParams] = useState<HeroSectionListParams>({
    page: 0,
    size: 10,
    status: '',
  });

  const { data, isLoading, isError } = useHeroSections(params);
  const deleteMutation = useDeleteHeroSection();

  const handleDelete = async (id: string, headline: string) => {
    const confirmed = await confirm({
      title: 'Delete Hero Section',
      message: `Are you sure you want to delete "${headline}"? This action cannot be undone.`,
      confirmLabel: 'Delete',
    });
    if (confirmed) {
      deleteMutation.mutate(id);
    }
  };

  if (isLoading) return <LoadingOverlay />;
  if (isError) return <div>Failed to load hero sections.</div>;

  return (
    <Box>
      <PageHeader
        title="Hero Sections"
        breadcrumbs={[{ label: 'Dashboard', path: '/dashboard' }, { label: 'Hero Sections' }]}
        actions={
          <Button
            variant="contained"
            startIcon={<AddIcon />}
            onClick={() => navigate('/hero-section/create')}
          >
            Add Hero Section
          </Button>
        }
      />

      {/* Filters */}
      <Stack direction="row" spacing={2} sx={{ mb: 2 }}>
        <Select
          size="small"
          value={params.status ?? ''}
          onChange={(e) => setParams((p) => ({ ...p, status: e.target.value as HeroSectionStatus | '', page: 0 }))}
          displayEmpty
          sx={{ minWidth: 160 }}
        >
          <MenuItem value="">All Statuses</MenuItem>
          <MenuItem value="DRAFT">Draft</MenuItem>
          <MenuItem value="ACTIVE">Active</MenuItem>
          <MenuItem value="EXPIRED">Expired</MenuItem>
        </Select>
      </Stack>

      <Card>
        <TableContainer>
          <Table>
            <TableHead>
              <TableRow>
                <TableCell>Headline</TableCell>
                <TableCell>Effective Date</TableCell>
                <TableCell>Expiration Date</TableCell>
                <TableCell>Status</TableCell>
                <TableCell align="right">Actions</TableCell>
              </TableRow>
            </TableHead>
            <TableBody>
              {data?.content.length === 0 ? (
                <TableRow>
                  <TableCell colSpan={5}>
                    <EmptyState message="No hero sections found." />
                  </TableCell>
                </TableRow>
              ) : (
                data?.content.map((row) => (
                  <TableRow key={row.id} hover>
                    <TableCell>{row.headline}</TableCell>
                    <TableCell>{row.effectiveDate}</TableCell>
                    <TableCell>{row.expirationDate}</TableCell>
                    <TableCell>
                      <StatusChip status={row.status} statusMap={HERO_STATUS_MAP} />
                    </TableCell>
                    <TableCell align="right">
                      <Tooltip title="Edit">
                        <IconButton
                          size="small"
                          onClick={() => navigate(`/hero-section/${row.id}/edit`)}
                        >
                          <EditIcon fontSize="small" />
                        </IconButton>
                      </Tooltip>
                      <Tooltip title="Delete">
                        <IconButton
                          size="small"
                          color="error"
                          onClick={() => handleDelete(row.id, row.headline)}
                        >
                          <DeleteIcon fontSize="small" />
                        </IconButton>
                      </Tooltip>
                    </TableCell>
                  </TableRow>
                ))
              )}
            </TableBody>
          </Table>
        </TableContainer>
        <TablePagination
          component="div"
          count={data?.totalElements ?? 0}
          page={params.page ?? 0}
          rowsPerPage={params.size ?? 10}
          onPageChange={(_, page) => setParams((p) => ({ ...p, page }))}
          onRowsPerPageChange={(e) =>
            setParams((p) => ({ ...p, size: parseInt(e.target.value), page: 0 }))
          }
        />
      </Card>
    </Box>
  );
}
```
