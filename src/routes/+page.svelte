<script lang="ts">
	import Chimp from './chimp.jpg';
	import Banana from './banana.jpg';
	import Galaxy from './galaxy.jpg';
	import Angel from './angel.jpg';
	import Bar from './bar.jpg';
	import Cat from './cat.jpg';
	import Fire from './fire.jpg';
	import Skull from './skull.jpg';
	import Svelte from './svelte.jpg';
	import { toggleMode, mode } from 'mode-watcher';
	import * as Dialog from '$lib/components/ui/dialog';
	import Slider from '$lib/components/ui/slider/slider.svelte';
	import Button from '$lib/components/ui/button/button.svelte';
	import Dices from 'lucide-svelte/icons/dices';
	import Upload from 'lucide-svelte/icons/upload';
	import Pause from 'lucide-svelte/icons/pause';
	import Play from 'lucide-svelte/icons/play';
	import Swords from 'lucide-svelte/icons/swords';
	import Github from 'lucide-svelte/icons/github';
	import RotateCcw from 'lucide-svelte/icons/rotate-ccw';
	import Moon from 'lucide-svelte/icons/moon';
	import Sun from 'lucide-svelte/icons/sun';
	import CircleHelp from 'lucide-svelte/icons/circle-help';
	// Dumb bug not letting me import icons in one line.

	const canvasSize: number = 800 as const;
	const images: string[] = [Angel, Bar, Cat, Fire, Skull, Svelte, Chimp, Banana, Galaxy] as const;

	let input: HTMLInputElement;
	let canvas: HTMLCanvasElement;
	let scrollArea: HTMLDivElement;
	let colorCanvas: HTMLCanvasElement;
	let currentRandomIndex: number = 0;
	let imageData: ImageData | null = null;

	let promptPlay: boolean = $state(false);
	let sizeSlider: number[] = $state([15]);
	let colorSlider: number[] = $state([25]);
	let imageURL: string | null = $state(null);
	let fightInterval: NodeJS.Timeout | null = $state(null);
	let colorBet: { r: number; g: number; b: number } | null = $state(null);
	let pixels: { r: number; g: number; b: number; strength: number; dirty: boolean }[] = $state([]);

	let pixelSize: number = $derived(canvasSize / sizeSlider[0]);
	let colors: { r: number; g: number; b: number; count: number }[] = $derived(
		pixels.reduce(
			(acc, pixel) => {
				const colorKey = `${pixel.r},${pixel.g},${pixel.b}`;
				const existingColor = acc.find((c) => `${c.r},${c.g},${c.b}` === colorKey);

				if (existingColor) {
					existingColor.count += 1; // Increment count if the color exists
				} else {
					acc.push({ r: pixel.r, g: pixel.g, b: pixel.b, count: 1 });
				}

				return acc;
			},
			[] as { r: number; g: number; b: number; count: number }[]
		)
	);

	$effect(() => {
		if (colors && colors.length === 1 && fightInterval) {
			clearInterval(fightInterval);
			fightInterval = null;
		}
	});

	$effect(() => {
		if (sizeSlider) initPixels();
	});

	$effect(() => {
		if (colorSlider) initPixels();
	});

	function handleImageUpload(event: Event) {
		if (fightInterval) clearInterval(fightInterval);
		fightInterval = null;
		promptPlay = true;

		const fileInput = event.target as HTMLInputElement;
		const file = fileInput?.files?.[0];

		if (file) {
			const reader = new FileReader();
			reader.onload = () => {
				imageURL = reader.result as string;
				initPixels();
			};
			reader.readAsDataURL(file);
		}
	}

	async function initPixels() {
		if (imageURL && colorCanvas) {
			const colorCtx = colorCanvas.getContext('2d')!;
			const img = new Image();
			img.src = imageURL;

			await new Promise<void>((resolve) => {
				img.onload = () => resolve();
			});

			colorCanvas.width = sizeSlider[0];
			colorCanvas.height = sizeSlider[0];
			colorCtx.drawImage(img, 0, 0, colorCanvas.width, colorCanvas.height);

			imageData = colorCtx.getImageData(0, 0, colorCanvas.width, colorCanvas.height);
			pixels = [];

			for (let i = 0; i < imageData.data.length; i += 4) {
				const r = Math.round(imageData.data[i] / colorSlider[0]) * colorSlider[0];
				const g = Math.round(imageData.data[i + 1] / colorSlider[0]) * colorSlider[0];
				const b = Math.round(imageData.data[i + 2] / colorSlider[0]) * colorSlider[0];
				pixels.push({ r, g, b, strength: 1, dirty: false }); // Initial dominance of 1
			}
			updateCanvas();
		}
	}

	function simulate() {
		if (fightInterval) clearInterval(fightInterval);

		function step() {
			const newPixels = pixels.map((pixel) => ({ ...pixel }));

			// Calculate strengths for all pixels first
			for (let y = 0; y < sizeSlider[0]; y++) {
				for (let x = 0; x < sizeSlider[0]; x++) {
					const pixelIndex = y * sizeSlider[0] + x;
					const currentPixel = newPixels[pixelIndex];
					const neighbors = [
						y > 0 ? newPixels[pixelIndex - sizeSlider[0]] : null, // Top
						y < sizeSlider[0] - 1 ? newPixels[pixelIndex + sizeSlider[0]] : null, // Bottom
						x > 0 ? newPixels[pixelIndex - 1] : null, // Left
						x < sizeSlider[0] - 1 ? newPixels[pixelIndex + 1] : null, // Right
						y > 0 && x > 0 ? newPixels[pixelIndex - sizeSlider[0] - 1] : null, // Top-left
						y > 0 && x < sizeSlider[0] - 1 ? newPixels[pixelIndex - sizeSlider[0] + 1] : null, // Top-right
						y < sizeSlider[0] - 1 && x > 0 ? newPixels[pixelIndex + sizeSlider[0] - 1] : null, // Bottom-left
						y < sizeSlider[0] - 1 && x < sizeSlider[0] - 1
							? newPixels[pixelIndex + sizeSlider[0] + 1]
							: null // Bottom-right
					]
						.filter(Boolean)
						.filter((neighbor) => neighbor !== null);

					if (!neighbors) continue;

					const friends = neighbors.filter(
						(neighbor) =>
							neighbor.r === currentPixel.r &&
							neighbor.g === currentPixel.g &&
							neighbor.b === currentPixel.b
					);

					// No friends lowers strength
					if (!friends && currentPixel.strength === 0) {
						currentPixel.strength--;
						// Fully surrounded by friends always increases strength
					} else if (friends.length === 8) {
						currentPixel.strength++;
					}
					// Some friends inceases strength if strength is lower than number of friends
					if (currentPixel.strength < friends.length) {
						currentPixel.strength++;
					}
				}
			}

			// Fight
			for (let y = 0; y < sizeSlider[0]; y++) {
				for (let x = 0; x < sizeSlider[0]; x++) {
					const pixelIndex = y * sizeSlider[0] + x;
					const currentPixel = newPixels[pixelIndex];

					if (currentPixel.dirty) continue;

					const neighbors = [
						y > 0 ? newPixels[pixelIndex - sizeSlider[0]] : null, // Top
						y < sizeSlider[0] - 1 ? newPixels[pixelIndex + sizeSlider[0]] : null, // Bottom
						x > 0 ? newPixels[pixelIndex - 1] : null, // Left
						x < sizeSlider[0] - 1 ? newPixels[pixelIndex + 1] : null, // Right
						y > 0 && x > 0 ? newPixels[pixelIndex - sizeSlider[0] - 1] : null, // Top-left
						y > 0 && x < sizeSlider[0] - 1 ? newPixels[pixelIndex - sizeSlider[0] + 1] : null, // Top-right
						y < sizeSlider[0] - 1 && x > 0 ? newPixels[pixelIndex + sizeSlider[0] - 1] : null, // Bottom-left
						y < sizeSlider[0] - 1 && x < sizeSlider[0] - 1
							? newPixels[pixelIndex + sizeSlider[0] + 1]
							: null // Bottom-right
					]
						.filter(Boolean)
						.filter((neighbor) => neighbor !== null);

					if (!neighbors) continue;

					const enemies = neighbors.filter(
						(neighbor) =>
							neighbor.r !== currentPixel.r ||
							neighbor.g !== currentPixel.g ||
							neighbor.b !== currentPixel.b
					);

					if (enemies.length === 0) continue;

					const weakestEnemy = enemies.reduce((weakest, enemy) => {
						return enemy.strength < weakest.strength ? enemy : weakest;
					});

					// Winning a fight changes the color of the loser to that of the winner
					// Winning a fight adds enemy strength to current strength for both pixels
					// The taken over pixel is "dirty" meaning it can not fight this round
					// Stalemate between two pixels causes them to both lose strength
					if (currentPixel.strength > weakestEnemy.strength) {
						currentPixel.strength += weakestEnemy.strength;
						weakestEnemy.strength = currentPixel.strength;

						weakestEnemy.dirty = true;

						weakestEnemy.r = currentPixel.r;
						weakestEnemy.g = currentPixel.g;
						weakestEnemy.b = currentPixel.b;
					} else if (currentPixel.strength === weakestEnemy.strength) {
						currentPixel.strength = 1;
						weakestEnemy.strength = 1;
					}
				}
			}

			// Reset dirt
			for (let y = 0; y < sizeSlider[0]; y++) {
				for (let x = 0; x < sizeSlider[0]; x++) {
					newPixels[y * sizeSlider[0] + x].dirty = false;
				}
			}

			pixels = newPixels;
			updateCanvas();
		}

		step();
		fightInterval = setInterval(step, 200);
	}

	function updateCanvas() {
		const ctx = canvas.getContext('2d')!;
		ctx.clearRect(0, 0, canvas.width, canvas.height);

		for (let y = 0; y < sizeSlider[0]; y++) {
			for (let x = 0; x < sizeSlider[0]; x++) {
				const index = y * sizeSlider[0] + x;
				const { r, g, b, strength } = pixels[index];
				ctx.fillStyle = `rgb(${r},${g},${b})`;
				ctx.fillRect(x * pixelSize, y * pixelSize, pixelSize, pixelSize);

				ctx.strokeStyle = `rgb(${r},${g},${b})`;
				ctx.strokeRect(x * pixelSize, y * pixelSize, pixelSize, pixelSize);
				// ctx.fillStyle = 'white';
				// ctx.fillText(strength.toString(), x * pixelSize, y * pixelSize + 10);
			}
		}
	}

	function handleCanvasClick(e: MouseEvent) {
		if (fightInterval || colors.length === 1) return;
		const ctx = canvas.getContext('2d')!;
		const rect = canvas.getBoundingClientRect();
		const x = ((e.clientX - rect.left) / rect.width) * canvasSize;
		const y = ((e.clientY - rect.top) / rect.height) * canvasSize;
		const pixelData = ctx.getImageData(x, y, 1, 1).data;
		colorBet = { r: pixelData[0], g: pixelData[1], b: pixelData[2] };
		const elementTop = document.getElementById(
			`${colorBet.r}${colorBet.g}${colorBet.b}`
		)?.offsetTop;
		if (!elementTop) return;
		scrollArea.scrollTo({ top: elementTop - rect.height / 4, behavior: 'smooth' });
	}
