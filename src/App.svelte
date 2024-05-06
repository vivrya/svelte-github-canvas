<script>
  import { onMount } from "svelte";
  import { fabric } from "fabric";
  import debounce from "lodash/debounce";

  let canvas;
  let members = [];
  let filteredMembers = [];
  let isDraggingCanvas = false;
  let lastMousePosition;
  let copiedMembers = [];

  // Set card dimensions
  const cardWidth = 500; // Width of each list item
  const cardHeight = 100; // Height of each list item, same as image height
  const cardSpacing = 10; // Space between list items

  // Function to filter members based on search input
  function filterMembers(query) {
    if (query.trim() === "") {
      // If the query is empty, render both filtered and copied members
      renderCopiedMembers();
      return;
    }

    // Filter the members based on the query
    filteredMembers = members.filter((member) =>
      member.login.toLowerCase().includes(query.toLowerCase())
    );

    renderMembers(filteredMembers);
  }

 function renderCopiedMembers() {
    const sidebar = document.querySelector(".sidebar");

    // Clear the sidebar
    sidebar.innerHTML = "";

    // Render each copied member as a card in the sidebar
    copiedMembers.forEach((member) => {
        const card = document.createElement("div");
        card.style.display = "flex";
        card.style.alignItems = "center";
        card.style.padding = "5px";
        card.style.margin = "5px";
        card.style.border = "1px solid #ccc";
        card.style.borderRadius = "5px";
        card.style.cursor = "pointer";

        // Create an image element for the member's avatar
        const img = document.createElement("img");
        img.src = member.avatar_url;
        img.alt = member.login;
        img.style.width = "50px";
        img.style.height = "50px";
        img.style.borderRadius = "50%";
        img.style.marginRight = "10px";

        // Create a span element for the member's login name
        const nameSpan = document.createElement("span");
        nameSpan.textContent = member.login;
        nameSpan.style.fontWeight = "bold";

        // Append the image and name to the card
        card.appendChild(img);
        card.appendChild(nameSpan);

        // Append the card to the sidebar
        sidebar.appendChild(card);
    });






  }

  // Function to render members on the canvas
  function renderMembers(membersToRender) {
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
    const width = window.innerWidth *0.70;
    const height = window.innerHeight * 0.8;
    const sidebarWidth = width * 0.27; // 30% of canvas width

    canvas = new fabric.Canvas("canvas", {
      width,
      height,
      isDrawingMode: false,
      selection: false,
      preserveObjectStacking: true,
    });

    // Event listener for keydown to detect Ctrl+S (or Cmd+S) key press
    window.addEventListener("keydown", (e) => {
		
      if ((e.ctrlKey || e.metaKey) && e.key === "s") {
		e.preventDefault()
        // Append filteredMembers array to copiedMembers array
        copiedMembers.push(...filteredMembers);
        console.log("Filtered members appended to copiedMembers array:", copiedMembers);
        renderCopiedMembers();
		}
		

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
    canvas.on("mouse:move", function (opt) {
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
    canvas.on("mouse:up", function () {
      isDraggingCanvas = false;
      lastMousePosition = null;
    });

    // Event listener for mouse wheel to zoom
    canvas.on("mouse:wheel", function (opt) {
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
          left: -10,
          fontSize: 20,
          fill: "black",
          originX: "left",
          originY: "center",
          textAlign: "left",
        });

        // Add the name text to the card group
        cardGroup.add(nameText);

        // Once the card is fully constructed, render the canvas
        canvas.renderAll();
      });

      // Allow the card group to be selectable and movable
      cardGroup.set({
        selectable: true,
        hasControls: false,
        hasBorders: true,
      });

      // Flag to track whether the card is being dragged
      let isDraggingCard = false;

      // Add event listener to track when the card is being moved
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

  // Function to animate card appearance (fade-in)
  function animateCardAppearance(card) {
    // Animate the opacity from 0 to 1
    card.animate("opacity", 1, {
      duration: 1500, // Duration of 500 ms
      easing: fabric.util.ease.easeOutCubic, // Use an easing function for a smooth transition
      onChange: () => {
        canvas.renderAll();
      },
    });
  }

  // Function to animate card disappearance (fade-out)
  function animateCardDisappearance(card) {
    // Animate the opacity from 1 to 0
    card.animate("opacity", 0, {
      duration: 1500, // Duration of 500 ms
      easing: fabric.util.ease.easeInCubic, // Use an easing function for a smooth transition
      onChange: () => {
        canvas.renderAll();
      },
      onComplete: () => {
        // Remove the card from the canvas after the animation completes
        canvas.remove(card);
      },
    });
  }

  // Function to animate card movement
  function animateCardMovement(card, newLeft, newTop) {
    // Animate the left and top properties
    card.animate(
      { left: newLeft, top: newTop },
      {
        duration: 1500, // Duration of 500 ms
        easing: fabric.util.ease.easeInOutCubic, // Use an easing function for a smooth transition
        onChange: () => {
          canvas.renderAll();
        },
      }
    );
  }

  // Modify createCard function to include appearance animation
  function createCard(member, leftPosition, topPosition) {
    const card = new fabric.MemberCard(member, {
      top: topPosition,
      left: leftPosition,
      opacity: 0, // Initially set the card opacity to 0 for fade-in effect
    });

    // Add the card to the canvas
    canvas.add(card);

    // Animate the appearance of the card
    animateCardAppearance(card);
  }

  // Add hover effect for the card
  function addHoverEffect(card) {
    // Define the original border properties (you may adjust these as desired)
    const originalStroke = null; // No border initially
    const originalStrokeWidth = 0; // No border initially

    // Define the hover border properties
    const hoverStroke = "black"; // Change to "black" for the black border on hover
    const hoverStrokeWidth = 3; // Keep this value the same for border width on hover

    // Set the initial border properties
    card.set({
      stroke: originalStroke,
      strokeWidth: originalStrokeWidth,
    });

    // Add mouseover event to change the border properties when hovering
    card.on("mouseover", () => {
      card.set({
        stroke: hoverStroke,
        strokeWidth: hoverStrokeWidth,
      });
      // Re-render the canvas to apply the changes
      canvas.renderAll();
    });

    // Add mouseout event to revert the border properties when no longer hovering
    card.on("mouseout", () => {
      card.set({
        stroke: originalStroke,
        strokeWidth: originalStrokeWidth,
      });
      // Re-render the canvas to apply the changes
      canvas.renderAll();
    });
  }
</script>



<style>
  .container {
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: center;
    margin-top: 20px;
    padding: 20px;
}


  input[type="text"] {
    padding: 10px;
    font-size: 16px;
    border-radius: 5px;
    border: 1px solid #ccc;
    margin-right: 10px;
    width: 300px;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  }

  canvas {
    width: 70vw; /* Set canvas width to 70% of viewport width */
    height: 80vh; /* Set canvas height to 80% of viewport height */
    border-radius: 10px;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
    margin-right: 10px; /* Add margin to separate canvas and sidebar */
  }

  .sidebar {
    width: 27vw; /* Set sidebar width to 30% of viewport width */
    height: 78vh; /* Set sidebar height to 80% of viewport height */
    padding: 10px;
    background-color: #f9f9f9;
    border-radius: 5px;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
    overflow-y: auto; /* Enable vertical scrolling if content overflows */
	margin-left:5px;
  }













/* Styles for the member details container */
.member-details {
  display: flex;
  align-items: center;
}

/* Styles for the member name */
.member-name {
  font-weight: bold;
}
</style>

<div class="container">
  <input
    type="text"
    placeholder="Search members..."
    on:input={(e) => filterMembers(e.target.value)}
  />
  </div>
  <div style="display:flex;flex-direction:row">
    <canvas id="canvas"></canvas>

  <div class="sidebar">
    <!-- Sidebar content goes here -->
  </div>
</div>

