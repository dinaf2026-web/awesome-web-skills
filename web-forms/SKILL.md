---
name: web-forms
description: Production form patterns for modern web apps — React Hook Form + Zod validation, Server Actions, multi-step forms, file uploads, and form UX that doesn't frustrate users.
origin: community
tags: [forms, react-hook-form, zod, server-actions, validation, ux]
---

# web-forms

Production form patterns for modern web apps. React Hook Form + Zod, Server Actions, multi-step flows, file uploads, and UX that doesn't frustrate users.

---

## 1. When to Use

Use this skill when:

- Building any form with more than one field
- You need client-side validation that mirrors server-side rules
- The form has async checks (username taken, email exists)
- You're wiring forms to Next.js Server Actions
- The flow is multi-step (wizard, checkout, onboarding)
- Users need to upload files with progress feedback

Do not use this skill for:
- Single-input search bars (uncontrolled input + debounce is fine)
- Read-only data tables that happen to have filters

---

## 2. Setup

### Install

```bash
npm install react-hook-form zod @hookform/resolvers
# shadcn/ui components used throughout:
npx shadcn-ui@latest add input label button textarea select
```

### Basic wiring

```tsx
// lib/schemas/contact.ts
import { z } from "zod"

export const contactSchema = z.object({
  name: z.string().min(2, "Name must be at least 2 characters"),
  email: z.string().email("Enter a valid email address"),
  message: z.string().min(10, "Message must be at least 10 characters"),
})

export type ContactFormValues = z.infer<typeof contactSchema>
```

```tsx
// components/forms/BasicForm.tsx
"use client"

import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { contactSchema, type ContactFormValues } from "@/lib/schemas/contact"

export function BasicForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
    reset,
  } = useForm<ContactFormValues>({
    resolver: zodResolver(contactSchema),
    defaultValues: { name: "", email: "", message: "" },
  })

  const onSubmit = async (data: ContactFormValues) => {
    // data is fully typed and validated
    await submitContact(data)
    reset()
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <input {...register("name")} />
      {errors.name && <p>{errors.name.message}</p>}
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "Sending…" : "Send"}
      </button>
    </form>
  )
}
```

---

## 3. Single Field Validation — Real Zod Schemas

```ts
// lib/schemas/fields.ts
import { z } from "zod"

// Email
export const emailSchema = z
  .string()
  .min(1, "Email is required")
  .email("Enter a valid email address")
  .max(254, "Email is too long")

// US phone (flexible — strips formatting before storing)
export const phoneSchema = z
  .string()
  .min(1, "Phone number is required")
  .transform((val) => val.replace(/\D/g, ""))
  .refine((val) => val.length === 10, "Enter a 10-digit US phone number")

// URL (requires https)
export const urlSchema = z
  .string()
  .url("Enter a valid URL")
  .refine((val) => val.startsWith("https://"), "URL must start with https://")

// Password with strength rules
export const passwordSchema = z
  .string()
  .min(8, "Password must be at least 8 characters")
  .refine((val) => /[A-Z]/.test(val), "Must contain an uppercase letter")
  .refine((val) => /[a-z]/.test(val), "Must contain a lowercase letter")
  .refine((val) => /[0-9]/.test(val), "Must contain a number")
  .refine(
    (val) => /[^A-Za-z0-9]/.test(val),
    "Must contain a special character"
  )

// Confirm password (used inside an object schema)
export const passwordConfirmSchema = z
  .object({
    password: passwordSchema,
    confirmPassword: z.string(),
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: "Passwords do not match",
    path: ["confirmPassword"],
  })

// US zip code
export const zipSchema = z
  .string()
  .regex(/^\d{5}(-\d{4})?$/, "Enter a valid ZIP code")

// Credit card (Luhn not checked — use Stripe Elements for real payments)
export const cardNumberSchema = z
  .string()
  .transform((val) => val.replace(/\s/g, ""))
  .refine((val) => /^\d{13,19}$/.test(val), "Enter a valid card number")
```

---

## 4. Full Form Example — Contact Form

