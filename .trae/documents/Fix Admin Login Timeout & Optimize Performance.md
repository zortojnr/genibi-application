# Admin Login Timeout Fix Plan

The user is encountering a specific error message: `"Login request timed out. Please check your network connection."` during admin login. This error was explicitly added in a previous step to handle infinite loading states. The fact that it's appearing means the `Promise.race` in `adminService.ts` is hitting the 15-second timeout before the login logic completes.

## Diagnosis
1.  **Current Implementation**: `adminService.login` wraps the entire login flow (Auth + Firestore check + Audit Log + Last Login update) in a `Promise.race` with a 15-second timeout.
2.  **Potential Causes**:
    *   **Network Latency**: 15 seconds might be too aggressive for users with slow connections, especially since multiple sequential Firestore operations are awaited (`signIn`, `getDoc`, `addDoc`, `updateDoc`).
    *   **Blocking Operations**: The audit log (`addDoc`) and last login update (`updateDoc`) are currently awaited *before* returning the user. If Firestore is slow, these block the login success.
    *   **Firestore Cold Start**: If the app hasn't been used in a while, the initial connection to Firestore might take longer than 15s.

## Resolution Strategy
1.  **Optimize Critical Path**: The user only needs to be authenticated and authorized to proceed. The audit logging and "last login" update are side effects that shouldn't block the UI transition.
    *   **Action**: Move `logAudit` and `updateDoc(lastLogin)` to run *asynchronously* (without `await` blocking the return) or catch their errors independently so they don't fail the main login promise.
    *   *Correction*: For safety, we usually want to await them, but for *performance* and *reliability* of the login UX, we can let them run in the background or at least don't let them trigger the global timeout if the auth part succeeded.
    *   **Better Approach**: Increase the timeout to 30s *OR* only timeout the critical `signIn` and `getDoc` checks.

2.  **Increase Timeout**: 15 seconds is a bit tight for a mobile network doing 3-4 round trips. Increasing to 30 seconds is a safe immediate fix.

3.  **Error Handling**: Improve the error message to be more descriptive if it's a timeout vs. a network error.

## Implementation Steps
1.  **Modify `src/services/adminService.ts`**:
    *   Increase timeout from `15000` to `30000` ms.
    *   Refactor `loginPromise`:
        *   Await `signInWithEmailAndPassword`.
        *   Await the `getDoc` (authorization check) - this is critical.
        *   **Do NOT await** `logAudit` and `updateDoc` for the final return. Fire them and let them resolve in the background to speed up the perceived login time.
        *   *Self-Correction*: If we don't await them, unhandled rejections might occur. We should attach `.catch()` to them.

2.  **Modify `src/pages/AdminLogin.tsx`**:
    *   The error handling logic is already there (`setError(err.message)`).
    *   We will ensure the UI gives feedback if it's taking a while (the spinner is already there).

## Refined Plan
1.  **Relax Timeout**: Increase to 30s.
2.  **Non-Blocking Side Effects**: Move audit logging and last-login updates *after* the critical auth check, and do not await them for the function return value. This ensures that once Auth + Permissions are verified, the user gets in immediately.

This directly addresses "Response times" and "Client-side network request implementations".

## Verification
*   **Test**: Simulate a slow network (if possible) or verify code structure ensures non-blocking side effects.
*   **Result**: Login should be faster because it no longer waits for the audit log write to confirm success.
