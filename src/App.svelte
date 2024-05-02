<script>
  import { onMount } from "svelte";
  import { fabric } from "fabric";
  import debounce from "lodash/debounce";
  
  let canvas;
  let members = [];
  let filteredMembers = [];
  let isDraggingCanvas = false;
  let lastMousePosition;
  
  // Set card dimensions
  const cardWidth = 500; // Width of each list item
  const cardHeight = 100; // Height of each list item, same as image height
  const cardSpacing = 10; // Space between list items
  
  // Function to filter members based on search input
  function filterMembers(query) {
    console.log("Filtering with query:", query);
    filteredMembers = members.filter((member) =>
      member.login.toLowerCase().includes(query.toLowerCase())
    );
  
    console.log("Filtered members:", filteredMembers);
    renderMembers(filteredMembers);
  }
  
  // Function to render members on the canvas
  function renderMembers(membersToRender) {
    console.log("Rendering members:", membersToRender);
    canvas.clear();
    
    let topPosition = 0;
    membersToRender.forEach((member) => {
      createCard(member, 0, topPosition);
      topPosition += cardHeight + cardSpacing;
    });
    
    canvas.renderAll();
  }
  
  onMount(() => {
    // Set up canvas dimensions
    const width = window.innerWidth * 0.8;
    const height = window.innerHeight * 0.8;
    
    canvas = new fabric.Canvas("canvas", {
      width,
      height,
      isDrawingMode: false,
      selection: false,
      preserveObjectStacking: true,
    });
  
    // Event listener for mouse down
    canvas.on("mouse:down", function(opt) {
      const evt = opt.e;
      const target = opt.target;
      
      if (!target) {
        // If no target object (clicking on canvas background), start panning
        isDraggingCanvas = true;
        lastMousePosition = {
          x: evt.clientX,
          y: evt.clientY,
        };
      }
    });
  
    // Event listener for mouse move
    canvas.on("mouse:move", function(opt) {
      const evt = opt.e;
      if (isDraggingCanvas) {
        // Calculate mouse movement and pan the canvas
        const deltaX = evt.clientX - lastMousePosition.x;
        const deltaY = evt.clientY - lastMousePosition.y;
        canvas.relativePan({ x: deltaX, y: deltaY });
        lastMousePosition = {
          x: evt.clientX,
          y: evt.clientY,
        };
      }
    });
  
    // Event listener for mouse up
    canvas.on("mouse:up", function() {
      isDraggingCanvas = false;
      lastMousePosition = null;
    });
  
    // Event listener for mouse wheel to zoom
    canvas.on("mouse:wheel", function(opt) {
      const evt = opt.e;
      let zoom = canvas.getZoom();
      zoom *= 0.999 ** evt.deltaY;
      zoom = Math.max(0.1, Math.min(20, zoom)); // Clamp zoom level
      
      canvas.zoomToPoint({ x: evt.offsetX, y: evt.offsetY }, zoom);
      evt.preventDefault();
      evt.stopPropagation();
    });
  
    // Fetch members data from the API
    fetch("https://api.github.com/orgs/mozilla/members?page=1")
      .then((response) => response.json())
      .then((data) => {
        members = data;
        filteredMembers = data;
        renderMembers(filteredMembers);
      })
      .catch((error) => {
        console.error("Error fetching members:", error);
      });
  });
  
  // Custom Fabric.js class for member name cards
  fabric.MemberCard = fabric.util.createClass(fabric.Group, {
    initialize: function (member, options) {
      options || (options = {});
      options.left = options.left || 10;
      options.top = options.top || 10;
      options.width = cardWidth;
      options.height = cardHeight;
  
      this.callSuper("initialize", [], options);
  
      // Create a group for the card
      const cardGroup = this;
  
      // Load and add avatar image to the card group
      fabric.Image.fromURL(member.avatar_url, (img) => {
        // Scale the image to fit the card height
        img.scaleToHeight(cardHeight);
  
        // Set the image's left position to 0 and top to 0
        img.set({ left: -200, top: -50 });
  
        // Add the image to the card group
        cardGroup.add(img);
  
        // Add member's name text to the card group
        const nameText = new fabric.Text(member.login, {
          left: img.getScaledWidth() + 10, // Position text next to the image
          top: 0,
          fontSize: 20,
          fill: "black",
          originX: "left",
          originY: "center",
          textAlign: "left",
        });
  
        // Add the name text to the card group
        cardGroup.add(nameText);
      });
  
      // Allow the card group to be selectable and movable
      cardGroup.set({
        selectable: true,
        hasControls: false,
        hasBorders: true,
      });
  
      // Flag to track whether the card is being dragged
      let isDraggingCard = false;
  
      // Add event listener to track when card is being moved
      cardGroup.on("moving", () => {
        isDraggingCard = true;
      });
  
      // Add event listener for mouse up to handle clicks
      cardGroup.on("mouseup", (e) => {
        // Check if the card is not being dragged (a click occurred)
        if (!isDraggingCard) {
          // Open the member's GitHub profile link on click
          window.open(member.html_url, "_blank");
        }
  
        // Reset flag on mouse up
        isDraggingCard = false;
      });
  
      // Apply hover effects
      addHoverEffect(cardGroup);
    },
  });
  
  // Function to create hover effects for the card
  function addHoverEffect(card) {
    card.on("mouseover", () => {
      card.set({ stroke: "blue", strokeWidth: 2 });
      canvas.renderAll();
    });
  
    card.on("mouseout", () => {
      card.set({ stroke: "black", strokeWidth: 1 });
      canvas.renderAll();
    });
  }
  
  function createCard(member, leftPosition, topPosition) {
    const card = new fabric.MemberCard(member, {
      top: topPosition,
      left: leftPosition,
    });
  
    // Add the card to the canvas
    canvas.add(card);
  }
  </script>
  
  <div class="container">
    <input
      type="text"
      placeholder="Search members..."
      on:input={(e) => filterMembers(e.target.value)}
    />
    <canvas id="canvas"></canvas>
  </div>
  
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
  