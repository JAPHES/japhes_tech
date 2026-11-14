# AWS EC2 deployment

This deployment runs the existing Django WSGI application with Gunicorn behind
Nginx on an Ubuntu EC2 instance. WhiteNoise continues to serve the collected,
cache-busted static files, so the same source remains compatible with Vercel.

The commands below assume Ubuntu 24.04/22.04, the default `ubuntu` user, and the
application path `/home/ubuntu/japhestech` used by the included systemd unit.

## 1. Create and secure the EC2 instance

1. Launch an Ubuntu EC2 instance and associate an Elastic IP so the public IP
   does not change after a stop/start.
2. Configure its security group with:
   - SSH (`22`) from your own public IP only.
   - HTTP (`80`) from `0.0.0.0/0` and `::/0`.
   - HTTPS (`443`) from `0.0.0.0/0` and `::/0`.
3. Do not expose port `8000`; Gunicorn listens only on `127.0.0.1`.

## 2. Install the application

SSH into the instance and run:

```bash
sudo apt update
sudo apt install -y git nginx python3-venv python3-pip
git clone https://github.com/JAPHES/FUTURE_FS_01.git /home/ubuntu/japhestech
cd /home/ubuntu/japhestech
python3 -m venv .venv
.venv/bin/pip install --upgrade pip
.venv/bin/pip install -r requirements.txt
```

Create a strong secret key:

```bash
.venv/bin/python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
```

Copy the environment template and replace the value inside the existing single
quotes on the `SECRET_KEY` line. The quotes prevent shell characters in the
generated key from being interpreted when the file is sourced. If DNS is not
pointing at EC2 yet, temporarily add the Elastic IP and EC2 public DNS name to
`ALLOWED_HOSTS` as comma-separated values.

```bash
sudo cp deploy/aws/japhestech.env.example /etc/japhestech.env
sudo nano /etc/japhestech.env
sudo chown root:ubuntu /etc/japhestech.env
sudo chmod 640 /etc/japhestech.env
```

Prepare Django:

```bash
set -a
source /etc/japhestech.env
set +a
.venv/bin/python manage.py migrate --noinput
.venv/bin/python manage.py collectstatic --noinput
.venv/bin/python manage.py check
```

## 3. Enable Gunicorn and Nginx

```bash
sudo cp deploy/aws/japhestech.service /etc/systemd/system/japhestech.service
sudo systemctl daemon-reload
sudo systemctl enable --now japhestech
sudo systemctl status japhestech --no-pager

sudo cp deploy/aws/nginx.conf /etc/nginx/sites-available/japhestech
sudo ln -sfn /etc/nginx/sites-available/japhestech /etc/nginx/sites-enabled/japhestech
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

Verify the app locally on the instance before changing DNS:

```bash
curl --fail --header 'Host: japhes.secora.dev' http://127.0.0.1:8000/health/
curl --fail --header 'Host: japhes.secora.dev' http://127.0.0.1/
```

The health response should be `{"status": "ok"}`.

## 4. Point the domain to EC2 and enable HTTPS

Change the `japhes` DNS record from the Vercel target to an `A` record containing
the EC2 Elastic IP. DNS selects which provider serves `japhes.secora.dev`; the
Vercel deployment can remain available on its `*.vercel.app` URL.

After the `A` record resolves to the EC2 Elastic IP:

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d japhes.secora.dev
sudo certbot renew --dry-run
```

Then edit `/etc/japhestech.env` and enable the HTTPS settings:

```text
SECURE_SSL_REDIRECT=True
SESSION_COOKIE_SECURE=True
CSRF_COOKIE_SECURE=True
```

Restart the app and verify it:

```bash
sudo systemctl restart japhestech
curl --fail https://japhes.secora.dev/health/
```

Keep `SECURE_HSTS_SECONDS=0` until HTTPS has been stable. It can later be raised
carefully; HSTS is cached by browsers and is not immediately reversible.

## 5. Deploy later updates

```bash
cd /home/ubuntu/japhestech
git pull --ff-only origin master
.venv/bin/pip install -r requirements.txt
set -a
source /etc/japhestech.env
set +a
.venv/bin/python manage.py migrate --noinput
.venv/bin/python manage.py collectstatic --noinput
.venv/bin/python manage.py check
sudo systemctl restart japhestech
curl --fail https://japhes.secora.dev/health/
```

Useful diagnostics:

```bash
sudo journalctl -u japhestech -n 100 --no-pager
sudo nginx -t
sudo tail -n 100 /var/log/nginx/error.log
```
