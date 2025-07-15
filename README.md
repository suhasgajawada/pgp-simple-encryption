Perfect. For this local POC, you want a **clean, no-Vault, no-SFTP**, **controller-triggered decryption** flow where everything runs inside your Spring Boot app using files from `resources/`.

Let’s do this in 3 clean steps.

---

## ✅ What You Want

| Component          | What it should do                                                         |
| ------------------ | ------------------------------------------------------------------------- |
| Encrypted file     | Placed in `src/main/resources/data/sample.pdf.pgp`                        |
| Keys + passphrase  | In `src/main/resources/keys/` as `.asc` and `.txt`                        |
| Decryption trigger | Hitting a REST endpoint `/decrypt` should:                                |
|                    | → Read encrypted `.pgp` from resources                                    |
|                    | → Load Org B's private key + passphrase                                   |
|                    | → Load Org A's public key                                                 |
|                    | → Decrypt + verify                                                        |
|                    | → Save decrypted output to `src/main/resources/data/sample.decrypted.pdf` |

---

## 🔨 1. Replace VaultKeyLoader with LocalKeyLoader

```java
package com.yourorg.pgpsftpprocessor.vault;

import org.springframework.stereotype.Service;

import java.io.BufferedReader;
import java.io.InputStream;
import java.io.InputStreamReader;

@Service
public class VaultKeyLoader {

    public String getOurPrivateKey() throws Exception {
        return readFile("keys/orgb_private.asc");
    }

    public String getOurPassphrase() throws Exception {
        return readFile("keys/passphrase.txt").trim();
    }

    public String getTheirPublicKey() throws Exception {
        return readFile("keys/orga_public.asc");
    }

    private String readFile(String path) throws Exception {
        InputStream is = getClass().getClassLoader().getResourceAsStream(path);
        if (is == null) throw new IllegalArgumentException("File not found: " + path);
        BufferedReader reader = new BufferedReader(new InputStreamReader(is));
        StringBuilder sb = new StringBuilder();
        String line;
        while ((line = reader.readLine()) != null) sb.append(line).append("\n");
        return sb.toString();
    }
}
```

> ✅ No Vault, no external I/O — everything is read from `resources/`.

---

## 🧾 2. Create the Decryption Controller

```java
package com.yourorg.pgpsftpprocessor.controller;

import com.yourorg.pgpsftpprocessor.pgp.PgpProcessorService;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.io.File;
import java.net.URL;

@RestController
public class DecryptionController {

    private final PgpProcessorService processor;

    public DecryptionController(PgpProcessorService processor) {
        this.processor = processor;
    }

    @GetMapping("/decrypt")
    public String decryptTestFile() {
        try {
            // Load sample.pdf.pgp from resources
            URL inputUrl = getClass().getClassLoader().getResource("data/sample.pdf.pgp");
            if (inputUrl == null) return "Encrypted file not found.";

            File inputFile = new File(inputUrl.toURI());
            File outputFile = new File("src/main/resources/data/sample.decrypted.pdf");

            processor.verifyAndDecrypt(inputFile, outputFile);
            return "Decryption successful. Output at: " + outputFile.getPath();
        } catch (Exception e) {
            e.printStackTrace();
            return "Decryption failed: " + e.getMessage();
        }
    }
}
```

> ✅ This controller runs when you `GET /decrypt`, reads the file from classpath, and writes output to disk.

---

## 🚫 3. Comment out / Remove these:

### A. ❌ Spring Vault dependencies from `pom.xml`

```xml
<!-- REMOVE -->
<dependency>
  <groupId>org.springframework.vault</groupId>
  <artifactId>spring-vault-core</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-vault-config</artifactId>
</dependency>
```

### B. ❌ Any `@EnableVaultConfiguration` or `VaultConfig` classes

Remove any config class that sets up `VaultTemplate`.

### C. ❌ Scheduled SFTP Polling

Remove or comment out the `SftpPollScheduler` class and any mention of `SftpService`.

---

## 📁 4. Your directory structure should look like this:

```
src/main/resources/
├── keys/
│   ├── orga_public.asc
│   ├── orgb_private.asc
│   └── passphrase.txt
├── data/
│   └── sample.pdf.pgp
```

---

## ✅ Now, run the app and test:

```bash
curl http://localhost:8080/decrypt
```

You should see:

```
Decryption successful. Output at: src/main/resources/data/sample.decrypted.pdf
```

> If signature verification fails, you’ll get a clear message with stack trace.

---

## 🔄 Recap of What Changed

| Component | Action                              |
| --------- | ----------------------------------- |
| Vault     | ✅ Replaced with local file reader   |
| SFTP      | ✅ Disabled completely               |
| Trigger   | ✅ Added `/decrypt` REST endpoint    |
| Output    | ✅ File written to `resources/data/` |

---

Let me know if you want:

* A ZIP containing valid `.asc` keys and `.pgp` file to test this
* Or to reverse this setup later and plug Vault/SFTP back in

This version is now perfect for local testing with 100% deterministic results.
