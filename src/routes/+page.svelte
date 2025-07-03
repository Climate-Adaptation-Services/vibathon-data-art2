<script lang="ts">
  import { onMount } from 'svelte';
  
  // Heartbeat parameters
  const width = 800;
  const height = 300;
  const baselineY = height / 2;
  
  // Animation state
  let progress = 0;
  let animationFrame: number;
  let heartlinePaths: {path: SVGPathElement | null, length: number}[] = [];
  
  // Heatwave data parameters
  type HeatwaveData = {
    startDate: string;
    endDate: string;
    duration: number;
    tropicalDays: number;
    highestTemp: number;
  };
  
  // Heartbeat parameters
  type HeartbeatParams = {
    pHeight: number;
    qDepth: number;
    rHeight: number;
    sDepth: number;
    tHeight: number;
    color: string;
    opacity: number;
    heatwaveData?: HeatwaveData;
  };
  
  // Data loading state
  let isLoading = true;
  let loadError: string | null = null;
  let heatwaves: HeatwaveData[] = [];
  let heartbeats: HeartbeatParams[] = [];
  
  // Generate a color based on temperature
  function getTemperatureColor(temp: number): string {
    // Map temperature range (30-42°C) to color intensity
    const minTemp = 30;
    const maxTemp = 42;
    const normalizedTemp = Math.min(Math.max(temp - minTemp, 0), maxTemp - minTemp) / (maxTemp - minTemp);
    
    // Generate color from blue (cooler) to red (hotter)
    const r = Math.floor(255 * normalizedTemp) + (255 * (1 - normalizedTemp) * 0.3);
    const g = Math.floor(100 * (1 - normalizedTemp));
    const b = Math.floor(50 * (1 - normalizedTemp));
    
    return `rgb(${r}, ${g}, ${b})`;
  }
  
  // Parse CSV data
  async function loadHeatwaveData() {
    const csvUrl = 'https://raw.githubusercontent.com/sophievanderhorst/data/refs/heads/main/heatwaves.csv';
    
    try {
      isLoading = true;
      loadError = null;
      
      const response = await fetch(csvUrl);
      if (!response.ok) {
        throw new Error(`Failed to fetch CSV: ${response.status} ${response.statusText}`);
      }
      
      const csvText = await response.text();
      const lines = csvText.trim().split('\n');
      const headers = lines[0].split(',');
      
      // Find column indices
      const startDateIndex = headers.findIndex(h => h === 'Van');
      const endDateIndex = headers.findIndex(h => h === 'Tot');
      const durationIndex = headers.findIndex(h => h === 'Duur');
      const tropicalDaysIndex = headers.findIndex(h => h === 'Aantal tropische dagen');
      const highestTempIndex = headers.findIndex(h => h === 'Hoogste temperatuur');
      
      if (startDateIndex === -1 || endDateIndex === -1 || highestTempIndex === -1) {
        throw new Error('Required columns not found in CSV');
      }
      
      heatwaves = lines.slice(1).map(line => {
        const values = line.split(',');
        return {
          startDate: values[startDateIndex],
          endDate: values[endDateIndex],
          duration: parseInt(values[durationIndex]) || 0,
          tropicalDays: parseInt(values[tropicalDaysIndex]) || 0,
          highestTemp: parseFloat(values[highestTempIndex]) || 0
        };
      });
      
      console.log(`Loaded ${heatwaves.length} heatwave records`);
      
      // Sort by temperature (descending)
      heatwaves.sort((a, b) => b.highestTemp - a.highestTemp);
      
      // Take top 30 or all if less than 30
      const topHeatwaves = heatwaves.slice(0, 30);
      
      // Generate heartbeats from heatwave data
      heartbeats = topHeatwaves.map(generateHeartbeatFromHeatwave);
      
    } catch (error) {
      console.error('Error loading CSV data:', error);
      loadError = error instanceof Error ? error.message : 'Unknown error loading data';
      
      // Fallback to random data if loading fails
      heartbeats = Array.from({ length: 30 }, () => generateRandomHeartbeatParams());
    } finally {
      isLoading = false;
    }
  }
  
  // Generate heartbeat parameters based on heatwave data
  function generateHeartbeatFromHeatwave(heatwave: HeatwaveData): HeartbeatParams {
    // Scale R wave height based on temperature (30-42°C range mapped to 50-150px)
    const minTemp = 30;
    const maxTemp = 42;
    const minHeight = 50;
    const maxHeight = 150;
    
    const tempNormalized = Math.min(Math.max(heatwave.highestTemp - minTemp, 0), maxTemp - minTemp) / (maxTemp - minTemp);
    const rHeight = minHeight + tempNormalized * (maxHeight - minHeight);
    
    // Other parameters can be partially randomized but influenced by temperature
    const tempFactor = tempNormalized * 0.7 + 0.3; // 0.3-1.0 range based on temperature
    
    return {
      pHeight: 10 + tempFactor * 15,       // P wave: 10-25px
      qDepth: 2 + tempFactor * 10,          // Q wave: 2-12px
      rHeight,                              // R wave: scaled by temperature
      sDepth: 20 + tempFactor * 30,         // S wave: 20-50px
      tHeight: 10 + tempFactor * 20,        // T wave: 10-30px
      color: getTemperatureColor(heatwave.highestTemp),
      opacity: 0.7 + tempFactor * 0.3,      // 0.7-1.0 opacity
      heatwaveData: heatwave
    };
  }
  
  // Generate random heartbeat parameters as fallback
  function generateRandomHeartbeatParams(): HeartbeatParams {
    // Random heights for each wave component with reasonable ranges
    return {
      pHeight: Math.random() * 20 + 5,       // P wave: 5-25px
      qDepth: Math.random() * 10 + 2,        // Q wave: 2-12px
      rHeight: Math.random() * 100 + 50,     // R wave: 50-150px
      sDepth: Math.random() * 30 + 20,       // S wave: 20-50px
      tHeight: Math.random() * 20 + 10,      // T wave: 10-30px
      color: getRandomColor(),               // Random color
      opacity: Math.random() * 0.5 + 0.5     // Random opacity: 0.5-1.0
    };
  }
  
  // Generate a random color with red/orange hues (fallback)
  function getRandomColor(): string {
    const r = 255;
    const g = Math.floor(Math.random() * 100);
    const b = Math.floor(Math.random() * 50);
    return `rgb(${r}, ${g}, ${b})`;
  }
  
  // Generate the heartline path for a specific set of parameters
  function generateHeartlinePath(params: HeartbeatParams): string {
    // Width of the entire visualization
    const lineWidth = width * 0.8;
    const startX = width * 0.1;
    
    // Extract parameters
    const { pHeight, qDepth, rHeight, sDepth, tHeight } = params;
    
    // Create segments for the PQRST pattern
    return [
      // Start with flat baseline
      `M ${startX} ${baselineY}`,
      `L ${startX + lineWidth * 0.2} ${baselineY}`,
      
      // P wave (small bump)
      `L ${startX + lineWidth * 0.25} ${baselineY - pHeight}`,
      `L ${startX + lineWidth * 0.3} ${baselineY}`,
      
      // PR segment (flat)
      `L ${startX + lineWidth * 0.35} ${baselineY}`,
      
      // Q wave (slight downward deflection)
      `L ${startX + lineWidth * 0.37} ${baselineY + qDepth}`,
      
      // R wave (sharp tall upward spike)
      `L ${startX + lineWidth * 0.4} ${baselineY - rHeight}`,
      
      // S wave (sharp downward deflection after R)
      `L ${startX + lineWidth * 0.43} ${baselineY + sDepth}`,
      
      // Return to baseline
      `L ${startX + lineWidth * 0.47} ${baselineY}`,
      
      // ST segment (flat)
      `L ${startX + lineWidth * 0.55} ${baselineY}`,
      
      // T wave (smaller bump)
      `L ${startX + lineWidth * 0.6} ${baselineY - tHeight}`,
      `L ${startX + lineWidth * 0.65} ${baselineY}`,
      
      // Final baseline
      `L ${startX + lineWidth} ${baselineY}`
    ].join(' ');
  }
  
  // Animation state for sequential animation
  let currentHeartbeatIndex = 0;
  let heartbeatProgress = 0;
  let pauseBetweenHeartbeats = 300; // ms to pause between heartbeats
  let lastHeartbeatFinishTime = 0;
  
  // Animation function for sequential heartlines
  function animateHeartlinesSequentially(): void {
    const now = performance.now();
    
    // If we're in a pause between heartbeats
    if (lastHeartbeatFinishTime > 0 && now - lastHeartbeatFinishTime < pauseBetweenHeartbeats) {
      animationFrame = requestAnimationFrame(animateHeartlinesSequentially);
      return;
    }
    
    // If we've completed all heartbeats, reset and start over
    if (currentHeartbeatIndex >= heartlinePaths.length) {
      currentHeartbeatIndex = 0;
      heartbeatProgress = 0;
      lastHeartbeatFinishTime = 0;
      
      // Reset all paths to hidden
      heartlinePaths.forEach(item => {
        if (item.path) {
          item.path.style.strokeDashoffset = item.length.toString();
        }
      });
      
      animationFrame = requestAnimationFrame(animateHeartlinesSequentially);
      return;
    }
    
    // Get the current heartbeat path to animate
    const currentPath = heartlinePaths[currentHeartbeatIndex];
    
    if (currentPath && currentPath.path) {
      if (heartbeatProgress < 1) {
        // Animate the current heartbeat
        heartbeatProgress += 0.01; // Speed of individual heartbeat drawing
        currentPath.path.style.strokeDashoffset = ((1 - heartbeatProgress) * currentPath.length).toString();
        animationFrame = requestAnimationFrame(animateHeartlinesSequentially);
      } else {
        // Current heartbeat animation complete, move to next
        lastHeartbeatFinishTime = now;
        currentHeartbeatIndex++;
        heartbeatProgress = 0;
        animationFrame = requestAnimationFrame(animateHeartlinesSequentially);
      }
    } else {
      // Skip any invalid paths
      currentHeartbeatIndex++;
      animationFrame = requestAnimationFrame(animateHeartlinesSequentially);
    }
  }
  
  // Setup animation after paths are rendered
  async function setupAnimation() {
    // Get all path elements
    const pathElements = document.querySelectorAll('.heartline-path');
    
    // Set up each path for animation
    heartlinePaths = Array.from(pathElements).map(element => {
      const path = element as SVGPathElement;
      const length = path.getTotalLength();
      path.style.strokeDasharray = length.toString();
      path.style.strokeDashoffset = length.toString();
      return { path, length };
    });
    
    // Start animation
    if (heartlinePaths.length > 0) {
      animationFrame = requestAnimationFrame(animateHeartlinesSequentially);
    }
  }

  // Initialize component
  onMount(() => {
    // Use an immediately invoked async function to handle async operations
    (async () => {
      try {
        // Load heatwave data from CSV
        await loadHeatwaveData();
        
        // Setup animation after a short delay to ensure DOM is ready
        setTimeout(() => {
          setupAnimation();
        }, 100);
      } catch (error) {
        console.error('Error during initialization:', error);
      }
    })();
    
    // Return cleanup function
    return () => {
      // Clean up on component unmount
      if (animationFrame) {
        cancelAnimationFrame(animationFrame);
      }
    };
  });