```tsx
// components/forms/ContactForm.tsx
"use client"

import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { z } from "zod"
import { Input } from "@/components/ui/input"
import { Label } from "@/components/ui/label"
import { Textarea } from "@/components/ui/textarea"
import { Button } from "@/components/ui/button"
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select"
import { cn } from "@/lib/utils"

const contactSchema = z.object({
  name: z.string().min(2, "Name must be at least 2 characters"),
  email: z.string().email("Enter a valid email address"),
  subject: z.enum(["general", "support", "billing", "other"], {
    required_error: "Select a subject",
  }),
  message: z
    .string()
    .min(10, "Message must be at least 10 characters")
    .max(2000, "Message cannot exceed 2000 characters"),
  consent: z.literal(true, {
    errorMap: () => ({ message: "You must agree to continue" }),
  }),
})

type ContactFormValues = z.infer<typeof contactSchema>

interface FieldErrorProps {
  message?: string
  id: string
}

function FieldError({ message, id }: FieldErrorProps) {
  if (!message) return null
  return (
    <p id={id} role="alert" className="mt-1 text-sm text-destructive">
      {message}
    </p>
  )
}

interface FormFieldProps {
  label: string
  error?: string
  required?: boolean
  hint?: string
  children: (props: {
    id: string
    errorId: string
    hasError: boolean
  }) => React.ReactNode
}

function FormField({ label, error, required, hint, children }: FormFieldProps) {
  const id = label.toLowerCase().replace(/\s+/g, "-")
  const errorId = `${id}-error`
  const hintId = `${id}-hint`
  const hasError = Boolean(error)

  return (
    <div className="space-y-1.5">
      <Label
        htmlFor={id}
        className={cn(hasError && "text-destructive")}
      >
        {label}
        {required && (
          <span aria-hidden="true" className="ml-1 text-destructive">
            *
          </span>
        )}
      </Label>
      {hint && (
        <p id={hintId} className="text-sm text-muted-foreground">
          {hint}
        </p>
      )}
      {children({ id, errorId, hasError })}
      <FieldError message={error} id={errorId} />
    </div>
  )
}

export function ContactForm() {
  const {
    register,
    handleSubmit,
    setValue,
    watch,
    formState: { errors, isSubmitting, isSubmitSuccessful },
    reset,
  } = useForm<ContactFormValues>({
    resolver: zodResolver(contactSchema),
  })

  const messageValue = watch("message", "")

  const onSubmit = async (data: ContactFormValues) => {
    await new Promise((r) => setTimeout(r, 1500)) // replace with real call
    console.log(data)
    reset()
  }

  if (isSubmitSuccessful) {
    return (
      <div role="status" className="rounded-lg border border-green-200 bg-green-50 p-6 text-center">
        <p className="font-medium text-green-800">Message sent. We'll be in touch within one business day.</p>
      </div>
    )
  }

  return (
    <form
      onSubmit={handleSubmit(onSubmit)}
      noValidate
      aria-label="Contact form"
      className="space-y-6"
    >
      <div className="grid gap-6 sm:grid-cols-2">
        <FormField label="Full name" error={errors.name?.message} required>
          {({ id, errorId, hasError }) => (
            <Input
              id={id}
              autoComplete="name"
              aria-describedby={hasError ? errorId : undefined}
              aria-invalid={hasError}
              className={cn(hasError && "border-destructive focus-visible:ring-destructive")}
              {...register("name")}
            />
          )}
        </FormField>

        <FormField label="Email address" error={errors.email?.message} required>
          {({ id, errorId, hasError }) => (
            <Input
              id={id}
              type="email"
              autoComplete="email"
              aria-describedby={hasError ? errorId : undefined}
              aria-invalid={hasError}
              className={cn(hasError && "border-destructive focus-visible:ring-destructive")}
              {...register("email")}
            />
          )}
        </FormField>
      </div>

      <FormField label="Subject" error={errors.subject?.message} required>
        {({ id, errorId, hasError }) => (
          <Select
            onValueChange={(val) =>
              setValue("subject", val as ContactFormValues["subject"], {
                shouldValidate: true,
              })
            }
          >
            <SelectTrigger
              id={id}
              aria-describedby={hasError ? errorId : undefined}
              aria-invalid={hasError}
              className={cn(hasError && "border-destructive")}
            >
              <SelectValue placeholder="Choose a subject" />
            </SelectTrigger>
            <SelectContent>
              <SelectItem value="general">General inquiry</SelectItem>
              <SelectItem value="support">Technical support</SelectItem>
              <SelectItem value="billing">Billing</SelectItem>
              <SelectItem value="other">Other</SelectItem>
            </SelectContent>
          </Select>
        )}
      </FormField>

      <FormField
        label="Message"
        error={errors.message?.message}
        hint="Minimum 10 characters"
        required
      >
        {({ id, errorId, hintId, hasError }: any) => (
          <div className="relative">
            <Textarea
              id={id}
              rows={5}
              aria-describedby={[hintId, hasError ? errorId : ""].filter(Boolean).join(" ")}
              aria-invalid={hasError}
              className={cn(hasError && "border-destructive focus-visible:ring-destructive")}
              {...register("message")}
            />
            <span className="absolute bottom-2 right-3 text-xs text-muted-foreground">
              {messageValue.length}/2000
            </span>
          </div>
        )}
      </FormField>

      <div className="flex items-start gap-2">
        <input
          id="consent"
          type="checkbox"
          className="mt-0.5 h-4 w-4 rounded border-input"
          aria-invalid={Boolean(errors.consent)}
          aria-describedby={errors.consent ? "consent-error" : undefined}
          {...register("consent")}
        />
        <div className="space-y-1">
          <Label htmlFor="consent" className="text-sm font-normal leading-snug">
            I agree to the{" "}
            <a href="/privacy" className="underline underline-offset-2">
              privacy policy
            </a>{" "}
            and consent to being contacted.
          </Label>
          <FieldError message={errors.consent?.message} id="consent-error" />
        </div>
      </div>

      <Button type="submit" disabled={isSubmitting} className="w-full sm:w-auto">
        {isSubmitting ? (
          <>
            <span className="mr-2 inline-block h-4 w-4 animate-spin rounded-full border-2 border-current border-t-transparent" aria-hidden="true" />
            Sending…
          </>
        ) : (
          "Send message"
        )}
      </Button>
    </form>
  )
}
```

---

## 5. Server Actions Integration (Next.js 14 App Router)

### The Server Action

```ts
// app/actions/contact.ts
"use server"

import { z } from "zod"
import { contactSchema } from "@/lib/schemas/contact"

export type ActionState = {
  status: "idle" | "success" | "error"
  errors?: Partial<Record<keyof z.infer<typeof contactSchema>, string[]>>
  message?: string
}

export async function submitContactAction(
  _prev: ActionState,
  formData: FormData
): Promise<ActionState> {
  const raw = {
    name: formData.get("name"),
    email: formData.get("email"),
    subject: formData.get("subject"),
    message: formData.get("message"),
    consent: formData.get("consent") === "on",
  }

  const result = contactSchema.safeParse(raw)

  if (!result.success) {
    return {
      status: "error",
      errors: result.error.flatten().fieldErrors as ActionState["errors"],
    }
  }

  try {
    // await db.contact.create({ data: result.data })
    await new Promise((r) => setTimeout(r, 500))
    return { status: "success", message: "Message received." }
  } catch {
    return { status: "error", message: "Something went wrong. Try again." }
  }
}
```

