---
title: 'Webhooks'
date: 2024-12-15T01:25:38+01:00
---

## Create the Webhook on ImproFans

Login into your ImproFans account webhook settings (https://improfans.de/u/settings/webhook\). Here, you have to add the webhook URL that should be triggered by ImproFans in case of a donation.

{{< callout type="warning" >}}
  Your webhook URL must use HTTPS scheme. Unencrypted HTTP endpoints will not be accepted.
{{< /callout >}}

## Usage

ImproFans will invoke your configured webhook URL when you have received a donation.

Your webhook will receive the following JSON payload by ImproFans:

```json
{
    "event": "benefit",
    "data": {
        "id": "6bf842c9-a09f-4ccb-8844-83b38dd4786c",
        "amount": 1.23,
        "currency": "EUR",
        "message": "This is a donation message.",
        "donor_id": "076ff569-7c58-4205-ae3a-e4fae554eb0a",
        "donor_name": "pixlcrashr",
        "created_at": "2024-12-15T01:04:55+00:00"
    }
}
```

where the payload is described in the following:

Field | Description | Data type
--- | --- | ---
`event` | Type of the webhook event. | String (one of: `benefit`)
`data.id` | ID of the benefit (= donation). | String ([UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier))
`data.amount` | Amount of the benefit. | Number
`data.currency` | Currency of the benefit. | String (only: `EUR`)
`data.message` | Message of the benefit. | String
`data.donor_id` | Donor id of the benefit. | String ([UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier))
`data.donor_name` | Donor name of the benefit. | String
`data.created_at` | When the benefit was created. | String ([ISO 8601](https://www.iso.org/obp/ui/#iso:std:iso:8601:-1:ed-1:v1:en))


{{< callout type="info" >}}
  Your webhook endpoint/receiver must be able to parse JSON (most platforms and programming languages support JSON parsing by default).
{{< /callout >}}

After you have successfully received the webhook, a [HTTP 200 OK](https://developer.mozilla.org/de/docs/Web/HTTP/Status/200) status code has to be returned as the result status. A status code different to 200 OK will mark the webhook event as failed and may (or may not) be re-send to your endpoint.

## Advanced: Verifying the integrity

{{< callout type="warning" >}}
  Although we **highly recommend** verifying the webhook payload, depending on the use-case, verifying the payload might not be necessary.
{{< /callout >}}

As your webhook must be publicly accessible for us to send any events, malicious actors may sent invalid payloads to your endpoints. Hence, we sign the HTTP payload using an ECDSA signature and put it in the HTTP header `X-Signature`.
For example:

```
X-Signature: B7BE60F73971924D9547C6C8C755866DF9DA56C19AD50B7B571827F6823D8434EA46AE1D4FADFBE4E46A9518747817A0F63FE6D4E59503982E07BDEBC8788C22
```

To verify that a webhook event was sent by us, simply download our public key which use for signing [here](https://improfans.de/api/webhooks/keys/public) and verify the signature. The public key is an EC (elliptic curve) key and is returned in standard PEM encoding.

Alternatively, this is our public key:

```
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEWZrLPi5uAvrLKQ5mgWLXICH3bkk2
oXl2WZLGwKysLdYPdsyc7Dc3UOPPFPdQH5Mjp24kDjiCVztCeNYc/PqR7Q==
-----END PUBLIC KEY-----
```

### Example

Here is a quick example of how to verify the signature of a webhook request in a programming language:

{{< tabs items="Go,JS" >}}

  {{< tab >}}
    ```go
    package main

    import (
        "crypto/ecdsa"
        "crypto/x509"
        "encoding/hex"
        "encoding/json"
        "encoding/pem"
        "fmt"
        "io"
        "log"
        "math/big"
        "net/http"
    )

    const publicKeyPEM = `-----BEGIN PUBLIC KEY-----
    MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEWZrLPi5uAvrLKQ5mgWLXICH3bkk2
    oXl2WZLGwKysLdYPdsyc7Dc3UOPPFPdQH5Mjp24kDjiCVztCeNYc/PqR7Q==
    -----END PUBLIC KEY-----`

    // WebhookPayload represents the incoming webhook data structure
    type WebhookPayload struct {
        Event string      `json:"event"`
        Data  BenefitData `json:"data"`
    }

    type BenefitData struct {
        ID        string  `json:"id"`
        Amount    float64 `json:"amount"`
        Currency  string  `json:"currency"`
        Message   string  `json:"message"`
        DonorID   string  `json:"donor_id"`
        DonorName string  `json:"donor_name"`
        CreatedAt string  `json:"created_at"`
    }

    func loadPublicKey() (*ecdsa.PublicKey, error) {
        block, _ := pem.Decode([]byte(publicKeyPEM))
        if block == nil {
            return nil, fmt.Errorf("failed to decode PEM block")
        }

        pub, err := x509.ParsePKIXPublicKey(block.Bytes)
        if err != nil {
            return nil, fmt.Errorf("failed to parse public key: %v", err)
        }

        ecdsaPub, ok := pub.(*ecdsa.PublicKey)
        if !ok {
            return nil, fmt.Errorf("public key is not ECDSA")
        }

        return ecdsaPub, nil
    }

    func verifySignature(publicKey *ecdsa.PublicKey, signature string, payload []byte) bool {
        // Convert hex signature to bytes
        sigBytes, err := hex.DecodeString(signature)
        if err != nil {
            return false
        }

        // ECDSA signature is two big integers (r,s) concatenated
        // Split signature into r and s components
        r := new(big.Int).SetBytes(sigBytes[:len(sigBytes)/2])
        s := new(big.Int).SetBytes(sigBytes[len(sigBytes)/2:])

        // Verify the signature
        return ecdsa.Verify(publicKey, payload, r, s)
    }

    func webhookHandler(w http.ResponseWriter, r *http.Request) {
        // Only accept POST requests
        if r.Method != http.MethodPost {
            http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
            return
        }

        // Read the request body
        body, err := io.ReadAll(r.Body)
        if err != nil {
            http.Error(w, "Failed to read request body", http.StatusBadRequest)
            return
        }
        defer r.Body.Close()

        // Get the signature from header
        signature := r.Header.Get("X-Signature")
        if signature == "" {
            http.Error(w, "Missing signature", http.StatusBadRequest)
            return
        }

        // Load public key
        publicKey, err := loadPublicKey()
        if err != nil {
            log.Printf("Failed to load public key: %v", err)
            http.Error(w, "Internal server error", http.StatusInternalServerError)
            return
        }

        // Verify signature
        if !verifySignature(publicKey, signature, body) {
            http.Error(w, "Invalid signature", http.StatusUnauthorized)
            return
        }

        // Parse the payload
        var payload WebhookPayload
        if err := json.Unmarshal(body, &payload); err != nil {
            http.Error(w, "Invalid payload", http.StatusBadRequest)
            return
        }

        // Validate event type
        if payload.Event != "benefit" {
            http.Error(w, "Invalid event type", http.StatusBadRequest)
            return
        }

        // Process the verified payload
        log.Printf("Received verified benefit: %+v\n", payload)
        w.WriteHeader(http.StatusOK)
    }

    func main() {
        http.HandleFunc("/webhook", webhookHandler)
        
        port := ":3000"
        log.Printf("Server starting on port %s", port)
        log.Fatal(http.ListenAndServe(port, nil))
    }
    ```
  {{< /tab >}}

  {{< tab >}}
    ```js
    const express = require('express');
    const crypto = require('crypto');

    const app = express();

    // Store the public key as a constant
    const PUBLIC_KEY = `-----BEGIN PUBLIC KEY-----
    MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEWZrLPi5uAvrLKQ5mgWLXICH3bkk2
    oXl2WZLGwKysLdYPdsyc7Dc3UOPPFPdQH5Mjp24kDjiCVztCeNYc/PqR7Q==
    -----END PUBLIC KEY-----`;

    // Parse JSON bodies with raw body buffer for verification
    app.use(express.json({
        verify: (req, res, buf) => {
            // Store raw body for verification
            req.rawBody = buf;
        }
    }));

    function verifySignature(signature, payload) {
        try {
            // Convert hex signature to buffer
            const sigBuffer = Buffer.from(signature, 'hex');
            
            // ECDSA signature is two equal-length integers (r,s)
            const signatureLength = sigBuffer.length;
            if (signatureLength % 2 !== 0) {
                throw new Error('Invalid signature length');
            }

            const halfLength = signatureLength / 2;
            
            // Split signature into r and s components
            const r = sigBuffer.slice(0, halfLength);
            const s = sigBuffer.slice(halfLength);

            // Create DER encoding of the signature
            const derSignature = crypto.BN.concat([r, s]);
            
            // Create the verifier
            const verify = crypto.createVerify('SHA256');
            verify.update(payload);
            
            // Verify using the DER-encoded signature
            return verify.verify({
                key: PUBLIC_KEY,
                dsaEncoding: 'ieee-p1363' // This ensures correct ECDSA signature format
            }, derSignature);
        } catch (error) {
            console.error('Signature verification failed:', error);
            return false;
        }
    }

    app.post('/webhook', (req, res) => {
        try {
            const signature = req.headers['x-signature'];
            if (!signature) {
                return res.status(400).json({ error: 'Missing signature header' });
            }

            // Verify signature using raw body
            const isValid = verifySignature(signature, req.rawBody);
            if (!isValid) {
                return res.status(401).json({ error: 'Invalid signature' });
            }

            // Process the verified webhook payload
            const payload = req.body;
            
            // Validate event type
            if (payload.event !== 'benefit') {
                return res.status(400).json({ error: 'Invalid event type' });
            }

            // Process the benefit data
            const benefitData = payload.data;
            console.log('Received verified benefit:', {
                id: benefitData.id,
                amount: benefitData.amount,
                currency: benefitData.currency,
                donorName: benefitData.donor_name,
                message: benefitData.message,
                createdAt: benefitData.created_at
            });

            // Send success response
            res.status(200).json({ status: 'success' });
        } catch (error) {
            console.error('Webhook processing error:', error);
            res.status(500).json({ error: 'Internal server error' });
        }
    });

    // Health check endpoint
    app.get('/health', (req, res) => {
        res.status(200).json({ status: 'healthy' });
    });

    const PORT = process.env.PORT || 3000;
    app.listen(PORT, () => {
        console.log(`Server is running on port ${PORT}`);
    });

    // Handle uncaught errors
    process.on('uncaughtException', (error) => {
        console.error('Uncaught Exception:', error);
        process.exit(1);
    });

    process.on('unhandledRejection', (reason, promise) => {
        console.error('Unhandled Rejection at:', promise, 'reason:', reason);
        process.exit(1);
    });
    ```
  {{< /tab >}}

{{< /tabs >}}
