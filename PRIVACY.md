# Privacy: what leaves your machine

Heystack runs on your own machine and stores its data in your own PostgreSQL database. It has **no telemetry**,
no analytics and no account with us; the people who make Heystack do not receive your data or know that you
run it.

Heystack only connects to other services when you set up a feature that needs them:

| Feature | Where it connects | What is sent |
|---|---|---|
| AI queries (off unless `ANTHROPIC_API_KEY` is set) | Anthropic API | Your question and the project information needed to answer it (names, descriptions, stacks, ports, repository and hosting links, notes). Nothing is sent until you ask a question. Leave the key empty to switch the feature off. |
| Status checks | The ports and URLs you register | A connection or a request to each address you registered. |
| Hosting integrations (Vercel, Netlify, Render, Railway, Fly, Cloudflare) | The provider's API | Your API token, to read project and deployment information. Tokens are stored encrypted. |
| GitHub deployment reporting (off by default) | GitHub's API | Your token, and deployment status for the repositories you configure. |
| Git Sync | Your own repositories | The machine pulls from your repositories with git, as you configured. GitHub sends push notifications to the webhook address you set up. |

Backups contain your data but not the encryption key. They stay where you put them (default: the `backups` folder of
the installation).

Questions: support@widgemo.com.
