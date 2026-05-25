## Using SSL autosigned for local development
To use SSL with autosigned certificates for local development, follow these steps:
1. **Generate a self-signed certificate**:
   You can use OpenSSL to generate a self-signed certificate. Run the following command in your terminal:
   ```bash
      openssl req -x509 -newkey rsa:2048 -nodes -keyout localhost.key -out localhost.crt -days 365 -subj "/CN=localhost"
   ```
   This command will create two files: localhost.key and localhost.crt.
2. **Start the app using angular cli**:
```bash
   ng serve --ssl --ssl-key localhost.key --ssl-cert localhost.crt
```

## CI validation note

This update is intentionally minimal and is used to validate Arcadia frontend workflow triggering for changes under `frontend/arcadiaFront/**`.

## Booking UX visual roadmap (mf-booking)

### Phase 1 (implemented)
- Professional selector (Step 2): initials avatar in each chip (no external images required).
- Service catalog (Step 1): semantic Material icon per service category (barber, wellness, medical, beauty, fallback).
- Professional name rendering rule: if GraphQL `fullName` is present, show it as source of truth; otherwise fallback to composed name.

### Phase 2 (next)
- Professional avatar with optional real photo URL (`photoUrl`/`imageUrl`) when backend contract provides it.
- Keep deterministic fallback chain for robustness:
   1. photo URL (if available and valid),
   2. initials avatar,
   3. generic `person` icon.
- Service cards: optional branded illustration/image support only from internal approved assets/CDN (no random external sources).
- Accessibility: preserve label readability and include `alt`/fallback behavior for broken images.

### Why this approach
- Avoids copyright/risk from arbitrary external images.
- Keeps fast loading and predictable UI in low-bandwidth environments.
- Allows progressive enhancement once backend exposes stable media fields.


