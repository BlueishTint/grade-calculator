<script lang="ts">
	import { onMount, tick } from 'svelte';

	// =====================================================================
	// STATE & DATA MODELS
	// =====================================================================

	let nextId = 5;
	let assignments = [
		{ id: 1, name: 'Homework Average', weight: 20, grade: '95', effort: 1 },
		{ id: 2, name: 'Midterm 1', weight: 25, grade: 'B+', effort: 1 },
		{ id: 3, name: 'Midterm 2', weight: 25, grade: '75/80*100', effort: 1 },
		{ id: 4, name: 'Final Exam', weight: 30, grade: '', effort: 1.5 } // Blank by default to show calculator
	];

	// Default letter scale (editable by user)
	let letterScale = $state([
		{ letter: 'A+', min: 97 }, { letter: 'A', min: 93 }, { letter: 'A-', min: 90 },
		{ letter: 'B+', min: 87 }, { letter: 'B', min: 83 }, { letter: 'B-', min: 80 },
		{ letter: 'C+', min: 77 }, { letter: 'C', min: 73 }, { letter: 'C-', min: 70 },
		{ letter: 'D+', min: 67 }, { letter: 'D', min: 63 }, { letter: 'D-', min: 60 },
		{ letter: 'F', min: 0 }
	]);

	let desiredGrade = $state(90);
	let showScaleSettings = $state(false);

	// DOM References for keyboard navigation
	let nameInputs: HTMLInputElement[] = $state([]);

	// =====================================================================
	// REACTIVE CALCULATIONS (The "Engine")
	// =====================================================================

	// 1. Parse all inputs (handle Letters, Percentages, and Math)
	let parsedAssignments = $derived(assignments.map((a) => {
		const isBlank = a.grade.trim() === '';
		let actualVal = null;
		let parseStatus = 'ok'; // 'ok', 'error', or 'letter'
		let parsedDisplay = '';

		if (!isBlank) {
			const rawStr = a.grade.trim().toUpperCase();
			
			// Check if it's a letter grade
			const letterMatch = letterScale.find(l => l.letter.toUpperCase() === rawStr);
			if (letterMatch) {
				actualVal = letterMatch.min;
				parseStatus = 'letter';
				parsedDisplay = `${actualVal}%`;
			} else {
				// Try evaluating as math/percentage
				try {
					// Sanitize to prevent malicious eval (only allow numbers and basic math operators)
					const sanitized = rawStr.replace(/[^0-9.\+\-\*\/\(\)]/g, '');
					if (sanitized) {
						// Safe evaluation of simple math
						actualVal = new Function('return ' + sanitized)();
						
						// Common UX fix: If they type 30/40, they usually mean out of 100 (75%). 
						// If result is <= 1.0 and they used division, auto-multiply by 100.
						if (actualVal <= 1.0 && actualVal > 0 && sanitized.includes('/')) {
							actualVal = actualVal * 100;
						}
						parsedDisplay = `${actualVal.toFixed(1)}%`;
					} else {
						parseStatus = 'error';
					}
				} catch (e) {
					parseStatus = 'error';
				}
			}
		}

		return { ...a, isBlank, actualVal, parseStatus, parsedDisplay };
	}));

	// 2. Current Status Calculations
	let completedAssignments = $derived(parsedAssignments.filter((a) => !a.isBlank && a.parseStatus !== 'error'));
	let totalCompletedWeight = $derived(completedAssignments.reduce((sum, a) => sum + (a.weight || 0), 0));
	let totalCompletedScore = $derived(completedAssignments.reduce((sum, a) => sum + (a.weight || 0) * a.actualVal, 0));
	
	let currentAverage = $derived(totalCompletedWeight > 0 ? totalCompletedScore / totalCompletedWeight : 0);

	// 3. Predictive Calculations (The "What do I need?" feature)
	let totalWeight = $derived(parsedAssignments.reduce((sum, a) => sum + (a.weight || 0), 0));
	let requiredTotalPoints = $derived(desiredGrade * totalWeight); // Total weighted points needed across all assignments
	let neededPointsFromBlanks = $derived(requiredTotalPoints - totalCompletedScore);

	let blankAssignments = $derived(parsedAssignments.filter((a) => a.isBlank));
	let effortWeightSum = $derived(blankAssignments.reduce((sum, a) => sum + (a.weight || 0) * (a.effort || 1), 0));
	let baseRequiredGrade = $derived(effortWeightSum > 0 ? neededPointsFromBlanks / effortWeightSum : 0);

	// Merge predictions back into the main array for rendering
	let finalCalculations = $derived(parsedAssignments.map((a) => {
		if (a.isBlank && effortWeightSum > 0) {
			const needed = baseRequiredGrade * (a.effort || 1);
			return { ...a, neededGrade: needed };
		}
		return { ...a, neededGrade: null };
	}));

	// =====================================================================
	// METHODS & EVENT HANDLERS
	// =====================================================================

	async function addAssignment() {
		assignments = [
			...assignments,
			{ id: nextId++, name: '', weight: 10, grade: '', effort: 1 }
		];
		// Wait for DOM update, then focus the new row
		await tick();
		nameInputs[assignments.length - 1]?.focus();
	}

	function removeAssignment(id: number) {
		assignments = assignments.filter((a) => a.id !== id);
	}

	function handleKeydown(e: KeyboardEvent, index: number) {
		// Keyboard Shortcut: Enter on any field moves to/creates next row
		if (e.key === 'Enter') {
			e.preventDefault();
			if (index === assignments.length - 1) {
				addAssignment();
			} else {
				nameInputs[index + 1]?.focus();
			}
		}
	}

	// Update letter scale value
	function updateLetterScale(index: number, value: string) {
		let num = parseFloat(value);
		if (!isNaN(num)) {
			letterScale[index].min = num;
			letterScale = [...letterScale]; // trigger reactivity
		}
	}
