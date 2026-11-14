<script lang="ts">
  // The Act check, over the whole page (`showModal`, which keeps focus inside
  // it). Escape or Cancel closes it and nothing changes. The office checks the
  // answers (POST /api/act-check); a wrong one shows why under it, and the
  // answers stay for the owner to change and check again.
  import { onMount } from 'svelte';
  import { ACT_CHECK, ACT_CHECK_LEAD, ACT_CHECK_TITLE } from '@oddboro/ledger/act-check';
  import { actCheckClosed } from './act-check.svelte.ts';
  import { call } from './api.ts';

  let dialog: HTMLDialogElement;
  let answers = $state<(boolean | undefined)[]>(ACT_CHECK.map(() => undefined));
  let wrong = $state<number[]>([]);
  let checking = $state(false);
  let problem = $state('');
  let passed = false;
  const answered = $derived(answers.every((a) => a !== undefined));

  onMount(() => dialog.showModal());

  async function check() {
    if (!answered || checking) return;
    checking = true;
    problem = '';
    try {
      const r = await call<{ passed: boolean; wrong: number[] }>('/api/act-check', { answers });
      wrong = r.wrong;
      if (r.passed) {
        passed = true;
        dialog.close();
      }
    } catch (e) {
      problem = e instanceof Error ? e.message : String(e);
    } finally {
      checking = false;
    }
  }

  function pick(i: number, value: boolean) {
    answers[i] = value;
    wrong = wrong.filter((w) => w !== i);
  }
</script>

<!-- Keep the office's window shortcuts out of the check while it is open. -->
<!-- svelte-ignore a11y_no_noninteractive_element_interactions: the dialog owns its keyboard events. -->
<dialog
  bind:this={dialog}
  aria-labelledby="act-check-title"
  class="m-auto max-h-[90dvh] w-[min(32rem,92vw)] overflow-y-auto rounded-2xl border border-hairline bg-panel-solid p-5 text-ink shadow-xl backdrop:bg-night/70"
  onkeydown={(e) => e.stopPropagation()}
  onclose={() => actCheckClosed(passed)}
>
  <h2 id="act-check-title" class="font-pixel text-xl">{ACT_CHECK_TITLE}</h2>
  <p class="mt-1 text-sm text-muted">{ACT_CHECK_LEAD}</p>
  <ol class="mt-4 space-y-4">
    {#each ACT_CHECK as item, i (i)}
      <li>
        <fieldset>
          <legend class="text-sm">{i + 1}. {item.says}</legend>
          <div class="mt-2 flex gap-2">
            {#each [true, false] as value (value)}
              <button
                type="button"
                class="rounded-lg px-3 py-1.5 text-xs {answers[i] === value
                  ? 'bg-sea/25 text-ink'
                  : 'bg-raise text-muted hover:text-ink'}"
                aria-pressed={answers[i] === value}
                disabled={checking}
                onclick={() => pick(i, value)}>{value ? 'True' : 'False'}</button
              >
            {/each}
          </div>
          {#if wrong.includes(i)}
            <p class="mt-1.5 text-xs text-coral">{item.ifWrong}</p>
          {/if}
        </fieldset>
      </li>
    {/each}
  </ol>
  {#if problem}<p class="mt-3 text-xs text-coral">{problem}</p>{/if}
  <div class="mt-5 flex gap-2">
    <button
      type="button"
      class="flex-1 rounded-lg bg-sea px-3 py-2 text-sm font-medium text-night disabled:opacity-40"
      disabled={!answered || checking}
      onclick={check}>Check my answers</button
    >
    <button
      type="button"
      class="rounded-lg bg-raise px-3 py-2 text-sm"
      disabled={checking}
      onclick={() => dialog.close()}>Cancel</button
    >
  </div>
</dialog>
