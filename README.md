# Stage Clock

A prop clock. A **display** page runs on an old phone and shows a fixed time (12h, red on black). An **admin** page changes the time or blanks the screen from anywhere, live.

Svelte 5 + Vite. Sync is Firebase Realtime DB over REST/SSE (no SDK).

## Usage

- Display: `https://your-site/?k=SECRET`. Tap once for fullscreen and keep-awake.
- Admin: `https://your-site/?k=SECRET#admin`. Time picker and Blank button.

`SECRET` is the room name. It acts as the password: anyone with the URL can change the clock, so use a long random string. To revoke access, change it.

## Dev (no database)

```sh
npm install
npm run dev
```

With no `VITE_DB_URL`, tabs of the same browser sync through `localStorage`. Open `http://localhost:5173/` and `http://localhost:5173/#admin` in two tabs. `?k=` is optional here.

## Deploy

1. Create a Firebase project and a Realtime Database. Paste `database.rules.json` into its Rules tab.
2. `cp .env.example .env` and set `VITE_DB_URL` to your database URL.
3. `npm run build`, then host `dist/` on any static host (Cloudflare Pages, Netlify, GitHub Pages). If the host builds it, set `VITE_DB_URL` as a build variable.

## Files

- `src/App.svelte`: display, admin and sync, all in one.
- `database.rules.json`: no listing, per-room read/write, value validation.