</script>

<!-- 
	COMPONENT EXTRACTION GUIDE
	For a production SvelteKit app, break this file into:
	1. src/lib/components/AssignmentRow.svelte (The individual row logic)
	2. src/lib/components/LetterScaleSettings.svelte (The settings modal/dropdown)
	3. src/lib/components/SummaryCard.svelte (The bottom results area)
	4. src/lib/stores/grades.js (Svelte store to hold assignments and scale logic globally)
-->

<div class="min-h-screen bg-gray-50 text-gray-900 p-4 md:p-8 font-sans">
	<div class="max-w-5xl mx-auto space-y-6">
		
		<!-- HEADER -->
		<div class="flex flex-col md:flex-row justify-between items-start md:items-center bg-white p-6 rounded-2xl shadow-sm border border-gray-100">
			<div>
				<h1 class="text-3xl font-bold text-gray-900 tracking-tight">Grade Calculator</h1>
				<p class="text-gray-500 mt-1 text-sm">Use letters (A+), percentages (95), or math (35/40) for grades. Leave empty to calculate what you need.</p>
			</div>
			<div class="mt-4 md:mt-0 flex gap-3">
				<button 
					onclick={() => showScaleSettings = !showScaleSettings}
					class="px-4 py-2 bg-gray-100 hover:bg-gray-200 text-gray-700 text-sm font-semibold rounded-lg transition flex items-center gap-2">
					<svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12.22 2h-.44a2 2 0 0 0-2 2v.18a2 2 0 0 1-1 1.73l-.43.25a2 2 0 0 1-2 0l-.15-.08a2 2 0 0 0-2.73.73l-.22.38a2 2 0 0 0 .73 2.73l.15.1a2 2 0 0 1 1 1.72v.51a2 2 0 0 1-1 1.74l-.15.09a2 2 0 0 0-.73 2.73l.22.38a2 2 0 0 0 2.73.73l.15-.08a2 2 0 0 1 2 0l.43.25a2 2 0 0 1 1 1.73V20a2 2 0 0 0 2 2h.44a2 2 0 0 0 2-2v-.18a2 2 0 0 1 1-1.73l.43-.25a2 2 0 0 1 2 0l.15.08a2 2 0 0 0 2.73-.73l.22-.39a2 2 0 0 0-.73-2.73l-.15-.08a2 2 0 0 1-1-1.74v-.5a2 2 0 0 1 1-1.74l.15-.09a2 2 0 0 0 .73-2.73l-.22-.38a2 2 0 0 0-2.73-.73l-.15.08a2 2 0 0 1-2 0l-.43-.25a2 2 0 0 1-1-1.73V4a2 2 0 0 0-2-2z"></path><circle cx="12" cy="12" r="3"></circle></svg>
					Letter Scale
				</button>
			</div>
		</div>

		<!-- LETTER SCALE SETTINGS (Collapsible) -->
		{#if showScaleSettings}
			<div class="bg-white p-6 rounded-2xl shadow-sm border border-gray-100 animate-fade-in">
				<h3 class="text-lg font-semibold mb-4">Edit Letter Grade Scale (Minimum %)</h3>
				<div class="grid grid-cols-2 sm:grid-cols-4 md:grid-cols-6 lg:grid-cols-7 gap-3">
					{#each letterScale as scale, i}
						<div class="flex flex-col">
							<label class="text-xs font-bold text-gray-500 mb-1" for="letter-scale">{scale.letter}</label>
							<input 
								id="letter-scale"
								type="number" 
								value={scale.min}
								oninput={(e) => updateLetterScale(i, (<HTMLInputElement>e.target).value)}
								class="border border-gray-200 rounded-md p-2 text-sm focus:ring-2 focus:ring-blue-500 focus:outline-none"
							/>
						</div>
					{/each}
				</div>
			</div>
		{/if}

		<!-- MAIN CALCULATOR GRID -->
		<div class="bg-white rounded-2xl shadow-sm border border-gray-100 overflow-hidden">
			<div class="overflow-x-auto">
				<table class="w-full text-left border-collapse">
					<thead>
						<tr class="bg-gray-50/80 border-b border-gray-100 text-xs uppercase tracking-wider text-gray-500 font-semibold">
							<th class="p-4 w-1/3">Assignment Name</th>
							<th class="p-4 w-24">Weight</th>
							<th class="p-4 w-40">Grade Input</th>
							<th class="p-4">Calculated Outcome / Effort</th>
							<th class="p-4 w-12 text-center"></th>
						</tr>
					</thead>
					<tbody class="divide-y divide-gray-50">
						{#each finalCalculations as item, index (item.id)}
							<tr class="hover:bg-blue-50/30 transition group">
								<!-- NAME -->
								<td class="p-3">
									<input 
										bind:this={nameInputs[index]}
										bind:value={item.name}
										onkeydown={(e) => handleKeydown(e, index)}
										placeholder="e.g., Final Exam" 
										class="w-full bg-transparent border-0 border-b-2 border-transparent hover:border-gray-200 focus:border-blue-500 focus:ring-0 p-2 transition"
									/>
								</td>
								
								<!-- WEIGHT -->
								<td class="p-3">
									<div class="relative flex items-center">
										<input 
											type="number" 
											bind:value={item.weight}
											onkeydown={(e) => handleKeydown(e, index)}
											class="w-full bg-transparent border-0 border-b-2 border-transparent hover:border-gray-200 focus:border-blue-500 focus:ring-0 p-2 pr-6 transition"
										/>
									</div>
								</td>

								<!-- GRADE INPUT -->
								<td class="p-3 relative">
									<input 
										type="text" 
										bind:value={item.grade}
										onkeydown={(e) => handleKeydown(e, index)}
										placeholder="Blank = predict" 
										class={`w-full bg-transparent border-0 border-b-2 hover:border-gray-200 focus:ring-0 p-2 transition ${item.parseStatus === 'error' ? 'text-red-600 focus:border-red-500 border-red-200' : 'focus:border-blue-500 border-transparent'}`}
									/>
									<!-- Tiny indicator showing how the math/letter parsed -->
									{#if !item.isBlank && item.parseStatus !== 'error' && item.parsedDisplay}
										<span class="absolute bottom-[-10px] left-2 text-[10px] text-gray-400 font-medium tracking-wide">
											= {item.parsedDisplay}
										</span>
									{:else if item.parseStatus === 'error'}
										<span class="absolute bottom-[-10px] left-2 text-[10px] text-red-500 font-medium tracking-wide">
											Invalid format
										</span>
									{/if}
								</td>

								<!-- OUTCOME & EFFORT -->
								<td class="p-3 align-middle">
									{#if item.isBlank}
										<div class="flex items-center gap-4 bg-orange-50/50 p-2 rounded-lg border border-orange-100">
											<div class="flex-1">
												<div class="flex justify-between items-center mb-1">
													<label class="text-xs font-bold text-orange-600 uppercase tracking-wide" for="effort-modifier">Effort Modifier</label>
													<span class="text-xs text-orange-700 font-medium">x{item.effort.toFixed(1)}</span>
												</div>
												<input
													id="effort-modifier"
													type="range" 
													min="0.1" max="3" step="0.1" 
													bind:value={item.effort}
													class="w-full accent-orange-500 h-1.5 bg-orange-200 rounded-lg appearance-none cursor-pointer"
												/>
											</div>
											<div class="text-right pl-4 border-l border-orange-200">
												<div class="text-xs text-orange-600 font-bold uppercase tracking-wide">You Need</div>
												<div class={`text-xl font-black ${item.neededGrade !== null && item.neededGrade > 100 ? 'text-red-600' : 'text-gray-900'}`}>
													{item.neededGrade !== null ? `${item.neededGrade.toFixed(1)}%` : '-'}
												</div>
											</div>
										</div>
									{:else}
										<div class="text-sm text-gray-400 italic px-2">Completed</div>
									{/if}
								</td>

								<!-- ACTIONS -->
								<td class="p-3 text-center">
									<button 
										onclick={() => removeAssignment(item.id)}
										class="p-2 text-gray-300 hover:text-red-500 hover:bg-red-50 rounded-lg transition opacity-0 group-hover:opacity-100 focus:opacity-100"
										title="Delete Assignment"
									>
										<svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M3 6h18"></path><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"></path><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"></path></svg>
									</button>
								</td>
							</tr>
						{/each}
					</tbody>
				</table>
			</div>
			
			<div class="p-4 bg-gray-50/50 border-t border-gray-100">
				<button 
					onclick={addAssignment}
					class="text-sm font-semibold text-blue-600 hover:text-blue-800 flex items-center gap-1 transition px-2 py-1 rounded-md hover:bg-blue-50"
				>
					<svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><line x1="12" y1="5" x2="12" y2="19"></line><line x1="5" y1="12" x2="19" y2="12"></line></svg>
					Add Assignment (or press Enter)
				</button>
			</div>
		</div>

		<!-- FOOTER SUMMARY CARDS -->
		<div class="grid grid-cols-1 md:grid-cols-2 gap-6">
			<!-- Current Status -->
			<div class="bg-white p-6 rounded-2xl shadow-sm border border-gray-100 flex flex-col justify-center">
				<h3 class="text-sm font-bold text-gray-400 uppercase tracking-wider mb-2">Current Average</h3>
				<div class="text-5xl font-black text-gray-900">
					{currentAverage.toFixed(1)}%
				</div>
				<p class="text-sm text-gray-500 mt-2">
					Based on <span class="font-semibold text-gray-700">{totalCompletedWeight}</span> weighted units completed.
				</p>
			</div>

			<!-- Target Status -->
			<div class="bg-gradient-to-br from-blue-600 to-indigo-700 p-6 rounded-2xl shadow-lg text-white relative overflow-hidden">
				<div class="relative z-10">
					<div class="flex justify-between items-start mb-4">
						<h3 class="text-sm font-bold text-blue-200 uppercase tracking-wider">Desired Final Grade</h3>
						<div class="flex items-center bg-white/20 rounded-lg p-1">
							<input 
								type="number" 
								bind:value={desiredGrade}
								class="w-16 bg-transparent border-none text-right text-white font-bold focus:ring-0 focus:outline-none placeholder-blue-300"
							/>
							<span class="pr-2 font-bold">%</span>
						</div>
					</div>

					{#if blankAssignments.length > 0}
						<div class="mt-4">
							<p class="text-blue-100 text-sm mb-1">To achieve your goal, you need an average of:</p>
							<div class="text-4xl font-black">
								{baseRequiredGrade > 0 ? baseRequiredGrade.toFixed(1) : 0}% 
								<span class="text-lg font-medium text-blue-200 ml-2">on remaining</span>
							</div>
							{#if baseRequiredGrade > 100}
								<p class="text-red-300 text-xs mt-2 font-bold bg-red-900/30 inline-block px-2 py-1 rounded">
									Warning: Requires over 100% on remaining assignments.
								</p>
							{/if}
						</div>
					{:else}
						<div class="mt-4 flex items-center gap-3">
							<div class="text-xl font-medium text-blue-100">All assignments graded!</div>
							{#if currentAverage >= desiredGrade}
								<span class="bg-green-400/20 text-green-200 px-3 py-1 rounded-full text-sm font-bold border border-green-400/30">Goal Met 🎉</span>
							{:else}
								<span class="bg-red-400/20 text-red-200 px-3 py-1 rounded-full text-sm font-bold border border-red-400/30">Goal Missed</span>
							{/if}
						</div>
					{/if}
				</div>
				<!-- Decorative background circle -->
				<div class="absolute -bottom-16 -right-16 w-48 h-48 bg-white/10 rounded-full blur-2xl"></div>
			</div>
		</div>

	</div>
</div>

<style>
	/* Hide standard HTML number input arrows for a cleaner look */
	input[type="number"]::-webkit-inner-spin-button,
	input[type="number"]::-webkit-outer-spin-button {
		-webkit-appearance: none;
		margin: 0;
	}
	input[type="number"] {
		appearance: textfield;
	}
	
	/* Simple fade-in animation */
	.animate-fade-in {
		animation: fadeIn 0.2s ease-out forwards;
	}
	@keyframes fadeIn {
		from { opacity: 0; transform: translateY(-5px); }
		to { opacity: 1; transform: translateY(0); }
	}
</style>