# Design a URL Shortener

Map billions of long URLs to short, unique keys — and redirect in milliseconds.

---

## 1. System Requirements

* **Functional:**
  * Given a long URL, generate a unique 7-character short alias (e.g., `https://sho.rt/2TX9aB1`).
  * When a user requests the short URL, redirect (`HTTP 301` or `302`) to the original target URL.
* **Non-Functional:**
  * Read-heavy workload (**100:1** read-to-write ratio).
  * Redirection latency under **10 ms**.
  * High availability.

---

## 2. Capacity & Key Space Math

Using **Base62** (`[a-z, A-Z, 0-9]`), a 7-character key provides:
* $62^7 = 3,521,614,606,208$ (~3.5 trillion unique URLs).

At 1,000 new URLs written per second, 3.5 trillion keys will last over **110 years**.

---

## 3. ID Generation & Base62 Encoding

Instead of hashing the URL with MD5/SHA-256 (which risks collisions when truncated to 7 characters), use a **Distributed Unique ID Generator** (such as Twitter Snowflake or range-allocated counters) and convert the 64-bit integer to Base62:

```java
public class Base62Encoder {
    private static final String ALPHABET = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";

    public static String encode(long id) {
        StringBuilder sb = new StringBuilder();
        while (id > 0) {
            sb.append(ALPHABET.charAt((int) (id % 62)));
            id /= 62;
        }
        return sb.reverse().toString();
    }
}
```

---

## 4. HTTP 301 vs. HTTP 302 Redirects

* **`301 Moved Permanently`:** The browser caches the redirect target locally. Reduces load on your servers, but you lose click-tracking analytics on repeat visits.
* **`302 Found (Temporary)`:** Every click hits your backend first, allowing real-time click analytics and dynamic target updates.
