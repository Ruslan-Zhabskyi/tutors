<script lang="ts">
  import { writable } from "svelte/store";
  import { supabase } from "$lib/db";
  import { marked } from "marked";
  export let data: any[];

  let selectedUrl: string | null = null;
  let filteredResponses: any[] = [];

  // Get unique URLs and features
  const urls = [...new Set(data.map((d) => d.contentUrl))];
  const features = [...new Set(data.map((d) => d.feature))];

  // Calculate response counts for each cell
  const heatmapData = urls.flatMap((url) => {
    return features.map((feature) => {
      const responses = data.filter((d) => d.contentUrl === url && d.feature === feature);
      return {
        url,
        feature,
        totalResponses: responses.length,
        intensity: responses.length === 0 ? 0 : Math.min(responses.length / 2, 1)
      };
    });
  });

  $: if (selectedUrl) {
    filteredResponses = data
      .filter((d) => d.contentUrl === selectedUrl && d.helpful === true)
      .map((d) => ({ ...d, selected: false })); // Add `selected` property
  }

  function handleCellClick(url: string) {
    selectedUrl = url;
  }

  function getShortUrl(url: string): string {
    try {
      const parsedUrl = new URL(url);
      return parsedUrl.pathname;
    } catch (error) {
      console.error("Invalid URL:", url);
      return url;
    }
  }

  function getTopicUrl(selectedUrl: string): string {
    try {
      const parsedUrl = new URL(selectedUrl);
      const segments = parsedUrl.pathname.split("/").filter(Boolean);
      if (segments[0] === "lab") {
        segments[0] = "topic";
      }
      const topicPath = segments.slice(0, 3).join("/");
      return `${parsedUrl.origin}/${topicPath}`;
    } catch (error) {
      console.error("Invalid URL:", selectedUrl);
      return selectedUrl;
    }
  }

  function getHeatColor(count: number): string {
    const maxIntensity = Math.max(...heatmapData.map((d) => d.totalResponses));
    const intensity = count === 0 ? 0 : count / maxIntensity;
    return `rgb(62 184 255 / ${intensity})`;
  }

  // LLM call
  let project_id: string = "68f58c24-1633-429d-bb39-cb0947f86d02";
  let model_id: string = "ibm/granite-3-8b-instruct";

  interface Message {
    refinedContentId: number | undefined;
    dateGenerated?: string;
    contentUrl?: string;
    topicUrl?: string;
    responseIds: string[];
    llmUsed?: string;
    generatedText?: string;
    helpful?: boolean;
  }

  let llmResponse = writable<Message>({
    refinedContentId: undefined,
    dateGenerated: undefined,
    contentUrl: undefined,
    topicUrl: undefined,
    responseIds: [],
    llmUsed: model_id,
    generatedText: "",
    helpful: undefined
  });

  let isLoading = writable(false);

  async function sendMessage(
    responses: string,
    selectedUrl: string,
    responseIds: string[],
    pageContent: string
  ): Promise<Message> {
    const userMessage = responses.trim();
    console.log("Selected responces for LLM analysis:", userMessage);
    isLoading.set(true);
    console.log("Page content:", pageContent);
    try {
      const response = await fetch("/api/summarise-search-background", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          model_id: model_id,
          project_id: project_id,
          prompt: `Here is the content of a lab that student was following: ${pageContent}. There was some text that student did
          not understand. Your task is to make small adjustments based on students' feedback: ${userMessage}. 
          Instructions:
          1. Carefully check lab content and students' feedback. 
          2. Identify areas where the content can be improved.
          3. Make small adjustments to the content.
          4. Ensure that the content is still relevant to the lab.
          5. Do not expose students questions or feedback.`
        })
      });

      const result = await response.json();
      const llmOutput = result?.results?.[0]?.generated_text || "No results found.";

      // Log the result to the console
      console.log("API result:", result);
      console.log("API Response:", llmOutput);

      // Save the response to the database
      const topicUrl = getTopicUrl(selectedUrl);
      const { data, error } = await supabase
        .from("AiContentRefiner")
        .insert({
          contentUrl: selectedUrl,
          topicUrl: topicUrl,
          responseIds: responseIds,
          llmUsed: model_id,
          generatedText: llmOutput,
          helpful: false
        })
        .select();

      if (error) {
        console.error("Error inserting data:", error.message);
      } else {
        console.log("Data inserted:", data);
      }
      console.log("Data inserted:", data);
      const refinedContentId = data?.[0]?.refinedContentId;
      console.log("Extracted responseId:", refinedContentId);
      const dateGenerated = data?.[0]?.dateGenerated;

      const llmMessage: Message = {
        generatedText: llmOutput,
        refinedContentId: refinedContentId,
        dateGenerated: dateGenerated,
        contentUrl: selectedUrl,
        topicUrl: topicUrl,
        responseIds: responseIds,
        llmUsed: model_id,
        helpful: false
      };

      console.log("llmMessage:", llmMessage);
      llmResponse.set(llmMessage);
      return llmMessage;
    } catch (error) {
      console.error("Error:", error);
      return {
        generatedText: "An error occurred while fetching data.",
        refinedContentId: undefined,
        dateGenerated: new Date().toISOString(),
        contentUrl: selectedUrl,
        topicUrl: selectedUrl,
        responseIds: [],
        llmUsed: model_id,
        helpful: false
      };
    } finally {
      isLoading.set(false);
    }
  }

  async function copyText(textToCopy: any) {
    try {
      await navigator.clipboard.writeText(textToCopy);
    } catch (err) {
      console.error("Failed to copy text:", err);
    }
  }

  //modal
  let modalOpen = writable(false);
  let modalContent = writable("");

  async function openModal(responses: string, responseIds: string[], pageContent: string) {
    try {
      const response = await sendMessage(responses, selectedUrl ?? "", responseIds, pageContent);
      llmResponse.set(response);
      modalContent.set(response.generatedText || "");
      modalOpen.set(true);
    } catch (error) {
      console.error("Error opening modal:", error);
      alert("An error occurred while opening the modal. Please try again.");
    } finally {
      isLoading.set(false);
    }
  }

  function closeModal() {
    modalOpen.set(false);
  }