### The Form Component

```tsx
// components/forms/ServerActionForm.tsx
"use client"

import { useActionState } from "react"
import { useEffect, useRef } from "react"
import { submitContactAction, type ActionState } from "@/app/actions/contact"
import { Input } from "@/components/ui/input"
import { Label } from "@/components/ui/label"
import { Textarea } from "@/components/ui/textarea"
import { Button } from "@/components/ui/button"

const initialState: ActionState = { status: "idle" }

export function ServerActionForm() {
  const [state, formAction, isPending] = useActionState(
    submitContactAction,
    initialState
  )
  const formRef = useRef<HTMLFormElement>(null)

  useEffect(() => {
    if (state.status === "success") {
      formRef.current?.reset()
    }
  }, [state.status])

  return (
    <form ref={formRef} action={formAction} noValidate className="space-y-4">
      {state.status === "error" && state.message && (
        <div role="alert" className="rounded-md border border-destructive/50 bg-destructive/10 p-3 text-sm text-destructive">
          {state.message}
        </div>
      )}

      {state.status === "success" && (
        <div role="status" className="rounded-md border border-green-500/50 bg-green-50 p-3 text-sm text-green-700">
          {state.message}
        </div>
      )}

      <div className="space-y-1.5">
        <Label htmlFor="name">Name</Label>
        <Input
          id="name"
          name="name"
          autoComplete="name"
          aria-invalid={Boolean(state.errors?.name)}
          aria-describedby={state.errors?.name ? "name-error" : undefined}
        />
        {state.errors?.name && (
          <p id="name-error" role="alert" className="text-sm text-destructive">
            {state.errors.name[0]}
          </p>
        )}
      </div>

      <div className="space-y-1.5">
        <Label htmlFor="email">Email</Label>
        <Input
          id="email"
          name="email"
          type="email"
          autoComplete="email"
          aria-invalid={Boolean(state.errors?.email)}
          aria-describedby={state.errors?.email ? "email-error" : undefined}
        />
        {state.errors?.email && (
          <p id="email-error" role="alert" className="text-sm text-destructive">
            {state.errors.email[0]}
          </p>
        )}
      </div>

      <div className="space-y-1.5">
        <Label htmlFor="message">Message</Label>
        <Textarea
          id="message"
          name="message"
          rows={4}
          aria-invalid={Boolean(state.errors?.message)}
          aria-describedby={state.errors?.message ? "message-error" : undefined}
        />
        {state.errors?.message && (
          <p id="message-error" role="alert" className="text-sm text-destructive">
            {state.errors.message[0]}
          </p>
        )}
      </div>

      <Button type="submit" disabled={isPending}>
        {isPending ? "Sending…" : "Send"}
      </Button>
    </form>
  )
}
```

---

## 6. Multi-Step Form Pattern

