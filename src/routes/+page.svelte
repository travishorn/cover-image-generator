<script lang="ts">
	import '@fontsource-variable/caveat';
	import '@fontsource-variable/climate-crisis';
	import '@fontsource-variable/comfortaa';
	import '@fontsource-variable/dancing-script';
	import '@fontsource-variable/eb-garamond';
	import '@fontsource-variable/fira-code';
	import '@fontsource-variable/funnel-display';
	import '@fontsource-variable/ibm-plex-sans';
	import '@fontsource-variable/inter';
	import '@fontsource-variable/jetbrains-mono';
	import '@fontsource-variable/lora';
	import '@fontsource-variable/merriweather';
	import '@fontsource-variable/montserrat';
	import '@fontsource-variable/noto-serif';
	import '@fontsource-variable/open-sans';
	import '@fontsource-variable/pixelify-sans';
	import '@fontsource-variable/playfair-display';
	import '@fontsource-variable/playpen-sans';
	import '@fontsource-variable/public-sans';
	import '@fontsource-variable/roboto';
	import '@fontsource-variable/roboto-mono';
	import '@fontsource-variable/roboto-slab';
	import { onDestroy } from 'svelte';

	const OUTPUT_WIDTH = 1200;
	const OUTPUT_HEIGHT = 630;
	const OUTPUT_ASPECT = OUTPUT_WIDTH / OUTPUT_HEIGHT;
	const PREVIEW_STAGE_ASPECT = 16 / 10;
	const WEBP_QUALITY = 0.92;
	const DEFAULT_TEXT_SIZE = 108;
	const TEXT_MIN_SIZE = 56;
	const TEXT_MAX_SIZE = 188;
	const TEXT_WIDTH_RATIO = 0.75;
	const TEXT_LINE_HEIGHT = 1.08;
	const SCRIM_OPACITY = 0.05;
	const SCRIM_FEATHER_OPACITY = 0.03;
	const SCRIM_BLUR_BASE = 6;
	const FALLBACK_FONT_FAMILY = "'Segoe UI', 'Helvetica Neue', Arial, sans-serif";

	const FONT_OPTIONS = [
		{ label: 'Funnel Display', family: "'Funnel Display'" },
		{ label: 'Montserrat', family: "'Montserrat Variable'" },
		{ label: 'Public Sans', family: "'Public Sans Variable'" },
		{ label: 'Open Sans', family: "'Open Sans Variable'" },
		{ label: 'Inter', family: "'Inter Variable'" },
		{ label: 'IBM Plex Sans', family: "'IBM Plex Sans Variable'" },
		{ label: 'Roboto', family: "'Roboto Variable'" },
		{ label: 'Roboto Slab', family: "'Roboto Slab Variable'" },
		{ label: 'Lora', family: "'Lora Variable'" },
		{ label: 'Merriweather', family: "'Merriweather Variable'" },
		{ label: 'Noto Serif', family: "'Noto Serif Variable'" },
		{ label: 'Playfair Display', family: "'Playfair Display Variable'" },
		{ label: 'EB Garamond', family: "'EB Garamond Variable'" },
		{ label: 'Comfortaa', family: "'Comfortaa Variable'" },
		{ label: 'Dancing Script', family: "'Dancing Script Variable'" },
		{ label: 'Caveat', family: "'Caveat Variable'" },
		{ label: 'Playpen Sans', family: "'Playpen Sans Variable'" },
		{ label: 'JetBrains Mono', family: "'JetBrains Mono Variable'" },
		{ label: 'Roboto Mono', family: "'Roboto Mono Variable'" },
		{ label: 'Fira Code', family: "'Fira Code Variable'" },
		{ label: 'Pixelify Sans', family: "'Pixelify Sans Variable'" },
		{ label: 'Climate Crisis', family: "'Climate Crisis Variable'" }
	] as const;

	let picker: HTMLInputElement | null = null;
	let frameEl = $state<HTMLDivElement | null>(null);

	let frameWidth = $state(0);
	let dropActive = $state(false);
	let isLoading = $state(false);
	let isExporting = $state(false);
	let errorMessage = $state<string | null>(null);

	let sourceUrl = $state<string | null>(null);
	let sourceName = $state('image');
	let sourceImage = $state<HTMLImageElement | null>(null);
	let sourceWidth = $state(0);
	let sourceHeight = $state(0);

	let drawScale = $state(1);
	let offsetX = $state(0);
	let offsetY = $state(0);

	let dragging = $state(false);
	let dragPointerId = $state<number | null>(null);
	let dragLastX = $state(0);
	let dragLastY = $state(0);
	let overlayText = $state('');
	let textSize = $state(DEFAULT_TEXT_SIZE);
	let selectedFontFamily = $state(FONT_OPTIONS[0].family);

	const hasImage = $derived(sourceImage !== null && sourceUrl !== null);
	const stageHeight = $derived(frameWidth > 0 ? frameWidth / PREVIEW_STAGE_ASPECT : 0);
	const cropWidth = $derived(
		frameWidth > 0 ? Math.min(frameWidth * 0.9, stageHeight * 0.9 * OUTPUT_ASPECT) : 0
	);
	const cropHeight = $derived(cropWidth > 0 ? cropWidth / OUTPUT_ASPECT : 0);
	const cropLeft = $derived((frameWidth - cropWidth) / 2);
	const cropTop = $derived((stageHeight - cropHeight) / 2);
	const previewScale = $derived(cropWidth > 0 ? cropWidth / OUTPUT_WIDTH : 1);
	const hasOverlayText = $derived(overlayText.trim().length > 0);
	const previewTextSize = $derived(textSize * previewScale);
	const previewTextBoxWidth = $derived(OUTPUT_WIDTH * TEXT_WIDTH_RATIO * previewScale);
	const previewScrimPaddingY = $derived(Math.max(3, previewTextSize * 0.12));
	const previewScrimPaddingX = $derived(Math.max(4, previewTextSize * 0.15));
	const previewScrimRadius = $derived(Math.max(6, previewTextSize * 0.1));

	$effect(() => {
		if (!frameEl) {
			return;
		}

		const observer = new ResizeObserver(() => {
			if (frameEl) {
				frameWidth = frameEl.clientWidth;
			}
		});

		observer.observe(frameEl);
		frameWidth = frameEl.clientWidth;

		return () => observer.disconnect();
	});

	onDestroy(() => {
		releaseImageUrl();
	});

	function releaseImageUrl() {
		if (sourceUrl) {
			URL.revokeObjectURL(sourceUrl);
			sourceUrl = null;
		}
	}

	function clamp(value: number, min: number, max: number) {
		return Math.min(max, Math.max(min, value));
	}

	function clampOffsets(nextX: number, nextY: number) {
		const renderedWidth = sourceWidth * drawScale;
		const renderedHeight = sourceHeight * drawScale;

		const minX = OUTPUT_WIDTH - renderedWidth;
		const minY = OUTPUT_HEIGHT - renderedHeight;

		offsetX = clamp(nextX, minX, 0);
		offsetY = clamp(nextY, minY, 0);
	}

	function getDownloadName() {
		const normalized = sourceName
			.replace(/\.[^/.]+$/, '')
			.toLowerCase()
			.replace(/[^a-z0-9-_]+/g, '-')
			.replace(/^-+|-+$/g, '')
			.slice(0, 48);

		return `${normalized || 'cover'}-1200x630.webp`;
	}

	function getWrappedLines(context: CanvasRenderingContext2D, text: string, maxWidth: number) {
		const lines: string[] = [];
		const paragraphs = text.split(/\r?\n/);

		for (const paragraph of paragraphs) {
			if (!paragraph.trim()) {
				lines.push('');
				continue;
			}

			const words = paragraph.trim().split(/\s+/);
			let currentLine = words[0] ?? '';

			for (const word of words.slice(1)) {
				const candidate = `${currentLine} ${word}`;
				if (context.measureText(candidate).width <= maxWidth) {
					currentLine = candidate;
				} else {
					lines.push(currentLine);
					currentLine = word;
				}
			}

			if (currentLine) {
				lines.push(currentLine);
			}
		}

		return lines;
	}

	function drawOverlayText(context: CanvasRenderingContext2D) {
		if (!hasOverlayText) {
			return;
		}

		context.save();
		context.font = `900 ${textSize}px ${selectedFontFamily}, ${FALLBACK_FONT_FAMILY}`;
		context.textAlign = 'center';
		context.textBaseline = 'top';

		const maxTextWidth = OUTPUT_WIDTH * TEXT_WIDTH_RATIO;
		const lines = getWrappedLines(context, overlayText, maxTextWidth);
		const lineWidths = lines.map((line) => context.measureText(line).width);
		const widestLine = Math.max(...lineWidths, 0);
		const lineHeight = textSize * TEXT_LINE_HEIGHT;
		const totalTextHeight = lines.length * lineHeight;
		const startY = (OUTPUT_HEIGHT - totalTextHeight) / 2;

		const padX = textSize * 0.12;
		const padY = textSize * 0.1;
		const scrimWidth = Math.min(
			OUTPUT_WIDTH * 0.7,
			Math.max(widestLine + padX * 2, textSize * 1.8)
		);
		const scrimHeight = totalTextHeight + padY * 2;
		const scrimX = (OUTPUT_WIDTH - scrimWidth) / 2;
		const scrimY = startY - padY;
		const scrimCenterX = OUTPUT_WIDTH / 2;
		const scrimCenterY = OUTPUT_HEIGHT / 2;
		const scrimOuterRadius = Math.max(scrimWidth * 0.72, scrimHeight * 1.1);
		const scrimInnerRadius = Math.max(textSize * 0.3, scrimOuterRadius * 0.2);

		const scrimGradient = context.createRadialGradient(
			scrimCenterX,
			scrimCenterY,
			scrimInnerRadius,
			scrimCenterX,
			scrimCenterY,
			scrimOuterRadius
		);
		scrimGradient.addColorStop(0, `rgba(0, 0, 0, ${SCRIM_OPACITY})`);
		scrimGradient.addColorStop(0.58, `rgba(0, 0, 0, ${SCRIM_FEATHER_OPACITY})`);
		scrimGradient.addColorStop(1, 'rgba(0, 0, 0, 0)');

		context.save();
		context.filter = `blur(${Math.max(SCRIM_BLUR_BASE, textSize * 0.07)}px)`;
		context.fillStyle = scrimGradient;
		context.fillRect(
			scrimX - scrimWidth * 0.3,
			scrimY - scrimHeight * 0.5,
			scrimWidth * 1.6,
			scrimHeight * 2
		);
		context.restore();

		context.fillStyle = '#ffffff';
		context.shadowColor = 'rgba(0, 0, 0, 0.45)';
		context.shadowBlur = Math.max(1, textSize * 0.045);
		context.shadowOffsetY = Math.max(1, textSize * 0.05);

		for (let index = 0; index < lines.length; index += 1) {
			context.fillText(lines[index], OUTPUT_WIDTH / 2, startY + index * lineHeight);
		}

		context.restore();
	}

	async function loadFile(file: File) {
		if (file.type && !file.type.startsWith('image/')) {
			errorMessage = 'Please use an image file.';
			return;
		}

		isLoading = true;
		errorMessage = null;

		const objectUrl = URL.createObjectURL(file);
		const image = new Image();
		image.decoding = 'async';

		try {
			await new Promise<void>((resolve, reject) => {
				image.onload = () => resolve();
				image.onerror = () => reject(new Error('The file could not be decoded as an image.'));
				image.src = objectUrl;
			});

			releaseImageUrl();
			sourceUrl = objectUrl;
			sourceImage = image;
			sourceName = file.name || 'image';
			sourceWidth = image.naturalWidth;
			sourceHeight = image.naturalHeight;

			const baseScale = Math.max(OUTPUT_WIDTH / sourceWidth, OUTPUT_HEIGHT / sourceHeight);
			drawScale = baseScale;

			const renderedWidth = sourceWidth * drawScale;
			const renderedHeight = sourceHeight * drawScale;

			offsetX = (OUTPUT_WIDTH - renderedWidth) / 2;
			offsetY = (OUTPUT_HEIGHT - renderedHeight) / 2;
			clampOffsets(offsetX, offsetY);
		} catch {
			URL.revokeObjectURL(objectUrl);
			errorMessage = 'This image could not be opened by your browser.';
		} finally {
			isLoading = false;
		}
	}

	async function onPickerChange(event: Event) {
		const target = event.currentTarget as HTMLInputElement;
		const file = target.files?.[0];
		if (!file) {
			return;
		}

		await loadFile(file);
		target.value = '';
	}

	function onDrop(event: DragEvent) {
		event.preventDefault();
		dropActive = false;

		const file = event.dataTransfer?.files?.[0];
		if (!file) {
			return;
		}

		void loadFile(file);
	}

	function onDragOver(event: DragEvent) {
		event.preventDefault();
		dropActive = true;
	}

	function onDragLeave(event: DragEvent) {
		event.preventDefault();
		dropActive = false;
	}

	function triggerPicker() {
		picker?.click();
	}

	function resetEditor() {
		errorMessage = null;
		sourceImage = null;
		sourceName = 'image';
		sourceWidth = 0;
		sourceHeight = 0;
		drawScale = 1;
		offsetX = 0;
		offsetY = 0;
		overlayText = '';
		textSize = DEFAULT_TEXT_SIZE;
		selectedFontFamily = FONT_OPTIONS[0].family;
		releaseImageUrl();
	}

	function beginDrag(event: PointerEvent) {
		if (!hasImage) {
			return;
		}

		dragging = true;
		dragPointerId = event.pointerId;
		dragLastX = event.clientX;
		dragLastY = event.clientY;

		(event.currentTarget as HTMLElement).setPointerCapture(event.pointerId);
	}

	function onDragMove(event: PointerEvent) {
		if (!dragging || event.pointerId !== dragPointerId || !hasImage) {
			return;
		}

		const dxPreview = event.clientX - dragLastX;
		const dyPreview = event.clientY - dragLastY;

		dragLastX = event.clientX;
		dragLastY = event.clientY;

		const safeScale = previewScale > 0 ? previewScale : 1;
		const dxOutput = dxPreview / safeScale;
		const dyOutput = dyPreview / safeScale;

		clampOffsets(offsetX + dxOutput, offsetY + dyOutput);
	}

	function endDrag(event: PointerEvent) {
		if (event.pointerId !== dragPointerId) {
			return;
		}

		dragging = false;
		dragPointerId = null;
	}

	async function exportWebp() {
		if (!sourceImage || isExporting) {
			return;
		}

		isExporting = true;
		errorMessage = null;

		try {
			const canvas = document.createElement('canvas');
			canvas.width = OUTPUT_WIDTH;
			canvas.height = OUTPUT_HEIGHT;

			const context = canvas.getContext('2d');
			if (!context) {
				throw new Error('Canvas is not available in this browser.');
			}

			context.drawImage(
				sourceImage,
				offsetX,
				offsetY,
				sourceWidth * drawScale,
				sourceHeight * drawScale
			);

			drawOverlayText(context);

			const blob = await new Promise<Blob | null>((resolve) => {
				canvas.toBlob(resolve, 'image/webp', WEBP_QUALITY);
			});

			if (!blob || blob.type !== 'image/webp') {
				throw new Error('WEBP export is unavailable in this browser.');
			}

			const downloadUrl = URL.createObjectURL(blob);
			const anchor = document.createElement('a');
			anchor.href = downloadUrl;
			anchor.download = getDownloadName();
			document.body.append(anchor);
			anchor.click();
			anchor.remove();
			URL.revokeObjectURL(downloadUrl);
		} catch (error) {
			errorMessage = error instanceof Error ? error.message : 'Export failed.';
		} finally {
			isExporting = false;
		}
	}
