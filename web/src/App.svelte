<script lang="ts">
  import { onMount } from 'svelte'

  let backendStatus = $state('Checking backend...')
  let backendConnected = $state(false)

  onMount(() => {
    fetch('/api/v1/status')
      .then(async (response) => {
        if (!response.ok) {
          throw new Error(`HTTP ${response.status}`)
        }

        return response.json()
      })
      .then((data) => {
        backendStatus = `${data.name} — ${data.stage}`
        backendConnected = true
      })
      .catch((error: unknown) => {
        backendStatus =
          error instanceof Error ? error.message : 'Connection failed'
      })
  })
</script>

<main>
  <h1>LogPocket</h1>

  <p>Lightweight, self-hosted logging.</p>

  <section>
    <h2>Backend status</h2>

    <p>
      {backendConnected ? 'Connected' : 'Disconnected'}
    </p>

    <pre>{backendStatus}</pre>
  </section>

  <section>
    <h2>Development progress</h2>

    <p>Phase 0 — Project initialization</p>
    <p>Go backend + Svelte frontend</p>
  </section>
</main>
