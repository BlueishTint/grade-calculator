<script lang="ts">
  import { tick } from 'svelte';

  // =====================================================================
  // STATE & DATA MODELS
  // =====================================================================

  // Helper to safely load from localStorage
  function load<T>(key: string, fallback: T): T {
    const saved = localStorage.getItem(key);
    try {
      return saved ? JSON.parse(saved) : fallback;
    } catch {
      return fallback;
    }
  }

  interface Assignment {
    id: number;
    name: string;
    weight: string;
    grade: string;
    effort: number;
  }

  interface LetterScale {
    letter: string;
    min: number;
  }

  interface GradeProfile {
    id: string;
    label: string;
    assignments: Assignment[];
    letterScale: LetterScale[];
    desiredGrade: number;
  }

  // Helper to generate a default profile
  function createNewProfile(name = 'New Course'): GradeProfile {
    return {
      id: crypto.randomUUID(),
      label: name,
      assignments: [{ id: 1, name: 'Final Exam', weight: '100', grade: '', effort: 1 }],
      letterScale: [
        { letter: 'A+', min: 97 },
        { letter: 'A', min: 93 },
        { letter: 'A-', min: 90 },
        { letter: 'B+', min: 87 },
        { letter: 'B', min: 83 },
        { letter: 'B-', min: 80 },
        { letter: 'C+', min: 77 },
        { letter: 'C', min: 73 },
        { letter: 'C-', min: 70 },
        { letter: 'D+', min: 67 },
        { letter: 'D', min: 63 },
        { letter: 'D-', min: 60 },
        { letter: 'F', min: 0 }
      ],
      desiredGrade: 90
    };
  }

  let profiles = $state(load('grade-profiles', [createNewProfile()]));

  let activeProfileId = $derived(load('active-profile-id', profiles[0].id));

  let currentProfile = $derived(profiles.find((p) => p.id === activeProfileId) ?? profiles[0]);

  let nextId = 5;

  // Svelte 5: Use $state for the source of truth
  let assignments = $derived(currentProfile.assignments);

  let letterScale = $derived(currentProfile.letterScale);

  let desiredGrade = $derived(currentProfile.desiredGrade);

  let showScaleSettings = $state(false);

  // DOM References for keyboard navigation
  let nameInputs: HTMLInputElement[] = $state([]);

  // =====================================================================
  // REACTIVE CALCULATIONS (The "Engine")
  // =====================================================================

  // 1. Helper to parse individual grades
  function parseGrade(gradeStr: string) {
    const str = gradeStr.trim().toUpperCase();
    if (!str) return { val: null, status: 'blank', display: '' };

    const letterMatch = letterScale.find((l) => l.letter.toUpperCase() === str);
    if (letterMatch) {
      return { val: letterMatch.min, status: 'letter', display: `${letterMatch.min}%` };
    }

    try {
      const sanitized = str.replace(/[^0-9.+\-*/()]/g, '');
      if (!sanitized) return { val: null, status: 'error', display: '' };

      let val = new Function('return ' + sanitized)();
      if (val <= 1.0 && val > 0 && sanitized.includes('/')) {
        val = val * 100;
      }
      return { val, status: 'ok', display: `${val.toFixed(1)}%` };
    } catch {
      return { val: null, status: 'error', display: '' };
    }
  }

  function parseWeight(weightStr: string) {
    let parsedWeight;
    try {
      const sanitizedWeight = weightStr.replace(/[^0-9.+\-*/()]/g, '');
      parsedWeight = new Function('return ' + sanitizedWeight)();
    } catch {
      return { val: null, status: 'error', display: '' };
    }

    return { val: parsedWeight, status: 'ok', display: `${parsedWeight}%` };
  }

  // 2. Computed values for the UI
  let processedData = $derived.by(() => {
    // First pass: Parse everything as usual
    const rawItems = assignments.map((a) => {
      const parsedWeight = parseWeight(a.weight);
      const parsed = parseGrade(a.grade);
      return { ...a, parsedWeight, ...parsed };
    });

    // Calculate total weight of all assignments except the last one
    const initialWeightSum = rawItems
      .slice(0, -1)
      .reduce((sum, i) => sum + (i.parsedWeight.val || 0), 0);

    // Second pass: Apply "Auto-Weight" to the last item if needed
    const items = rawItems.map((item, index) => {
      const isLast = index === assignments.length - 1;
      const hasNoWeight = !item.weight || item.parsedWeight.val === 0;

      if (isLast && hasNoWeight) {
        const remainder = Math.max(0, 100 - initialWeightSum);
        return {
          ...item,
          parsedWeight: { val: remainder, status: 'ok', display: `${remainder}% (auto)` }
        };
      }
      return item;
    });

    // Standard calculations using the finalized 'items' array
    const completed = items.filter((i) => i.status !== 'blank' && i.status !== 'error');
    const weightCompleted = completed.reduce((sum, i) => sum + (i.parsedWeight.val || 0), 0);
    const scoreCompleted = completed.reduce(
      (sum, i) => sum + (i.parsedWeight.val || 0) * (i.val || 0),
      0
    );
    const currentAvg = weightCompleted > 0 ? scoreCompleted / weightCompleted : 0;

    const totalWeight = items.reduce((sum, i) => sum + (i.parsedWeight.val || 0), 0);
    const neededTotal = desiredGrade * totalWeight;
    const neededFromBlanks = neededTotal - scoreCompleted;

    const blanks = items.filter((i) => i.status === 'blank');
    const totalEffortWeight = blanks.reduce(
      (sum, i) => sum + (i.parsedWeight.val || 0) * (i.effort || 1),
      0
    );
    const baseReq = totalEffortWeight > 0 ? neededFromBlanks / totalEffortWeight : 0;

    return {
      items: items.map((i) => ({
        ...i,
        prediction: i.status === 'blank' && totalEffortWeight > 0 ? baseReq * (i.effort || 1) : null
      })),
      currentAvg,
      weightCompleted,
      baseReq,
      hasBlanks: blanks.length > 0
    };
  });

  $effect(() => {
    localStorage.setItem('grade-profiles', JSON.stringify(profiles));
    localStorage.setItem('active-profile-id', activeProfileId);
  });

  // =====================================================================
  // METHODS
  // =====================================================================

  async function addAssignment() {
    assignments.push({ id: nextId++, name: '', weight: '10', grade: '', effort: 1 });
    await tick();
    nameInputs[assignments.length - 1]?.focus();
  }

  function removeAssignment(id: number) {
    const index = assignments.findIndex((a) => a.id === id);
    if (index !== -1) assignments.splice(index, 1);
  }

  function handleKeydown(e: KeyboardEvent, index: number) {
    if (e.key === 'Enter') {
      e.preventDefault();
      if (index === assignments.length - 1) {
        addAssignment();
      } else {
        nameInputs[index + 1]?.focus();
      }
    }
  }

  function addProfile() {
    const newP = createNewProfile(`Course ${profiles.length + 1}`);
    profiles.push(newP);
    activeProfileId = newP.id;
  }

  function deleteProfile(id: string) {
    if (profiles.length <= 1) return; // Keep at least one
    profiles = profiles.filter((p) => p.id !== id);
    activeProfileId = profiles[0].id;
  }

  function selectProfile(id: string) {
    activeProfileId = id;
  }

  function resetData() {
    if (confirm('Are you sure you want to clear all data?')) {
      localStorage.clear();
      location.reload(); // Quickest way to re-initialize the state
    }
  }
