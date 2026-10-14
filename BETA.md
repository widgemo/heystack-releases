# Trying Heystack: what to test and how to tell us

Thank you for trying Heystack. It is new software that runs on your own machine, and it sends us nothing, so we only
learn what is wrong when you tell us.

## What to try (about an hour, then use it normally)

1. **Install it** the way that suits your machine: Docker (see [DOCKER.md](DOCKER.md)) or the installer
   (see [INSTALL.md](INSTALL.md)). Note anything that was unclear or surprising, even if it worked.
2. **Create your account** with the one-time setup code and sign in.
3. **Add two or three of your real projects** (by hand or with the Scan page): ports, repositories, hosting links.
4. **Look at the status page** and let it run for a few days. Does it tell you something useful?
5. **Run the doctor**: `heystack-doctor`. Does it say anything you do not understand?
6. **Make a backup and restore it** (`heystack-backup`, `heystack-restore`) on a copy if you can, to see it work.
7. **Update** when a new version appears (run the installer of the new package, or `docker compose pull && docker compose up -d`).

## How to tell us

- **A problem:** open an issue here and choose "Bug report" or "Install or update problem". Run
  `heystack-doctor --report` and paste the output; it leaves out secrets, addresses and your home folder, but read it
  before you post it.
- **An idea or a confusing moment:** open an "Idea" issue, or email support@widgemo.com.
- **Something private or a security problem:** email support@widgemo.com (see [SECURITY.md](SECURITY.md)).

## What helps most

What you were trying to do, what you expected, what happened, and your machine (for example "Raspberry Pi 5,
Raspberry Pi OS" or "Ubuntu 24.04 PC with Docker"). Short and specific beats long and thorough.
