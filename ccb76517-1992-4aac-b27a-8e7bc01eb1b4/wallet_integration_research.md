# Apple Pay and Google Pay Flow in Hyperswitch

Based on a thorough review of the Hyperswitch repository, here is a deep dive into how Apple Pay and Google Pay are implemented, specifically covering wallet session initiation, merchant domain validation, and cryptogram-based authorization across multiple gateways.

## 1. Wallet Session Initiation (`session_flow.rs`)

Wallet session initiation varies significantly between Apple Pay and Google Pay due to the different ways these platforms handle client-server authentication.

### Apple Pay Session Initiation
When an Apple Pay payment is initiated on the web, Apple requires a valid merchant session before displaying the payment sheet. This is handled by `create_applepay_session_token`.
- The router checks the `connector_metadata` to determine the flow (Simplified vs Manual).
- It constructs an `ApplepaySessionRequest` containing the merchant identifier, business country, and `initiative_context` (the merchant's domain).
- It sends a `POST` request to Apple's `paymentservices/paymentSession` endpoint.
- **Mutual TLS (mTLS):** The request uses the merchant's (or Hyperswitch's) Apple Pay merchant certificate and private key to authenticate.
- The resulting Apple Pay session is passed back to the client to render the Apple Pay sheet.

### Google Pay Session Initiation
Google Pay operates differently. It does not require a server-to-server call to generate a session token. Instead, `create_gpay_session_token` generates the necessary configuration for the Google Pay frontend SDK.
- It extracts the `GooglePayProviderDetails` from the connector's configuration.
- It formulates a `GooglePaySessionResponse` containing `tokenization_specification` (e.g., `gateway` and `gatewayMerchantId`).
- This configuration is passed to the client so the Google Pay JS API knows how to encrypt the payment token for the specific processor.

---

## 2. Merchant Domain Validation for Apple Pay (`verification.rs`)

Apple Pay on the Web requires that every domain hosting the Apple Pay button be explicitly verified by Apple.

- The route `verify_merchant_creds_for_applepay` is used to register a merchant's domain.
- It reads the Apple Pay merchant configurations (certificate and key).
- It creates an `ApplepayMerchantVerificationConfigs` payload with the `domain_names` and `encrypt_to` fields.
- A `POST` request is sent to Apple's merchant verification endpoint (e.g., `/paymentservices/registerMerchant`).
- Upon success, the domain is persisted in the database via `check_existence_and_add_domain_to_db` and linked to the Merchant Connector Account (MCA). This ensures that Hyperswitch knows the domain is cleared for processing Apple Pay transactions.

---

## 3. Cryptogram-Based Authorization via REST and gRPC

When a user authenticates the payment via FaceID, TouchID, or Android biometrics, the wallet provides a token containing a secure **cryptogram**.

### Decryption Models
Hyperswitch supports two primary decryption models for Wallet Tokens:
1. **Application Decryption (Hyperswitch decrypts):** Hyperswitch decrypts the Apple Pay token using its managed certificates (`PaymentMethodToken::ApplePayDecrypt`). It extracts the `application_primary_account_number` (DPAN/Network Token), `online_payment_cryptogram`, and expiration dates.
2. **Gateway Decryption (Pass-through):** The token is sent in its raw, encrypted format to the gateway. For example, Google Pay tokens are often base64 encoded and passed directly to the gateway's tokenization field.

### Connector Transformers
When building the authorization request for a specific gateway (e.g., `Cybersource`, `Stripe`, `Worldpay`), the transformers handle the cryptogram mapping:
- In `crates/hyperswitch_connectors/src/connectors/cybersource/transformers.rs`, if Hyperswitch decrypted the Apple Pay token, it maps the `online_payment_cryptogram` to Cybersource's `cryptogram` field and the DPAN to the `number` field.
- If it's a pass-through (e.g., Google Pay), it wraps the base64-encoded encrypted token in a `FluidData` object with a specific descriptor (e.g., `GooglePayTokenPaymentInformation`), allowing Cybersource to decrypt it on their end.
- These payloads are then transmitted to the respective gateways via REST (most common) or gRPC protocols, executing the actual financial authorization.

---

## 4. The 14 Gateways

Your resume mentions shipping across 14 gateways. Based on the presence of `ApplePayWalletData` and `GooglePayWalletData` parsing in the connectors directory, the following gateways actively support these wallet cryptogram flows in the codebase:

1. **Cybersource**
2. **Stripe**
3. **Worldpay / WorldpayXML**
4. **NMI**
5. **Nuvei**
6. **Barclaycard**
7. **Wells Fargo**
8. **Bank of America**
9. **ACI**
10. **Fiuu**
11. **Zen**
12. **Tesouro**
13. **BlueSnap**
14. **Noon**

*(Other notable mentions that support some variation of wallets include Paysafe, Nexinets, and Adyen).*
