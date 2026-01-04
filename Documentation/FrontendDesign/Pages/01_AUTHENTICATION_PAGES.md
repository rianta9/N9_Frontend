# Authentication Pages

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 2.0 |
| Last Updated | 2026-01-04 |
| Status | Approved |
| Owner | Frontend Engineering Team |
| Review Cycle | Quarterly |
| Backend Reference | [02_USERS_COMPONENT.md](../../../../Backend/N9/Documentation/BackendDesign/Components/02_USERS_COMPONENT.md) |

---

## 2. Overview

### 2.1 Purpose
This document specifies the **authentication pages** for the N9 platform, including login, registration, password reset, email verification, MFA flows, and OAuth integration. Aligned with backend Users component for secure authentication.

### 2.2 User Flows

```
┌─────────────────────────────────────────────────────────────────┐
│                    AUTHENTICATION FLOWS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  New User Flow:                                                  │
│  Landing → Register → Email Verify → Complete Profile → Home     │
│                                                                  │
│  Returning User Flow:                                            │
│  Landing → Login → Home                                          │
│  Landing → Login → MFA Verify → Home                            │
│                                                                  │
│  Password Recovery Flow:                                         │
│  Login → Forgot Password → Email Sent → Reset Password → Login   │
│                                                                  │
│  OAuth Flow:                                                     │
│  Landing → OAuth Provider → Callback → Home/Complete Profile     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 Related Backend APIs

#### Authentication Endpoints
| Endpoint | Method | Purpose |
|----------|--------|----------|
| `/api/v1/auth/register` | POST | User registration |
| `/api/v1/auth/login` | POST | User login |
| `/api/v1/auth/logout` | POST | User logout |
| `/api/v1/auth/refresh` | POST | Refresh access token |
| `/api/v1/auth/forgot-password` | POST | Request password reset |
| `/api/v1/auth/reset-password` | POST | Reset password |
| `/api/v1/auth/verify-email` | POST | Verify email address |
| `/api/v1/auth/resend-verification` | POST | Resend verification email |

#### MFA Endpoints
| Endpoint | Method | Purpose |
|----------|--------|----------|
| `/api/v1/auth/mfa/setup` | POST | Setup MFA |
| `/api/v1/auth/mfa/verify` | POST | Verify MFA code |
| `/api/v1/auth/mfa/recovery` | POST | Use recovery code |
| `/api/v1/auth/mfa/disable` | POST | Disable MFA |

#### OAuth Endpoints
| Endpoint | Method | Purpose |
|----------|--------|----------|
| `/api/v1/auth/oauth/google` | GET | Google OAuth redirect |
| `/api/v1/auth/oauth/google/callback` | GET | Google OAuth callback |
| `/api/v1/auth/oauth/apple` | GET | Apple OAuth redirect |
| `/api/v1/auth/oauth/apple/callback` | GET | Apple OAuth callback |

---

## 3. Routes Configuration

```typescript
// Auth routes (GuestGuard - redirect if authenticated)
const authRoutes = [
  { path: '/auth/login', element: <LoginPage /> },
  { path: '/auth/register', element: <RegisterPage /> },
  { path: '/auth/forgot-password', element: <ForgotPasswordPage /> },
  { path: '/auth/reset-password', element: <ResetPasswordPage /> },
  { path: '/auth/verify-email', element: <VerifyEmailPage /> },
  { path: '/auth/mfa', element: <MFAVerifyPage /> },
];
```

---

## 4. Login Page

### 4.1 Specification

| Attribute | Value |
|-----------|-------|
| Route | `/auth/login` |
| Auth Required | No (Guest only) |
| Mobile Support | Full |
| SEO | No index |

### 4.2 User Story
> As a returning user, I want to sign in with my email and password so that I can access my account and continue reading.

### 4.3 Layout

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│                         ┌─────────────┐                          │
│                         │    Logo     │                          │
│                         └─────────────┘                          │
│                                                                  │
│                    Welcome Back to N9                            │
│                  Sign in to continue reading                     │
│                                                                  │
│           ┌─────────────────────────────────────┐               │
│           │  📧 Email                            │               │
│           │  ┌─────────────────────────────────┐│               │
│           │  │ you@example.com                 ││               │
│           │  └─────────────────────────────────┘│               │
│           │                                     │               │
│           │  🔒 Password                        │               │
│           │  ┌─────────────────────────────────┐│               │
│           │  │ ••••••••                    👁   ││               │
│           │  └─────────────────────────────────┘│               │
│           │                                     │               │
│           │  ☑ Remember me    Forgot password? │               │
│           │                                     │               │
│           │  ┌─────────────────────────────────┐│               │
│           │  │         Sign In                 ││               │
│           │  └─────────────────────────────────┘│               │
│           │                                     │               │
│           │  ─────────── or ───────────        │               │
│           │                                     │               │
│           │  ┌─────────┐  ┌─────────┐          │               │
│           │  │ Google  │  │  Apple  │          │               │
│           │  └─────────┘  └─────────┘          │               │
│           │                                     │               │
│           │  Don't have an account? Sign up    │               │
│           └─────────────────────────────────────┘               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.4 Component Hierarchy

```tsx
<AuthLayout>
  <div className="auth-container">
    <Logo />
    <div className="auth-header">
      <h1>Welcome Back to N9</h1>
      <p>Sign in to continue reading</p>
    </div>
    
    <LoginForm>
      <FormField name="email" type="email" label="Email" />
      <FormField name="password" type="password" label="Password" />
      <div className="form-row">
        <Checkbox name="rememberMe" label="Remember me" />
        <Link to="/auth/forgot-password">Forgot password?</Link>
      </div>
      <Button type="submit" variant="primary" fullWidth>
        Sign In
      </Button>
    </LoginForm>
    
    <Divider>or</Divider>
    
    <OAuthButtons>
      <OAuthButton provider="google" />
      <OAuthButton provider="apple" />
    </OAuthButtons>
    
    <p className="auth-footer">
      Don't have an account? <Link to="/auth/register">Sign up</Link>
    </p>
  </div>
