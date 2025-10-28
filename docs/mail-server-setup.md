# IPUniverse Mail Server

This folder sets up a [Stalwart Mail](https://www.stalw.art/) instance backed by PostgreSQL so that both incoming and outgoing mail for `ipuniverse.org` is handled by your own Docker stack (deployable to Coolify).

The components run inside `compose.yml`:

| Service    | Purpose                                                          |
| ---------- | ---------------------------------------------------------------- |
| `postgres` | Stores accounts, queue metadata, and message blobs for Stalwart. |
| `mail`     | Stalwart Mail server (SMTP, IMAP, JMAP, admin UI).               |

## 1. Bootstrap

1. Create a `.env` file (or export shell vars) with a strong password for the mail database:

   ```bash
   export MAIL_DB_PASSWORD='change_this'
   ```

2. Either export `MAIL_DB_PASSWORD` in your shell or create a `.env` file so Docker Compose picks it up (defaults to `stalwart_pass` if unset).
3. Start the stack locally first:

   ```bash
   docker compose up -d postgres
   docker compose up -d mail
   ```

   On first boot Stalwart seeds `/opt/stalwart/config` with defaults inside the mounted `stalwart/` directory.
   The compose file maps the admin UI to host port `8081`; adjust if that port is also in use.

## 2. Secure the Admin UI

1. Open `http://localhost:8081` (or the Coolify forwarded port) and log in with:
   - username: `admin`
   - password: `change_me_admin`
2. **Immediately** change the password under **Account → Security**.
3. Under **System → Storage**, switch every store (`Directory`, `Mailbox`, `Queue`, `Lookup`, `Envelopes`) to `PostgreSQL` and use this DSN:  
   `postgres://stalwart:YOUR_PASSWORD@postgres:5432/stalwart`  
   (replace `YOUR_PASSWORD` with the same value used in `MAIL_DB_PASSWORD`).
4. Restart the `mail` container after saving.

> Stalwart automatically creates the required tables the first time it connects to the database.

## 3. Configure Domains & Accounts

1. In the admin UI go to **Directory → Domains** and add `ipuniverse.org` as the primary domain.
2. Create the SMTP user for your app (`no-reply@ipuniverse.org`) under **Directory → Accounts**.
3. Optional: create additional inbox users if you want to log in via IMAP/JMAP.

## 4. TLS Certificates

You need a valid certificate for `mail.ipuniverse.org`. Two common approaches:

- **Coolify-managed**: terminate TLS in Coolify and proxy ports 465/587/993/8080 to the container (ensure STARTTLS still works).
- **Let's Encrypt inside the host**: use `certbot` or `acme.sh` to issue a certificate and mount it into the container:

  ```bash
  stalwart/config/certs/fullchain.pem
  stalwart/config/certs/privkey.pem
  ```

  Then, in the admin UI, register the certificate under **Security → TLS Certificates** and assign it to the SMTP/IMAP listeners.

## 5. DNS Records

Configure these records at your DNS provider (replace the TTL with your preferred value):

| Type   | Name                  | Value                                                          | Notes                                               |
| ------ | --------------------- | -------------------------------------------------------------- | --------------------------------------------------- |
| `A`    | `mail`                | Public IPv4 of the host/Coolify deployment                     | Required for MX and PTR matching.                   |
| `AAAA` | `mail`                | (optional) IPv6                                                | Needed if your host has IPv6.                       |
| `MX`   | `@`                   | `mail.ipuniverse.org.` (priority 10)                           | Routes inbound mail.                                |
| `TXT`  | `@`                   | `v=spf1 mx -all`                                               | Allows your server to send mail.                    |
| `TXT`  | `_dmarc`              | `v=DMARC1; p=quarantine; rua=mailto:postmaster@ipuniverse.org` | Tune policy once reputation is stable.              |
| `TXT`  | `selector._domainkey` | (DKIM public key)                                              | Generate inside Stalwart under **Security → DKIM**. |
| `CAA`  | `@`                   | `0 issue "letsencrypt.org"`                                    | Optional but recommended.                           |

After the DNS propagates, run external checks (for example, mail-tester.com) to confirm SPF/DKIM/DMARC.

## 6. Next.js (Nodemailer) Integration

Add an SMTP transport that authenticates against the Stalwart submission port (587):

```ts
// lib/mailer.ts
import nodemailer from "nodemailer";

export const mailer = nodemailer.createTransport({
  host: "mail.ipuniverse.org",
  port: 587,
  secure: false, // STARTTLS
  auth: {
    user: "no-reply@ipuniverse.org",
    pass: process.env.MAIL_ACCOUNT_PASSWORD!, // store in Coolify secrets
  },
  tls: {
    rejectUnauthorized: true,
  },
});
```

Use it in your OTP/transactional handler:

```ts
await mailer.sendMail({
  from: '"IPUniverse" <no-reply@ipuniverse.org>',
  to: userEmail,
  subject: "Your OTP",
  text: `Your code is ${otp}`,
});
```

## 7. Accessing Incoming Mail from the App

Stalwart exposes:

- **IMAP (993)** – traditional clients.
- **JMAP** – JSON API; enable it under **Protocols → JMAP** in the admin UI and expose the configured port.
- **Admin REST API (8081 → container 8080)** – for automation (create accounts, read queue, etc.).

For app-side processing you can:

1. Enable JMAP and query mail via HTTP (recommended if you need structured data).
2. Use IMAP libraries (e.g., `imapflow`) from your Node backend.
3. Query PostgreSQL directly for custom analytics (tables are prefixed with `sw_`). Use views instead of direct writes.

## 8. Coolify Deployment Notes

- Add both containers to the same Docker network in Coolify.
- Expose the necessary ports (25, 465, 587, 993, 8081, and optional 4190 for Sieve, 8000 for JMAP) with firewall rules allowing the Internet to reach 25/465/587/993.
- Persist the volumes (`stalwart/config`, `stalwart/data`, `stalwart/logs`, `postgres/data`).
- Configure health checks (e.g., TCP on 587 and 993).
- If Coolify proxies the app through Traefik, set the internal service port to 8080 or add a label like `traefik.http.services.stalwart-admin.loadbalancer.server.port=8080` so the proxy reaches the admin UI.

## 9. Operational Tips

- Monitor logs via `docker compose logs -f mail`.
- Set up backups for the PostgreSQL volume.
- Regularly rotate the `no-reply` password and any admin passwords.
- Enable rate limits/anti-abuse rules inside Stalwart before going live.
- Use external tools (MXToolbox, Postmaster Tools) to verify deliverability.

With these steps, `mail.ipuniverse.org` becomes a full-featured SMTP/IMAP/JMAP server that your Next.js app can use for both outbound and inbound email while persisting everything in PostgreSQL. Update the credentials and policies before going live.
