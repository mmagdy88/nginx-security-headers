# nginx-security-headers

My nginx security header configs, built up over years of managing production web servers. I keep coming back to the same questions on new setups so I'm documenting everything here.

This isn't a framework or a tool — it's a snippet library you drop into your nginx config. The goal is understanding what each header does, not just copy-pasting a block and hoping for the best.

Tested on nginx 1.18+ and 1.24+, Ubuntu 20.04/22.04, Debian 11/12. Most of it applies anywhere.

---

## Structure

```
├── snippets/
│   ├── security-headers.conf     # Core headers — include this in every server block
│   ├── ssl-modern.conf           # TLS 1.2/1.3 only, strong ciphers, OCSP stapling
│   └── proxy-hide.conf           # Strip backend info headers from proxied responses
├── examples/
│   ├── server-block-full.conf    # Complete server block putting it all together
│   ├── wordpress.conf            # WordPress-specific CSP adjustments
│   └── reverse-proxy.conf        # When nginx sits in front of another app server
└── README.md
```

---

## Headers

### X-Content-Type-Options

```nginx
add_header X-Content-Type-Options "nosniff" always;
```

Stops the browser from guessing the content type. Without this, if you serve a file as `text/plain` and it contains JavaScript, some browsers will execute it anyway. Set it and forget it — there's no reason not to.

---

### X-Frame-Options

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
```

Prevents your site from being embedded in an `<iframe>` on another domain. The attack this blocks is clickjacking — someone puts your login page in a transparent iframe on their site, users think they're clicking their own page. `SAMEORIGIN` allows your own pages to iframe each other, `DENY` blocks iframes entirely.

---

### Strict-Transport-Security (HSTS)

```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

Tells browsers to only ever connect over HTTPS, even if someone types `http://`. The browser caches this for `max-age` seconds (1 year here).

Don't add this until HTTPS is working correctly on all subdomains. Once browsers cache it, getting back to HTTP requires the max-age to expire. I've broken staging environments this way by adding it too early.

If you're confident, you can add `preload` to the end and submit to the [HSTS preload list](https://hstspreload.org/). This puts your domain in browsers' built-in lists, not just the cache. That one is permanent — harder to undo.

---

### Referrer-Policy

```nginx
add_header Referrer-Policy "no-referrer-when-downgrade" always;
```

Controls what URL the browser sends in the `Referer` header when users click links. With this setting, the full URL is sent to HTTPS destinations, but nothing is sent when going from HTTPS to HTTP (no point leaking your URL to an unencrypted destination).

For sensitive apps (anything with user data in the URL), tighten this to `same-origin` or `no-referrer`.

---

### Permissions-Policy

```nginx
add_header Permissions-Policy "geolocation=(), microphone=(), camera=(), payment=()" always;
```

Disables browser features the site doesn't use. Empty `()` means nobody — not even your own origin — can access that feature. This matters most for compromised third-party JS: if an ad script or analytics library gets poisoned, it can't silently activate the camera.

Adjust based on what your app actually needs. A map application needs `geolocation=(*self*)`.

---

### Content-Security-Policy

```nginx
add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self'; connect-src 'self'; frame-ancestors 'none';" always;
```

The heavy one. CSP tells the browser exactly where it's allowed to load resources from. Scripts, styles, images, fonts, API calls — each has its own directive. If a source isn't listed, the browser blocks it.

`'unsafe-inline'` on `style-src` is a compromise for sites using inline CSS. Avoid `'unsafe-inline'` on `script-src` — that's the one that actually matters for XSS.

A strict CSP will break most sites on first apply. WordPress, anything with Google Fonts, anything with inline scripts — all will break. Start with [Report-Only mode](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy-Report-Only) to see what would be blocked before enforcing. The WordPress example in `/examples/wordpress.conf` has a more permissive version for that specific case.

---

### Server token suppression

```nginx
# In the http{} block in nginx.conf
server_tokens off;
```

Removes the nginx version from error pages and the `Server` response header. This goes from `Server: nginx/1.24.0` to `Server: nginx`. Not a real security control — anyone can fingerprint your nginx version through behavior — but it removes the low-hanging fruit for automated scanners.

If you want to remove the `Server` header entirely, you need the `ngx_headers_more` module:

```nginx
more_clear_headers 'Server';
```

That module isn't in the default Ubuntu/Debian nginx packages. You'll need to either compile it in or use a package from nginx.org.

---

## SSL/TLS config

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
ssl_prefer_server_ciphers off;

ssl_stapling on;
ssl_stapling_verify on;
resolver 1.1.1.1 8.8.8.8 valid=300s;

ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;
ssl_session_tickets off;
```

`ssl_prefer_server_ciphers off` lets the client pick from the allowlist. Modern clients will pick TLS 1.3 automatically. `ssl_session_tickets off` disables session tickets — the key rotation required to maintain forward secrecy on tickets is easy to get wrong, so I just disable them.

Generate your DH params if you're supporting older clients:

```bash
openssl dhparam -out /etc/nginx/dhparam.pem 2048
```

Then add to your server block:
```nginx
ssl_dhparam /etc/nginx/dhparam.pem;
```

---

## Testing

After deploying, check your work:

- [securityheaders.com](https://securityheaders.com) — grades your headers, shows what's missing
- [SSL Labs](https://www.ssllabs.com/ssltest/) — full TLS analysis, grades your cipher config
- `curl -I https://yourdomain.com` — quick header check from the terminal

Aim for A on securityheaders.com and A/A+ on SSL Labs.

---

## Usage

Copy `snippets/` to `/etc/nginx/snippets/` and add to your server block:

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    include snippets/security-headers.conf;
    include snippets/ssl-modern.conf;

    # rest of your config
}
```

---

## References

- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/)
- [Mozilla SSL Config Generator](https://ssl-config.mozilla.org/)
- [MDN: HTTP Headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers)
- [CSP Evaluator](https://csp-evaluator.withgoogle.com/) — paste your CSP policy, get feedback