<script lang="ts">
	let video: HTMLVideoElement;
	let canvas: HTMLCanvasElement;
	let stream: MediaStream | null = null;
	let photo = $state<string | null>(null);
	let error = $state<string | null>(null);

	async function start() {
		try {
			stream = await navigator.mediaDevices.getUserMedia({
				video: { facingMode: 'environment' },
				audio: false
			});
			video.srcObject = stream;
		} catch (err) {
			error = err instanceof Error ? err.message : 'Could not access camera';
		}
	}

	function snap() {
		canvas.width = video.videoWidth;
		canvas.height = video.videoHeight;
		canvas.getContext('2d')?.drawImage(video, 0, 0);
		photo = canvas.toDataURL('image/jpeg');
	}

	function stop() {
		stream?.getTracks().forEach((track) => track.stop());
		stream = null;
	}

	$effect(() => stop);
</script>

<div class="flex flex-col gap-3">
	<video bind:this={video} autoplay playsinline class="max-w-sm rounded bg-black"
		><track kind="captions" /></video
	>
	<canvas bind:this={canvas} class="hidden"></canvas>

	<div class="flex gap-2">
		<button onclick={start} class="rounded bg-blue-600 px-3 py-1 text-white">Start</button>
		<button onclick={snap} class="rounded bg-green-600 px-3 py-1 text-white">Capture</button>
		<button onclick={stop} class="rounded bg-gray-600 px-3 py-1 text-white">Stop</button>
	</div>

	{#if error}<p class="text-red-500">{error}</p>{/if}
	{#if photo}<img src={photo} alt="Captured" class="max-w-sm rounded" />{/if}
</div>
