<script>
	export let url;
	export let backlink_count;
	export let colored = false;

  function getUrlParts(url) {
      const cleanUrl = url.replace(/^\/|\/$/g, '');
      const parts = cleanUrl.split('/');
      const category = parts[0] || '/';
      const subcategory = url;
      return { category, subcategory };
  }

  const {category, subcategory} = getUrlParts(url);
</script>

<div class="card {colored? 'colored' : ''}" on:click>
	<div class="card-content">
		<div class="media-content mb-3">
			<p class="title is-6">{category}</p>
		</div>
		<div class="content">
			<p>{subcategory}</p>
			<p>{backlink_count} liens</p>
		</div>
	</div>
</div>


<style>
	.card {
			max-width: 250px;
	}

  .colored {
      border: 1px solid seagreen;
  }
</style>
