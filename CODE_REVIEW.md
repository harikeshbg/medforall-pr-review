---
title: "PR Review — Quick Add Patient (QuickAddPatient.tsx)"
assignment: "Assignment 5 — Code Review Exercise"
---

# PR Review: QuickAddPatient.tsx (Medium-Length, Easy to Read)

## Summary (what’s wrong overall)
This PR is a decent start, but it’s **not production-ready**. The biggest problems are:
- **No TypeScript safety** (everything is `any`/implicit).
- **Bad API error handling** (failures can look like success).
- **SSN is collected unsafely** (serious security/compliance issue).
- **Accessibility is missing** (labels + error announcement).
- **Derived state bug risk** (writing `age` into `formData` via `useEffect`).

---

## 1) Critical issues (must fix before merge)

### 1.1 Add TypeScript types (props, state, events)
**What:** `onSuccess`, `formData`, `handleChange`, `handleSubmit` are untyped; `formData` starts as `{}`.  
**Why it matters:** You can submit malformed payloads, get runtime undefined errors, and lose all TS benefits.  
**How to fix (example):**
```ts
type QuickAddPatientProps = {
  onSuccess: (patient: { id: string }) => void;
};

type PatientCreatePayload = {
  firstName: string;
  lastName: string;
  dob: string;   // YYYY-MM-DD from input[type=date]
  email: string;
  phone?: string;
  insuranceId?: string;
};

const [formData, setFormData] = useState<PatientCreatePayload>({
  firstName: "",
  lastName: "",
  dob: "",
  email: "",
  phone: "",
  insuranceId: "",
});
```

---

### 1.2 Fix `fetch()` (headers + `res.ok` + safe error handling)
**What:** Missing `Content-Type`, no `res.ok` check, always calls `res.json()`.  
**Why it matters:** On a 400/500 response, the UI may still treat it like success and redirect incorrectly.  
**How to fix:**
```ts
const res = await fetch("/api/patients", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(payload),
});

if (!res.ok) {
  const msg = await res.text().catch(() => "");
  throw new Error(msg || `Failed to create patient (${res.status})`);
}

const data: { id: string } = await res.json();
```

---

### 1.3 Remove SSN from this “Quick Add” form
**What:** SSN is captured in plaintext and stored in React state.  
**Why it matters:** This is **high-risk sensitive data**. Even for mock apps, reviewers will flag this immediately because it creates real-world security/compliance concerns (logging, analytics, autofill, accidental exposure).  
**How to fix:** Remove SSN from this component. If SSN is required, move it to a dedicated, secured workflow (masking, strict access control, audit logging, backend encryption, etc.).

---

### 1.4 Do not store `age` inside `formData` via `useEffect`
**What:** The code writes `{...formData, age: ...}` whenever DOB changes.  
**Why it matters:**  
- `age` is derived data (DOB is the source of truth).  
- The `useEffect` pattern can cause stale state overwrites.  
**How to fix:** Compute `age` for display only and do not submit it.
```ts
const age = formData.dob ? calculateAge(formData.dob) : undefined;
```

---

### 1.5 Loading state should be handled with `finally` and double-submit guard
**What:** `setLoading(false)` is duplicated in `try` and `catch`, and there’s no early return when already loading.  
**Why it matters:** Duplicate submissions can create duplicate patients. Loading state can become fragile during refactors.  
**How to fix:**
```ts
if (loading) return;
setLoading(true);
setError(null);
try {
  ...
} catch (e) {
  ...
} finally {
  setLoading(false);
}
```

---

## 2) Improvements (should fix)

### 2.1 Better validation + clearer feedback
**What:** Only checks presence; no DOB future-date validation; no field-level errors.  
**Why it matters:** Bad data goes to backend, causing avoidable failures and bad UX.  
**How to fix:**  
- Validate DOB is not in the future  
- Optionally validate reasonable age range  
- Show field-level error messages (not just one generic string)

---

### 2.2 Use controlled inputs
**What:** Inputs don’t have `value`, only `onChange`.  
**Why it matters:** Harder to reset form, harder to keep UI/state consistent, harder to test.  
**How to fix:** Bind `value={formData.firstName}` etc. and keep state initialized to empty strings.

---

### 2.3 Use functional state updates in `handleChange`
**What:** `setFormData({ ...formData, ... })` can use stale `formData` during rapid updates.  
**Why it matters:** Rare but real bugs; functional update is the correct React pattern.  
**How to fix:**
```ts
setFormData(prev => ({ ...prev, [name]: value }));
```

---

## 3) Suggestions (nice to have)

### 3.1 Extract API call to a helper
**What:** fetch logic is inside the component.  
**Why it matters:** Less reusable, harder to test.  
**How:** `lib/patients.ts -> createPatient(payload)`.

### 3.2 Improve UX with a success message / reset
**What:** After submit it redirects; if this ever changes, there’s no clear “success” behavior.  
**How:** toast or inline confirmation + reset.

---

## 4) Security concerns (call these out explicitly)
- **SSN in client-side state is a red flag** (remove from quick add).
- **Raw `err.message` shown to users** can leak internals. Prefer a user-safe message and log detailed errors separately.
- If auth is cookie-based, consider **CSRF protections** for POST endpoints (depends on project auth model).

---

## 5) Performance concerns (small but real)
- Writing derived `age` into state causes extra renders; compute it inline instead.
- Object spreading on each keystroke is fine for this size, but use functional updates for correctness.

---

## 6) Accessibility issues (important)

### 6.1 Labels not associated with inputs
**What:** No `id` on input and no `htmlFor` on label.  
**Why it matters:** Screen readers and click-to-focus behavior break.  
**Fix:**
```tsx
<label htmlFor="firstName">First Name *</label>
<input id="firstName" name="firstName" ... />
```

### 6.2 Error not announced to assistive tech
**What:** error is a red `<div>` only.  
**Fix:** Use `role="alert"` or `aria-live`.
```tsx
{error && <div role="alert">{error}</div>}
```

---

## 7) Testing recommendations (what to test)
- **Validation:** missing required fields should block submit + show errors.
- **API success:** sends correct request (method, headers, body) and navigates to `/patients/:id`.
- **API failure:** non-2xx response shows error and does not navigate.
- **Loading:** button disables and double-submit is prevented.
- **Basic a11y:** labels are linked and error uses `role="alert"`.

---

## Merge recommendation
**Request changes.** Fix TypeScript typing, fetch correctness, remove SSN handling, remove derived-age state, and address a11y issues before merging.
