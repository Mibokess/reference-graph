<div class="w-screen h-screen">
  {#if !subscriptionKey}
    <div class="flex justify-center w-full pt-16">
      <div class="max-w-xs md:max-w-md lg:max-w-xl flex-grow flex-shrink-0">
        <input placeholder="Enter subscription key" type="input" on:keyup={setSubscriptionKey} id="site-search" aria-label="Search for papers" class="focus:outline-0 border border-transparent focus:bg-white focus:border-gray-300 placeholder-gray-600 rounded-lg bg-gray-300 py-2 px-4 block w-full appearance-none leading-normal ds-input">
        <p>You can generate a new key <a class="text-blue-600" href="https://msr-apis.portal.azure-api.net/products/project-academic-knowledge">here.</a></p>
      </div>
    </div>
  {:else}
    <div>
      {#if !clickedPaper}
        <div class="flex justify-center w-full pt-16">
          <h1 class="text-3xl font-light">Paper Graph</h1>
        </div>
      {/if}
      <div class="w-full h-full">
        <div class="flex justify-center {!clickedPaper ? 'mt-0' : 'mt-8'}">
          <Search {subscriptionKey} on:clickedPaper={createGraph}/>
        </div>
        <div class="top-0 w-full h-full z-0">
          <Graph {paperId} {subscriptionKey} maxNumberOfRequests={!clickedPaper ? 20 : 100}/>
        </div>
      </div>
    </div>
  {/if}
</div>

<script>
  import { onMount } from 'svelte';

  import Search from './Search.svelte';
  import Graph from './Graph.svelte';
  
  let clickedPaper = false;
  let paperId = 40134741;
  let subscriptionKey;

  onMount(() => {
    subscriptionKey = getSubscriptionKey();
  });

  function createGraph(event) {
    clickedPaper = true;
    paperId = event.detail.text;
  }

  function getSubscriptionKey() {
    return localStorage.getItem('subscriptionKey');
  }

  function setSubscriptionKey(event) {
    if (event.key == "Enter" && event.target.value) {
      subscriptionKey = event.target.value;
      localStorage.setItem('subscriptionKey', subscriptionKey);
    }
  }
</script>