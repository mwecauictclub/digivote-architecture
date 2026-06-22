# Email Provider Options

MWECAU DigiVote supports multiple email backends for transactional mail (verifications, password resets, election notifications, etc.).

## Supported Providers

| Provider     | How it works                              | Recommended for          | Setup Notes |
|--------------|-------------------------------------------|--------------------------|-------------|
| **Brevo**    | django-anymail + Brevo API (HTTPS)        | Default / production     | Free tier available (300 emails/day). Set `EMAIL_BACKEND=anymail.backends.brevo.EmailBackend` + API key. |
| **SMTP**     | Standard Django SMTP backend              | Simple setups, testing   | Any SMTP server (Gmail, custom mail server, Mailgun SMTP, etc.). Use `django.core.mail.backends.smtp.EmailBackend`. |
| **AWS SES**  | Boto3 + Django SES backend or Anymail     | High volume, AWS users   | Requires AWS credentials + SES verified domain. Good for scalability. |

## Configuration

All email sending is performed through Celery tasks to avoid blocking the web thread.

Key settings (in `.env` + `mw_es/settings.py`):

```env
# Brevo (current default in many deploys)
EMAIL_BACKEND=anymail.backends.brevo.EmailBackend
BREVO_API_KEY=...

# Or classic SMTP
EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
EMAIL_HOST=smtp.example.com
EMAIL_HOST_USER=...
EMAIL_HOST_PASSWORD=...

# AWS SES example (via django-ses or anymail)
EMAIL_BACKEND=django_ses.SESBackend
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
```

## Celery Queue

Email tasks are routed to the `email_queue` (separate from heavier notification fan-out).

This allows independent scaling of email sending.

## Best Practices

- Always send email via background tasks.
- Use a verified "from" domain.
- Monitor delivery rates (Brevo and AWS provide excellent dashboards).
- Fall back to console backend in development.

Changing providers only requires updating environment variables — no code changes needed for the core flows.
