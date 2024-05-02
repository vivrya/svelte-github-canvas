<script>
  import { onMount, createEventDispatcher } from 'svelte';
  import { fabric } from 'fabric';

  let canvas;
  let members = []; // Store all members fetched from API
  let filteredMembers = []; // Store filtered members based on search input
  const dispatch = createEventDispatcher();

  // Function to filter members based on search input
  function filterMembers(query) {
    filteredMembers = members.filter(member =>
      member.login.toLowerCase().includes(query.toLowerCase())
    );
    renderMembers(filteredMembers);
  }

  // Function to render members on canvas
  function renderMembers(membersToRender) {
    canvas.clear(); // Clear canvas before rendering new members
    let topPosition = 10; // Initial top position
    membersToRender.forEach(member => {
      createCard(member, topPosition);
      topPosition += 120; // Adjust this value as needed for spacing between cards
    });
  }

  onMount(() => {
    const width = window.innerWidth * 0.8;
    const height = window.innerHeight * 0.8;
    canvas = new fabric.Canvas('canvas', { width, height });

    fetch('https://api.github.com/orgs/mozilla/members?page=1')
      .then(response => response.json())
      .then(data => {
        members = data; // Store fetched members
        filteredMembers = data; // Initially, display all members
        renderMembers(filteredMembers); // Render members on canvas
      })
      .catch(error => {
        console.error('Error fetching members:', error);
      });
  });

  function createCard(member, topPosition) {
    // Create a group for the card
    const cardGroup = new fabric.Group([], {
      left: 10,
      top: topPosition,
	  margin:"10px",
    });

    // Add avatar image
    fabric.Image.fromURL(member.avatar_url, function(img) {
      img.scaleToWidth(100);
      img.scaleToHeight(100);
      cardGroup.addWithUpdate(img);

      // Add name text
      const text = new fabric.Text(member.login, {
        left: 10,
        top: 110,
        fontSize: 16,
        fill: '#333',
      });
      cardGroup.addWithUpdate(text);
	   console.log("cardgrp",cardGroup)

      // Add click event to open profile in new tab
      

      canvas.add(cardGroup);
    });
  }
</script>

<style>
  .container {
    display: flex;
    flex-direction: column;
    align-items: center;
    margin-top: 20px;
  }

  input[type="text"] {
    padding: 10px;
    font-size: 16px;
    border-radius: 5px;
    border: 1px solid #ccc;
    margin-bottom: 20px;
    width: 300px;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  }

  canvas {
    border-radius: 10px;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  }
</style>

<div class="container">
  <input type="text" placeholder="Search members..." on:input={e => filterMembers(e.target.value)} />
  <canvas id="canvas"></canvas>
</div>
