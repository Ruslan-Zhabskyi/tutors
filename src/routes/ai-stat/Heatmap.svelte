<script lang="ts">
  export let data: any[];

  let selectedUrl: string | null = null;
  let filteredResponses: any[] = [];

  // Get unique URLs and features
  const urls = [...new Set(data.map(d => d.contentUrl))];
  const features = [...new Set(data.map(d => d.feature))];

  // Calculate response counts for each cell
  const heatmapData = urls.flatMap(url => {
    return features.map(feature => {
      const responses = data.filter(d => d.contentUrl === url && d.feature === feature);
      return {
        url,
        feature,
        totalResponses: responses.length,
        intensity: responses.length === 0 ? 0 : Math.min(responses.length / 2, 1)
      };
    });
  });

  $: if (selectedUrl) {
    filteredResponses = data.filter(d => d.contentUrl === selectedUrl && d.helpful === true);
  }

  function handleCellClick(url: string) {
    selectedUrl = url;
  }

function getShortUrl(url: string): string {
  try {
    const parsedUrl = new URL(url);
    return parsedUrl.pathname;
  } catch (error) {
    console.error('Invalid URL:', url);
    return url;
  }
}

  function getHeatColor(count: number): string {
    const maxIntensity = Math.max(...heatmapData.map(d => d.totalResponses));
    const intensity = count === 0 ? 0 : (count / maxIntensity);
    return `rgb(62 184 255 / ${intensity})`;
  }
</script>

<div class="space-y-8">
  <div class="card p-4 overflow-x-auto">
    <table class="table table-compact">
      <thead>
        <tr>
          <th></th>
          {#each features as feature}
            <th class="text-center whitespace-nowrap">{feature}</th>
          {/each}
        </tr>
      </thead>
      <tbody>
        {#each urls as url}
          <tr>
            <th class="whitespace-nowrap text-right" title={url}>{getShortUrl(url)}</th>
            {#each features as feature}
              {@const cell = heatmapData.find(d => d.url === url && d.feature === feature)}
              <td 
                class="text-center cursor-pointer transition-colors hover:brightness-90"
                style="background-color: {getHeatColor(cell?.totalResponses || 0)};"
                on:click={() => handleCellClick(url)}
                title="Total Responses: {cell?.totalResponses || 0}"
              >
                {cell?.totalResponses || 0}
              </td>
            {/each}
          </tr>
        {/each}
      </tbody>
    </table>
  </div>

  {#if selectedUrl && filteredResponses.length > 0}
    <div class="card p-4">
      <h3 class="h3 mb-4">Helpful Responses for {getShortUrl(selectedUrl)}</h3>
      <div class="space-y-4">
        {#each filteredResponses as response}
          <div class="card p-4 variant-filled">
            <div class="space-y-2">
              <p><span class="font-bold">Feature:</span> {response.feature}</p>
              <p><span class="font-bold">LLM:</span> {response.llmUsed}</p>
              <p><span class="font-bold">User Message:</span> {response.userMessage}</p>
              <p><span class="font-bold">Response:</span> {response.content}</p>
            </div>
          </div>
        {/each}
      </div>
    </div>
      <div class="flex justify-center mt-4">
        <button class="btn variant-filled-primary" on:click={() => alert(

          `Selected URL: ${selectedUrl}\n\n` +
          filteredResponses.map(r => 

            `User Message: ${r.userMessage}\n` +
            `Response: ${r.content}\n\n`
          ).join('\n')
 
        )}>
          Alert!
        </button>
      </div>
  {/if}

</div>