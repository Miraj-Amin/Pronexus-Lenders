# Phoenix Lender Index — Supabase edition

A single-page app (`index.html`) on GitHub Pages, with Supabase for the database, sign-in and real-time sync, plus one Edge Function for the plain-English deal reader.

```
index.html                         the app (matching logic and UI unchanged from the original)
supabase/schema.sql                tables, access list, row-level security, realtime
supabase/functions/parse-deal/     Edge Function that calls Claude to read deal descriptions
```

## Setup (about 15 minutes)

**1. Create the database**
Open Supabase → SQL Editor → paste `supabase/schema.sql` → Run.
Then add your team (anyone not on this list can't sign in):
```sql
insert into public.allowed_users (email, name, is_admin) values
  ('miraj@yourfirm.co.uk', 'Miraj',  true),
  ('hanish@yourfirm.co.uk','Hanish', false);
```
`name` is used to sign the enquiry emails.

**2. Configure sign-in**
Authentication → URL Configuration:
- Site URL: `https://<your-github-user>.github.io/<repo>/`
- Redirect URLs: add the same URL
Authentication → Providers → Email: keep enabled. The built-in mailer has a low hourly limit, so set up custom SMTP (e.g. Resend or Postmark) before rolling out to the whole team.

**3. Point the page at your project**
In `index.html`, find `window.PLI_CONFIG` and paste your Project URL and the anon public key from Project Settings → API. The anon key is meant to be public, because row-level security protects the data.

**4. Deploy the deal reader**
```bash
supabase link --project-ref <your-ref>
supabase secrets set ANTHROPIC_API_KEY=sk-ant-...
# optional: supabase secrets set ANTHROPIC_MODEL=claude-haiku-5-5
supabase functions deploy parse-deal
```
The function checks the caller is signed in **and** on the access list before calling Claude.

**5. Publish**
Push `index.html` to a GitHub repo → Settings → Pages → deploy from the `main` branch.

## How data is stored

| Table | Contents |
|---|---|
| `lenders` | `id`, `name`, `profile` as columns; every criterion (LTVs, products, team, flags) in a `data` jsonb document |
| `notes` | One note per lender, with who last edited it |
| `verified` | Date, note and who confirmed the criteria |
| `deals` | Deal name and a snapshot of the deal-sheet form |
| `deal_approaches` | One row per lender approached on a deal: status, terms, date |
| `allowed_users` | The access list |

Changes reach everyone's open page within a second through Supabase Realtime.

## Managing access
- **Add someone:** insert a row into `allowed_users`. They can then request a link.
- **Remove someone:** delete their row. They lose data access immediately, even with a live session.
