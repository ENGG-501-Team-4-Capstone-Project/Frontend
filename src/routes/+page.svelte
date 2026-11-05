<script lang="ts">
	type Confidence = 'high' | 'low';
	interface Field {
		id: string;
		label: string;
		value: string;
		units: string;
		confidence: Confidence;
	}

	const rigImages = [
		{ src: '/1.jpg', label: '1.jpg (good)' },
		{ src: '/2.jpg', label: '2.jpg (bad)' }
	];

	let result = $state('');
	let sendResult = $state('');
	let loading = $state(false);
	let sending = $state(false);
	let fields = $state<Field[]>([]);
	let selectedImage = $state(rigImages[0].src);

	const confirmed = $derived({
		fields: fields.map((field) => ({
			id: field.id,
			label: field.label.trim(),
			value: field.value.trim() === '' ? null : field.value.trim(),
			units: field.units.trim() === '' ? null : field.units.trim(),
			confidence: field.confidence
		}))
	});

	function pretty(text: string): string {
		try {
			return JSON.stringify(JSON.parse(text), null, 2);
		} catch {
			return text;
		}
	}

	async function toBase64(url: string): Promise<string> {
		const blob = await (await fetch(url)).blob();
		return new Promise((resolve, reject) => {
			const reader = new FileReader();
			reader.onload = () => resolve((reader.result as string).split(',')[1]);
			reader.onerror = reject;
			reader.readAsDataURL(blob);
		});
	}

	async function readTag() {
		loading = true;
		result = '';
		sendResult = '';
		fields = [];
		try {
			const image = await toBase64('/tag.jpg');
			const response = await fetch('http://localhost:3000/read-tag', {
				method: 'POST',
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify({ image })
			});
			const text = await response.text();
			result = `${response.status} ${response.statusText}\n\n${pretty(text)}`;

			try {
				const data = JSON.parse(text);
				if (response.ok && Array.isArray(data.fields)) {
					fields = data.fields.map((field: any) => ({
						id: field.id,
						label: field.label ?? '',
						value: field.value ?? '',
						units: field.units ?? '',
						confidence: field.confidence === 'high' ? 'high' : 'low'
					}));
				}
			} catch {}
		} catch (e) {
			result = `Request failed: ${e}`;
		} finally {
			loading = false;
		}
	}

	async function sendConfirmed() {
		sending = true;
		sendResult = '';
		try {
			const image = await toBase64(selectedImage);
			const tag = await toBase64('/tag.jpg');
			const response = await fetch('http://localhost:3000/evaluate-rig', {
				method: 'POST',
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify({
					tagInformation: confirmed,
					images: [image, tag],
					rigType: 'WIRE_ROPE_SLING'
				})
			});
			sendResult = `${response.status} ${response.statusText}\n\n${pretty(await response.text())}`;
		} catch (e) {
			sendResult = `Request failed: ${e}`;
		} finally {
			sending = false;
		}
	}
</script>

<div class="p-8">
	<img class="h-xl w-lg" src="/tag.jpg" alt="tag" />
	<br />
	<button class="cursor-pointer border-2" onclick={readTag} disabled={loading}>Read tag</button>

	{#if fields.length}
		<h3>Fields</h3>
		<table class="border-2">
			<thead>
				<tr><th>label</th><th>value</th><th>units</th></tr>
			</thead>
			<tbody>
				{#each fields as field}
					<tr>
						<td><input bind:value={field.label} /></td>
						<td><input bind:value={field.value} /></td>
						<td><input bind:value={field.units} /></td>
					</tr>
				{/each}
			</tbody>
		</table>

		<h3>Rig image to send</h3>
		{#each rigImages as img}
			<label>
				<input type="radio" bind:group={selectedImage} value={img.src} />
				{img.label}
				<br />
				<img class="h-xl w-lg" src={img.src} alt={img.label} />
			</label>
		{/each}
		<br />

		<button class="cursor-pointer border-2" onclick={sendConfirmed} disabled={sending}>
			Send confirmed fields
		</button>

		<h3>Request tag information</h3>
		<pre>{JSON.stringify(confirmed, null, 2)}</pre>

		<h3>EVALUATION</h3>
		<pre>{sendResult}</pre>
	{/if}
</div>
