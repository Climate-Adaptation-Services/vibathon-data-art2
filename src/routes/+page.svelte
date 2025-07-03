<script lang="ts">
  import { onMount } from "svelte"

  // Heartbeat parameters
  const width = 800
  const height = 300
  const baselineY = height / 2
  // Center point for the heartbeat visualization
  const centerX = width / 2

  // Animation state
  let progress = 0
  let animationFrame: number

  // Heatwave data parameters
  interface HeatwaveData {
    startDate: string;
    endDate: string;
    duration: number;
    tropicalDays: number;
    highestTemp: number;
  }
  
  // Global variables for temperature range
  let globalMinTemp = 30;
  let globalMaxTemp = 40;

  // Heartbeat parameters
  type HeartbeatParams = {
    pHeight: number
    qDepth: number
    rHeight: number
    sDepth: number
    tHeight: number
    color: string
    opacity: number
    heatwaveData?: HeatwaveData
  }

  // Data loading state
  let isLoading = true
  let loadError: string | null = null
  let heatwaves: HeatwaveData[] = []
  let heartbeats: HeartbeatParams[] = []

  // Generate a color based on temperature
  function getTemperatureColor(temp: number): string {
    // Map temperature range (30-42°C) to color intensity
    const minTemp = 30
    const maxTemp = 42
    const normalizedTemp = Math.min(Math.max(temp - minTemp, 0), maxTemp - minTemp) / (maxTemp - minTemp)

    // Generate color from blue (cooler) to red (hotter)
    const r = Math.floor(255 * normalizedTemp) + 255 * (1 - normalizedTemp) * 0.3
    const g = Math.floor(100 * (1 - normalizedTemp))
    const b = Math.floor(50 * (1 - normalizedTemp))

    return `rgb(${r}, ${g}, ${b})`
  }

  // Parse CSV data
  async function loadHeatwaveData() {
    const csvUrl = "https://raw.githubusercontent.com/sophievanderhorst/data/refs/heads/main/heatwaves.csv"

    try {
      isLoading = true
      loadError = null

      const response = await fetch(csvUrl)
      if (!response.ok) {
        throw new Error(`Failed to fetch CSV: ${response.status} ${response.statusText}`)
      }

      const csvText = await response.text()
      const lines = csvText.trim().split("\n")
      const headers = lines[0].split(",") // Use comma as separator

      // Find column indices
      const startDateIndex = headers.findIndex((h) => h === "Van")
      const endDateIndex = headers.findIndex((h) => h === "Tot")
      const durationIndex = headers.findIndex((h) => h === "Duur")
      const tropicalDaysIndex = headers.findIndex((h) => h === "Aantal tropische dagen")
      const highestTempIndex = headers.findIndex((h) => h === "Hoogste temperatuur")

      if (startDateIndex === -1 || endDateIndex === -1 || highestTempIndex === -1) {
        throw new Error("Required columns not found in CSV")
      }

      heatwaves = lines.slice(1).map((line, index) => {
        const values = line.split(','); // GitHub CSV uses comma separators
        const highestTemp = parseFloat(values[highestTempIndex]) || 0;
        
        // Log every 10th record for debugging
        if (index % 10 === 0) {
          console.log(`Record ${index} - Date: ${values[startDateIndex]}, Temp: ${highestTemp}°C`);
        }
        
        return {
          startDate: values[startDateIndex],
          endDate: values[endDateIndex],
          duration: parseInt(values[durationIndex]) || 0,
          tropicalDays: parseInt(values[tropicalDaysIndex]) || 0,
          highestTemp: highestTemp
        };
      });

      console.log(`Loaded ${heatwaves.length} heatwave records`)

      // Log the first few records to inspect the data
      console.log("First 5 heatwave records:", heatwaves.slice(0, 5))

      // Find min and max temperatures for better scaling
      const temperatures = heatwaves.map(hw => hw.highestTemp);
      const minDataTemp = Math.min(...temperatures);
      const maxDataTemp = Math.max(...temperatures);
      console.log(`Actual temperature range in data: ${minDataTemp}°C to ${maxDataTemp}°C`);
      
      // Store the temperature range globally for use in heartbeat generation
      globalMinTemp = minDataTemp;
      globalMaxTemp = maxDataTemp;
      
      // Sort by temperature (descending)
      heatwaves.sort((a, b) => b.highestTemp - a.highestTemp)

      // Generate heartbeats from all heatwave data
      heartbeats = heatwaves.map(generateHeartbeatFromHeatwave)
    } catch (error) {
      console.error("Error loading CSV data:", error)
      loadError = error instanceof Error ? error.message : "Unknown error loading data"

      // Fallback to random data if loading fails
      heartbeats = Array.from({ length: 30 }, () => generateRandomHeartbeatParams())
    } finally {
      isLoading = false
    }
  }

  // Generate heartbeat parameters based on heatwave data
  function generateHeartbeatFromHeatwave(heatwave: HeatwaveData): HeartbeatParams {
    // Use random peak heights instead of temperature-based scaling
    const minHeight = 30;   // Minimum height for peaks
    const maxHeight = 120;  // Maximum height for peaks
    
    // Generate a random height for this heartbeat
    const rHeight = minHeight + Math.random() * (maxHeight - minHeight);
    
    // Generate random variations for other parameters
    const randomFactor = Math.random() * 0.7 + 0.3; // 0.3-1.0 range
    
    // Generate a random color in the red-orange-yellow spectrum
    const hue = Math.floor(Math.random() * 60); // 0-60 range (red to yellow)
    const color = `hsl(${hue}, 80%, 50%)`;
    
    console.log(`Heartbeat for ${heatwave.startDate}: Height=${rHeight.toFixed(0)}px, Color=${color}`);

    return {
      pHeight: 10 + randomFactor * 15, // P wave: 10-25px
      qDepth: 5 + randomFactor * 10, // Q wave: 5-15px
      rHeight, // R wave: random height
      sDepth: 10 + randomFactor * 15, // S wave: 10-25px
      tHeight: 15 + randomFactor * 20, // T wave: 15-35px
      color: color,
      opacity: 0.8,
      heatwaveData: heatwave
    }
  }

  // Generate random heartbeat parameters as fallback
  function generateRandomHeartbeatParams(): HeartbeatParams {
    // Random heights for each wave component with reasonable ranges
    return {
      pHeight: Math.random() * 20 + 5, // P wave: 5-25px
      qDepth: Math.random() * 10 + 2, // Q wave: 2-12px
      rHeight: Math.random() * 100 + 50, // R wave: 50-150px
      sDepth: Math.random() * 30 + 20, // S wave: 20-50px
      tHeight: Math.random() * 20 + 10, // T wave: 10-30px
      color: getRandomColor(), // Random color
      opacity: Math.random() * 0.5 + 0.5, // Random opacity: 0.5-1.0
    }
  }

  // Generate a random color with red/orange hues (fallback)
  function getRandomColor(): string {
    const r = 255
    const g = Math.floor(Math.random() * 100)
    const b = Math.floor(Math.random() * 50)
    return `rgb(${r}, ${g}, ${b})`
  }

  // Generate the heartline path for a specific set of parameters
  function generateHeartlinePath(params: HeartbeatParams): string {
    // Center the heartbeat in the SVG
    // Width of the entire visualization - centered
    const lineWidth = width * 0.8;
    const startX = (width - lineWidth) / 2; // Center the heartbeat
    
    // Extract parameters
    const { pHeight, qDepth, rHeight, sDepth, tHeight } = params;
    
    // Log the actual R peak position
    const rPeakY = baselineY - rHeight;
    console.log(`R peak will be drawn at Y: ${rPeakY} (baselineY: ${baselineY} - rHeight: ${rHeight})`);
    
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
      `L ${startX + lineWidth} ${baselineY}`,
    ].join(" ")
  }

  // Animation state for sequential animation
  let currentHeartbeatIndex = 0
  let heartbeatProgress = 0
  let fadeOutProgress = 0
  let isFadingOut = false
  let pauseBetweenHeartbeats = 500 // ms to pause between heartbeats
  let lastHeartbeatFinishTime = 0

  // Animation function for sequential heartlines
  function animateHeartlinesSequentially(): void {
    const now = performance.now()

    // If we're in a pause between heartbeats
    if (lastHeartbeatFinishTime > 0 && now - lastHeartbeatFinishTime < pauseBetweenHeartbeats) {
      animationFrame = requestAnimationFrame(animateHeartlinesSequentially)
      return
    }

    // If we've completed all heartbeats, reset and start over
    if (currentHeartbeatIndex >= heartbeats.length) {
      currentHeartbeatIndex = 0
      heartbeatProgress = 0
      fadeOutProgress = 0
      isFadingOut = false
      lastHeartbeatFinishTime = 0

      animationFrame = requestAnimationFrame(animateHeartlinesSequentially)
      return
    }

    // Log current heartbeat data
    if (heartbeats.length > 0 && currentHeartbeatIndex < heartbeats.length) {
      const currentHeatwave = heartbeats[currentHeartbeatIndex].heatwaveData;
      const currentHeartbeat = heartbeats[currentHeartbeatIndex];
      
      // Log at the start of each heartbeat animation
      if (currentHeatwave && heartbeatProgress === 0 && !isFadingOut) {
        console.log('-------------------------------------');
        console.log(`HEARTBEAT ${currentHeartbeatIndex + 1}/${heartbeats.length}`);
        console.log('Heatwave Data:', {
          startDate: currentHeatwave.startDate,
          endDate: currentHeatwave.endDate,
          duration: currentHeatwave.duration + ' days',
          tropicalDays: currentHeatwave.tropicalDays,
          highestTemp: currentHeatwave.highestTemp + '°C'
        });
        console.log('Visualization Parameters:', {
          rHeight: Math.round(currentHeartbeat.rHeight) + 'px',
          color: currentHeartbeat.color,
          opacity: currentHeartbeat.opacity.toFixed(2)
        });
        console.log('-------------------------------------');
      }
    }

    // First draw the heartbeat, then fade it out
    if (!isFadingOut) {
      // Drawing phase
      if (heartbeatProgress < 1) {
        // Gradually increase progress
        heartbeatProgress += 0.02 // Speed of individual heartbeat drawing
        animationFrame = requestAnimationFrame(animateHeartlinesSequentially)
      } else {
        // Drawing complete, now start fading out
        isFadingOut = true
        animationFrame = requestAnimationFrame(animateHeartlinesSequentially)
      }
    } else {
      // Fading out phase
      if (fadeOutProgress < 1) {
        fadeOutProgress += 0.02 // Speed of fade out
        animationFrame = requestAnimationFrame(animateHeartlinesSequentially)
      } else {
        // Fade out complete, move to next heartbeat
        lastHeartbeatFinishTime = now
        currentHeartbeatIndex++
        heartbeatProgress = 0
        fadeOutProgress = 0
        isFadingOut = false
        animationFrame = requestAnimationFrame(animateHeartlinesSequentially)
      }
    }
  }

  // Setup animation after paths are rendered
  async function setupAnimation() {
    // Start animation
    animationFrame = requestAnimationFrame(animateHeartlinesSequentially)
  }

  // Initialize component
  onMount(() => {
    // Use an immediately invoked async function to handle async operations
    ;(async () => {
      try {
        // Load heatwave data from CSV
        await loadHeatwaveData()

        // Setup animation after a short delay to ensure DOM is ready
        setTimeout(() => {
          setupAnimation()
        }, 100)
      } catch (error) {
        console.error("Error during initialization:", error)
      }
    })()

    // Return cleanup function
    return () => {
      // Clean up on component unmount
      if (animationFrame) {
        cancelAnimationFrame(animationFrame)
      }
    }
  })
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

    <svg {width} {height} viewBox="0 0 {width} {height}" preserveAspectRatio="xMidYMid meet">
      <!-- Black background -->
      <rect width="100%" height="100%" fill="black" />

      <!-- Subtle grid lines -->
      {#each Array(15) as _, i}
        <line x1="0" y1={i * (height / 15)} x2={width} y2={i * (height / 15)} stroke="rgba(0, 100, 0, 0.15)" stroke-width="1" />
      {/each}
      {#each Array(20) as _, i}
        <line x1={i * (width / 20)} y1="0" x2={i * (width / 20)} y2={height} stroke="rgba(0, 100, 0, 0.15)" stroke-width="1" />
      {/each}

      <!-- Single heartbeat that will change through animation -->
      {#if heartbeats.length > 0 && currentHeartbeatIndex < heartbeats.length}
        <g class="heartbeat-group">
          <!-- Debug visualization of the R peak height -->
          <line
            x1="0"
            y1={baselineY - heartbeats[currentHeartbeatIndex].rHeight}
            x2="20"
            y2={baselineY - heartbeats[currentHeartbeatIndex].rHeight}
            stroke="yellow"
            stroke-width="1"
            opacity="0.5"
          />

          <path
            d={generateHeartlinePath(heartbeats[currentHeartbeatIndex])}
            stroke={heartbeats[currentHeartbeatIndex].color}
            stroke-width="2"
            fill="none"
            opacity={isFadingOut
              ? heartbeats[currentHeartbeatIndex].opacity * (1 - fadeOutProgress)
              : heartbeats[currentHeartbeatIndex].opacity * Math.min(heartbeatProgress * 2, 1)}
            class="heartline-path"
            style="stroke-dasharray: {heartbeatProgress < 1 ? '1000' : '0'}; 
                   stroke-dashoffset: {heartbeatProgress < 1 ? 1000 * (1 - heartbeatProgress) : '0'}"
          />

          <!-- Text labels removed as requested -->
        </g>
      {/if}
    </svg>

    <div class="legend">
      <h2>Heatwave ECG Visualization</h2>
      <p>Each heartbeat represents a heatwave, with R-peak height scaled to the highest temperature.</p>
      <p>
        Data source: <a href="https://raw.githubusercontent.com/sophievanderhorst/data/refs/heads/main/heatwaves.csv" target="_blank"
          >Heatwave Dataset</a
        >
      </p>
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
    margin: 0 auto; /* Center horizontally */
    max-width: 1200px; /* Limit maximum width */
  }

  svg {
    width: 100%;
    max-width: 1000px;
    height: 70vh;
    margin: 0 auto; /* Center horizontally */
    display: block; /* Needed for margin auto to work */
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

  .loading,
  .error {
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
