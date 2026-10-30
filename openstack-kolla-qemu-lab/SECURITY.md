# 🔐 Security and publication

This is sanitized lab documentation, not a production hardening baseline.

## Never publish

- `/etc/kolla/passwords.yml`, `clouds.yaml`, OpenRC files or application credentials.
- Private keys, session cookies, API tokens or noVNC token URLs.
- Database dumps and raw debug logs containing connection credentials.
- Real client information or provider/account identifiers.

Do not upload `/etc/kolla` wholesale. Manually inspect browser uploads even with `.gitignore`.

## Still pending

- Rotate the exposed admin password, synchronizing Keystone, saved Kolla secrets and client configuration.
- Configure HTTPS and restrict API/dashboard/console ingress.
- Create separate users/projects, quotas and tested tenant isolation.
- Design backups and image/patch lifecycle.
- Reassess the outer provider port's disabled port security for the final host-NAT design.
- Keep the no-proxy Kolla settings consistent with disabled obsolete proxy systemd units.

Use verified SSH keys and tunnels when TLS is unavailable. Do not expose MariaDB/RabbitMQ administration ports publicly.

If a credential leaks: rotate it, verify dependent clients, then address Git history. Deleting the visible file alone is insufficient.
