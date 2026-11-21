# Redmine on Wodby

What Wodby sets up for Redmine on this service.

## Image

The service runs the prebuilt `wodby/redmine` image, with nothing built for it: Redmine's official image (Debian, Ruby,
Redmine and its gems) with precompiled assets, running Puma as the `wodby` user (uid 1000).

Plugins and themes need an image of their own built `FROM wodby/redmine:<version>`, with plugins copied to
`/usr/src/redmine/plugins` and themes to `/usr/src/redmine/themes` (plus `bundle install` for plugins with gems). Use it
from a custom service based on this one. The install action runs the plugins' migrations.

## Configuration

`config/database.yml` and `config/configuration.yml` read environment variables when Redmine starts; everything else is
set in Redmine under Administration › Settings.

- **Database.** PostgreSQL from the `db` link: `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`. When the
  linked server has a CA certificate (`DB_SSL_CA`), Redmine connects with `verify-full` TLS.
- **Attachments.** `/mnt/files`, the `files` volume shared by every replica through the linked storage service.
- **Processes.** Puma runs `workers` processes (setting, default 2) with 5 threads each (`RAILS_MAX_THREADS`) on port
  3000. Redmine serves its own stylesheets and scripts. `/-/health` answers without the database.
- **Sessions.** `SECRET_KEY_BASE` is a generated token; changing it signs everybody out. The "Stay logged in" cookie is
  sent over HTTPS only.
- **Email.** With a mail server linked as `sendmail`, `SMTP_HOST` and `SMTP_PORT` become Redmine's SMTP server, without
  STARTTLS. Without one Redmine sends no email; set `SMTP_*` variables for another server (see the
  [wodby/redmine](https://github.com/wodby/redmine) README). Emails are sent during the request that causes them
  (`REDMINE_QUEUE_ADAPTER=inline`), so a restart never loses one; a failed delivery is logged and the request still
  succeeds.

## Installation and upgrades

After every deployment the **Install or upgrade Redmine** action (`redmine-update`) runs the core and plugin
migrations, then:

- **First deployment:** loads Redmine's default trackers, statuses, roles and workflows in the `language` setting
  (default `en`), and replaces Redmine's stock `admin`/`admin` account with the `admin-login` (default `admin`),
  `admin-password` (required) and `admin-email` settings in the same process, so the stock password never works.
  Redmine's host name and protocol are set to the app's main address (`WODBY_PRIMARY_URL`, passed with
  `routeMetadata`) and the email sender to `noreply@` that host.
- Without an admin password, or with one Redmine refuses (shorter than 8 characters), the stock account is locked and
  the action fails.
- **Later deployments:** only the migrations, for example after a new image. The host name and the sender are not set
  again once they are stored, and changing the admin settings does not change the account; change them in Redmine.

Redmine has no background workers or scheduled jobs on this service. Redmine's own rake tasks, such as
`redmine:send_reminders` or `redmine:email:receive_imap`, can be added as cron jobs: `bin/rake redmine:send_reminders`.

## Limits

- PostgreSQL only.
- Plugins and themes need an image built on `wodby/redmine`.
- Repositories (Git, Subversion) work only with paths inside the container; there is no shared repository volume.
- No development workspace.

## Check the result

- `curl -s localhost:3000/-/health` in the container returns `OK`.
- The deployment's post-deploy log shows lines starting with `[redmine-update]`.
- `bin/rails console` opens a Rails console in `/usr/src/redmine`.
