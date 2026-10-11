# Payment Gateway Integration — Multi-Provider Payment Handling

**Domain:** Airline reservation and booking systems  
**Engineering focus:** Java, Spring, payment integration, backend extensibility, browser redirects, transaction auditing  
**Portfolio type:** Architecture case study and illustrative implementation

> **Scope and confidentiality:** This case study describes payment-handling patterns informed by my airline reservation integration experience. The diagrams and accompanying Java example are illustrative, not a verified representation of a production deployment. No private code, credentials, or internal endpoints are included. Any code published alongside this case study should be independently authored and cleared for public release.

## 1. Background and engineering problem

An airline booking application may need to support payment providers with different integration contracts. Some providers host the card-entry page and redirect the customer back after payment. Others expose a direct backend API, allowing the business application to submit payment information through a provider-specific client library. A third pattern combines backend initiation with a provider-hosted checkout page.

Without an abstraction, each payment option can introduce conditional logic into booking controllers, duplicating auditing, provider selection, response handling, and booking-status updates. The design goal is to keep the booking flow consistent while allowing each provider to implement its own payment initiation and completion behavior.

## 2. Integration patterns

| Pattern | Where payment details are entered | Provider interaction | Browser redirected to gateway? | Completion handling |
| --- | --- | --- | --- | --- |
| **External / hosted checkout** | Payment gateway's hosted page | Business backend audits initiation and builds configured redirect URL | **Yes** | Gateway returns browser to client confirmation URL; client handles result |
| **Internal / direct API** | Business application's payment page | Backend uses a payment gateway stub/client library to submit payment data | **No** | Backend receives result, audits and updates payment/booking state, then navigates browser to confirmation |
| **Internal–external / hybrid** | Business application initially collects payment information; provider may complete checkout on hosted pages | Backend invokes gateway client library to initiate processing and obtain a redirect URL | **Yes** | Gateway redirects browser back; client handles post-payment response |

**Terminology:** “Internal” describes the integration mode from the business application's perspective, not ownership of the external provider's infrastructure. The hybrid flow should be used only where the provider's supported integration contract permits this arrangement.

## 3. Architecture overview

The example Java design uses a **Strategy-style handler interface** and a **Spring-managed registry** of implementations. Booking-related code selects a payment option by its identifier rather than hard-coding provider-specific behavior.

```text
Booking UI / Backend
        |
        v
PaymentOptionsProvider
        |  select by getPaymentOptionName()
        v
PaymentGatewayHandlerTemplete
        |
        +-- MyExternalPaymentgatewayHandler
        |       audit -> configured redirect URL -> post-payment handling
        |
        +-- MyInternalExternalPaymentgatewayHandler
        |       audit -> gateway stub -> hosted checkout redirect
        |
        +-- MyInternalPaymentgatewayHandler
                audit -> gateway stub -> direct result / post-processing

Shared collaborators: AuditService, BookingService, PaymentGatewayStubLib
```

The sample's interface name `PaymentGatewayHandlerTemplete` is retained exactly as supplied, although `PaymentGatewayHandlerTemplate` would be a clearer spelling for a new implementation.

### Core contract

```java
interface PaymentGatewayHandlerTemplete {
    String getPaymentOptionName();
    PaymentResponse getPaymentProcess(PaymentRequest request);
    PostPaymentResponse getPostPaymentProcess(PostPaymentRequest request);
}
```

- `getPaymentOptionName()` supplies a stable key for selecting a provider.
- `getPaymentProcess(...)` initiates the provider-specific payment flow.
- `getPostPaymentProcess(...)` handles the return/completion phase where applicable.

The example `PaymentOptionsProvider` receives all Spring beans implementing this interface and constructs a map keyed by payment option name. This makes new integrations easier to add without modifying the selection mechanism. The supplied example comments identify enabled-provider filtering and validation as possible extensions; these are **not yet implemented**.

## 4. Flow A — External payment gateway (hosted checkout)

![External hosted-checkout payment flow](/asserts/external-payment-gateway.png)

**Sequence**

1. The customer reviews a booking and clicks **Pay**. **No card details are entered in the business application.**
2. The backend records/audits payment initiation and retrieves the configured gateway redirect URL.
3. The browser navigates to the gateway-hosted payment page.
4. The customer enters payment details and completes payment on the provider's page.
5. The provider redirects the browser to the application's confirmation/return URL. The client processes the result and presents booking confirmation or failure.

**Mapping to sample code:** `MyExternalPaymentgatewayHandler.getPaymentProcess()` audits the request and builds a URL using `pg.external.redirect.url` and `bookingId`. `getPostPaymentProcess()` audits the supplied completion request, then calls `BookingService.confirmBooking()` or `failBooking()` based on its status.

**Important implementation distinction:** A browser redirect is **not** authoritative proof that payment succeeded. A production application should verify the transaction through an authenticated provider callback or server-side status inquiry before marking the booking paid. The provided sample does not implement that verification.

## 5. Flow B — Internal payment gateway (direct backend API)

![Direct backend payment gateway integration](assets/internal-payment-gateway.png)

**Sequence**

1. The customer enters payment information in the business application's payment form and clicks **Pay**.
2. The frontend submits the payment request to the application's backend.
3. The backend invokes a gateway stub/client library with the required payment parameters. **The browser is not redirected to the gateway.**
4. The payment provider processes the request and returns a success, failure, or other result to the backend.
5. The backend audits the response, updates transaction and booking state as appropriate, and returns the outcome.
6. The business application directs the browser to its own confirmation page.