</AuthLayout>
```

### 4.5 State Management

| State | Type | Source | Scope |
|-------|------|--------|-------|
| formData | `LoginFormData` | React Hook Form | Component |
| isSubmitting | `boolean` | Mutation | Component |
| error | `string | null` | Mutation | Component |

### 4.6 Form Validation

```typescript
const loginSchema = z.object({
  email: z.string().email('Please enter a valid email'),
  password: z.string().min(1, 'Password is required'),
  rememberMe: z.boolean().optional(),
});
```

### 4.7 API Integration

| Action | Endpoint | Trigger | Response |
|--------|----------|---------|----------|
| Login | `POST /auth/login` | Form submit | `{ accessToken, refreshToken, user }` |
| OAuth | External | Button click | Redirect to provider |

### 4.8 Interactions

| Action | Trigger | Result |
|--------|---------|--------|
| Submit form | Click "Sign In" | Validate → API call → Store tokens → Redirect |
| Toggle password | Click eye icon | Show/hide password |
| OAuth login | Click provider button | Redirect to OAuth flow |
| Forgot password | Click link | Navigate to forgot password |
| Sign up | Click link | Navigate to register |

### 4.9 Loading & Error States

| State | Display |
|-------|---------|
| Submitting | Button shows spinner + "Signing in..." |
| Invalid credentials | Alert: "Invalid email or password" |
| Account locked | Alert: "Account locked. Try again in X minutes" |
| MFA required | Redirect to MFA verification page |
| Network error | Toast: "Connection error. Please try again" |

### 4.10 Accessibility

- Form labels associated with inputs
- Error messages linked via `aria-describedby`
- Password toggle button with `aria-label`
- Focus management on error
- Enter key submits form

---

## 5. Register Page

### 5.1 Specification

| Attribute | Value |
|-----------|-------|
| Route | `/auth/register` |
| Auth Required | No (Guest only) |
| Mobile Support | Full |
| SEO | No index |

### 5.2 User Story
> As a new visitor, I want to create an account so that I can save my reading progress and interact with stories.

### 5.3 Layout

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│                         ┌─────────────┐                          │
│                         │    Logo     │                          │
│                         └─────────────┘                          │
│                                                                  │
│                     Create Your Account                          │
│                    Join thousands of readers                     │
│                                                                  │
│           ┌─────────────────────────────────────┐               │
│           │  Display Name                       │               │
│           │  ┌─────────────────────────────────┐│               │
│           │  │ John Doe                        ││               │
│           │  └─────────────────────────────────┘│               │
│           │                                     │               │
│           │  Email                              │               │
│           │  ┌─────────────────────────────────┐│               │
│           │  │ you@example.com                 ││               │
│           │  └─────────────────────────────────┘│               │
│           │                                     │               │
│           │  Password                           │               │
│           │  ┌─────────────────────────────────┐│               │
│           │  │ ••••••••                    👁   ││               │
│           │  └─────────────────────────────────┘│               │
│           │  ✓ 8+ chars  ✓ Uppercase  ○ Number │               │
│           │                                     │               │
│           │  Confirm Password                   │               │
│           │  ┌─────────────────────────────────┐│               │
│           │  │ ••••••••                    👁   ││               │
│           │  └─────────────────────────────────┘│               │
│           │                                     │               │
│           │  ☑ I agree to the Terms of Service │               │
│           │    and Privacy Policy               │               │
│           │                                     │               │
│           │  ┌─────────────────────────────────┐│               │
│           │  │       Create Account            ││               │
│           │  └─────────────────────────────────┘│               │
│           │                                     │               │
│           │  ─────────── or ───────────        │               │
│           │                                     │               │
│           │  ┌─────────┐  ┌─────────┐          │               │
│           │  │ Google  │  │  Apple  │          │               │
│           │  └─────────┘  └─────────┘          │               │
│           │                                     │               │
│           │  Already have an account? Sign in  │               │
│           └─────────────────────────────────────┘               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.4 Form Validation

```typescript
const registerSchema = z.object({
  displayName: z
    .string()
    .min(2, 'Display name must be at least 2 characters')
    .max(50, 'Display name must be at most 50 characters')
    .regex(/^[a-zA-Z0-9_\s]+$/, 'Only letters, numbers, underscores allowed'),
  email: z.string().email('Please enter a valid email'),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/[A-Z]/, 'Must contain an uppercase letter')
    .regex(/[a-z]/, 'Must contain a lowercase letter')
    .regex(/[0-9]/, 'Must contain a number'),
  confirmPassword: z.string(),
  acceptTerms: z.literal(true, {
    errorMap: () => ({ message: 'You must accept the terms' }),
  }),
}).refine((data) => data.password === data.confirmPassword, {
  message: 'Passwords do not match',
  path: ['confirmPassword'],
});
```

### 5.5 Password Strength Indicator

```tsx
<PasswordStrengthIndicator password={password}>
  <Requirement met={password.length >= 8}>8+ characters</Requirement>
  <Requirement met={/[A-Z]/.test(password)}>Uppercase letter</Requirement>
  <Requirement met={/[a-z]/.test(password)}>Lowercase letter</Requirement>
  <Requirement met={/[0-9]/.test(password)}>Number</Requirement>
