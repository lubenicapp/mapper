<script>

		import {graphMock} from "./lib/graphData.js";
		import Card from './lib/Card.svelte'

    let status = 'STOPPED'
    let graph = {}
    let lastLink = ''
    let searchTerm = ''

    export class Mapper {
        constructor(root) {
            this.root = root;
            this.graph = {};
            this.seen = new Set();
            this.status = 'STOPPED'
        }

        async map() {
            this.status = 'RUNNING';
            status = this.status;
            const toVisit = [this.root];
						let first = true
            while (toVisit.length > 0) {
                const page = toVisit.shift();
                lastLink = page
                if (this.seen.has(page)) {
                    continue;
                }

                let links = []
                if (first){
		                links = await this.fetchLinks(page);
                    first = false
                }
                else{
                    links = await this.fetchLinksNoNav(page)
                }
                const cleanedLinks = links.map(link => link.split('#')[0]).map(link => link.split('?')[0]);
                const internalLinks = cleanedLinks.filter(link => this.sameDomain(this.root, link));


                if (!this.graph[page]) {
                    this.graph[page] = [];
                }
                internalLinks.forEach(link => {
                    this.graph[page].push(link);
                    if (!this.seen.has(link)) {
                        toVisit.push(link);
                    }
                });

                this.seen.add(page);
                console.log(page);
            }

            this.cleanGraph();
            console.log("Finished mapping", this.graph);
            this.status = 'DONE';
            status = this.status;
        }

        async fetchLinks(url) {
            try {
                const response = await fetch(url);
                if (response.status !== 200) {
                    console.error(`Could not fetch page at ${url}`);
                    return [];
                }

                const text = await response.text();
                const parser = new DOMParser();
                const doc = parser.parseFromString(text, 'text/html');

                const allLinks = Array.from(doc.querySelectorAll('a[href]'));
                const fullUrls = allLinks.map(link => new URL(link.getAttribute('href'), url).href);

                return fullUrls;
            } catch (error) {
                console.error(`Error fetching links from ${url}:`, error);
                return [];
            }
        }

        async fetchLinksNoNav(url) {
            try {
                const response = await fetch(url);
                if (response.status !== 200) {
                    console.error(`Could not fetch page at ${url}`);
                    return [];
                }

                const text = await response.text();
                const parser = new DOMParser();
                const doc = parser.parseFromString(text, 'text/html');

                // Get all the links in the document
                const allLinks = Array.from(doc.querySelectorAll('a[href]'));

                // Get links inside the div with role="banner" and footer
                const bannerLinks = Array.from(doc.querySelectorAll('div[role="banner"] a[href]'));
                const footerLinks = Array.from(doc.querySelectorAll('footer a[href]'));

                // Combine the banner and footer links to exclude them
                const excludedLinks = new Set([...bannerLinks, ...footerLinks]);

                // Filter out the links that are inside banner and footer
                const filteredLinks = allLinks.filter(link => !excludedLinks.has(link));

                // Convert the remaining links to full URLs
                const fullUrls = filteredLinks.map(link => new URL(link.getAttribute('href'), url).href);

                return fullUrls;
            } catch (error) {
                console.error(`Error fetching links from ${url}:`, error);
                return [];
            }
        }

        cleanGraph() {
            const cleanedGraph = {};

            for (const [key, values] of Object.entries(this.graph)) {
                const cleanedKey = key.replace(this.root, '');
                const cleanedValues = values.map(link => link.replace(this.root, ''));
                cleanedGraph[cleanedKey] = cleanedValues;
            }

            this.graph = cleanedGraph;
            graph = this.graph
        }

        sameDomain(baseUrl, targetUrl) {
            return new URL(baseUrl).origin === new URL(targetUrl).origin;
        }

        sleep(ms) {
            return new Promise(resolve => setTimeout(resolve, ms));
        }
    }

    const root = 'https://www.wobee.fr';
    const mapper = new Mapper(root);

		let backLinks = []
		let highlight = '#'


</script>

<div class="container">
	{#if status == 'RUNNING'}
		<p>Mapping in progress...</p>
		<p>{lastLink}</p>
	{:else if status === 'DONE'}
		<input class="input" type="text" placeholder="search ex:blog" bind:value={searchTerm} />

		<button class="button"> A-Z </button>
		<button class="button"> 1-9 </button>

	{:else if status == 'STOPPED'}
		<input class="input" type="text" bind:value={mapper.root}>
		<button class='button' on:click={() => {mapper.map()}}>Start Mapping</button>
	{/if}

</div>

<div class="columns">
	<div class="column is-10">
		<div class="grid-container">
			{#each Object.entries(graph) as [key, value]}
				{#if searchTerm == '' || key.toLowerCase().includes(searchTerm.toLowerCase())}
					<Card
						url={key}
						backlink_count={[...new Set(value)].length}
						on:click={()=>{backLinks = [...new Set(value)]; highlight = key}}
						colored={highlight==key}
					/>
				{/if}
			{/each}
		</div>
	</div>
	<div class="column is-2 sticky-list">
		<ul>
			{#each backLinks as link}
				<li class="mb-1"> { link } </li>
			{/each}
		</ul>
	</div>
</div>




<style>
    .grid-container {
        display: grid;
        grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
        gap: 1rem;
        padding: 1rem;
    }

    .input {
		    width: 450px;
    }

    .sticky-list {
        position: sticky;
        top: 0;
        max-height: 100vh;
        overflow-y: auto;
        padding: 1rem;
        background-color: white;
        border-left: 1px solid #ddd;
    }
</style>
