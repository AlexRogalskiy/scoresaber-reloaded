<script>
  import {navigate} from "svelte-routing";
  import {fade} from 'svelte/transition'
  import ssrConfig from '../ssr-config'

  document.body.classList.add('slim');
</script>

<svelte:head>
  <title>Friends - {ssrConfig.name}</title>
</svelte:head>

<article transition:fade>
  <div class="box has-shadow">
    <h1 class="title is-3">Not implemented yet :-(</h1>
    <p>Come back later.</p>

    <p><a href="" on:click|preventDefault={() => navigate('/')}>Back to Home</a></p>
  </div>
</article>

<style>
    article {
        text-align: center;
    }

    p {
        text-align: center;
    }
</style>