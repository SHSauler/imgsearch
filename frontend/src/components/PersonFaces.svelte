<script lang="ts">
  import { getImage, listFaces, listPeople, reassignFace, splitPerson } from "../lib/api";
  import { lightboxPin, refreshPeople } from "../lib/stores";
  import { fallbackToOriginal, mediaUrl, pinFromImage } from "../lib/utils";
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
  let opening = $state<number | null>(null);
  let active = true;

  const photoCount = $derived(new Set(faces.map((f) => f.image_id)).size);
  const selectedFaces = $derived(faces.filter((f) => selected.has(f.id)));
  const selectedPhotos = $derived([...new Set(selectedFaces.map((f) => f.original_name))]);

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

  async function openPhoto(face: FaceView) {
    if (opening !== null) return;
    opening = face.id;
    try {
      const record = await getImage(face.image_id);
      lightboxPin.set(pinFromImage(record));
    } catch (err) {
      if (active) error = err instanceof Error ? err.message : "Could not open photo";
    } finally {
      if (active) opening = null;
    }
  }

  /** Padded face crop for object-view-box (inset = top right bottom left). */
  function cropStyle(face: FaceView): string {
    const pad = 0.25;
    const x0 = Math.max(0, face.box_x - face.box_w * pad);
    const y0 = Math.max(0, face.box_y - face.box_h * pad);
    const x1 = Math.min(1, face.box_x + face.box_w * (1 + pad));
    const y1 = Math.min(1, face.box_y + face.box_h * (1 + pad));
    const pct = (v: number) => `${(v * 100).toFixed(1)}%`;
    return `object-view-box: inset(${pct(y0)} ${pct(1 - x1)} ${pct(1 - y1)} ${pct(x0)});`;
  }

  function faceLabel(face: FaceView): string {
    return `${face.original_name}, face ${face.id}, ${Math.round(face.confidence * 100)}% match`;
  }

  function photoDate(face: FaceView): string {
    return (face.captured_at || "").slice(0, 10);
  }

  const button = "px-3 py-2 border border-line rounded-lg cursor-pointer disabled:opacity-50";
</script>

<section data-person-faces aria-label="Face management" class="px-5 sm:px-9 pb-16 max-w-5xl w-full">
  <h2 class="text-base font-semibold mt-6 mb-1">Faces</h2>
  {#if !loading && !error}
    <p class="text-sm text-muted mb-3">
      {faces.length} {faces.length === 1 ? "face" : "faces"} from {photoCount} {photoCount === 1 ? "photo" : "photos"}{selected.size > 0 ? ` · ${selected.size} selected${selectedPhotos.length > 0 ? ` from ${selectedPhotos.join(", ")}` : ""}` : ""}.
      Select faces to correct the grouping: move them to the right person, or split them into a new one.
    </p>
  {:else}
    <p class="text-sm text-muted mb-3">Select faces to correct the grouping: move them to the right person, or split them into a new one.</p>
  {/if}
  {#if loading}<p role="status">Loading faces…</p>{/if}
  {#if error}<p role="alert" class="text-bad">{error}</p>{/if}
  {#if !loading && faces.length === 0 && !error}<p class="text-muted">No faces grouped here yet.</p>{/if}
  {#if faces.length > 0}
    <ul class="list-none p-0 grid grid-cols-2 sm:grid-cols-4 gap-3">
      {#each faces as face (face.id)}
        {@const src = mediaUrl(face.thumbnail_path || face.storage_path)}
        {@const original = mediaUrl(face.storage_path)}
        <li data-face-card={face.id} class="min-w-0">
          <button
            type="button"
            data-face={face.id}
            aria-pressed={selected.has(face.id)}
            aria-label={`Select ${faceLabel(face)}`}
            title={faceLabel(face)}
            onclick={() => toggle(face.id)}
            class="block w-full rounded-lg overflow-hidden border-2 cursor-pointer p-0 bg-surface-2 {selected.has(face.id) ? 'border-accent' : 'border-transparent'}"
          >
            {#if src}
              <img
                src={src}
                alt=""
                loading="lazy"
                onerror={(event) => fallbackToOriginal(event.currentTarget, original)}
                style={cropStyle(face)}
                class="block w-full aspect-square object-cover"
              />
            {:else}
              <span class="grid place-items-center w-full aspect-square text-muted-2 text-sm">—</span>
            {/if}
          </button>
          <div class="mt-1 min-w-0">
            <button
              type="button"
              class="block max-w-full truncate text-left text-[11px] text-muted hover:text-ink hover:underline cursor-pointer bg-transparent border-0 p-0"
              title={`Open ${face.original_name}`}
              onclick={() => void openPhoto(face)}
              disabled={opening !== null}
            >{face.original_name || `Photo ${face.image_id}`}{photoDate(face) ? ` · ${photoDate(face)}` : ""}</button>
            <span class="block text-[11px] text-muted-2">{Math.round(face.confidence * 100)}% match</span>
          </div>
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
