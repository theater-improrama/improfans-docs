---
title: 'Webhooks'
date: 2024-12-18T01:25:38+01:00
next: streamelements-integration
---

## Webhook bei ImproFans erstellen

Melde dich in deinem ImproFans-Account bei den Webhook-Einstellungen (https://improfans.de/u/settings/webhook) an. Hier musst du die Webhook-URL hinzufügen, die im Falle einer Spende von ImproFans aufgerufen werden soll.

{{< callout type="warning" >}}
  Deine Webhook-URL muss das HTTPS-Protokoll benutzen. Unverschlüsselte HTTP-Endpunkte werden nicht akzeptiert.
{{< /callout >}}

## Verwendung

ImproFans ruft deine konfigurierte Webhook-URL auf, sobald du eine Spende erhalten hast.

Dein Webhook wird folgende JSON-Payload von ImproFans empfangen:

```json
{
    "event": "benefit",
    "data": {
        "id": "6bf842c9-a09f-4ccb-8844-83b38dd4786c",
        "amount": 1.23,
        "currency": "EUR",
        "name": "pixlcrashr",
        "message": "This is a donation message.",
        "donor_id": "076ff569-7c58-4205-ae3a-e4fae554eb0a",
        "is_subscription": false,
        "created_at": "2024-12-15T01:04:55+00:00"
    }
}
```

Hier sind die Felder für das `benefit`-Event näher beschrieben:

| Feld | Beschreibung | Datentyp |
| --- | --- | --- |
| `event` | Typ des Webhook-Events. | String (nur: `benefit`) |
| `data.id` | ID des Benefits (= Spende). | String ([UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier)) |
| `data.amount` | Betrag des Benefits. | Number |
| `data.currency` | Währung des Benefits. | String (nur: `EUR`) |
| `data.name` | Spendername des Benefits. Kann `null` sein, falls kein Name übergeben wurde (z. B. eine anonyme Spende). | String, `null` |
| `data.message` | Nachricht zur Spende. | String |
| `data.donor_id` | Optionale Spender-ID des Benefits. | String ([UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier)), `null` |
| `data.is_subscription` | Zeigt an, ob diese Spende Teil eines Abonnements ist (WIP). | String ([UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier)), `null` |
| `data.created_at` | Zeitstempel, wann der Benefit erstellt wurde. | String ([ISO 8601](https://www.iso.org/obp/ui/#iso:std:iso:8601:-1:ed-1:v1:en)) |

{{< callout type="info" >}}
Dein Webhook-Endpunkt/Empfänger muss in der Lage sein, JSON zu parsen (die meisten Plattformen und Programmiersprachen unterstützen von Haus aus das Parsing von JSON).
{{< /callout >}}

Wurde der Webhook erfolgreich empfangen, muss ein [HTTP 200 OK](https://developer.mozilla.org/de/docs/Web/HTTP/Status/200)-Statuscode als Ergebnis zurückgegeben werden. Ein Statuscode, der von 200 OK abweicht, markiert das Webhook-Event als fehlgeschlagen und kann erneut an deinen Endpunkt gesendet werden.

## Fortgeschritten: Validierung der Integrität

{{< callout type="warning" >}}
  Obwohl wir **dringend empfehlen**, die Webhook-Payload zu validieren, ist dies je nach Anwendungsfall nicht zwingend erforderlich.
{{< /callout >}}

Da dein Webhook öffentlich zugänglich sein muss, damit wir Events senden können, könnten bösartige Akteure ungültige Payloads an deinen Endpunkt senden. Daher signieren wir das HTTP-Payload mittels einer ECDSA-Signatur und stellen diese im HTTP-Header `X-Signature` bereit.

Beispiel:

```
X-Signature: B7BE60F73971924D9547C6C8C755866DF9DA56C19AD50B7B571827F6823D8434EA46AE1D4FADFBE4E46A9518747817A0F63FE6D4E59503982E07BDEBC8788C22
```

Dies ist unser öffentlicher Schlüssel – ein EC (Elliptic Curves Public Key):

```
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEWZrLPi5uAvrLKQ5mgWLXICH3bkk2
oXl2WZLGwKysLdYPdsyc7Dc3UOPPFPdQH5Mjp24kDjiCVztCeNYc/PqR7Q==
-----END PUBLIC KEY-----
```

### Beispiel

Hier ein kurzes Beispiel, wie du in einer Programmiersprache die Signatur einer Webhook-Anfrage überprüfen kannst:

{{< tabs items="Go,JS" >}}

  {{< tab >}}
    ```go
    package main

    import (
        "crypto/ecdsa"
        "crypto/sha256"
        "crypto/x509"
        "encoding/hex"
        "encoding/pem"
        "io"
        "log"
        "math/big"
        "net/http"
    )

    const publicKeyPEM = `-----BEGIN PUBLIC KEY-----
    MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEWZrLPi5uAvrLKQ5mgWLXICH3bkk2
    oXl2WZLGwKysLdYPdsyc7Dc3UOPPFPdQH5Mjp24kDjiCVztCeNYc/PqR7Q==
    -----END PUBLIC KEY-----`

    var publicKey *ecdsa.PublicKey

    func main() {
        // Parse the public key once at startup
        var err error
        publicKey, err = parsePublicKey(publicKeyPEM)
        if err != nil {
            log.Fatalf("Failed to parse public key: %v", err)
        }

        http.HandleFunc("/webhook", handleWebhook)
        
        log.Println("Server starting on :8080")
        log.Fatal(http.ListenAndServe(":8080", nil))
    }

    func handleWebhook(w http.ResponseWriter, r *http.Request) {
        // Only accept POST requests
        if r.Method != http.MethodPost {
            http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
            return
        }

        // Read the signature from header
        signature := r.Header.Get("X-Signature")
        if signature == "" {
            http.Error(w, "Missing signature", http.StatusBadRequest)
            return
        }

        // Read the request body
        body, err := io.ReadAll(r.Body)
        if err != nil {
            http.Error(w, "Failed to read request body", http.StatusBadRequest)
            return
        }
        defer r.Body.Close()

        // Verify the signature
        valid, err := verifySignature(body, signature)
        if err != nil {
            log.Printf("Signature verification error: %v", err)
            http.Error(w, "Invalid signature format", http.StatusBadRequest)
            return
        }

        if !valid {
            log.Printf("Invalid signature for payload: %s", string(body))
            http.Error(w, "Invalid signature", http.StatusUnauthorized)
            return
        }

        // Signature is valid, log the payload
        log.Printf("Received valid webhook: %s", string(body))
        w.WriteHeader(http.StatusOK)
    }

    func verifySignature(message []byte, signatureHex string) (bool, error) {
        // Decode the hex signature
        signature, err := hex.DecodeString(signatureHex)
        if err != nil {
            return false, err
        }

        // Signature should be exactly 64 bytes (r and s are 32 bytes each)
        if len(signature) != 64 {
            return false, err
        }

        // Split signature into r and s
        r := new(big.Int).SetBytes(signature[:32])
        s := new(big.Int).SetBytes(signature[32:])

        // Calculate the hash of the message
        hash := sha256.Sum256(message)

        // Verify the signature
        return ecdsa.Verify(publicKey, hash[:], r, s), nil
    }

    func parsePublicKey(pemKey string) (*ecdsa.PublicKey, error) {
        block, _ := pem.Decode([]byte(pemKey))
        if block == nil {
            return nil, err
        }

        pub, err := x509.ParsePKIXPublicKey(block.Bytes)
        if err != nil {
            return nil, err
        }

        ecdsaPub, ok := pub.(*ecdsa.PublicKey)
        if !ok {
            return nil, err
        }

        return ecdsaPub, nil
    }
    ```
  {{< /tab >}}

  {{< tab >}}
    ```js
    const express = require('express');
    const crypto = require('crypto');

    const app = express();

    // Configure Express to get raw body for verification
    app.use(express.json({
        verify: (req, res, buf) => {
            // Store raw body for signature verification
            req.rawBody = buf;
        }
    }));

    // The public key used for verification
    const publicKeyPEM = `-----BEGIN PUBLIC KEY-----
    MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEWZrLPi5uAvrLKQ5mgWLXICH3bkk2
    oXl2WZLGwKysLdYPdsyc7Dc3UOPPFPdQH5Mjp24kDjiCVztCeNYc/PqR7Q==
    -----END PUBLIC KEY-----`;

    function verifySignature(message, signatureHex) {
        try {
            // Convert hex signature to buffer
            const signature = Buffer.from(signatureHex, 'hex');
            
            // Signature should be exactly 64 bytes (r and s are 32 bytes each)
            if (signature.length !== 64) {
                throw new Error('Invalid signature length');
            }

            // Split signature into r and s components
            const r = signature.slice(0, 32);
            const s = signature.slice(32);

            // Create verifier
            const verifier = crypto.createVerify('SHA256');
            verifier.update(message);

            // Verify the signature (using raw/ieee-p1363 format)
            return verifier.verify({ key: publicKeyPEM, dsaEncoding: 'ieee-p1363' }, Buffer.concat([r, s]));
        } catch (error) {
            console.error('Signature verification error:', error);
            return false;
        }
    }

    app.post('/webhook', (req, res) => {
        try {
            // Get signature from header
            const signature = req.headers['x-signature'];
            if (!signature) {
                console.log('Missing signature header');
                return res.status(400).json({ error: 'Missing signature' });
            }

            // Verify signature using raw body
            const isValid = verifySignature(req.rawBody, signature);
            
            if (!isValid) {
                console.log('Invalid signature for payload:', req.body);
                return res.status(401).json({ error: 'Invalid signature' });
            }

            // Log the valid webhook
            console.log('Received valid webhook:', req.body);
            
            // Send success response
            res.status(200).json({ status: 'success' });
        } catch (error) {
            console.error('Webhook processing error:', error);
            res.status(500).json({ error: 'Internal server error' });
        }
    });

    // Add basic error handling
    app.use((err, req, res, next) => {
        console.error('Unhandled error:', err);
        res.status(500).json({ error: 'Internal server error' });
    });

    // Start the server
    const PORT = process.env.PORT || 8080;
    app.listen(PORT, () => {
        console.log(`Server is running on port ${PORT}`);
    });
    ```
  {{< /tab >}}

{{< /tabs >}}