```tsx
// components/forms/MultiStepForm.tsx
"use client"

import { useState } from "react"
import { useForm, FormProvider } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { z } from "zod"
import { Button } from "@/components/ui/button"
import { Input } from "@/components/ui/input"
import { Label } from "@/components/ui/label"
import { cn } from "@/lib/utils"

// --- Schemas per step ---

const step1Schema = z.object({
  firstName: z.string().min(1, "Required"),
  lastName: z.string().min(1, "Required"),
  email: z.string().email("Enter a valid email"),
})

const step2Schema = z.object({
  company: z.string().min(1, "Required"),
  role: z.string().min(1, "Required"),
  teamSize: z.enum(["solo", "2-10", "11-50", "51+"], {
    required_error: "Select team size",
  }),
})

const step3Schema = z.object({
  password: z.string().min(8, "At least 8 characters"),
  confirmPassword: z.string(),
}).refine((d) => d.password === d.confirmPassword, {
  message: "Passwords do not match",
  path: ["confirmPassword"],
})

const fullSchema = step1Schema.merge(step2Schema).merge(step3Schema)
type FormValues = z.infer<typeof fullSchema>

const STEPS = [
  { label: "Personal", schema: step1Schema },
  { label: "Company", schema: step2Schema },
  { label: "Security", schema: step3Schema },
]

// --- Progress Indicator ---

function StepIndicator({ current, total, labels }: { current: number; total: number; labels: string[] }) {
  return (
    <nav aria-label="Form progress" className="mb-8">
      <ol className="flex items-center gap-2">
        {labels.map((label, i) => {
          const state =
            i < current ? "complete" : i === current ? "current" : "upcoming"
          return (
            <li key={label} className="flex items-center gap-2">
              <div
                aria-current={state === "current" ? "step" : undefined}
                className={cn(
                  "flex h-8 w-8 items-center justify-center rounded-full text-sm font-medium transition-colors",
                  state === "complete" && "bg-primary text-primary-foreground",
                  state === "current" && "border-2 border-primary text-primary",
                  state === "upcoming" && "border border-muted-foreground/40 text-muted-foreground"
                )}
              >
                {state === "complete" ? (
                  <svg className="h-4 w-4" viewBox="0 0 20 20" fill="currentColor" aria-hidden="true">
                    <path fillRule="evenodd" d="M16.707 5.293a1 1 0 010 1.414l-8 8a1 1 0 01-1.414 0l-4-4a1 1 0 011.414-1.414L8 12.586l7.293-7.293a1 1 0 011.414 0z" clipRule="evenodd" />
                  </svg>
                ) : (
                  i + 1
                )}
              </div>
              <span
                className={cn(
                  "hidden text-sm sm:block",
                  state === "current" ? "font-medium text-primary" : "text-muted-foreground"
                )}
              >
                {label}
              </span>
              {i < total - 1 && (
                <div
                  className={cn(
                    "h-px flex-1 min-w-[2rem] transition-colors",
                    i < current ? "bg-primary" : "bg-border"
                  )}
                  aria-hidden="true"
                />
              )}
            </li>
          )
        })}
      </ol>
    </nav>
  )
}

// --- Step Components ---

function Step1() {
  const { register, formState: { errors } } = useFormContext<FormValues>()
  return (
    <fieldset className="space-y-4">
      <legend className="sr-only">Personal information</legend>
      <div className="grid gap-4 sm:grid-cols-2">
        <div className="space-y-1.5">
          <Label htmlFor="firstName">First name</Label>
          <Input id="firstName" {...register("firstName")} aria-invalid={Boolean(errors.firstName)} />
          {errors.firstName && <p className="text-sm text-destructive">{errors.firstName.message}</p>}
        </div>
        <div className="space-y-1.5">
          <Label htmlFor="lastName">Last name</Label>
          <Input id="lastName" {...register("lastName")} aria-invalid={Boolean(errors.lastName)} />
          {errors.lastName && <p className="text-sm text-destructive">{errors.lastName.message}</p>}
        </div>
      </div>
      <div className="space-y-1.5">
        <Label htmlFor="email">Email</Label>
        <Input id="email" type="email" {...register("email")} aria-invalid={Boolean(errors.email)} />
        {errors.email && <p className="text-sm text-destructive">{errors.email.message}</p>}
      </div>
    </fieldset>
  )
}

function Step2() {
  const { register, setValue, watch, formState: { errors } } = useFormContext<FormValues>()
  return (
    <fieldset className="space-y-4">
      <legend className="sr-only">Company information</legend>
      <div className="space-y-1.5">
        <Label htmlFor="company">Company</Label>
        <Input id="company" {...register("company")} aria-invalid={Boolean(errors.company)} />
        {errors.company && <p className="text-sm text-destructive">{errors.company.message}</p>}
      </div>
      <div className="space-y-1.5">
        <Label htmlFor="role">Your role</Label>
        <Input id="role" {...register("role")} aria-invalid={Boolean(errors.role)} />
        {errors.role && <p className="text-sm text-destructive">{errors.role.message}</p>}
      </div>
      <div className="space-y-2" role="group" aria-labelledby="teamSize-label">
        <p id="teamSize-label" className="text-sm font-medium">Team size</p>
        {(["solo", "2-10", "11-50", "51+"] as const).map((size) => (
          <label key={size} className="flex cursor-pointer items-center gap-2">
            <input
              type="radio"
              value={size}
              {...register("teamSize")}
              className="h-4 w-4 text-primary"
            />
            <span className="text-sm">{size}</span>
          </label>
        ))}
        {errors.teamSize && <p className="text-sm text-destructive">{errors.teamSize.message}</p>}
      </div>
    </fieldset>
  )
}

function Step3() {
  const { register, formState: { errors } } = useFormContext<FormValues>()
  return (
    <fieldset className="space-y-4">
      <legend className="sr-only">Create password</legend>
      <div className="space-y-1.5">
        <Label htmlFor="password">Password</Label>
        <Input id="password" type="password" autoComplete="new-password" {...register("password")} aria-invalid={Boolean(errors.password)} />
        {errors.password && <p className="text-sm text-destructive">{errors.password.message}</p>}
      </div>
      <div className="space-y-1.5">
        <Label htmlFor="confirmPassword">Confirm password</Label>
        <Input id="confirmPassword" type="password" autoComplete="new-password" {...register("confirmPassword")} aria-invalid={Boolean(errors.confirmPassword)} />
        {errors.confirmPassword && <p className="text-sm text-destructive">{errors.confirmPassword.message}</p>}
      </div>
    </fieldset>
  )
}

// Import at top; using inline here for brevity
import { useFormContext } from "react-hook-form"

const STEP_COMPONENTS = [Step1, Step2, Step3]

// --- Main Component ---

export function MultiStepForm() {
  const [currentStep, setCurrentStep] = useState(0)

  const methods = useForm<FormValues>({
    resolver: zodResolver(fullSchema),
    mode: "onTouched",
    defaultValues: {
      firstName: "", lastName: "", email: "",
      company: "", role: "",
      password: "", confirmPassword: "",
    },
  })

  const { handleSubmit, trigger } = methods

  const goNext = async () => {
    const stepFields = Object.keys(STEPS[currentStep].schema.shape) as (keyof FormValues)[]
    const valid = await trigger(stepFields)
    if (valid) setCurrentStep((s) => s + 1)
  }

  const goBack = () => setCurrentStep((s) => s - 1)

  const onSubmit = async (data: FormValues) => {
    console.log("Final submission:", data)
  }

  const StepComponent = STEP_COMPONENTS[currentStep]

  return (
    <FormProvider {...methods}>
      <div className="mx-auto max-w-lg">
        <StepIndicator
          current={currentStep}
          total={STEPS.length}
          labels={STEPS.map((s) => s.label)}
        />

        <form onSubmit={handleSubmit(onSubmit)} noValidate className="space-y-6">
          <StepComponent />

          <div className="flex justify-between pt-2">
            {currentStep > 0 ? (
              <Button type="button" variant="outline" onClick={goBack}>
                Back
              </Button>
            ) : (
              <div />
            )}

            {currentStep < STEPS.length - 1 ? (
              <Button type="button" onClick={goNext}>
                Next
              </Button>
            ) : (
              <Button type="submit">Create account</Button>
            )}
          </div>
        </form>

        <p className="mt-4 text-center text-sm text-muted-foreground">
          Step {currentStep + 1} of {STEPS.length}
        </p>
      </div>
    </FormProvider>
  )
}
```

