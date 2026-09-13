# WE ARE. — Feed Interactions Spec
### Likes · Comments · Reposts · Shares

This covers the data model, API, and real-time events needed to wire the prototype's
`toggleLike`, `submitComment`, `toggleRepost`, and share button to a real backend.

---

## 1. Data model

```sql
create table posts (
  id            uuid primary key default gen_random_uuid(),
  author_id     uuid not null references users(id),
  community_id  uuid references communities(id),
  type          text not null check (type in ('text','image','video','poll','link')),
  content       text,
  media_url     text,
  link_title    text,
  link_domain   text,
  created_at    timestamptz not null default now(),
  deleted_at    timestamptz
);

create table likes (
  user_id    uuid not null references users(id),
  post_id    uuid not null references posts(id),
  created_at timestamptz not null default now(),
  primary key (user_id, post_id)
);

create table comments (
  id                uuid primary key default gen_random_uuid(),
  post_id           uuid not null references posts(id),
  user_id           uuid not null references users(id),
  parent_comment_id uuid references comments(id),  -- null = top-level
  content           text not null,
  created_at        timestamptz not null default now(),
  deleted_at        timestamptz
);

create table reposts (
  user_id           uuid not null references users(id),
  original_post_id  uuid not null references posts(id),
  quote_content     text,                            -- null = plain repost, set = quote-repost
  created_at        timestamptz not null default now(),
  primary key (user_id, original_post_id)
);

create table shares (
  id          uuid primary key default gen_random_uuid(),
  user_id     uuid not null references users(id),
  post_id     uuid not null references posts(id),
  target_type text not null check (target_type in ('external_link','message','community')),
  target_id   uuid,        -- conversation_id or community_id, if applicable
  created_at  timestamptz not null default now()
);

create table saves (
  user_id    uuid not null references users(id),
  post_id    uuid not null references posts(id),
  created_at timestamptz not null default now(),
  primary key (user_id, post_id)
);
```

**Counts**: don't store `like_count`/`comment_count` columns that can drift — compute via
`count(*)` with an index on `post_id`, and cache the result (Redis, 30–60s TTL) on hot posts.
Recompute on write, not on every read.

**Repost vs. share**: a *repost* re-publishes the post to the user's own feed/profile (like/unlike
is idempotent via the primary key). A *share* is a lighter action — send to a DM or copy a link —
logged for analytics/notifications but doesn't duplicate the post anywhere.

---

## 2. Endpoints

| Action | Method | Path | Body | Notes |
|---|---|---|---|---|
| Like a post | `PUT` | `/posts/:id/like` | — | Idempotent — safe to call if already liked |
| Unlike | `DELETE` | `/posts/:id/like` | — | Idempotent |
| List likers | `GET` | `/posts/:id/likes?cursor=` | — | Paginated |
| Add comment | `POST` | `/posts/:id/comments` | `{content, parent_comment_id?}` | Returns created comment |
| List comments | `GET` | `/posts/:id/comments?cursor=` | — | Top-level first; fetch replies via `parent_comment_id` |
| Edit comment | `PATCH` | `/comments/:id` | `{content}` | Author only |
| Delete comment | `DELETE` | `/comments/:id` | — | Soft delete (`deleted_at`), keep thread intact |
| Repost | `PUT` | `/posts/:id/repost` | `{quote_content?}` | Idempotent |
| Undo repost | `DELETE` | `/posts/:id/repost` | — | |
| Share | `POST` | `/posts/:id/share` | `{target_type, target_id?}` | Fire-and-forget log + triggers side effect (e.g. sends the DM) |
| Get post w/ viewer state | `GET` | `/posts/:id` | — | See response shape below |

### Example: `GET /posts/:id`
```json
{
  "id": "8f2c...",
  "author": { "id": "u_12", "name": "Dee Okoro", "username": "deeokoro" },
  "type": "text",
  "content": "Spent two years thinking I was the only person building this alone...",
  "counts": { "likes": 64, "comments": 1, "reposts": 3, "shares": 8 },
  "viewer_state": { "liked": false, "reposted": false, "saved": false }
}
```

### Example: `PUT /posts/:id/like` response
```json
{ "post_id": "8f2c...", "liked": true, "like_count": 65 }
```

---

## 3. Real-time / notification side effects

Each write emits an event (via WebSocket channel `post:{id}` and/or a queue for async work):

| Event | Triggers |
|---|---|
| `post.liked` | Push updated `like_count` to anyone viewing the post; create a notification for the author (skip if `author_id === user_id`) |
| `comment.created` | Push new comment to the post's comment thread; notify post author + parent-comment author if it's a reply |
| `post.reposted` | Insert the repost into the reposting user's own feed; notify original author |
| `post.shared` | If `target_type = message`, deliver as a message with a post-preview card; if `community`, cross-post reference |

Batch notifications for the same actor + action within a short window (e.g. "Theo and 4 others
liked your post") instead of firing one row per like — this is what keeps a notifications feed
useful instead of noisy.

---

## 4. Edge cases & guardrails

- **Double-tap protection**: `like`/`repost` use `PUT` on a composite primary key, so a duplicate
  request just no-ops instead of erroring — no need for client-side debounce logic beyond disabling
  the button mid-request.
- **Rate limiting**: cap comments/likes per user per minute (e.g. 1 comment/5s, 1 like/1s) to blunt
  spam bots without affecting real usage.
- **Soft delete everywhere**: deleting a post or comment sets `deleted_at` rather than removing the
  row, so reply threads and repost chains don't break. Filter `deleted_at is null` in all reads.
  Full erasure (GDPR-style "delete my data") is a separate, deliberate hard-delete job.
- **Deleted parent posts**: if a reposted or shared post's original is deleted, show
  "This post is no longer available" in place of the content rather than erroring the whole feed item.
- **Comment depth**: cap nested replies at 2–3 levels in the UI even if the schema allows arbitrary
  depth — deep threads become unreadable on mobile.
