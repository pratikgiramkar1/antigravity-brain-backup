# Wallet Data Security & Liability Shift

When operating in **DIRECT Mode** (where Hyperswitch decrypts the wallet token), passing the decrypted data correctly to the underlying Payment Gateway is critical to ensuring the merchant receives a "Liability Shift".

## What is a Liability Shift?
In a standard Card-Not-Present (CNP) e-commerce transaction, if fraud occurs, the **Merchant** is financially liable (they lose the money and pay a chargeback fee). 
However, when a user pays via Apple Pay or Google Pay, their device forces biometric authentication (Face ID / Touch ID). Because the customer's identity was strongly verified, the card networks (Visa/Mastercard) grant a **Liability Shift**. If fraud occurs on an authenticated wallet transaction, the **Card Issuer (the Bank)** absorbs the financial loss, protecting the merchant.

To receive this protection, the merchant's gateway must receive proof of the authentication. This proof is the **ECI Indicator** and the **Cryptogram**.

---

## The ECI Indicator (Electronic Commerce Indicator)
The ECI is a 2-digit string that tells the gateway the security level of the transaction.

### Visa ECI Values
* `05`: **Fully Authenticated** (Biometric Wallet/3DS successful) - **Liability Shift Granted**
* `06`: **Attempted Authentication** (System couldn't complete it, but merchant tried) - **Liability Shift Granted**
* `07`: **Not Authenticated** (Standard risky transaction) - **No Liability Shift**

### Mastercard ECI Values
* `02`: **Fully Authenticated** - **Liability Shift Granted**
* `01`: **Attempted Authentication** - **Liability Shift Granted**
* `00` (or empty): **Not Authenticated** - **No Liability Shift**

*Note: Dropping the ECI indicator during internal routing will strip the merchant of their liability shift, resulting in higher fees or unfair fraud losses.*

---

## The gRPC Contract
Hyperswitch uses a microservice architecture where the core router communicates with the Unified Connector Service (`ucs`) via **gRPC**. The data structures are strictly defined in `payment_methods.proto`.

### Google Pay Decrypted Payload
When passing a decrypted Google Pay token to the `ucs`, the following structure is used:
```protobuf
message GooglePayDecryptedData {
  SecretString card_exp_month = 1;
  SecretString card_exp_year = 2;
  CardNumberType application_primary_account_number = 3; // The DPAN
  optional SecretString cryptogram = 4; // The dynamic signature
  optional string eci_indicator = 5; // The security flag (e.g. "05")
}
```

### Apple Pay Decrypted Payload
The Apple Pay structure contains the exact same critical data, grouped slightly differently:
```protobuf
message ApplePayDecryptedData {
  CardNumberType application_primary_account_number = 1;
  SecretString application_expiration_month = 2;
  SecretString application_expiration_year = 3;
  ApplePayCryptogramData payment_data = 4;
}

message ApplePayCryptogramData {
  SecretString online_payment_cryptogram = 1;
  optional string eci_indicator = 2;
}
```
