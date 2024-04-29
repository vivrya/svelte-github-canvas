

<!-- App.svelte -->
<script>
    import { writable } from 'svelte/store';
    import axios from 'axios';
    import Fabric from 'fabric';
	import { onMount } from 'svelte';
    const members = writable([]);
    let searchText = '';
    let canvas;

    onMount(async () => {
        const response = await axios.get('https://api.github.com/orgs/mozilla/members?page=1');
        members.set(response.data);

        // Initialize Fabric.js canvas
        const width = window.innerWidth * 0.8; // 80% of window width
        const height = window.innerHeight * 0.8; // 80% of window height
        canvas = new Fabric.Canvas('canvas', { width, height });

        // Render name cards for each member
        updateCanvas();
    });

    $: filteredMembers = $members.filter(member => member.login.toLowerCase().includes(searchText.toLowerCase()));

    let isDragging = false;
    let offsetX = 0;
    let offsetY = 0;
    let originalPositions = {};

    const updateCanvas = () => {
        canvas.clear();
        originalPositions = {};

        filteredMembers.forEach((member, index) => {
            const text = new Fabric.Text(member.login, { left: 50, top: 50 + index * 30 });
            text.githubUrl = member.html_url;
            canvas.add(text);
            originalPositions[member.login] = { left: text.left, top: text.top };
        });
    };

    canvas?.on('mouse:down', function(options) {
        if (options.target) {
            isDragging = true;
            const { left, top } = options.target.getCenterPoint();
            offsetX = options.e.clientX - left;
            offsetY = options.e.clientY - top;
        }
    });

    canvas?.on('mouse:move', function(options) {
        if (isDragging && options.target) {
            options.target.set({
                left: options.e.clientX - offsetX,
                top: options.e.clientY - offsetY
            });
            canvas.renderAll();
        }
    });

    canvas?.on('mouse:up', function(options) {
        if (isDragging) {
            isDragging = false;
            const { left, top } = options.target;
            originalPositions[options.target.text] = { left, top };
        }
    });

    // Reset card positions when search text changes
    $: {
        if (!searchText) {
            Object.keys(originalPositions).forEach(text => {
                const { left, top } = originalPositions[text];
                const card = canvas.getObjects().find(obj => obj.text === text);
                if (card) {
                    card.set({ left, top });
                    canvas.renderAll();
                }
            });
        }
    }

    // Make the canvas responsive
    window.addEventListener('resize', () => {
        const width = window.innerWidth * 0.8; // 80% of window width
        const height = window.innerHeight * 0.8; // 80% of window height
        canvas.setDimensions({ width, height });
        canvas.renderAll();
    });
</script>

<style>
    /* Global styles */
    body {
        font-family: 'Arial', sans-serif;
        margin: 0;
        padding: 0;
        background-color: #f8f9fa;
    }

    /* Header styles */
    h1 {
        text-align: center;
        padding: 20px 0;
        background-color: #0366d6;
        color: white;
        margin: 0;
    }

    /* Search input styles */
    input[type="text"] {
        padding: 10px;
        font-size: 16px;
        border: 1px solid #ccc;
        border-radius: 5px;
        width: 80%;
        max-width: 400px;
        margin: 20px auto;
        display: block;
    }

    /* Member card styles */
    .member-card {
        display: flex;
        align-items: center;
        margin: 10px auto;
        padding: 10px;
        background-color: #fff;
        border-radius: 5px;
        box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
        max-width: 400px;
    }

    .member-card img {
        border-radius: 50%;
        margin-right: 10px;
    }

    .member-card a {
        text-decoration: none;
        color: #0366d6;
        font-weight: bold;
    }

    /* Canvas styles */
    canvas {
        display: block;
        margin: 20px auto;
        border: 1px solid #ccc;
        border-radius: 5px;
        box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
        max-width: 80%;
        max-height: 400px;
    }
</style>


<h1>Svelte GitHub Canvas</h1>

<input type="text" bind:value={searchText} placeholder="Search members..." />

{#each filteredMembers as member}
    <div class="member-card">
        <img src={member.avatar_url} alt={member.login} width="50" height="50" />
        <a href={member.html_url} target="_blank">{member.login}</a>
    </div>
{/each}

<canvas id="canvas"></canvas>


