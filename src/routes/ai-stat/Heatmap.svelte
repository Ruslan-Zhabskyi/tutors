<script lang="ts">
  import { writable } from "svelte/store";
  import {supabase} from '$lib/db';
  export let data: any[];

  let selectedUrl: string | null = null;
  let filteredResponses: any[] = [];
  
// let pageContent: string = '';

// $: if (selectedUrl) {
//   filteredResponses = data.filter(d => d.contentUrl === selectedUrl && d.helpful === true);
//   fetchPageContent(selectedUrl); 
// }

// async function fetchPageContent(url: string) {
//   try {
//     console.log('Fetching content from URL:', url); // Log the URL
//     const response = await fetch(url);
//     if (!response.ok) throw new Error(`Error: ${response.statusText}`);
//     pageContent = await response.text();
//   } catch (error) {
//     console.error("Failed to load content:", error);
//     pageContent = "Error loading content.";
//   }
// }
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


  // LLM call
  let project_id: string = '68f58c24-1633-429d-bb39-cb0947f86d02';
  let model_id: string = 'ibm/granite-3-8b-instruct';

      interface Message {
    role: 'user' | 'assistant' | 'system';
    userMessage?: string;
    content: string;
    responseId?: number;
    responseDate?: string;
    contentUrl?: string;
    llmUsed?: string;
    feature?: string;
    helpful?: boolean;
  }

    let llmResponse = writable<Message>({
    role: 'assistant',
    userMessage: undefined,
    content: '',
    responseId: undefined,
    responseDate: undefined,
    contentUrl: undefined,
    llmUsed: undefined,
    feature: undefined,
    helpful: undefined
  });

  let isLoading = writable(false);
  async function sendMessage(responses:string): Promise<Message> {
    const userMessage = responses.trim();
    isLoading.set(true);

    try {
      const response = await fetch('/api/summarise-search-background', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          model_id: model_id,
          project_id: project_id,
          prompt: `Create page for tutors website based on responces to student questions: "${userMessage}". I want it to be not question - answer, but as seamless article`,
        }),
      });

      const result = await response.json();
      const llmOutput = result?.results?.[0]?.generated_text || "No results found.";

      // Log the result to the console
      console.log("API Response:", llmOutput);
      
    // Save the response to the database

    const { data, error } = await supabase
    .from('GenAiResponses')
    .insert(
      {   role: 'assistant',
          userMessage: userMessage,
          content:llmOutput,
          contentUrl: window.location.href,
          llmUsed: model_id,
          feature: 'Tutors AI for Content Creators',
          helpful: false,
        },
    )
    .select()

    if (error) {
  console.error("Error inserting data:", error.message); 
} else {
  console.log("Data inserted:", data);
}
       console.log("Data inserted:", data);
       const responseId = data?.[0]?.responseId; 
       console.log("Extracted responseId:", responseId);
       const responseDate = data?.[0]?.responseDate; 
       
    const llmMessage: Message = {
          role: 'assistant',
          userMessage: userMessage,
          content:llmOutput,
          responseId: responseId,
          responseDate: responseDate,
          contentUrl: window.location.href,
          llmUsed: model_id,
          feature: 'Tutors AI for Content Creators',
          helpful: false,
        };

      console.log("llmMessage:", llmMessage);  
      llmResponse.set(llmMessage);
      return llmMessage;


    } catch (error) {
      console.error('Error:', error);
          return {
        role: 'assistant',
        content: "An error occurred while fetching data.",
        responseId: undefined,
        responseDate: new Date().toISOString(),
        contentUrl: window.location.href,
        llmUsed: model_id,
        helpful: false,
      };
    } finally {
      isLoading.set(false);
    }
  }

  // async function copyText(textToCopy: any) {
  //   try {
  //     await navigator.clipboard.writeText(textToCopy);
  //   } catch (err) {
  //     console.error('Failed to copy text:', err);
  //   }
  //   showEli5Button.set(false);
  // }

async function updateMessageHelpful(responseId: string, helpful: boolean) {
  const { data, error } = await supabase
    .from('GenAiResponses')
    .update({ helpful }) 
    .eq('responseId', responseId)
    .select();

  if (error) {
    console.error("Error updating helpful status:", error);
  } else {
    console.log("Update successful:", data);
  }
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
 <button
  class="btn variant-filled-primary"
  on:click={() =>
    sendMessage(
      `Selected URL: ${selectedUrl}\n\n` +
      // `Page Content:\n${pageContent}\n\n` + 
      filteredResponses
        .map((r) => {
          if (r.feature === 'eli5') {
            return (
              `Uncleartext: ${r.userMessage}\n` +
              `Liked LLM Response: ${r.content}\n\n`
            );
          } else {
            return (
              `User Question: ${r.userMessage}\n` +
              `Liked LLM Response: ${r.content}\n\n`
            );
          }
        })
        .join('\n')
    )
  }
>
  Alert!
</button>
      </div>
  {/if}

</div>