</PasswordStrengthIndicator>
```

### 5.6 Success Flow

1. User submits valid form
2. API creates account
3. Show success message: "Account created! Check your email to verify."
4. Redirect to email verification pending page

---

## 6. Forgot Password Page

### 6.1 Specification

| Attribute | Value |
|-----------|-------|
| Route | `/auth/forgot-password` |
| Auth Required | No |
| Mobile Support | Full |
| SEO | No index |

### 6.2 Layout

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│                         ┌─────────────┐                          │
│                         │    Logo     │                          │
│                         └─────────────┘                          │
│                                                                  │
│                     Forgot Your Password?                        │
│            Enter your email to receive a reset link              │
│                                                                  │
│           ┌─────────────────────────────────────┐               │
│           │  Email                              │               │
│           │  ┌─────────────────────────────────┐│               │
│           │  │ you@example.com                 ││               │
│           │  └─────────────────────────────────┘│               │
│           │                                     │               │
│           │  ┌─────────────────────────────────┐│               │
│           │  │      Send Reset Link            ││               │
│           │  └─────────────────────────────────┘│               │
│           │                                     │               │
│           │  ← Back to Sign In                 │               │
│           └─────────────────────────────────────┘               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3 Success State

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│                         ┌─────────────┐                          │
│                         │    ✉️       │                          │
│                         └─────────────┘                          │
│                                                                  │
│                      Check Your Email                            │
│                                                                  │
│          We've sent a password reset link to                     │
│          j***@example.com                                        │
│                                                                  │
│          Didn't receive the email?                               │
│          Check spam folder or [Resend email]                     │
│                                                                  │
│          ← Back to Sign In                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. Reset Password Page

### 7.1 Specification

| Attribute | Value |
|-----------|-------|
| Route | `/auth/reset-password?token={token}` |
| Auth Required | No |
| Mobile Support | Full |
| SEO | No index |

### 7.2 Layout

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│                         ┌─────────────┐                          │
│                         │    Logo     │                          │
│                         └─────────────┘                          │
│                                                                  │
│                    Reset Your Password                           │
│                  Enter your new password                         │
│                                                                  │
│           ┌─────────────────────────────────────┐               │
│           │  New Password                       │               │
│           │  ┌─────────────────────────────────┐│               │
│           │  │ ••••••••                    👁   ││               │
│           │  └─────────────────────────────────┘│               │
│           │  ✓ 8+ chars  ✓ Uppercase  ○ Number │               │
│           │                                     │               │
│           │  Confirm Password                   │               │
│           │  ┌─────────────────────────────────┐│               │
│           │  │ ••••••••                    👁   ││               │
│           │  └─────────────────────────────────┘│               │
│           │                                     │               │
│           │  ┌─────────────────────────────────┐│               │
│           │  │      Reset Password             ││               │
│           │  └─────────────────────────────────┘│               │
│           └─────────────────────────────────────┘               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.3 Error States

| Error | Display |
|-------|---------|
| Invalid token | "This link is invalid or has expired. [Request new link]" |
| Expired token | "This link has expired. [Request new link]" |
| Already used | "This link has already been used. [Request new link]" |

---

## 8. Email Verification Page

### 8.1 Specification

| Attribute | Value |
|-----------|-------|
| Route | `/auth/verify-email?token={token}` |
| Auth Required | No |
| Mobile Support | Full |
| SEO | No index |

### 8.2 States

#### Verifying State
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│                         ┌─────────────┐                          │
│                         │   ⏳        │                          │
│                         └─────────────┘                          │
│                                                                  │
│                   Verifying your email...                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### Success State
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│                         ┌─────────────┐                          │
│                         │    ✓       │                          │
│                         └─────────────┘                          │
│                                                                  │
│                    Email Verified!                               │
│                                                                  │
│          Your email has been verified successfully.              │
│          You can now sign in to your account.                    │
│                                                                  │
│                  [Continue to Sign In]                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### Error State
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│                         ┌─────────────┐                          │
│                         │    ✗       │                          │
│                         └─────────────┘                          │
│                                                                  │
│                  Verification Failed                             │
│                                                                  │
│          This verification link is invalid or expired.           │
│                                                                  │
│                  [Resend Verification Email]                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9. MFA Verification Page

### 9.1 Specification

| Attribute | Value |
|-----------|-------|
| Route | `/auth/mfa` |
| Auth Required | Partial (after initial login) |
| Mobile Support | Full |
| SEO | No index |

### 9.2 Layout

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│                         ┌─────────────┐                          │
│                         │    🔐      │                          │
│                         └─────────────┘                          │
│                                                                  │
│                   Two-Factor Authentication                      │
│                                                                  │
│          Enter the 6-digit code from your authenticator         │
│                                                                  │
│           ┌─────────────────────────────────────┐               │
│           │                                     │               │
│           │    ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐  │               │
│           │    │  │ │  │ │  │ │  │ │  │ │  │  │               │
│           │    └──┘ └──┘ └──┘ └──┘ └──┘ └──┘  │               │
│           │                                     │               │
│           │  ┌─────────────────────────────────┐│               │
│           │  │           Verify                ││               │
│           │  └─────────────────────────────────┘│               │
│           │                                     │               │
│           │  Can't access your authenticator?  │               │
│           │  [Use recovery code]               │               │
│           │                                     │               │
│           └─────────────────────────────────────┘               │
│                                                                  │
│                     ← Use different account                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 9.3 Code Input Component

```tsx
<OTPInput
  length={6}
  autoFocus
  onComplete={(code) => verifyMFA(code)}
  className="otp-input-group"
