

<!-- App.svelte -->
<script>
import { onMount } from 'svelte';
	import { writable } from 'svelte/store';
    import axios from 'axios';
    import Fabric from 'fabric';

    const members = writable([]);
    let searchText = '';
    let canvas;

    onMount(async () => {
        const response = await axios.get('https://api.github.com/orgs/mozilla/members?page=1');
        members.set(response.data);

        // Initialize Fabric.js canvas
        canvas = new Fabric.Canvas('canvas', { width: 800, height: 600 });

        // Render name cards for each member
        $: {
            canvas.clear();
            filteredMembers.forEach((member, index) => {
                const text = new Fabric.Text(member.login, { left: 50, top: 50 + index * 30 });
                canvas.add(text);
            });
        }
    });

    $: filteredMembers = $members.filter(member => member.login.toLowerCase().includes(searchText.toLowerCase()));
</script>

<style>
    canvas {
        border: 1px solid #ccc;
        margin-top: 20px;
    }
</style>

<h1>Svelte GitHub Canvas</h1>

<input type="text" bind:value={searchText} placeholder="Search members..." />

{#each filteredMembers as member}
    <div>
        <img src={member.avatar_url} alt={member.login} width="50" height="50" />
        <a href={member.html_url} target="_blank">{member.login}</a>
    </div>
{/each}