**Mapping to sample code:** `MyInternalPaymentgatewayHandler` calls `PaymentGatewayStubLib.initiatePayment(...)` and immediately invokes its post-payment method. **The supplied code is only a mock:** it discards the stub's return value, constructs a hard-coded `SUCCESS` post-payment request, and returns a placeholder URL. It does **not** actually transmit card data or verify a provider authorization. The diagram describes the intended direct-integration pattern, not functionality fully implemented by this snippet.

**Security note:** Collecting raw card data on the business application's own pages can significantly increase PCI DSS scope. A real implementation requires a PCI-compliant design and should prefer tokenization or provider-hosted fields where supported. Never log PAN, CVV, or other sensitive authentication data.

## 6. Flow C — Internal–external payment gateway (hybrid)

![Hybrid backend-initiation and hosted-checkout flow](assets/hybrid-payment-gateway.png)

**Sequence**

1. The customer initiates payment from the business application, which may collect provider-required payment inputs.
2. The backend uses the gateway stub library to initiate a transaction and obtain a provider checkout URL.
3. The browser is redirected to the provider-hosted page to complete the payment.
4. The gateway processes the transaction and returns the customer to the application's return URL.
5. The backend audits the completion request and updates the booking according to the **verified** payment outcome.

**Mapping to sample code:** `MyInternalExternalPaymentgatewayHandler` calls `stubLib.initiatePayment(bookingId, amount)` and returns the resulting URL. Its post-payment method audits the supplied status and invokes booking success/failure handling. The provided stub constructs a mock URL; it does not make a real provider API request.

## 7. Shared responsibilities and design decisions

| Concern | Approach shown or intended | Benefit |
| --- | --- | --- |
| Payment-option selection | Spring injects handler implementations into `PaymentOptionsProvider` | Isolates provider-specific logic |
| Contract consistency | Shared initiation and post-payment methods | Common application-facing integration points |
| Provider configuration | External redirect URL supplied via `@Value` | Avoids hard-coded URLs in the external handler's main logic |
| Transaction auditing | `AuditService` invoked on initiation and completion | Consistent trace points (illustrative only) |
| Booking updates | `BookingService` confirms or fails bookings | Separates payment integration from booking operations |
| Provider client | `PaymentGatewayStubLib` encapsulates initiation | Provides a seam for provider-specific adapters |

### Engineering trade-offs

- **Redirect checkout** generally keeps card-entry activity on the provider's hosted page, but requires robust return handling and independently verified status.
- **Direct backend processing** can offer a more integrated user experience, but requires stronger handling of sensitive payment data and provider API failures.
- **Hybrid initiation** allows provider-specific setup before hosted checkout, but has both backend integration and browser-return complexity.
- **A common handler interface** improves extensibility, although synchronous and redirect flows return different kinds of outcomes; a production contract may benefit from an explicit result type (`REDIRECT_REQUIRED`, `COMPLETED`, `PENDING`, `FAILED`) instead of using `redirectUrl` for every case.

## 8. Reliability, security and production hardening

The supplied Java snippet demonstrates structural intent, **not a production-ready payment solution**. Before production use, the following would need to be designed and tested:

- **Verified payment outcome:** Validate signed callbacks/webhooks or query the gateway server-to-server; do not trust browser-provided `status` alone.
- **Idempotency:** Prevent duplicate captures, booking confirmations, and post-payment side effects during retries or repeated callbacks.
- **Transaction lifecycle:** Persist initiated, pending, authorised, captured, failed, cancelled, and reconciled states as applicable to each provider.
- **Request integrity:** Validate booking ownership, server-side amount/currency, and transaction references; do not rely solely on browser input.
- **Sensitive data protection:** Avoid recording payment-card details in application logs or audit records; apply appropriate PCI DSS controls.
- **Return URL safety:** Restrict redirects to approved destinations and protect against open redirects and parameter tampering.
- **Resilience:** Define timeouts, safe retries, error mappings, reconciliation jobs, and recovery from interrupted browser sessions.
- **Observability:** Use correlation IDs and sanitized audit events to trace initiation, gateway response, and booking state transitions.
- **Configuration:** Validate gateway enablement and credentials, handle duplicate handler names, and keep secrets out of source code.

## 9. Evidence and implementation notes

The supporting sample includes:

- `PaymentGatewayHandlerTemplete` and DTOs (`PaymentRequest`, `PaymentResponse`, `PostPaymentRequest`, `PostPaymentResponse`).
- Three Spring service implementations corresponding to the flows above.
- `PaymentOptionsProvider`, which registers handlers by payment-option name.
- Placeholder `PaymentGatewayStubLib`, `AuditService`, and `BookingService` classes.

**What the example does not establish:** real provider SDK integration, end-to-end card handling, callback signature verification, persistence, metrics, deployment configuration, automated tests, or measurable business outcomes. These should not be claimed as completed solely from the example.

### Suggested future demonstrations

- Original Spring Boot demo with mock payment providers and a runnable README.
- Sequence diagrams showing successful, failed, cancelled, and delayed-confirmation payments.
- Unit tests for handler selection and booking-state transitions.
- Integration tests for webhook replay and idempotency.
- A short architectural decision record comparing hosted checkout and direct API integrations.

## 10. Engineering contribution and takeaway

My airline reservation experience included payment integration upgrades, 3-D Secure-related changes, and integration between payment services and a legacy booking platform. This case study illustrates how I reason about differing gateway integration patterns and isolate provider-specific behavior behind a common backend contract. It should be read as an **engineering design explanation**, not as proof that every component of the illustrative code or diagrams was deployed unchanged.

---

[← Back to engineering portfolio](../../README.md) · [Professional engineering profile](../../01-professional-profile/engineering-profile.md)

