# Support

Support is free. Heystack is a small project with one maintainer and few users so far, and every question helps make it better.
That may change later, when the product is more robust and a lot of people depend on it; if it does, it will be announced
here, and nothing you use today will start to cost money.

## How to get help

- **Something is wrong**: run `heystack-doctor` (in Docker: `docker compose exec heystack heystack-doctor`). It checks the
  machine, the settings, the service, the database, the stored secrets, the backups and the tunnel, and says what to do about
  each problem.
- **Read first**: [INSTALL.md](INSTALL.md) (including "Something wrong?"), [FAQ.md](FAQ.md) and [LIMITATIONS.md](LIMITATIONS.md).
- **Still stuck, a bug, or an idea**: open an
  [issue](https://github.com/widgemo/heystack-releases/issues/new/choose) and paste the output of `heystack-doctor --report`.
  The report leaves out secrets, addresses and your home folder, but read it before you post it.
- **Prefer e-mail**: support@widgemo.com. Send the same report.
- **A security problem**: do not open a public issue; see [SECURITY.md](SECURITY.md).

Messages are read and answered as time allows, usually within a few days, with no promise about how fast.

## What we cannot do

- We cannot see your installation: Heystack sends nothing to us. Everything we know is what you tell us.
- We do not support modified copies, and we cannot help with the apps you deploy with it.
