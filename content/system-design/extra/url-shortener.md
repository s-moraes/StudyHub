# Design a URL Shortener

Map billions of long URLs to short, unique keys — and redirect in milliseconds.

![URL Shortener Architecture](../assets/url-shortener.webp)

## Functional Requirements
- Given a long URL, generate a shorter alias.
- Given a short alias, redirect the user to the original URL (HTTP 301 Permanent Redirect).

## Core Encoding Strategy
Base62 encoding using characters `[0-9a-zA-Z]` allows 62^7 combinations (~3.5 trillion URLs) with just 7 characters.

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