---

## 7. File Upload

```tsx
// components/forms/FileUpload.tsx
"use client"

import { useState, useRef, useCallback, DragEvent, ChangeEvent } from "react"
import { z } from "zod"
import { Button } from "@/components/ui/button"
import { cn } from "@/lib/utils"

// --- Validation ---

const MAX_FILE_SIZE = 5 * 1024 * 1024 // 5 MB
const ALLOWED_TYPES = ["image/jpeg", "image/png", "image/webp", "application/pdf"]

const fileSchema = z.object({
  size: z.number().max(MAX_FILE_SIZE, "File must be under 5 MB"),
  type: z.string().refine(
    (t) => ALLOWED_TYPES.includes(t),
    "Only JPEG, PNG, WebP, and PDF files are allowed"
  ),
})

// --- Types ---

interface UploadFile {
  id: string
  file: File
  preview?: string
  progress: number
  error?: string
  uploaded: boolean
}

// --- Helpers ---

function formatBytes(bytes: number) {
  if (bytes < 1024) return `${bytes} B`
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`
  return `${(bytes / (1024 * 1024)).toFixed(1)} MB`
}

function validateFile(file: File): string | null {
  const result = fileSchema.safeParse({ size: file.size, type: file.type })
  return result.success ? null : result.error.errors[0].message
}

// --- Simulated upload (replace with real implementation) ---

async function uploadFile(
  file: File,
  onProgress: (pct: number) => void
): Promise<string> {
  return new Promise((resolve, reject) => {
    let progress = 0
    const interval = setInterval(() => {
      progress += Math.random() * 20
      if (progress >= 100) {
        clearInterval(interval)
        onProgress(100)
        resolve(`https://cdn.example.com/${file.name}`)
      } else {
        onProgress(Math.round(progress))
      }
    }, 200)
  })
}

// --- Component ---

