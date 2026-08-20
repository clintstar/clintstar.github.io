# FLOWS: Functions and Variables

This document lists functions and variables referenced in the FLOWS.md examples from pi-apps/demo and describes their shape and purpose. The snippets in FLOWS.md are examples—not full implementations—so several variables are referenced but defined outside the snippets.

## Frontend (frontend/src/Shop or frontend/src/shop/index.ts)

- signIn (async function)
  - Parameters: none
  - Returns: Promise<void>
  - Description: Calls `window.Pi.authenticate(scopes, onIncompletePaymentFound)` to obtain an `AuthResult`, posts it to backend via `signInUser(authResponse)`, and updates local user state via `setUser(authResponse.user)`.

- signInUser (function)
  - Parameters: authResult: any
  - Returns: result of `setShowModal(false)` (used to close UI modal)
  - Description: Sends `authResult` to backend `/signin` endpoint using `axiosClient.post("/signin", { authResult }, config)` and closes sign-in modal.

- scopes (variable)
  - Type: string[] (example: `["username","payments"]`)
  - Description: Scopes passed to `Pi.authenticate` that determine which keys appear on `AuthResult`.

- authResponse (variable)
  - Type: AuthResult (SDK type)
  - Description: Result of `Pi.authenticate(...)` containing user data and `accessToken` (depending on scopes).

- onIncompletePaymentFound (function)
  - Parameters: payment: PaymentDTO
  - Returns: Promise (axios post)
  - Description: Callback passed to `Pi.authenticate` invoked when an incomplete payment is detected; posts the `payment` to backend `/incomplete` for handling.

- orderProduct (async function)
  - Parameters: memo: string, amount: number, paymentMetadata: MyPaymentMetadata
  - Returns: Promise<void>
  - Description: Builds `paymentData` and `callbacks`, calls `window.Pi.createPayment(paymentData, callbacks)`, and logs returned `payment`.

- paymentData (variable)
  - Shape: { amount: number, memo: string, metadata: MyPaymentMetadata }
  - Description: Payment descriptor passed to `Pi.createPayment`.

- callbacks (variable)
  - Shape: { onReadyForServerApproval, onReadyForServerCompletion, onCancel, onError }
  - Description: Callback handlers passed to `Pi.createPayment`.

- payment (variable)
  - Type: PaymentDTO (returned by `Pi.createPayment`)
  - Description: The created payment object.

- onReadyForServerApproval (function)
  - Parameters: paymentId: string
  - Returns: axios post Promise
  - Description: Called when payment identifier is available; posts `{ paymentId }` to backend `/approve` to trigger server-side approval.

- onReadyForServerCompletion (function)
  - Parameters: paymentId: string, txid: string
  - Returns: axios post Promise
  - Description: Called when transaction is submitted to blockchain; posts `{ paymentId, txid }` to backend `/complete` for server-side completion.

- onCancel (function)
  - Parameters: paymentId: string
  - Returns: axios post Promise
  - Description: Called when payment is canceled; posts `{ paymentId }` to backend `/cancelled_payment`.

- onError (function)
  - Parameters: error: Error, payment?: PaymentDTO
  - Description: Logs error and inspects optional `payment` for debugging. Not required to be handled server-side.

- user, setUser, setShowModal (React state variables / setters)
  - Description: UI state used in snippets; not defined in the examples.

- axiosClient (HTTP client)
  - Description: Axios instance used for both app backend calls and Pi Platform API calls; assumed defined elsewhere.

- config (variable)
  - Description: Axios request configuration (headers, timeouts); defined elsewhere.

- window.Pi (SDK)
  - Methods used:
    - `Pi.authenticate(scopes, onIncompletePaymentFound)` → AuthResult
    - `Pi.createPayment(paymentData, callbacks)` → PaymentDTO

## Backend (backend/src/index.ts)

- POST /signin handler (Express async function)
  - Parameters: req, res
  - Description: Verifies the user's `accessToken` by calling Pi Platform `GET /v2/me` using `axiosClient.get('/v2/me', { headers: { Authorization: `Bearer ${currentUser.accessToken}` } })`. On error returns 401, otherwise returns 200.
  - Note: `currentUser` is referenced but not defined in the snippet; it should be derived from the posted `authResult` or session.

- POST /approve handler (Express async function)
  - Parameters: req, res
  - Description: Reads `paymentId` from `req.body.paymentId` and calls Pi Platform `POST /v2/payments/${paymentId}/approve` to notify Pi that server is ready to approve the payment.

- POST /complete handler (Express async function)
  - Parameters: req, res
  - Description: Reads `paymentId` and `txid` from the request body and calls Pi Platform `POST /v2/payments/${paymentId}/complete` with `{ txid }` to complete the payment.

- POST /incomplete handler (Express async function)
  - Parameters: req, res
  - Description: Handles incomplete payments posted by `onIncompletePaymentFound`. Extracts `payment`, validates blockchain transaction by fetching `payment.transaction._link` (`txURL`) and comparing memo/identifier to local order, marks order paid in DB, and calls Pi Platform `POST /v2/payments/${paymentId}/complete` with `{ txid }`.
  - Variables used in snippet: `payment`, `paymentId`, `txid`, `txURL`, `horizonResponse`, `paymentIdOnBlock`, `order` (DB lookup) — some are placeholders and must be implemented fully in the app.

## Variables referenced but not defined in FLOWS.md examples

- currentUser: must be set from frontend-supplied `authResult` (or session) and contain `accessToken`.
- order: application-specific DB order record used to match incomplete payments.
- MyPaymentMetadata: application-defined metadata type used when creating payments.
- AuthResult, PaymentDTO, UserDTO: DTO types described in Pi SDK / Platform API docs.
- axios: the HTTP library; `axios.create({ timeout: 20000 })` is used in the `/incomplete` handler to fetch transaction data.

## Notes and next steps

- The snippets are examples and require wiring into the real frontend and backend code (parsing the posted `authResult`, extracting `accessToken`, storing/looking up `order` data, and providing a properly configured `axiosClient` and `config`).
- If you want, I can:
  - create TypeScript/Express route stubs for the backend endpoints (`/signin`, `/approve`, `/complete`, `/incomplete`), or
  - add typed interfaces for `AuthResult` and `PaymentDTO` (based on platform docs), or
  - open a PR in `clintstar/clintstar.github.io` with this file placed in the repository root.