/>
```

### 9.4 Interactions

| Action | Trigger | Result |
|--------|---------|--------|
| Enter digit | Keypress | Move to next input |
| Backspace | Keypress | Clear & move to previous |
| Paste code | Ctrl+V | Fill all inputs |
| Complete entry | 6 digits entered | Auto-submit |
| Use recovery | Click link | Show recovery code input |

---

## 10. Shared Components

### 10.1 AuthLayout

```tsx
interface AuthLayoutProps {
  children: React.ReactNode;
}

export function AuthLayout({ children }: AuthLayoutProps) {
  return (
    <div className="min-h-screen flex">
      {/* Left: Form */}
      <div className="flex-1 flex items-center justify-center p-8">
        <div className="w-full max-w-md">
          {children}
        </div>
      </div>
      
      {/* Right: Brand Image (desktop only) */}
      <div className="hidden lg:flex flex-1 bg-primary-600 items-center justify-center">
        <div className="text-white text-center">
          <h2 className="text-4xl font-bold">Discover Stories</h2>
          <p className="mt-4 text-primary-100">
            Join millions of readers and writers
          </p>
        </div>
      </div>
    </div>
  );
}
```

### 10.2 OAuthButton

```tsx
interface OAuthButtonProps {
  provider: 'google' | 'apple' | 'facebook';
}