export function FileUpload({ maxFiles = 5 }: { maxFiles?: number }) {
  const [files, setFiles] = useState<UploadFile[]>([])
  const [isDragging, setIsDragging] = useState(false)
  const inputRef = useRef<HTMLInputElement>(null)

  const addFiles = useCallback(
    (incoming: FileList | File[]) => {
      const fileArray = Array.from(incoming)
      const remaining = maxFiles - files.length
      const toAdd = fileArray.slice(0, remaining)

      const newFiles: UploadFile[] = toAdd.map((file) => ({
        id: `${file.name}-${Date.now()}-${Math.random()}`,
        file,
        preview: file.type.startsWith("image/") ? URL.createObjectURL(file) : undefined,
        progress: 0,
        error: validateFile(file) ?? undefined,
        uploaded: false,
      }))

      setFiles((prev) => [...prev, ...newFiles])

      // Start upload for valid files immediately
      newFiles
        .filter((f) => !f.error)
        .forEach((uploadItem) => {
          uploadFile(uploadItem.file, (progress) => {
            setFiles((prev) =>
              prev.map((f) =>
                f.id === uploadItem.id ? { ...f, progress } : f
              )
            )
          })
            .then(() => {
              setFiles((prev) =>
                prev.map((f) =>
                  f.id === uploadItem.id ? { ...f, uploaded: true, progress: 100 } : f
                )
              )
            })
            .catch(() => {
              setFiles((prev) =>
                prev.map((f) =>
                  f.id === uploadItem.id ? { ...f, error: "Upload failed. Try again." } : f
                )
              )
            })
        })
    },
    [files.length, maxFiles]
  )

  const onDrop = (e: DragEvent<HTMLDivElement>) => {
    e.preventDefault()
    setIsDragging(false)
    if (e.dataTransfer.files) addFiles(e.dataTransfer.files)
  }

  const onDragOver = (e: DragEvent<HTMLDivElement>) => {
    e.preventDefault()
    setIsDragging(true)
  }

  const onDragLeave = () => setIsDragging(false)

  const onInputChange = (e: ChangeEvent<HTMLInputElement>) => {
    if (e.target.files) addFiles(e.target.files)
    e.target.value = "" // allow re-selecting the same file
  }

  const removeFile = (id: string) => {
    setFiles((prev) => {
      const file = prev.find((f) => f.id === id)
      if (file?.preview) URL.revokeObjectURL(file.preview)
      return prev.filter((f) => f.id !== id)
    })
  }

  const atLimit = files.length >= maxFiles

  return (
    <div className="space-y-4">
      {/* Drop zone */}
      <div
        role="button"
        tabIndex={0}
        aria-label="Upload files — click or drag and drop"
        aria-disabled={atLimit}
        onDrop={onDrop}
        onDragOver={onDragOver}
        onDragLeave={onDragLeave}
        onClick={() => !atLimit && inputRef.current?.click()}
        onKeyDown={(e) => {
          if ((e.key === "Enter" || e.key === " ") && !atLimit) {
            inputRef.current?.click()
          }
        }}
        className={cn(
          "flex cursor-pointer flex-col items-center justify-center gap-3 rounded-lg border-2 border-dashed p-8 text-center transition-colors",
          isDragging && !atLimit
            ? "border-primary bg-primary/5"
            : "border-muted-foreground/25 hover:border-primary/50",
          atLimit && "cursor-not-allowed opacity-50"
        )}
      >
        <svg className="h-8 w-8 text-muted-foreground" fill="none" viewBox="0 0 24 24" stroke="currentColor" aria-hidden="true">
          <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={1.5} d="M3 16.5v2.25A2.25 2.25 0 005.25 21h13.5A2.25 2.25 0 0021 18.75V16.5m-13.5-9L12 3m0 0l4.5 4.5M12 3v13.5" />
        </svg>
        <div>
          <p className="font-medium text-sm">
            {isDragging ? "Drop files here" : "Click to upload or drag and drop"}
          </p>
          <p className="mt-1 text-xs text-muted-foreground">
            JPEG, PNG, WebP, PDF — max 5 MB each — up to {maxFiles} files
          </p>
        </div>
        <input
          ref={inputRef}
          type="file"
          multiple
          accept={ALLOWED_TYPES.join(",")}
          onChange={onInputChange}
          className="sr-only"
          tabIndex={-1}
        />
      </div>

      {/* File list */}
      {files.length > 0 && (
        <ul className="space-y-2" aria-label="Uploaded files">
          {files.map((item) => (
            <li
              key={item.id}
              className={cn(
                "flex items-center gap-3 rounded-lg border p-3",
                item.error && "border-destructive/50 bg-destructive/5"
              )}
            >
              {/* Preview or icon */}
              <div className="h-10 w-10 shrink-0 overflow-hidden rounded">
                {item.preview ? (
                  <img src={item.preview} alt="" className="h-full w-full object-cover" />
                ) : (
                  <div className="flex h-full w-full items-center justify-center bg-muted text-xs font-medium uppercase text-muted-foreground">
                    {item.file.name.split(".").pop()}
                  </div>
                )}
              </div>

              {/* File info */}
              <div className="min-w-0 flex-1">
                <p className="truncate text-sm font-medium">{item.file.name}</p>
                <p className="text-xs text-muted-foreground">{formatBytes(item.file.size)}</p>

                {/* Progress bar */}
                {!item.error && !item.uploaded && (
                  <div className="mt-1.5">
                    <div
                      role="progressbar"
                      aria-valuenow={item.progress}
                      aria-valuemin={0}
                      aria-valuemax={100}
                      aria-label={`Uploading ${item.file.name}`}
                      className="h-1.5 w-full overflow-hidden rounded-full bg-muted"
                    >
                      <div
                        className="h-full bg-primary transition-all duration-200"
                        style={{ width: `${item.progress}%` }}
                      />
                    </div>
                  </div>
                )}

                {item.uploaded && (
                  <p className="mt-0.5 text-xs text-green-600">Uploaded</p>
                )}

                {item.error && (
                  <p role="alert" className="mt-0.5 text-xs text-destructive">
                    {item.error}
                  </p>
                )}
              </div>

              {/* Remove */}
              <button
                type="button"
                onClick={() => removeFile(item.id)}
                aria-label={`Remove ${item.file.name}`}
                className="shrink-0 rounded p-1 text-muted-foreground hover:text-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring"
              >
                <svg className="h-4 w-4" viewBox="0 0 20 20" fill="currentColor" aria-hidden="true">
                  <path fillRule="evenodd" d="M4.293 4.293a1 1 0 011.414 0L10 8.586l4.293-4.293a1 1 0 111.414 1.414L11.414 10l4.293 4.293a1 1 0 01-1.414 1.414L10 11.414l-4.293 4.293a1 1 0 01-1.414-1.414L8.586 10 4.293 5.707a1 1 0 010-1.414z" clipRule="evenodd" />
                </svg>
              </button>
            </li>
          ))}
        </ul>
      )}
    </div>
  )
}
```

---

## 8. Async Validation — Username Availability

```tsx
// components/forms/UsernameField.tsx
"use client"

import { useState, useEffect, useRef } from "react"
import { useFormContext } from "react-hook-form"
import { Input } from "@/components/ui/input"
import { Label } from "@/components/ui/label"
import { cn } from "@/lib/utils"

// Simulated API check — replace with real fetch
async function checkUsernameAvailable(username: string): Promise<boolean> {
  await new Promise((r) => setTimeout(r, 400))
  const taken = ["admin", "user", "dina", "test"]
  return !taken.includes(username.toLowerCase())
}

type AsyncState = "idle" | "checking" | "available" | "taken" | "error"