</script>

<svelte:head>
  <title>Grade Calculator</title>
  <meta name="description" content="Calculates grades and requried grades" />
</svelte:head>
<div class="min-h-screen bg-gray-50 p-4 font-sans text-gray-900 md:p-8">
  <div class="mx-auto max-w-5xl space-y-6">
    <!-- HEADER -->
    <div
      class="flex flex-col items-start justify-between rounded-2xl border border-gray-100 bg-white p-6 shadow-sm md:flex-row md:items-center"
    >
      <div>
        <h1 class="text-3xl font-bold tracking-tight text-gray-900">Grade Calculator</h1>
        <p class="mt-1 text-sm text-gray-500">
          Use letters (A+), percentages (95), or math (35/40) for grades. Leave empty to predict.
        </p>
      </div>
      <button
        onclick={() => (showScaleSettings = !showScaleSettings)}
        class="mt-4 flex items-center gap-2 rounded-lg bg-gray-100 px-4 py-2 text-sm font-semibold text-gray-700 transition hover:bg-gray-200 md:mt-0"
      >
        <svg
          xmlns="http://www.w3.org/2000/svg"
          width="16"
          height="16"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
          stroke-linecap="round"
          stroke-linejoin="round"
          ><path
            d="M12.22 2h-.44a2 2 0 0 0-2 2v.18a2 2 0 0 1-1 1.73l-.43.25a2 2 0 0 1-2 0l-.15-.08a2 2 0 0 0-2.73.73l-.22.38a2 2 0 0 0 .73 2.73l.15.1a2 2 0 0 1 1 1.72v.51a2 2 0 0 1-1 1.74l-.15.09a2 2 0 0 0-.73 2.73l.22.38a2 2 0 0 0 2.73.73l.15-.08a2 2 0 0 1 2 0l.43.25a2 2 0 0 1 1 1.73V20a2 2 0 0 0 2 2h.44a2 2 0 0 0 2-2v-.18a2 2 0 0 1 1-1.73l.43-.25a2 2 0 0 1 2 0l.15.08a2 2 0 0 0 2.73-.73l.22-.39a2 2 0 0 0-.73-2.73l-.15-.08a2 2 0 0 1-1-1.74v-.5a2 2 0 0 1 1-1.74l.15-.09a2 2 0 0 0 .73-2.73l-.22-.38a2 2 0 0 0-2.73-.73l-.15.08a2 2 0 0 1-2 0l-.43-.25a2 2 0 0 1-1-1.73V4a2 2 0 0 0-2-2z"
          ></path><circle cx="12" cy="12" r="3"></circle></svg
        >
        Letter Scale
      </button>
      <button onclick={resetData} class="text-xs text-red-500 hover:underline">
        Reset All Data
      </button>
    </div>

    <!-- LETTER SCALE SETTINGS -->
    {#if showScaleSettings}
      <div class="animate-fade-in rounded-2xl border border-gray-100 bg-white p-6 shadow-sm">
        <h3 class="mb-4 text-lg font-semibold">Edit Letter Grade Scale (Minimum %)</h3>
        <div class="grid grid-cols-2 gap-3 sm:grid-cols-4 md:grid-cols-6 lg:grid-cols-7">
          {#each letterScale as scale (scale.letter)}
            <div class="flex flex-col">
              <label class="mb-1 text-xs font-bold text-gray-500" for="scale-{scale.letter}"
                >{scale.letter}</label
              >
              <input
                id="scale-{scale.letter}"
                type="number"
                bind:value={scale.min}
                class="rounded-md border border-gray-200 p-2 text-sm focus:ring-2 focus:ring-blue-500 focus:outline-none"
              />
            </div>
          {/each}
        </div>
      </div>
    {/if}

    <div class="mb-6 flex flex-wrap gap-2">
      {#each profiles as profile (profile.id)}
        <div class="group flex items-center gap-1">
          <button
            title="Select Profile"
            onclick={() => selectProfile(profile.id)}
            class="rounded-lg px-4 py-2 font-medium transition
                {activeProfileId === profile.id
              ? 'bg-blue-600 text-white shadow-md'
              : 'bg-white text-gray-600 hover:bg-gray-100'}"
          >
            <input
              bind:value={profile.label}
              class="w-24 cursor-pointer border-none bg-transparent p-0 text-inherit focus:ring-0"
            />
          </button>

          {#if profiles.length > 1}
            <button
              onclick={() => deleteProfile(profile.id)}
              class="p-1 text-gray-400 hover:text-red-500"
            >
              ✕
            </button>
          {/if}
        </div>
      {/each}

      <button
        onclick={addProfile}
        class="rounded-lg border-2 border-dashed border-gray-300 px-4 py-2 text-gray-500 transition hover:border-blue-400 hover:text-blue-500"
      >
        + New Course
      </button>
    </div>

    <!-- MAIN CALCULATOR -->
    <div class="overflow-hidden rounded-2xl border border-gray-100 bg-white shadow-sm">
      <div class="overflow-x-auto">
        <table class="w-full border-collapse text-left">
          <thead>
            <tr
              class="border-b border-gray-100 bg-gray-50/80 text-xs font-semibold tracking-wider text-gray-500 uppercase"
            >
              <th class="w-1/3 p-4">Assignment Name</th>
              <th class="w-24 p-4">Weight</th>
              <th class="w-40 p-4">Grade Input</th>
              <th class="p-4">Calculated Outcome / Effort</th>
              <th class="w-12 p-4 text-center"></th>
            </tr>
          </thead>
          <tbody class="divide-y divide-gray-50">
            {#each assignments as assignment, index (assignment.id)}
              {@const result = processedData.items[index]}
              <tr class="group transition hover:bg-blue-50/30">
                <td class="p-3">
                  <input
                    bind:this={nameInputs[index]}
                    bind:value={assignment.name}
                    onkeydown={(e) => handleKeydown(e, index)}
                    placeholder="e.g., Final Exam"
                    class="w-full border-0 border-b-2 border-transparent bg-transparent p-2 transition hover:border-gray-200 focus:border-blue-500 focus:ring-0"
                  />
                </td>
                <td class="relative p-3">
                  <input
                    type="text"
                    bind:value={assignment.weight}
                    onkeydown={(e) => handleKeydown(e, index)}
                    class="w-full border-0 border-b-2 border-transparent bg-transparent p-2 transition hover:border-gray-200 focus:border-blue-500 focus:ring-0"
                  />
                  {#if result.parsedWeight.display}
                    <span
                      class="absolute -bottom-0.5 left-2 text-[10px] font-medium tracking-wide text-gray-400"
                    >
                      = {result.parsedWeight.display}
                    </span>
                  {:else if result.parsedWeight.status === 'error'}
                    <span
                      class="absolute -bottom-0.5 left-2 text-[10px] font-medium tracking-wide text-red-500"
                    >
                      Invalid format
                    </span>
                  {/if}
                </td>
                <td class="relative p-3">
                  <input
                    type="text"
                    bind:value={assignment.grade}
                    onkeydown={(e) => handleKeydown(e, index)}
                    placeholder="Blank = predict"
                    class={`w-full border-0 border-b-2 bg-transparent p-2 transition hover:border-gray-200 focus:ring-0 ${result.status === 'error' ? 'border-red-200 text-red-600 focus:border-red-500' : 'border-transparent focus:border-blue-500'}`}
                  />
                  {#if result.display}
                    <span
                      class="absolute -bottom-0.5 left-2 text-[10px] font-medium tracking-wide text-gray-400"
                    >
                      = {result.display}
                    </span>
                  {:else if result.status === 'error'}
                    <span
                      class="absolute -bottom-0.5 left-2 text-[10px] font-medium tracking-wide text-red-500"
                    >
                      Invalid format
                    </span>
                  {/if}
                </td>
                <td class="p-3 align-middle">
                  {#if result.status === 'blank'}
                    <div
                      class="flex items-center gap-4 rounded-lg border border-orange-100 bg-orange-50/50 p-2"
                    >
                      <div class="flex-1">
                        <div class="mb-1 flex items-center justify-between">
                          <label
                            class="text-xs font-bold tracking-wide text-orange-600 uppercase"
                            for="eff-{assignment.id}">Effort</label
                          >
                          <span class="text-xs font-medium text-orange-700"
                            >x{assignment.effort.toFixed(1)}</span
                          >
                        </div>
                        <input
                          id="eff-{assignment.id}"
                          type="range"
                          min="0.1"
                          max="3"
                          step="0.1"
                          bind:value={assignment.effort}
                          class="h-1.5 w-full cursor-pointer appearance-none rounded-lg bg-orange-200 accent-orange-500"
                        />
                      </div>
                      <div class="border-l border-orange-200 pl-4 text-right">
                        <div class="text-xs font-bold tracking-wide text-orange-600 uppercase">
                          Need
                        </div>
                        <div
                          class={`text-xl font-black ${result.prediction !== null && result.prediction > 100 ? 'text-red-600' : 'text-gray-900'}`}
                        >
                          {result.prediction !== null ? `${result.prediction.toFixed(1)}%` : '-'}
                        </div>
                      </div>
                    </div>
                  {:else}
                    <div class="px-2 text-sm text-gray-400 italic">Completed</div>
                  {/if}
                </td>
                <td class="p-3 text-center">
                  <button
                    title="Add"
                    onclick={() => removeAssignment(assignment.id)}
                    class="rounded-lg p-2 text-gray-300 opacity-0 transition group-hover:opacity-100 hover:bg-red-50 hover:text-red-500 focus:opacity-100"
                  >
                    <svg
                      xmlns="http://www.w3.org/2000/svg"
                      width="18"
                      height="18"
                      viewBox="0 0 24 24"
                      fill="none"
                      stroke="currentColor"
                      stroke-width="2"
                      stroke-linecap="round"
                      stroke-linejoin="round"
                      ><path d="M3 6h18"></path><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"
                      ></path><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"></path></svg
                    >
                  </button>
                </td>
              </tr>
            {/each}
          </tbody>
        </table>
      </div>
      <div class="border-t border-gray-100 bg-gray-50/50 p-4">
        <button
          onclick={addAssignment}
          class="flex items-center gap-1 text-sm font-semibold text-blue-600 transition hover:text-blue-800"
        >
          <svg
            xmlns="http://www.w3.org/2000/svg"
            width="16"
            height="16"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
            ><line x1="12" y1="5" x2="12" y2="19"></line><line x1="5" y1="12" x2="19" y2="12"
            ></line></svg
          >
          Add Assignment (Enter)
        </button>
      </div>
    </div>

    <!-- FOOTER -->
    <div class="grid grid-cols-1 gap-6 md:grid-cols-2">
      <div class="rounded-2xl border border-gray-100 bg-white p-6 shadow-sm">
        <h3 class="mb-2 text-sm font-bold tracking-wider text-gray-400 uppercase">
          Current Average
        </h3>
        <div class="text-5xl font-black text-gray-900">{processedData.currentAvg.toFixed(1)}%</div>
        <p class="mt-2 text-sm text-gray-500">
          Weight completed: <span class="font-bold">{processedData.weightCompleted}</span>
        </p>
      </div>

      <div
        class="rounded-2xl bg-gradient-to-br from-blue-600 to-indigo-700 p-6 text-white shadow-lg"
      >
        <div class="mb-4 flex items-start justify-between">
          <h3 class="text-sm font-bold tracking-wider text-blue-200 uppercase">Desired Final</h3>
          <div class="flex items-center rounded-lg bg-white/20 px-2 py-1">
            <input
              type="number"
              bind:value={desiredGrade}
              class="w-12 border-none bg-transparent p-0 text-right font-bold focus:ring-0"
            />
            <span class="ml-1 font-bold">%</span>
          </div>
        </div>
        {#if processedData.hasBlanks}
          <p class="mb-1 text-sm text-blue-100">Target average for remaining:</p>
          <div class="text-4xl font-black">{processedData.baseReq.toFixed(1)}%</div>
          {#if processedData.baseReq > 100}
            <p
              class="mt-2 inline-block rounded bg-red-900/30 px-2 py-1 text-xs font-bold text-red-300"
            >
              Requires >100%
            </p>
          {/if}
        {:else}
          <div class="mt-4 text-xl font-bold">
            {processedData.currentAvg >= desiredGrade ? 'Goal Met! 🎉' : 'Below Goal'}
          </div>
        {/if}
      </div>
    </div>
  </div>
</div>

<style>
  input[type='number']::-webkit-inner-spin-button,
  input[type='number']::-webkit-outer-spin-button {
    -webkit-appearance: none;
    margin: 0;
  }
  input[type='number'] {
    appearance: textfield;
  }
  .animate-fade-in {
    animation: fadeIn 0.2s ease-out forwards;
  }
  @keyframes fadeIn {
    from {
      opacity: 0;
      transform: translateY(-5px);
    }
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }
</style>
