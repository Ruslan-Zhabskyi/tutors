<script lang="ts">
  import Heatmap from './Heatmap.svelte';
  import { supabase } from '$lib/db';
  import { writable } from 'svelte/store';
  import { onMount } from 'svelte';
  
  let responses = writable<any[]>([]);
  let isLoading = writable(true);
  
  async function fetchResponses() {
    const { data: GenAiResponses, error } = await supabase
      .from('GenAiResponses')
        .select('*')
        .eq('helpful', true);

    if (error) {
      console.error('Error fetching responses:', error.message);
      return;
    }

    if (GenAiResponses) {
      // Ensure the data is in JSON format
      const jsonData = JSON.parse(JSON.stringify(GenAiResponses));
      responses.set(jsonData); // Store the parsed JSON data
      console.log('Fetched responses in JSON format:', jsonData);
    }

    isLoading.set(false);
  }

onMount(() => {
  fetchResponses();
});
</script>

<div class="card to-accent-50 dark:to-accent-900 from-primary-50 dark:from-primary-900 m-4 border bg-gradient-to-l">
  <div class="container mx-auto mt-2 items-center justify-between lg:flex">
    <div class="mx-4">
      <main class="container mx-auto p-4">
        <h1 class="h1 mb-4">LLM Response Heatmap</h1>
        <p class="mb-8">Click on a cell to see detailed responses for that URL</p>
        {#if $isLoading}
          <p>Loading responses...</p>
        {:else}
          <Heatmap data={$responses} />
        {/if}
      </main>
    </div>
  </div>
</div>