export function UsernameField() {
  const { register, watch, setError, clearErrors, formState: { errors } } =
    useFormContext()

  const [asyncState, setAsyncState] = useState<AsyncState>("idle")
  const timerRef = useRef<ReturnType<typeof setTimeout>>()
  const latestRef = useRef<string>("")

  const username = watch("username", "")

  useEffect(() => {
    clearTimeout(timerRef.current)

    if (username.length < 3) {
      setAsyncState("idle")
      clearErrors("username")
      return
    }

    setAsyncState("checking")
    latestRef.current = username

    timerRef.current = setTimeout(async () => {
      const current = username
      try {
        const available = await checkUsernameAvailable(current)
        // Ignore stale responses
        if (latestRef.current !== current) return

        if (available) {
          setAsyncState("available")
          clearErrors("username")
        } else {
          setAsyncState("taken")
          setError("username", {
            type: "manual",
            message: `"${current}" is already taken`,
          })
        }
      } catch {
        if (latestRef.current !== current) return
        setAsyncState("error")
        setError("username", {
          type: "manual",
          message: "Could not check availability. Try again.",
        })
      }
    }, 500)

    return () => clearTimeout(timerRef.current)
  }, [username, setError, clearErrors])

  const hasError = Boolean(errors.username)
  const errorId = "username-error"
  const hintId = "username-hint"

  return (
    <div className="space-y-1.5">
      <Label htmlFor="username">Username</Label>
      <p id={hintId} className="text-xs text-muted-foreground">
        3–20 characters, letters and numbers only
      </p>

      <div className="relative">
        <Input
          id="username"
          autoComplete="username"
          aria-invalid={hasError}
          aria-describedby={[hintId, hasError ? errorId : ""].filter(Boolean).join(" ")}
          className={cn(
            "pr-9",
            hasError && "border-destructive focus-visible:ring-destructive",
            asyncState === "available" && "border-green-500 focus-visible:ring-green-500"
          )}
          {...register("username", {
            required: "Username is required",
            minLength: { value: 3, message: "At least 3 characters" },
            maxLength: { value: 20, message: "At most 20 characters" },
            pattern: {
              value: /^[a-zA-Z0-9_]+$/,
              message: "Letters, numbers, and underscores only",
            },
          })}
        />

        {/* Status indicator */}
        <div className="absolute right-2.5 top-1/2 -translate-y-1/2" aria-hidden="true">
          {asyncState === "checking" && (
            <svg className="h-4 w-4 animate-spin text-muted-foreground" fill="none" viewBox="0 0 24 24">
              <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4" />
              <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z" />
            </svg>
          )}
          {asyncState === "available" && (
            <svg className="h-4 w-4 text-green-500" viewBox="0 0 20 20" fill="currentColor">
              <path fillRule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z" clipRule="evenodd" />
            </svg>
          )}
          {(asyncState === "taken" || asyncState === "error") && (
            <svg className="h-4 w-4 text-destructive" viewBox="0 0 20 20" fill="currentColor">
              <path fillRule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zM8.707 7.293a1 1 0 00-1.414 1.414L8.586 10l-1.293 1.293a1 1 0 101.414 1.414L10 11.414l1.293 1.293a1 1 0 001.414-1.414L11.414 10l1.293-1.293a1 1 0 00-1.414-1.414L10 8.586 8.707 7.293z" clipRule="evenodd" />
            </svg>
          )}
        </div>
      </div>

      {/* Status text — screen-reader accessible */}
      <div aria-live="polite" aria-atomic="true" className="text-sm">
        {asyncState === "checking" && (
          <p className="text-muted-foreground">Checking availability…</p>
        )}
        {asyncState === "available" && (
          <p className="text-green-600">"{username}" is available</p>
        )}
        {hasError && (
          <p id={errorId} role="alert" className="text-destructive">
            {String(errors.username?.message)}
          </p>
        )}
      </div>
    </div>
  )
}
```

---

## 9. Form UX Rules

### Label placement

```tsx
// Always above the input — never floating-only or placeholder-only
<div className="space-y-1.5">
  <Label htmlFor="email">Email address</Label>
  <Input id="email" type="email" placeholder="you@example.com" />
  {/* Placeholder is supplemental context, not the label */}
</div>
```

### Inline errors — show immediately after blur, not only on submit

```tsx
// In useForm config:
const form = useForm({
  mode: "onTouched", // validate on blur, then on every change after first touch
})
```

### Helper text — always visible, above the error

```tsx
<div className="space-y-1.5">
  <Label htmlFor="password">Password</Label>
  <p id="password-hint" className="text-xs text-muted-foreground">
    At least 8 characters with a number and symbol
  </p>
  <Input
    id="password"
    type="password"
    aria-describedby="password-hint password-error"
  />
  <p id="password-error" role="alert" className="text-sm text-destructive">
    {/* error here */}
  </p>
</div>
```

### Disabled vs loading states

```tsx
// Disabled: user cannot interact — convey why
<Button disabled aria-disabled="true">
  Save changes
</Button>

// Loading: action in progress — show spinner + label
<Button disabled>
  <Loader2 className="mr-2 h-4 w-4 animate-spin" aria-hidden="true" />
  <span>Saving…</span>
</Button>
```

### Auto-focus — first field on mount, first error after submit

```tsx
// Auto-focus first field
<Input id="name" autoFocus {...register("name")} />

// Focus first error after failed submit
const onInvalid = () => {
  const firstError = Object.keys(errors)[0]
  const el = document.getElementById(firstError)
  el?.focus()
}

<form onSubmit={handleSubmit(onSubmit, onInvalid)}>
```

### Required field marking

```tsx
// Mark required with both visual asterisk and text for screen readers
<Label htmlFor="email">
  Email
  <span aria-hidden="true" className="ml-1 text-destructive">*</span>
</Label>
// At form level:
<p className="text-xs text-muted-foreground">
  Fields marked <span aria-hidden="true">*</span>
  <span className="sr-only">with an asterisk</span> are required.
</p>
```

---

## 10. Accessibility

### aria-describedby pattern

```tsx
// Wire hint + error to the input so screen readers announce both
<div className="space-y-1.5">
  <Label htmlFor="phone">Phone number</Label>
  <p id="phone-hint" className="text-xs text-muted-foreground">
    Format: (555) 555-5555
  </p>
  <Input
    id="phone"
    type="tel"
    aria-describedby={
      [
        "phone-hint",
        errors.phone ? "phone-error" : "",
      ]
        .filter(Boolean)
        .join(" ")
    }
    aria-invalid={Boolean(errors.phone)}
  />
  {errors.phone && (
    <p id="phone-error" role="alert" className="text-sm text-destructive">
      {errors.phone.message}
    </p>
  )}
