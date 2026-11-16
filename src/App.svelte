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

  // 7-segment: each segment is a hexagon; unlit ones drawn dim like a real LED display
  const hex = (x1, y1, x2, y2) => {
    const h = y1 === y2
    const [a, b] = h ? [x1 + 1.4, x2 - 1.4] : [y1 + 1.4, y2 - 1.4]
    const pt = (u, v) => (h ? `${u},${y1 + v}` : `${x1 + v},${u}`)
    return [pt(a - 1, 0), pt(a, -1), pt(b, -1), pt(b + 1, 0), pt(b, 1), pt(a, 1)].join(' ')
  }
  const SEG = { a: hex(1, 1, 9, 1), b: hex(9, 1, 9, 9), c: hex(9, 9, 9, 17), d: hex(1, 17, 9, 17), e: hex(1, 9, 1, 17), f: hex(1, 1, 1, 9), g: hex(1, 9, 9, 9) }
  const ON = ['abcdef', 'bc', 'abged', 'abgcd', 'fgbc', 'afgcd', 'afgedc', 'abc', 'abcdefg', 'abcdfg']

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
        {#each Array.from({ length: 12 }, (_, i) => i + 1) as h}<option>{String(h).padStart(2, '0')}</option>{/each}
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
      <div class="time">
        {#each parts.t as ch}
          {#if ch === ':'}
            <svg viewBox="0 0 4 18" class="colon"><circle cx="2" cy="5" r="1.2" /><circle cx="2" cy="13" r="1.2" /></svg>
          {:else}
            <svg viewBox="0 0 10 18">
              {#each Object.entries(SEG) as [k, pts]}<polygon points={pts} class:lit={ON[ch].includes(k)} />{/each}
            </svg>
          {/if}
        {/each}
      </div>
      <div class="ap"><span class:lit={parts.ap === 'AM'}>AM</span><span class:lit={parts.ap === 'PM'}>PM</span></div>
    {/if}
  </div>
{/if}

<style>
  :global(html, body) { margin: 0; height: 100%; background: #000; color: #fff; font-family: system-ui, sans-serif; }
  .display { height: 100vh; display: flex; align-items: center; justify-content: center; gap: 2vw; color: #f00; cursor: none; user-select: none; }
  .time { display: flex; gap: 1.2vw; transform: skewX(-6deg); }
  .time svg { height: min(100vh - 40px, (84vw - 40px) / 2.55); /* ponytail: 20px padding; width term budgets 4 digits + colon + AM/PM + gaps + skew */ fill: #f001; overflow: visible; }
  .time .lit { fill: #f00; filter: drop-shadow(0 0 .6px #f00); }
  .colon circle { fill: #f00; }
  .ap { display: flex; flex-direction: column; gap: 1vh; font: 700 5vw/1 system-ui, sans-serif; letter-spacing: .1em; align-self: center; }
  .ap span { color: #f001; }
  .ap .lit { color: #f00; text-shadow: 0 0 1vw #f008; }
  .admin { max-width: 24rem; margin: 0 auto; padding: 1rem; display: grid; gap: 1rem; }
  .pick { display: flex; gap: .5rem; }
  select, button { font: inherit; font-size: 2rem; padding: 1rem; border-radius: .5rem; border: 0; }
  button { background: #333; color: #fff; }
  button.on { background: #b00; }
  .msg { padding: 1rem; opacity: .7; word-break: break-all; }
</style>
