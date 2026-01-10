# Lean.x Integration Analysis & Implementation Plan

This document outlines the strategy for integrating the Lean.x "API Own Page" payment flow into the Canvas Builder prototype (`app.jsx`).

## 1. Credential Mapping

We will update the `PaymentsPanel` to accept the specific credentials required by Lean.x, replacing the generic "Merchant ID/API Key" fields.

**User Credentials (To be used as defaults/hardcoded for demo):**
*   **Collection UUID:** `Dc-94C881BE5B-Lx`
*   **Auth Token:** `LP-D70DAABE-MM|03253b16-7eb7-4244-837e-0ace0343ae5c|c2d2fb521d1a09786cae63de912af1dcaaeb57adb61033f09c893685d499c8d2a904b5622256bb0bca2abf02e5ce671f6c5170a9339c58305739d12ea97a52b4`
*   **Hash Key:** `c2d2fb521d1a09786cae63de912af1dcaaeb57adb61033f09c893685d499c8d2a904b5622256bb0bca2abf02e5ce671f6c5170a9339c58305739d12ea97a52b4`

*(Note: The `UUID` provided `03253b16...` seems to be part of the Auth Token structure, but we will store it if needed.)*

## 2. Component Modifications

### A. `PaymentsPanel` (Configuration)
**Current:** Generic form (Merchant ID, Public Key, Secret Key).
**New:** Detailed configuration for Lean.x.
*   **Inputs Needed:**
    *   Auth Token (Header: `auth-token`)
    *   Collection UUID (Body: `collection_uuid`)
    *   Hash Key (For verifying callbacks - simulated in frontend)
*   **Action:** Update the state schema in `PaymentsPanel` and the corresponding input fields.

### B. `CheckoutModal` (The Payment Flow)
**Current:** Static Credit Card form (Number, Expiry, CVC).
**New:** Dynamic Bank Selection (The "API Own Page" Flow).

**New Workflow inside Modal:**
1.  **On Mount:**
    *   Call `POST https://api.leanx.io/api/v1/merchant/list-payment-services`.
    *   Headers: `auth-token`.
    *   Body: `{"payment_type": "WEB_PAYMENT", "payment_status": "active", ...}`.
    *   **Loading State:** Show a spinner while fetching banks.
2.  **Display:**
    *   Render a grid/list of banks returned from the API (using `payment_service_name` and logos if available).
    *   User clicks a bank -> Sets `selectedBankId`.
3.  **On Submit ("Pay Now"):**
    *   Call `POST https://api.leanx.io/api/v1/merchant/create-bill-silent`.
    *   Body: Includes `collection_uuid`, `payment_service_id` (selected above), `amount`, `full_name`, etc.
    *   **Response:** Extract `redirect_url`.
4.  **Redirect:**
    *   `window.location.href = response.data.redirect_url`.

## 3. Technical Implementation Details

### API Helper Function
Since we are in a pure frontend prototype, we will use `fetch` directly.
*Warning: In a real production app, these calls should go through a backend to hide the Auth Token and avoid CORS issues. For this prototype, we assume the API allows CORS or we will use a proxy if needed.*

```javascript
const leanxApi = async (endpoint, body, authToken) => {
    const response = await fetch(`https://api.leanx.io/api/v1/merchant/${endpoint}`, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'auth-token': authToken
        },
        body: JSON.stringify(body)
    });
    return response.json();
};
```

### State Updates
We need to lift the `paymentSettings` state up from `PaymentsPanel` to the main `App` component so that `CheckoutModal` can access the `Auth Token` and `Collection UUID`.

**App.js Structure Update:**
```javascript
const App = () => {
    // ...
    const [integrationSettings, setIntegrationSettings] = useState({
        leanx: {
            authToken: "LP-D70DAABE-...", // Default provided
            collectionUuid: "Dc-94C881BE5B-Lx",
            hashKey: "..."
        }
    });
    // ...
}
```

## 4. Proposed `CheckoutModal` internal logic

```javascript
// Pseudo-code for the new modal content
useEffect(() => {
    if (isOpen) {
        // Fetch Banks
        fetchPaymentServices().then(data => setBanks(data));
    }
}, [isOpen]);

const handlePay = async () => {
    const billData = {
        collection_uuid: settings.collectionUuid,
        payment_service_id: selectedBank.payment_service_id,
        amount: total,
        invoice_ref: `INV-${Date.now()}`,
        // ... user details
    };
    
    const response = await createBillSilent(billData);
    if (response.data?.redirect_url) {
        window.location.href = response.data.redirect_url;
    }
};
```

## 5. Next Steps
1.  **Refactor `App.jsx`**: Move settings state to root.
2.  **Update `PaymentsPanel`**: Bind to new settings structure.
3.  **Update `CheckoutModal`**: Implement the 2-step API flow (Fetch List -> Create Bill).
