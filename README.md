# WE ARE. — Messaging System + Production Deployment

This package adds a simple real-time messaging system to a static HTML/CSS/JS website using Supabase.

## Production architecture
- Frontend: GitHub Pages (or any static host)
- Database/Auth/Realtime: Supabase
- Authentication: email/password
- Messages: private 1-to-1 conversations
- Realtime updates: Supabase Realtime

## 1. Create Supabase project
Create a free Supabase project, then open SQL Editor and run `supabase/schema.sql`.

## 2. Configure the site
Copy `supabase/config.example.js` to `supabase/config.js` and put in:
- Supabase Project URL
- Supabase anon/publishable key

Never put a Supabase service-role key in frontend code.

Then include these scripts in your page:

```html
<script type="module" src="./messaging/messaging.js"></script>
```

See `messaging/example.html` for the required HTML structure.

## 3. GitHub Pages deployment
Upload the files to your repository. Push to the branch configured for Pages.

GitHub:
Settings → Pages → Deploy from a branch → select your production branch and `/ (root)`.

Your site can then use the messaging system directly from GitHub Pages.

## Security
The SQL schema uses Row Level Security (RLS). Users can only read conversations/messages they belong to and can only send messages as themselves.

For a real production launch, also configure:
- a custom domain if desired
- Supabase email confirmation/password recovery
- rate limits / abuse protection
- a report/block system
- privacy policy and terms
