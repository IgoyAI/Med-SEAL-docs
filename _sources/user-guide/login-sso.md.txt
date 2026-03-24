# Logging In with Single Sign-On

All Med-SEAL services share a single login through the **SSO (Single Sign-On)** system. You create one account and use it across OpenEMR, the AI Dashboard, and the Patient Portal.

---

## Creating Your Account

Your account is provisioned by a system administrator. When your account is created you will receive a welcome email containing:

- Your **username** (typically your work email or patient ID)
- A **temporary password**
- The login URL for your organisation

If you have not received a welcome email, contact your system administrator.

---

## Logging In

### Patient Portal Native (Mobile App)

1. Open the **Med-SEAL** app on your iOS or Android device.
2. On the welcome screen, tap **Sign In**.
3. Enter your **username** and **password**.
4. Tap **Sign In**.

If this is your first login, you will be prompted to set a new password.

### OpenEMR (Clinician Web)

1. Navigate to your OpenEMR URL (e.g. `http://localhost:8081`).
2. Enter your **username** and **password**.
3. Click **Login**.

### AI Dashboard (Admin Web)

1. Navigate to `http://localhost:3001`.
2. Enter your SSO credentials.
3. Click **Sign In**.

---

## Resetting Your Password

If you forget your password:

1. On the login screen, click **Forgot password?**
2. Enter your registered email address.
3. Check your email for a reset link (valid for 24 hours).
4. Follow the link and enter a new password.

If the reset email does not arrive within a few minutes, check your spam folder or contact your administrator.

---

## Session Timeout

For security, your session automatically expires after **60 minutes of inactivity**. You will be redirected to the login screen. Your work is saved before expiry.

---

## Troubleshooting

| Problem | Solution |
|---|---|
| "Invalid credentials" | Check Caps Lock; try password reset |
| Account locked after 5 attempts | Contact your administrator to unlock |
| Blank screen after login | Clear browser cache or reinstall the app |
| "Service unavailable" | The backend may be starting up; wait 2 minutes and retry |
