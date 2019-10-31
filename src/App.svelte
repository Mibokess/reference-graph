<div>
  {#if clickedPaper}
    <div class="flex justify-center w-full pt-16">
      <h1 class="text-3xl font-light">Paper Graph</h1>
    </div>
    <div class="flex justify-center h-full w-full mt-8 md:mt-12 lg:mt-16">
      <Search subscriptionKey={subscriptionKey} on:clickedPaper={createGraph}/>
    </div>
  {:else}
    <div>  
      <Graph originalPaperId={originalPaperId} subscriptionKey={subscriptionKey}/>
    </div>
  {/if}
</div>

<script>
  import Search from './Search.svelte';
  import Graph from './Graph.svelte';
  
  let clickedPaper = false;
  let originalPaperId = 40134741;
  let subscriptionKey = "554c1cf34356401ab6fc1dd31982f3ce";

  function createGraph(event) {
    console.log(event.detail.text);
    clickedPaper = true;
    originalPaperId = event.detail.text;
  }
</script>