</div>
```

### fieldset + legend for grouped controls

```tsx
// Use fieldset/legend for radio groups, checkboxes, and related fields
<fieldset className="space-y-3">
  <legend className="text-sm font-medium">Notification preferences</legend>
  {["email", "sms", "push"].map((type) => (
    <label key={type} className="flex items-center gap-2 cursor-pointer">
      <input
        type="checkbox"
        value={type}
        {...register("notifications")}
        className="h-4 w-4 rounded"
      />
      <span className="text-sm capitalize">{type}</span>
    </label>
  ))}
</fieldset>
```

### Keyboard navigation

```tsx
// Ensure custom controls are keyboard-reachable
// Select, combobox, date pickers — use shadcn/ui which handles this

// For custom dropdown:
<div
  role="combobox"
  aria-expanded={isOpen}
  aria-haspopup="listbox"
  aria-controls="options-list"
  tabIndex={0}
  onKeyDown={(e) => {
    if (e.key === "Enter" || e.key === " ") toggleOpen()
    if (e.key === "Escape") closeMenu()
    if (e.key === "ArrowDown") focusNext()
    if (e.key === "ArrowUp") focusPrev()
  }}
>
  {/* trigger */}
</div>
<ul
  id="options-list"
  role="listbox"
  aria-label="Select an option"
>
  {options.map((opt) => (
    <li
      key={opt.value}
      role="option"
      aria-selected={opt.value === selected}
      tabIndex={-1}
    >
      {opt.label}
    </li>
  ))}
</ul>
```

### Live regions for async feedback

```tsx
// Announce status changes without moving focus
<div aria-live="polite" aria-atomic="true" className="sr-only">
  {submitStatus === "success" && "Form submitted successfully."}
  {submitStatus === "error" && "Submission failed. Review errors below."}
</div>
```

### Error summary for long forms

```tsx
// When there are multiple errors, show a summary at the top and move focus to it
function ErrorSummary({ errors }: { errors: Record<string, any> }) {
  const ref = useRef<HTMLDivElement>(null)
  const errorList = Object.entries(errors)

  useEffect(() => {
    if (errorList.length > 0) {
      ref.current?.focus()
    }
  }, [errorList.length])

  if (errorList.length === 0) return null

  return (
    <div
      ref={ref}
      tabIndex={-1}
      role="alert"
      aria-labelledby="error-summary-title"
      className="rounded-lg border border-destructive/50 bg-destructive/10 p-4 focus-visible:outline-none"
    >
      <h2 id="error-summary-title" className="font-medium text-destructive">
        There {errorList.length === 1 ? "is" : "are"} {errorList.length}{" "}
        {errorList.length === 1 ? "error" : "errors"} in this form
      </h2>
      <ul className="mt-2 space-y-1 text-sm text-destructive">
        {errorList.map(([field, error]) => (
          <li key={field}>
            <a href={`#${field}`} className="underline underline-offset-2">
              {String(error?.message)}
            </a>
          </li>
        ))}
      </ul>
    </div>
  )
}
```

---

## 11. Anti-Patterns

### Placeholder-only labels — never do this

```tsx
// WRONG: placeholder disappears on type; screen readers may not announce it
<Input placeholder="Email address" />

// CORRECT: visible label always present
<div className="space-y-1.5">
  <Label htmlFor="email">Email address</Label>
  <Input id="email" placeholder="you@example.com" />
</div>
```

### Errors only on submit — frustrates users

```tsx
// WRONG: user finds out about all errors at once, after investing effort
const form = useForm({ mode: "onSubmit" })

// CORRECT: validate on blur so users get feedback field by field
const form = useForm({ mode: "onTouched" })
```

### No loading feedback — users click multiple times

```tsx
// WRONG: button does nothing visible during async
<Button onClick={handleSubmit(onSubmit)}>Save</Button>

// CORRECT: disable and show spinner
<Button type="submit" disabled={isSubmitting}>
  {isSubmitting ? (
    <>
      <span className="mr-2 inline-block h-4 w-4 animate-spin rounded-full border-2 border-current border-t-transparent" aria-hidden="true" />
      Saving…
    </>
  ) : (
    "Save"
  )}
</Button>
```

### Generic error messages — tell users what to do

```tsx
// WRONG: user doesn't know how to fix this
z.string().email("Invalid email")

// CORRECT: specific and actionable
z.string().email("Enter a valid email address like you@example.com")
```

### Resetting errors on re-render instead of on user action

```tsx
// WRONG: clear all errors when component re-renders
useEffect(() => { clearErrors() }, [someState])

// CORRECT: clear specific field error when user fixes it — RHF does this
// automatically when mode is "onTouched" or "onChange"
```

### Not preserving data on back navigation in multi-step forms

```tsx
// WRONG: remounting the form on step change wipes values
{currentStep === 0 && <Step1 />}
{currentStep === 1 && <Step2 />}

// CORRECT: use FormProvider + a single useForm at the parent level
// and keep all steps mounted but hidden, or store values in the
// parent form state via getValues() before advancing
const { getValues } = useForm()
const goNext = async () => {
  const valid = await trigger(currentStepFields)
  if (valid) {
    const snapshot = getValues() // values are preserved in RHF internal state
    setCurrentStep((s) => s + 1)
  }
}
```

### Not using noValidate on the form element

```tsx
// WRONG: browser native validation interferes with RHF's validation UI
<form onSubmit={handleSubmit(onSubmit)}>

// CORRECT: disable native validation, use RHF + Zod exclusively
<form onSubmit={handleSubmit(onSubmit)} noValidate>
```

### Misusing disabled for read-only display

```tsx
// WRONG: disabled inputs are excluded from form submission and hard to style
<Input disabled value={user.email} />

// CORRECT: use readOnly when the value should be submitted but not editable
<Input readOnly value={user.email} aria-readonly="true" />
```
