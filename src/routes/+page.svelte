<script lang="ts">
	let sharedUrl: string = $state('');
	let googleMapsUrl: string = $state('');

	type LatLng = {
		lat: number;
		lng: number;
	};

	const validateGoogleMapUrl = (url: string): boolean => {
		const regex =
			/^https:\/\/www\.google\.com\/maps\/d\/u\/\d+\/edit\?mid=[a-zA-Z0-9_-]+(&usp=sharing)?$/;
		return regex.test(url);
	};

	const getGoogleMapMid = (url: string): string | null => {
		const { searchParams } = new URL(url);
		const mid = searchParams.get('mid');
		return mid;
	};

	const toKmlUrl = (mid: string) => `https://www.google.com/maps/d/kml?mid=${mid}&forcekml=1`;

	const fetchKmlFile = async (url: string): Promise<string> => {
		const response = await fetch(url);
		if (!response.ok) {
			throw new Error(`Failed to fetch KML file: ${response.statusText}`);
		}
		return response.text();
	};

	const parseKML = (kmlString: string) => {
		const parser = new DOMParser();
		const xmlDoc = parser.parseFromString(kmlString, 'application/xml');
		return xmlDoc;
	};

	const getLocation = (xmlDoc: Document) => {
		const placeMarks = xmlDoc.getElementsByTagName('Placemark');
		const locations: LatLng[] = Array.from(placeMarks).map((placeMark) => {
			const coords = placeMark.getElementsByTagName('coordinates')[0]?.textContent || '';
			const [lng, lat] = coords.split(',').map(Number);
			return { lat, lng };
		});
		return locations;
	};

	const makeGoogleMapsUrl = (points: LatLng[]) => {
		const baseUrl = 'https://www.google.com/maps/dir';
		const waypoints = points.map((point) => `${point.lat},${point.lng}`).join('/');
		return `${baseUrl}/${waypoints}`;
	};

	const run = async () => {
		if (!sharedUrl) {
			throw new Error('URL is required');
		}

		// const ok = validateGoogleMapUrl(sharedUrl);
		// if (!ok) {
		// 	throw new Error('Invalid Google Maps URL');
		// }

		const mid = getGoogleMapMid(sharedUrl);
		if (!mid) {
			throw new Error('No mid found in the Google Maps URL');
		}

		const kmlUrl = toKmlUrl(mid);
		const kml = await fetchKmlFile(kmlUrl);
		const kmlData = parseKML(kml);
		const points: LatLng[] = getLocation(kmlData);
		googleMapsUrl = makeGoogleMapsUrl(points);

		setTimeout(() => {
			window.location.href = googleMapsUrl;
		}, 3000);
	};

	const handleClick = async (e: Event) => {
		e.preventDefault();

		try {
			await run();
		} catch (error) {
			alert(error instanceof Error ? error.message : 'An unexpected error occurred');
		}
	};
</script>

<main class="flex h-screen flex-col items-center justify-center gap-4 p-4">
	<h1 class="text-2xl font-bold">Google マイマップを Google マップに変換</h1>
	<p class="text-gray-600">
		Google マイマップの共有URLを入力すると、Google マップのルートURLに変換します。
	</p>

	<form onsubmit={handleClick} class="w-full max-w-md space-y-4">
		<input
			type="text"
			bind:value={sharedUrl}
			placeholder="Google マイマップの共有URL"
			class="w-full rounded-md border border-gray-300 p-2 focus:border-blue-500 focus:outline-none"
			required
		/>
		<button
			type="submit"
			class="w-full rounded-md bg-blue-500 p-2 text-white transition-colors hover:bg-blue-600"
		>
			実行
		</button>
	</form>

	{#if googleMapsUrl}
		<a
			href={googleMapsUrl}
			target="_blank"
			rel="noopener noreferrer"
			class="w-full max-w-md rounded-md bg-blue-100 p-2 break-words text-blue-700 hover:bg-blue-200"
		>
			{googleMapsUrl}
		</a>
	{/if}
</main>