</script>

<div class="max-w-80 select-none">
	<div class="relative">
		<canvas
			bind:this={canvas}
			width={canvasSize}
			height={canvasSize}
			onclick={handleCanvasClick}
			class="aspect-square w-full rounded-md border bg-popover"
		></canvas>
		{#if !imageURL}
			<div
				class="absolute left-1/2 top-1/2 flex -translate-x-1/2 -translate-y-1/2 flex-col items-center justify-center"
			>
				<Swords class="h-10 w-10 text-muted-foreground" />
				<p>Pixel Attack</p>
			</div>
		{/if}
		<div
			bind:this={scrollArea}
			class="mask no-scrollbar left-full top-0 hidden h-80 min-w-20 translate-x-2 overflow-auto py-2 transition-[padding] focus-visible:pl-3 focus-visible:outline-none sm:absolute sm:block {colors.length ===
				1 || fightInterval
				? 'pointer-events-none'
				: ''}"
		>
			{#each colors as color}
				<button
					id="{color.r}{color.g}{color.b}"
					onclick={() => {
						if (fightInterval || colors.length === 1) return;
						if (colorBet?.r === color.r && colorBet?.g === color.g && colorBet?.b === color.b) {
							colorBet = null;
						} else {
							colorBet = { r: color.r, g: color.g, b: color.b };
						}
					}}
					class="flex items-center gap-2 transition-[padding] hover:pl-2 focus-visible:pl-2 focus-visible:outline-none"
				>
					<div
						style="background: rgb({color.r},{color.g},{color.b}); width: {(color.count / 100) *
							16 +
							2}px"
						class="h-4 rounded-sm transition-[width]"
					></div>
					<p class="truncate font-mono text-sm text-muted-foreground">
						{color.count}
						{colors.length === 1 ? '🥇' : ''}
						{colorBet?.r === color.r && colorBet?.g === color.g && colorBet?.b === color.b
							? '💵'
							: ''}
					</p>
				</button>
			{/each}
		</div>
	</div>

	<!-- Controls -->
	<div class="mt-4 flex gap-2">
		<div class="space-y-2">
			<div class="flex gap-2 rounded-md">
				<Button
					size="icon"
					variant="outline"
					onclick={() => {
						promptPlay = true;
						if (fightInterval) {
							clearInterval(fightInterval);
							fightInterval = null;
						}
						if (currentRandomIndex < images.length - 1) {
							currentRandomIndex++;
						} else {
							currentRandomIndex = 0;
						}
						imageURL = images[currentRandomIndex];
					}}
				>
					<Dices class="h-4 w-4" />
				</Button>

				<Button
					size="icon"
					variant="outline"
					onclick={() => input.click()}
					class="relative overflow-hidden"
				>
					<img
						src={imageURL}
						alt=""
						class="absolute left-0 top-0 h-full w-full transition-opacity
				{imageURL ? 'opacity-100' : 'opacity-0'}"
					/>
					<Upload class="h-4 w-4" />
					<input
						hidden
						type="file"
						accept="image/*"
						bind:this={input}
						onchange={handleImageUpload}
					/>
				</Button>

				<Button
					size="icon"
					disabled={!imageURL}
					variant="outline"
					class={promptPlay ? 'bg-green-500 text-white hover:bg-green-600' : ''}
					onclick={async () => {
						promptPlay = false;
						if (fightInterval) {
							clearTimeout(fightInterval);
							fightInterval = null;
						} else if (colors.length === 1) {
							initPixels();
						} else {
							simulate();
						}
					}}
				>
					{#if fightInterval}
						<Pause class="h-4 w-4" />
					{:else if colors.length === 1}
						<RotateCcw class="h-4 w-4" />
					{:else}
						<Play class="h-4 w-4" />
					{/if}
				</Button>
			</div>

			<div class="flex gap-2 rounded-md">
				<Dialog.Root>
					<Dialog.Trigger>
						{#snippet child({ props })}
							<Button size="icon" variant="outline" {...props}>
								<CircleHelp class="h-4 w-4" />
							</Button>
						{/snippet}
					</Dialog.Trigger>
					<Dialog.Content class="max-w-md">
						<Dialog.Header>
							<Dialog.Title>What is this?</Dialog.Title>
							<Dialog.Description>
								Just a small demo inspired by <a
									class="underline hover:text-foreground"
									href="https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life"
									>Conway's Game of Life.</a
								> An image is broken down into larger pixels, then they fight. The winning team is the
								color that takes over the entire board.
							</Dialog.Description>

							<br />
							<Dialog.Title>How it works:</Dialog.Title>
							<div class="space-y-2">
								<div class="flex gap-2">
									<p class="shrink-0">🧮</p>
									<p class="text-sm">
										Every round we calculate the strength of each pixel based off the number of
										allies surrounding.
									</p>
								</div>

								<div class="flex gap-2">
									<p class="shrink-0">⚔️</p>
									<p class="text-sm">
										Then we fight! Each pixel picks the weakest enemy near it and fights it. Taking
										over a pixel gives it momentum.
									</p>
								</div>

								<div class="flex gap-2">
									<p class="shrink-0">💵</p>
									<p class="text-sm">
										Click on a color before the round starts to bet on it winning.
									</p>
								</div>
							</div>
						</Dialog.Header>
					</Dialog.Content>
				</Dialog.Root>

				<Button size="icon" variant="outline" onclick={toggleMode}>
					{#if $mode === 'dark'}
						<Moon class="h-4 w-4" />
					{:else}
						<Sun class="h-4 w-4" />
					{/if}
				</Button>
			</div>
		</div>

		<!-- Sliders -->
		<div
			class="grow space-y-2 px-4 transition-opacity {fightInterval
				? 'pointer-events-none opacity-50'
				: ''}"
		>
			<div class="flex h-10 items-center">
				<Slider bind:value={sizeSlider} min={10} max={25} />
			</div>
			<div class="flex h-10 items-center">
				<Slider bind:value={colorSlider} min={10} max={125} />
			</div>
		</div>
	</div>
</div>

<Button
	size="sm"
	variant="ghost"
	onclick={toggleMode}
	href="https:unin.io/@jason"
	class="absolute bottom-2 right-2 text-muted-foreground"
>
	Made by @jason
</Button>

<canvas hidden bind:this={colorCanvas}></canvas>

<style>
	.mask {
		mask-image: linear-gradient(
			to bottom,
			transparent 0%,
			black 0.5rem,
			black calc(100% - 0.5rem),
			transparent 100%
		);
	}

	.no-scrollbar::-webkit-scrollbar {
		display: none;
	}

	.no-scrollbar {
		-ms-overflow-style: none;
		-webkit-overflow-scrolling: touch;
		scrollbar-width: none;
	}
</style>
