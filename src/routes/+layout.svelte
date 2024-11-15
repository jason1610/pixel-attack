<script lang="ts">
	import '../app.css';
	import { ModeWatcher } from 'mode-watcher';

	let { children } = $props();
</script>

<ModeWatcher defaultMode="dark" />
<div class="flex min-h-svh items-center justify-center p-4">
	{@render children()}
</div>
