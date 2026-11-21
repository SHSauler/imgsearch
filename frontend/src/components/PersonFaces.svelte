<script lang="ts">
  import { listFaces, listPeople, reassignFace, splitPerson } from "../lib/api";
  import { refreshPeople } from "../lib/stores";
  import { mediaUrl } from "../lib/utils";
  import type { FaceView, Person } from "../lib/types";

  let { personId }: { personId: number } = $props();

  let faces = $state<FaceView[]>([]);
  let people = $state<Person[]>([]);
  let selected = $state<Set<number>>(new Set());
  let loading = $state(true);
  let pending = $state(false);
  let error = $state("");
  let target = $state("");
  let splitName = $state("");
  let active = true;

  async function load(signal?: AbortSignal) {
    loading = true;
    error = "";
    try {
      const [faceRes, peopleRes] = await Promise.all([
        listFaces({ personId, limit: 200, signal }),
        listPeople(signal),
      ]);
      if (!active) return;
      faces = faceRes.faces;
      people = peopleRes.people.filter((p) => p.id !== personId);
      selected = new Set();
    } catch (err) {
      if (!active || (err as { name?: string }).name === "AbortError") return;
      error = err instanceof Error ? err.message : "Could not load faces";
    } finally {
      if (active) loading = false;
    }
  }

  $effect(() => {
    const ac = new AbortController();
    active = true;
    void load(ac.signal);
    return () => {
      active = false;
      ac.abort();
    };
  });

  function toggle(id: number) {
    const next = new Set(selected);
    if (next.has(id)) next.delete(id);
    else next.add(id);
    selected = next;
  }

  async function move(targetId: number | null) {
    if (selected.size === 0 || pending) return;
    pending = true;
    error = "";
    try {
      for (const id of selected) await reassignFace(id, targetId);
      await load();
      refreshPeople();
    } catch (err) {
      if (active) error = err instanceof Error ? err.message : "Could not move faces";
    } finally {
      if (active) pending = false;
    }
  }

  async function split(event: SubmitEvent) {
    event.preventDefault();
    if (selected.size === 0 || pending) return;
    const trimmed = splitName.trim();
    if (trimmed && Array.from(trimmed).length > 100) {
      error = "Use a person name of 1–100 characters.";
      return;
    }
    pending = true;
    error = "";
    try {
      await splitPerson(personId, [...selected], trimmed || undefined);
      if (active) splitName = "";
      await load();
      refreshPeople();
    } catch (err) {
      if (active) error = err instanceof Error ? err.message : "Could not split faces";
    } finally {
      if (active) pending = false;
    }
  }

  const button = "px-3 py-2 border border-line rounded-lg cursor-pointer disabled:opacity-50";
</script>

<section data-person-faces aria-label="Face management" class="px-5 sm:px-9 pb-16 max-w-5xl w-full">
  <h2 class="text-base font-semibold mt-6 mb-1">Faces</h2>
  <p class="text-sm text-muted mb-3">Select faces to correct the grouping: move them to the right person, or split them into a new one.</p>
  {#if loading}<p role="status">Loading faces…</p>{/if}
  {#if error}<p role="alert" class="text-bad">{error}</p>{/if}
  {#if !loading && faces.length === 0 && !error}<p class="text-muted">No faces grouped here yet.</p>{/if}
  {#if faces.length > 0}
    <ul class="list-none p-0 grid grid-cols-3 sm:grid-cols-5 gap-2">
      {#each faces as face (face.id)}
        <li>
          <button type="button" data-face={face.id} aria-pressed={selected.has(face.id)} aria-label={`Face ${face.id}${selected.has(face.id) ? ", selected" : ""}`} onclick={() => toggle(face.id)} class="block w-full rounded-lg overflow-hidden border-2 cursor-pointer {selected.has(face.id) ? 'border-accent' : 'border-transparent'}">
            {#if face.thumbnail_path}
              <img src={mediaUrl(face.thumbnail_path)} alt="" loading="lazy" class="w-full aspect-square object-cover" />
            {:else}
              <span class="grid place-items-center w-full aspect-square bg-surface-2 text-muted-2">—</span>
            {/if}
          </button>
        </li>
      {/each}
    </ul>
    <div class="flex flex-wrap items-center gap-2 mt-3">
      <button class={button} disabled={pending || selected.size === 0} onclick={() => void move(null)}>Unassign selected</button>
      <label class="sr-only" for="face-move-target">Move selected to</label>
      <select id="face-move-target" bind:value={target} disabled={pending || selected.size === 0} class="px-3 py-2 bg-surface border border-line-2 rounded-lg">
        <option value="">Move selected to…</option>
        {#each people as p (p.id)}
          <option value={p.id}>{p.name}</option>
        {/each}
      </select>
      <button class={button} disabled={pending || selected.size === 0 || !target} onclick={() => void move(Number(target))}>Move</button>
    </div>
    <form onsubmit={split} class="flex flex-wrap items-center gap-2 mt-2">
      <input aria-label="New person name for split" placeholder="New person name (blank for automatic)" bind:value={splitName} disabled={pending || selected.size === 0} class="min-w-0 flex-1 px-3 py-2 bg-surface border border-line-2 rounded-lg" />
      <button type="submit" disabled={pending || selected.size === 0} class="px-3 py-2 bg-ink text-white rounded-lg cursor-pointer disabled:opacity-50">Split selected into new person</button>
    </form>
  {/if}
</section>