</script>

<main>
  <div class="container">
    {#if isLoading}
      <div class="loading">
        <p>Loading heatwave data...</p>
      </div>
    {:else if loadError}
      <div class="error">
        <p>Error loading data: {loadError}</p>
        <p>Using random heartbeat data instead.</p>
      </div>
    {/if}
    
    <svg width={width} height={height}>
      <!-- Black background -->
      <rect width="100%" height="100%" fill="black" />
      
      <!-- Subtle grid lines -->
      {#each Array(15) as _, i}
        <line 
          x1="0" 
          y1={i * (height / 15)} 
          x2={width} 
          y2={i * (height / 15)} 
          stroke="rgba(0, 100, 0, 0.15)" 
          stroke-width="1" 
        />
      {/each}
      {#each Array(20) as _, i}
        <line 
          x1={i * (width / 20)} 
          y1="0" 
          x2={i * (width / 20)} 
          y2={height} 
          stroke="rgba(0, 100, 0, 0.15)" 
          stroke-width="1" 
        />
      {/each}
      
      <!-- Multiple heartlines stacked on top of each other -->
      {#each heartbeats as heartbeat, i}
        <g class="heartbeat-group">
          <path 
            d={generateHeartlinePath(heartbeat)} 
            stroke={heartbeat.color} 
            stroke-width="2"
            fill="none"
            opacity={heartbeat.opacity}
            class="heartline-path"
          />
          
          <!-- Temperature label for each heartbeat -->
          {#if heartbeat.heatwaveData}
            <text 
              x={width * 0.1 + width * 0.8 * 0.4} 
              y={baselineY - heartbeat.rHeight - 10} 
              fill={heartbeat.color}
              font-size="10"
              text-anchor="middle"
              opacity={heartbeat.opacity}
              class="temp-label"
            >
              {heartbeat.heatwaveData.highestTemp}°C
            </text>
            
            <!-- Date label -->
            <text 
              x={width * 0.1 + width * 0.8 * 0.8} 
              y={baselineY - 5} 
              fill={heartbeat.color}
              font-size="8"
              text-anchor="end"
              opacity={heartbeat.opacity * 0.8}
              class="date-label"
            >
              {heartbeat.heatwaveData.startDate}
            </text>
          {/if}
        </g>
      {/each}
    </svg>
    
    <div class="legend">
      <h2>Heatwave ECG Visualization</h2>
      <p>Each heartbeat represents a heatwave, with R-peak height scaled to the highest temperature.</p>
      <p>Data source: <a href="https://raw.githubusercontent.com/sophievanderhorst/data/refs/heads/main/heatwaves.csv" target="_blank">Heatwave Dataset</a></p>
    </div>
  </div>
</main>

<style>
  :global(body) {
    margin: 0;
    padding: 0;
    background-color: #000;
    color: #fff;
    font-family: Arial, sans-serif;
  }
  
  main {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    min-height: 100vh;
    padding: 0;
    box-sizing: border-box;
    overflow: hidden;
  }
  
  .container {
    width: 100%;
    height: 100vh;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    position: relative;
  }
  
  svg {
    width: 100%;
    max-width: 1000px;
    height: 70vh;
  }
  
  .heartline-path {
    filter: drop-shadow(0 0 8px rgba(255, 0, 0, 0.7));
    will-change: stroke-dashoffset;
    transition: opacity 0.3s;
  }
  
  .heartline-path:hover {
    opacity: 1 !important;
    filter: drop-shadow(0 0 12px rgba(255, 255, 255, 0.9));
    z-index: 10;
  }
  
  .temp-label {
    filter: drop-shadow(0 0 3px rgba(255, 50, 50, 0.9));
    font-weight: bold;
  }
  
  .date-label {
    font-style: italic;
  }
  
  .legend {
    position: absolute;
    bottom: 20px;
    left: 0;
    right: 0;
    text-align: center;
    background-color: rgba(0, 0, 0, 0.7);
    padding: 10px;
    border-top: 1px solid rgba(255, 50, 50, 0.5);
  }
  
  .legend h2 {
    margin: 0 0 10px 0;
    font-size: 1.2rem;
    color: #ff5050;
  }
  
  .legend p {
    margin: 5px 0;
    font-size: 0.9rem;
    color: #ccc;
  }
  
  .legend a {
    color: #ff8080;
    text-decoration: none;
  }
  
  .legend a:hover {
    text-decoration: underline;
  }
  
  .loading, .error {
    position: absolute;
    top: 20px;
    left: 0;
    right: 0;
    text-align: center;
    background-color: rgba(0, 0, 0, 0.7);
    padding: 10px;
    z-index: 10;
  }
  
  .error {
    color: #ff5050;
    border: 1px solid #ff5050;
  }
</style>