export function OAuthButton({ provider }: OAuthButtonProps) {
  const { icon, label, colors } = getProviderConfig(provider);
  
  return (
    <Button
      variant="outline"
      onClick={() => initiateOAuth(provider)}
      className="w-full"
    >
      {icon}
      <span>Continue with {label}</span>
    </Button>
  );
}
```

---

## 11. Responsive Behavior

| Breakpoint | Layout Changes |
|------------|----------------|
| Mobile (<768px) | Full-width form, hide brand image |
| Tablet (768-1024px) | Centered form (480px max), hide brand image |
| Desktop (>1024px) | Split layout: form left, brand image right |

---

## 12. Security Considerations

### 12.1 Token Management

```typescript
// Secure token storage
const tokenConfig = {
  accessToken: {
    storage: 'memory', // Never localStorage
    expiry: 15 * 60 * 1000, // 15 minutes
  },
  refreshToken: {
    storage: 'httpOnly cookie',
    expiry: 7 * 24 * 60 * 60 * 1000, // 7 days
    secure: true,
    sameSite: 'strict',
  },
};
```

### 12.2 Rate Limiting

| Action | Limit | Window | Lockout |
|--------|-------|--------|----------|
| Login attempts | 5 | 15 min | 30 min |
| Password reset | 3 | 1 hour | 24 hours |
| Email verification | 5 | 1 hour | 1 hour |
| MFA attempts | 5 | 5 min | 15 min |

### 12.3 Password Requirements

```typescript
const passwordRequirements = {
  minLength: 8,
  maxLength: 128,
  requireUppercase: true,
  requireLowercase: true,
  requireNumber: true,
  requireSpecial: false,
  preventCommon: true, // Check against common passwords
  preventUserInfo: true, // Check against username/email
};
```

---

## 13. References

### 13.1 Related Design Documents

| Document | Purpose |
|----------|----------|
| [02_DESIGN_SYSTEM_GUIDELINES.md](../02_DESIGN_SYSTEM_GUIDELINES.md) | UI components |
| [03_STATE_MANAGEMENT_ROUTING.md](../03_STATE_MANAGEMENT_ROUTING.md) | Auth state |

### 13.2 Backend Component References

| Document | Purpose |
|----------|----------|
| [02_USERS_COMPONENT.md](../../../../Backend/N9/Documentation/BackendDesign/Components/02_USERS_COMPONENT.md) | User authentication APIs |

---

## 14. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|----------|
| 1.0 | 2025-12-31 | Frontend Team | Initial authentication pages |
| 2.0 | 2026-01-04 | Frontend Team | Added comprehensive API endpoints (MFA, OAuth), security considerations, token management, rate limiting, password requirements, backend component references |
