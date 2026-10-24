<script>
  const DB = import.meta.env.VITE_DB_URL
  // room name in the URL is the secret (capability URL); no auth. Add Firebase Auth if it must be revocable.
  const q = new URLSearchParams(location.search)
  const room = q.get('k') || (DB ? null : 'dev')
  let admin = $state(location.hash === '#admin')
  addEventListener('hashchange', () => (admin = location.hash === '#admin'))
  const url = `${DB}/${room}.json`

  let s = $state({ t: '12:00', blank: false })

  // No DB configured: sync between tabs of the same browser via localStorage (dev only).
  const LS = 'stage-clock:' + room
  $effect(() => {
    if (!room) return
    if (!DB) {
      s = JSON.parse(localStorage.getItem(LS) || 'null') ?? s
      const on = (e) => e.key === LS && e.newValue && (s = JSON.parse(e.newValue))
      addEventListener('storage', on)
      return () => removeEventListener('storage', on)
    }
    const es = new EventSource(url)
    es.addEventListener('put', (e) => {
      const { path, data } = JSON.parse(e.data)
      if (path === '/' && data) s = data
    })
    // poll too; some phones' browsers silently stall SSE (sleep, data saver). Drop if SSE proves reliable.
    const poll = () => fetch(url).then((r) => r.json()).then((d) => d && (s = d)).catch(() => {})
    const iv = setInterval(poll, 3000)
    return () => (es.close(), clearInterval(iv))
  })

  const push = (patch) => {
    s = { ...s, ...patch }
    if (!DB) return localStorage.setItem(LS, JSON.stringify(s))
    fetch(url, { method: 'PUT', body: JSON.stringify(s) })
  }

  // "19:42" -> "7:42" + "PM"
  const parts = $derived.by(() => {
    const [h, m] = s.t.split(':').map(Number)
    return { t: `${String(h % 12 || 12).padStart(2, "0")}:${String(m).padStart(2, '0')}`, ap: h < 12 ? 'AM' : 'PM' }
  })

  const set = ({ h12 = +parts.t.slice(0, 2), m = +parts.t.slice(3), pm = parts.ap === 'PM' }) =>
    push({ t: `${String((h12 % 12) + (pm ? 12 : 0)).padStart(2, '0')}:${String(m).padStart(2, '0')}` })

  const fs = () => document.documentElement.requestFullscreen?.()
  const wake = () => navigator.wakeLock?.request('screen').catch(() => {})
</script>

{#if !room}
  <p class="msg">Missing <code>?k=roomname</code> in URL.</p>
{:else if admin}
  <main class="admin">
    <h1>Clock admin</h1>
    <!-- selects, not <input type=time>, which hides AM/PM on 24h-locale browsers -->
    <div class="pick">
      <select value={parts.t.slice(0, 2)} onchange={(e) => set({ h12: +e.target.value })}>
        {#each Array.from({ length: 12 }, (_, i) => i + 1) as h}<option value={h}>{String(h).padStart(2, '0')}</option>{/each}
      </select>
      <select value={parts.t.slice(3)} onchange={(e) => set({ m: +e.target.value })}>
        {#each Array.from({ length: 60 }, (_, i) => i) as m}<option value={String(m).padStart(2, '0')}>{String(m).padStart(2, '0')}</option>{/each}
      </select>
      <select value={parts.ap} onchange={(e) => set({ pm: e.target.value === 'PM' })}>
        <option>AM</option><option>PM</option>
      </select>
    </div>
    <button class:on={s.blank} onclick={() => push({ blank: !s.blank })}>
      {s.blank ? 'Screen is BLANK — tap to show' : 'Blank screen'}
    </button>
    <p class="msg">Display: <code>{location.origin}{location.pathname}{DB ? '?k=' + room : ''}</code></p>
  </main>
{:else}
  <!-- svelte-ignore a11y_click_events_have_key_events, a11y_no_static_element_interactions -->
  <div class="display" onclick={() => { fs(); wake() }}>
    {#if !s.blank}
      <span class="time">{parts.t}</span><span class="ap">{parts.ap}</span>
    {/if}
  </div>
{/if}

<style>
  :global(html, body) { margin: 0; height: 100%; background: #000; color: #fff; font-family: system-ui, sans-serif; }
  .display { height: 100vh; display: flex; align-items: center; justify-content: center; gap: 2vw; color: #f00; cursor: none; user-select: none; }
  .time { font: 700 28vw/1 'Courier New', monospace; text-shadow: 0 0 3vw #f008; }
  .ap { font: 700 7vw/1 'Courier New', monospace; align-self: flex-end; margin-bottom: 18vh; }
  .admin { max-width: 24rem; margin: 0 auto; padding: 1rem; display: grid; gap: 1rem; }
  .pick { display: flex; gap: .5rem; }
  select, button { font: inherit; font-size: 2rem; padding: 1rem; border-radius: .5rem; border: 0; }
  button { background: #333; color: #fff; }
  button.on { background: #b00; }
  .msg { padding: 1rem; opacity: .7; word-break: break-all; }
</style>
