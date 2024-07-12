<!-- ExamplePage.svelte -->
<script>
    let resolution = '720p';
  
    function changeResolution(newResolution) {
      resolution = newResolution;
      // Additional logic can be added here if needed
    }
  </script>
  
  <style>
    /* Styles can be added here if needed */
  </style>
  
  <div class="resolution-container">
    <p>Current Resolution: {resolution}</p>
    <button on:click={() => changeResolution('720p')}>720p</button>
    <button on:click={() => changeResolution('1080p')}>1080p</button>
    <button on:click={() => changeResolution('4k')}>4k</button>
  </div>
  