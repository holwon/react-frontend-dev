---
name: form-validation
description: Form Handling and Validation — React Hook Form (register/Controller/useFormContext), Zod schema validation, async validation, field arrays (useFieldArray), form state (isDirty/isSubmitting), integration with UI component libraries. Use when building forms, implementing field validation, or handling complex form logic.
---

# Form Handling and Validation

## React Hook Form + Zod (Recommended Combination)

### Basic Setup

```tsx
// Path: src/features/user/components/CreateUserForm.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

// 1. Define Zod Schema (Single source of truth)
const createUserSchema = z.object({
  name: z.string().min(2, 'Name must be at least 2 characters').max(50, 'Name must not exceed 50 characters'),
  email: z.string().email('Please enter a valid email address'),
  age: z.number({ required_error: 'Age is required' }).min(18, 'Must be at least 18 years old').max(100),
  role: z.enum(['admin', 'user', 'viewer'], { message: 'Please select a valid role' }),
  bio: z.string().max(200).optional(),
});

// 2. Infer type from Schema (Single source of truth, no duplicate definitions)
type CreateUserFormData = z.infer<typeof createUserSchema>;

// 3. Component implementation
const CreateUserForm = ({ onSuccess }: { onSuccess: () => void }) => {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting, isDirty, isValid },
    reset,
  } = useForm<CreateUserFormData>({
    resolver: zodResolver(createUserSchema),
    defaultValues: {
      role: 'user',
    },
    mode: 'onBlur', // Trigger validation on blur
  });

  const onSubmit = async (data: CreateUserFormData) => {
    try {
      await userApi.create(data);
      reset();
      onSuccess();
    } catch (error) {
      // Handle server errors
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <input {...register('name')} placeholder="Name" />
        {errors.name && <span className="error">{errors.name.message}</span>}
      </div>

      <div>
        <input {...register('email')} type="email" placeholder="Email" />
        {errors.email && <span className="error">{errors.email.message}</span>}
      </div>

      <button type="submit" disabled={isSubmitting || !isDirty || !isValid}>
        {isSubmitting ? 'Submitting...' : 'Create User'}
      </button>
    </form>
  );
};
```

---

## Controller (Integration with Controlled UI Components)

Suitable for components that do not support native refs (e.g., Select, DatePicker, custom components).

```tsx
// Path: src/features/user/components/UserRoleForm.tsx
import { Controller, useForm } from 'react-hook-form';

const UserRoleForm = () => {
  const { control, handleSubmit } = useForm<FormData>({
    resolver: zodResolver(schema),
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* Integrate with custom Select component */}
      <Controller
        name="role"
        control={control}
        render={({ field, fieldState }) => (
          <CustomSelect
            value={field.value}
            onChange={field.onChange}
            onBlur={field.onBlur}
            error={fieldState.error?.message}
            options={ROLE_OPTIONS}
          />
        )}
      />

      {/* Integrate with DatePicker */}
      <Controller
        name="birthDate"
        control={control}
        render={({ field }) => (
          <DatePicker
            selected={field.value}
            onChange={field.onChange}
          />
        )}
      />
    </form>
  );
};
```

---

## useFieldArray (Dynamic Field Arrays)

```tsx
// Path: src/features/product/components/ProductTagsForm.tsx
import { useFieldArray, useForm } from 'react-hook-form';

interface FormData {
  tags: { name: string; color: string }[];
}

const ProductTagsForm = () => {
  const { register, control } = useForm<FormData>({
    defaultValues: { tags: [{ name: '', color: '#000000' }] },
  });

  const { fields, append, remove, move } = useFieldArray({
    control,
    name: 'tags',
  });

  return (
    <div>
      {fields.map((field, index) => (
        <div key={field.id}> {/* Must use field.id instead of array index */}
          <input {...register(`tags.${index}.name`)} placeholder="Tag name" />
          <input {...register(`tags.${index}.color`)} type="color" />
          <button type="button" onClick={() => remove(index)}>Delete</button>
        </div>
      ))}
      <button type="button" onClick={() => append({ name: '', color: '#000000' })}>
        Add Tag
      </button>
    </div>
  );
};
```

---

## useFormContext (Cross-Component Forms)

```tsx
// Path: src/features/checkout/components/CheckoutForm.tsx
import { FormProvider, useForm, useFormContext } from 'react-hook-form';

// Root component: Provides Form Context
const CheckoutForm = () => {
  const methods = useForm<CheckoutFormData>({ resolver: zodResolver(checkoutSchema) });

  return (
    <FormProvider {...methods}>
      <form onSubmit={methods.handleSubmit(onSubmit)}>
        <ShippingSection />    {/* Deeply nested components can access the form directly */}
        <PaymentSection />
        <OrderSummary />
      </form>
    </FormProvider>
  );
};

// Deeply nested component: Direct access to the parent form
const ShippingSection = () => {
  const { register, formState: { errors } } = useFormContext<CheckoutFormData>();

  return (
    <section>
      <input {...register('shipping.address')} />
      {errors.shipping?.address && <span>{errors.shipping.address.message}</span>}
    </section>
  );
};
```

---

## Zod Schema Advanced Usage

```typescript
// Conditional Validation
const schema = z.object({
  type: z.enum(['individual', 'company']),
  companyName: z.string().optional(),
}).refine(
  data => data.type !== 'company' || !!data.companyName,
  { message: 'Company name is required for company users', path: ['companyName'] }
);

// Async Validation (Server-side uniqueness check)
const usernameSchema = z.object({
  username: z.string().min(3).superRefine(async (val, ctx) => {
    const isTaken = await checkUsernameAvailability(val);
    if (isTaken) {
      ctx.addIssue({ code: 'custom', message: 'Username is already taken' });
    }
  }),
});

// Password Confirmation
const passwordSchema = z.object({
  password: z.string().min(8),
  confirmPassword: z.string(),
}).refine(
  data => data.password === data.confirmPassword,
  { message: 'Passwords do not match', path: ['confirmPassword'] }
);

// Transformation (String to Number)
const schema = z.object({
  age: z.string().transform(val => parseInt(val, 10)).pipe(z.number().min(18)),
});
```

---

## Form State Cheat Sheet

```tsx
const { formState } = useForm();

formState.isDirty          // Any field value differs from default
formState.isValid          // All fields pass validation
formState.isSubmitting     // Currently submitting (onSubmit is executing)
formState.isSubmitted      // Submission has been attempted
formState.isSubmitSuccessful // onSubmit completed successfully
formState.errors           // Field errors object
formState.dirtyFields      // Set of modified fields
```

---

## Common Anti-patterns

| Anti-pattern | Correct Approach |
|--------|---------|
| Manually managing `useState` for each field | Use React Hook Form |
| Handwritten validation logic (if/else spaghetti) | Use Zod schemas to centrally define validation rules |
| Duplicating form types and validation rules definitions | `z.infer<typeof schema>` as a single source of truth |
| `mode: 'onChange'` (Frequent re-renders) | Use `mode: 'onBlur'` or `mode: 'onSubmit'` |
| Ignoring the `isSubmitting` state | Submit button `disabled={isSubmitting}` |