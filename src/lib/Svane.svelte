<script>
	import { fly } from 'svelte/transition';
	import { readable } from 'svelte/store';
	import resolveConfig from 'tailwindcss/resolveConfig';
	import tailwindConfig from 'tailwind-config';

	const config = resolveConfig(tailwindConfig);
	console.log(config.theme.screens);
	/** @type {Object.<string, {Object.<string, number>}>} */
	export let breakpoints = config.theme.screens;

	// // TODO: Get from Tailwind config
	// /** @type {Object.<string, {Object.<string, number>}>} */
	// export let breakpoints;

	/** @type {boolean} */
	export let stayOpen = false;

	/** @type {number} */
	export let duration = 2500;

	/** @type {('top'|'bottom')} */
	export let position = 'top';

	export const convertBreakpoints = (breakpoints) => {
    const convertedBreakpoints = {};

    const breakpointKeys = Object.keys(breakpoints);
    for (let i = 0; i < breakpointKeys.length; i++) {
        const currentKey = breakpointKeys[i];
        const currentVal = parseInt(breakpoints[currentKey]);

        if (i === 0) {
            // First breakpoint
			const nextKey = breakpointKeys[i + 1];
            const nextVal = parseInt(breakpoints[nextKey]);
            convertedBreakpoints[currentKey] = { max: nextVal - 1 };
        } else if (i === breakpointKeys.length - 1) {
            // Last breakpoint
            convertedBreakpoints[currentKey] = { min: currentVal };
        } else {
			// Other Breakpoints
            const nextKey = breakpointKeys[i + 1];
            const nextVal = parseInt(breakpoints[nextKey]);
            convertedBreakpoints[currentKey] = { min: currentVal, max: nextVal - 1 };
        }
    }
    console.log("convertedBreakpoints", convertedBreakpoints);
    return convertedBreakpoints;
};
	const testVar = convertBreakpoints(breakpoints)
	console.log("testVar", testVar)

	let show = false;
	let mediaQueries = {};

	for (const [name, sizes] of Object.entries(testVar)) {
		let mediaQuery = '';
		if (sizes.min) {
			mediaQuery += `(min-width: ${sizes.min}px)`;
			if (sizes.max) {
				mediaQuery += ' and ';
			}
		}

		if (sizes.max) {
			mediaQuery += `(max-width: ${sizes.max}px)`;
		}

		mediaQueries[name] = mediaQuery;
	}

	function calculateViewportPercentage(arr, innerWidth) {
		let matchingBreakpoint = null;

		for (const breakpoint in arr) {
			const values = arr[breakpoint];
			const min = values.min || 0;
			const max = values.max || Infinity;
			
			if (min <= innerWidth && innerWidth <= max) {
				matchingBreakpoint = values;

				break;
			}
		}

		if (matchingBreakpoint === null) {
			return `The given innerWidth (${innerWidth}) doesn't match any breakpoints`;
		}

		const rangeStart = matchingBreakpoint.min || 0;
		const rangeEnd = matchingBreakpoint.max;

		if (!rangeEnd) {
			return 1;
		}

		return (innerWidth - rangeStart) / (rangeEnd - rangeStart);
	}

	function showForDuration() {
		if (show) return;
		show = true;

		if (!stayOpen) {
			setTimeout(() => {
				show = false;
			}, duration);
		}
	}

	function breakpointMatcher() {
		if (typeof window === 'undefined') {
			return readable(null);
		}

		return readable('', (set) => {
			const breakpointNames = Object.keys(mediaQueries);

			Object.keys(mediaQueries).forEach(
				(mql) => (mediaQueries[mql] = window.matchMedia(mediaQueries[mql]))
			);

			const setCurrentBreakPoint = () => {
				for (const size of breakpointNames) {
					const mql = mediaQueries[size];
					if (mql?.matches) {
						set(size);
					}
				}
			};

			setCurrentBreakPoint();
			showForDuration();

			window.addEventListener('resize', () => {
				setCurrentBreakPoint();
				showForDuration();
			});
		});
	}

	export const shortcut = (_node, params) => {
		let handler;
		const removeHandler = () => window.removeEventListener('keydown', handler),
			setHandler = () => {
				removeHandler();
				if (!params) return;

				handler = (e) => {
					if (
						!!params.alt != e.altKey ||
						!!params.shift != e.shiftKey ||
						!!params.control != (e.ctrlKey || e.metaKey) ||
						params.code != e.code
					)
						return;
					e.preventDefault();
					if (params?.callback) {
						params.callback();
					}
				};

				window.addEventListener('keydown', handler);
			};

		setHandler();

		return {
			update: setHandler,
			destroy: removeHandler
		};
	};

	const currentBreakpointName = breakpointMatcher();

	let innerWidth;
	let innerHeight;

	$: percent = Math.round(calculateViewportPercentage(breakpoints, innerWidth) * 100);
</script>

<svelte:window
	bind:innerWidth
	bind:innerHeight
	use:shortcut={{ shift: true, code: 'KeyT', callback: () => (show = !show) }}
/>

{#if show}
	<div
		class="absolute top-0 w-screen bg-gray-800 text-white p-4"
		class:top-0={position === 'top'}
		class:bottom-0={position === 'bottom'}
		transition:fly={{ y: position === 'top' ? -200 : 200, duration: 400 }}
	>
		<div class="flex space-x-4 items-center">
			{#each Object.keys(mediaQueries) as breakpoint}
				{#if $currentBreakpointName === breakpoint}
					<div class="relative w-full bg-slate-700 border border-blue-500 rounded-xl">
						<div
							class="absolute h-full bg-blue-800 text-center rounded-xl"
							style={`width: ${percent}%`}
						/>
						<div
							class="relative flex space-x-6 lg:space-x-8 xl:space-x-10 2xl:space-x-16 items-center justify-center text-slate-50"
						>
							<span class="font-semibold text-xs">{percent}%</span>
							<span class="font-bold">{breakpoint}</span>
							<span class="font-semibold text-xs">{innerWidth} x {innerHeight}</span>
						</div>
					</div>
				{:else}
					<div class="py-0 px-4 border border-slate-400 text-slate-400 rounded-xl">
						{breakpoint}
					</div>
				{/if}
			{/each}
		</div>
	</div>
{/if}