</script>

<div class="space-y-8">
  <div class="card overflow-x-auto p-4">
    <table class="table-compact table">
      <thead>
        <tr>
          <th></th>
          {#each features as feature}
            <th class="whitespace-nowrap text-center">{feature}</th>
          {/each}
        </tr>
      </thead>
      <tbody>
        {#each urls as url}
          <tr>
            <th class="whitespace-nowrap text-right" title={url}>{getShortUrl(url)}</th>
            {#each features as feature}
              {@const cell = heatmapData.find((d) => d.url === url && d.feature === feature)}
              <td
                class="cursor-pointer text-center transition-colors hover:brightness-90"
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
        {#each filteredResponses as response, index}
          <div class="variant-filled card flex items-start p-4">
            <input type="checkbox" bind:checked={response.selected} class="mr-4 mt-1" />
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

    <div class="mt-4 flex justify-center">
      {#if $isLoading}
        <button class="variant-filled-primary btn" disabled>
          <span class="spinner-border spinner-border-sm mr-2"></span>
          Generating Content...
        </button>
      {:else}
        <button
          class="btn rounded bg-gray-500 px-4 py-2 text-white"
          on:click={async () => {
            const selectedResponses = filteredResponses.filter((r) => r.selected);

            if (selectedResponses.length === 0) {
              alert("Please select at least one response.");
              return;
            }

            const responseIds = selectedResponses.map((r) => r.responseId);

            const pageContent = data.find((d) => d.contentUrl === selectedUrl)?.pageContent || "";

            const responsesString = selectedResponses
              .map((r) => `User Question: ${r.userMessage}\nLiked LLM Response: ${r.content}\n\n`)
              .join("\n");

            console.log("Sending responses to LLM:", responsesString);

            isLoading.set(true);
            try {
              await openModal(responsesString, responseIds, pageContent);
            } catch (error) {
              console.error("Error generating content:", error);
              alert("An error occurred while generating content. Please try again.");
              isLoading.set(false);
            }
          }}
        >
          Generate New Content
        </button>
      {/if}
    </div>
  {/if}

  {#if $modalOpen}
    <div
      class="modal modal-open fixed inset-0 flex items-center justify-center bg-black bg-opacity-50"
      on:click={closeModal}
    >
      <div class="modal-box max-h-[80vh] w-2/3 overflow-y-auto rounded bg-white p-6 shadow-lg" on:click|stopPropagation>
        <h3 class="mb-4 text-lg font-bold">Generated Content</h3>
        <div class="prose max-w-none">
          <!-- {@html $modalContent} -->
          {@html marked($modalContent)}
        </div>
        <div class="mt-4 flex justify-end space-x-2">
          <button class="btn-outline btn" on:click={() => copyText($modalContent)}>
            <i class="fa-solid fa-copy"></i>
          </button>
          <!-- <button 
          class="btn btn-primary"
          on:click={closeModal}
        >
          Close
        </button> -->
        </div>
      </div>
    </div>
  {/if}
</div>
