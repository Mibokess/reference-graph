<div class="h-screen w-screen content-center">
  <div class="flex justify-center w-full pt-16">
    <h1 class="text-3xl font-light">Paper Graph</h1>
  </div>
  <div class="flex justify-center h-full w-full mt-8 md:mt-12 lg:mt-16">
    {#if clickedPaper}
      <Search on:clickedPaper={createGraph}/>
    {:else}
      <Graph originalPaperId={originalPaperId}/>
    {/if}
  </div>
</div>

<script>
  import Search from './Search.svelte';
  import Graph from './Graph.svelte';
  
  let clickedPaper = false;
  let originalPaperId;

  function createGraph(event) {
    console.log(event.detail.text);
    clickedPaper = true;
    originalPaperId = event.detail.text;
  }
</script>