</script>

<main
	class="min-h-screen bg-[radial-gradient(circle_at_10%_10%,#2b3645_0%,#131a25_40%,#0b1018_100%)] text-slate-100"
>
	<section class="mx-auto flex w-full max-w-6xl flex-col gap-6 px-4 py-8 sm:px-6 sm:py-10 lg:px-10">
		<div class="grid gap-5 lg:grid-cols-[minmax(0,1fr)_320px]">
			<div
				class={`relative overflow-hidden rounded-2xl border bg-slate-950/45 p-3 shadow-2xl backdrop-blur-sm sm:p-4 ${dropActive ? 'border-cyan-300/80 ring-2 ring-cyan-300/60' : 'border-slate-700/70'}`}
				role="region"
				aria-label="Image upload and drop area"
				ondrop={onDrop}
				ondragover={onDragOver}
				ondragleave={onDragLeave}
			>
				{#if !hasImage}
					<button
						type="button"
						class="flex aspect-16/10 w-full flex-col items-center justify-center gap-3 rounded-xl border border-dashed border-slate-500/70 bg-slate-900/70 px-4 text-center transition hover:border-cyan-300/80 hover:bg-slate-900"
						onclick={triggerPicker}
					>
						<span class="text-lg font-medium">Drop an image here</span>
						<span class="text-sm text-slate-300">or click to browse from your filesystem</span>
					</button>
				{:else}
					<div
						class="relative mx-auto aspect-16/10 w-full max-w-245 cursor-grab overflow-hidden rounded-xl border border-slate-500/70 bg-slate-950 shadow-inner active:cursor-grabbing"
						role="application"
						aria-label="Drag image to position within fixed crop window"
						bind:this={frameEl}
						onpointerdown={beginDrag}
						onpointermove={onDragMove}
						onpointerup={endDrag}
						onpointercancel={endDrag}
						style:touch-action="none"
					>
						{#if sourceUrl}
							<img
								src={sourceUrl}
								alt="Uploaded preview"
								class="pointer-events-none absolute top-0 left-0 select-none"
								style:width={`${sourceWidth * drawScale * previewScale}px`}
								style:height={`${sourceHeight * drawScale * previewScale}px`}
								style:transform={`translate(${cropLeft + offsetX * previewScale}px, ${cropTop + offsetY * previewScale}px)`}
								style:filter="brightness(0.34)"
								draggable="false"
							/>
							<div
								class="pointer-events-none absolute overflow-hidden rounded-lg border border-slate-100/80 shadow-[0_0_0_200vmax_rgba(6,10,16,0.48)]"
								style:left={`${cropLeft}px`}
								style:top={`${cropTop}px`}
								style:width={`${cropWidth}px`}
								style:height={`${cropHeight}px`}
							>
								<img
									src={sourceUrl}
									alt="Crop frame preview"
									class="absolute top-0 left-0 select-none"
									style:width={`${sourceWidth * drawScale * previewScale}px`}
									style:height={`${sourceHeight * drawScale * previewScale}px`}
									style:transform={`translate(${offsetX * previewScale}px, ${offsetY * previewScale}px)`}
									draggable="false"
								/>
								{#if hasOverlayText}
									<div
										class="pointer-events-none absolute inline-block w-fit text-center font-black text-white"
										style:left="50%"
										style:top="50%"
										style:width="max-content"
										style:max-width={`${previewTextBoxWidth}px`}
										style:font-size={`${previewTextSize}px`}
										style:line-height={TEXT_LINE_HEIGHT}
										style:padding={`${previewScrimPaddingY}px ${previewScrimPaddingX}px`}
										style:border-radius={`${previewScrimRadius}px`}
										style:text-shadow={`${Math.max(1, previewTextSize * 0.05)}px ${Math.max(1, previewTextSize * 0.05)}px ${Math.max(3, previewTextSize * 0.18)}px rgba(0,0,0,0.6)`}
										style:transform="translate(-50%, -50%)"
										style:white-space="pre-wrap"
										style:overflow-wrap="normal"
										style:word-break="normal"
										style:font-family={`${selectedFontFamily}, ${FALLBACK_FONT_FAMILY}`}
									>
										<div
											class="absolute inset-0 -z-10"
											style:transform="scale(1.3, 1.55)"
											style:filter={`blur(${Math.max(1, previewTextSize * 0.1)}px)`}
											style:border-radius={`${previewScrimRadius * 4}px`}
											style:background={`radial-gradient(ellipse at center, rgba(0,0,0,${SCRIM_OPACITY}) 0%, rgba(0,0,0,${SCRIM_FEATHER_OPACITY}) 55%, rgba(0,0,0,0) 100%)`}
										></div>
										{overlayText}
									</div>
								{/if}
							</div>
						{/if}
					</div>
				{/if}
			</div>

			<aside
				class="space-y-4 rounded-2xl border border-slate-700/70 bg-slate-950/55 p-4 shadow-xl backdrop-blur-sm sm:p-5"
			>
				<div class="space-y-3">
					<input
						bind:this={picker}
						type="file"
						accept="image/*"
						class="hidden"
						onchange={onPickerChange}
					/>
					<button
						type="button"
						class="w-full rounded-lg border border-slate-500/80 px-3 py-2 text-sm font-medium transition hover:border-slate-300"
						onclick={resetEditor}
						disabled={!hasImage || isLoading}
					>
						Reset
					</button>
					<button
						type="button"
						class="w-full rounded-lg bg-emerald-400 px-3 py-2 text-sm font-semibold text-slate-950 transition hover:bg-emerald-300 disabled:cursor-not-allowed disabled:bg-emerald-400/40"
						onclick={exportWebp}
						disabled={!hasImage || isLoading || isExporting}
					>
						{isExporting ? 'Exporting WEBP...' : 'Download WEBP (1200x630)'}
					</button>

					<div class="space-y-2 rounded-lg border border-slate-700/80 bg-slate-900/70 p-3">
						<label
							for="overlay-text"
							class="block text-xs font-medium tracking-wide text-slate-200 uppercase"
						>
							Text
						</label>
						<textarea
							id="overlay-text"
							rows="4"
							class="w-full resize-y rounded-md border border-slate-600 bg-slate-950/80 px-2 py-2 text-sm text-white transition outline-none focus:border-cyan-300"
							placeholder="Type headline text..."
							bind:value={overlayText}
						></textarea>

						<div class="space-y-4">
							<div class="space-y-1">
								<label
									for="overlay-font"
									class="block text-xs font-medium tracking-wide text-slate-200 uppercase"
								>
									Font
								</label>
								<select
									id="overlay-font"
									class="w-full rounded-md border border-slate-600 bg-slate-950/80 px-2 py-2 text-sm text-white transition outline-none focus:border-cyan-300"
									bind:value={selectedFontFamily}
								>
									{#each FONT_OPTIONS as option (option.family)}
										<option value={option.family}>{option.label}</option>
									{/each}
								</select>
							</div>

							<div class="space-y-1">
								<div class="flex items-center justify-between">
									<label
										for="overlay-size"
										class="block text-xs font-medium tracking-wide text-slate-200 uppercase"
									>
										Text size
									</label>
									<span class="text-xs text-slate-300">{textSize}px</span>
								</div>
								<input
									id="overlay-size"
									type="range"
									min={TEXT_MIN_SIZE}
									max={TEXT_MAX_SIZE}
									step="1"
									class="w-full accent-cyan-300"
									bind:value={textSize}
								/>
							</div>
						</div>
					</div>
				</div>

				{#if isLoading}
					<p class="text-sm text-cyan-300">Loading image...</p>
				{/if}

				{#if errorMessage}
					<p
						class="rounded-md border border-rose-300/50 bg-rose-400/15 px-3 py-2 text-sm text-rose-100"
					>
						{errorMessage}
					</p>
				{/if}
			</aside>
		</div>
	</section>
</main>
