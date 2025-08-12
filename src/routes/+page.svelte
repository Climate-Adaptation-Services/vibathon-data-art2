<script lang="ts">
  import { onMount } from "svelte"
  import * as d3 from "d3"

  // Audio for heartbeat sound
  let heartbeatSound: HTMLAudioElement
  let isPlaying = false
  
  // Reference to mini graph container for D3
  let miniGraphContainer: HTMLElement

  // Heartbeat parameters
  const width = 800
  const height = 300
  const baselineY = height / 2
  // Center point for the heartbeat visualization
  const centerX = width / 2

  // Animation state
  let progress = 0
  let hasStarted = false

  // Heatwave data parameters
  interface HeatwaveData {
    startDate: string
    endDate: string
    duration: number
    tropicalDays: number
    highestTemp: number
  }

  // Global variables for temperature range
  let globalMinTemp = 30
  let globalMaxTemp = 40

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
    year: string // Added year for timing and display
    duration?: number // Duration in days for wave length scaling
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

    // Generate much brighter colors from deep orange to deep red
    // Deep orange: rgb(255, 140, 0) to Deep red: rgb(220, 20, 20)
    const r = 255 - Math.floor(35 * normalizedTemp) // Slightly decrease red for deeper reds
    const g = Math.floor(140 * (1 - normalizedTemp)) // Decrease green as temp increases
    const b = Math.floor(30 * (1 - normalizedTemp)) // Very low blue component
    
    // Add brightness and saturation
    return `rgb(${r}, ${g}, ${b})`
  }

  // Format date to show month name without year (e.g., "June 15" instead of "2023-06-15")
  function formatDate(dateString: string): string {
    const date = new Date(dateString)
    const monthNames = [
      "January", "February", "March", "April", "May", "June",
      "July", "August", "September", "October", "November", "December"
    ]
    const month = monthNames[date.getMonth()]
    const day = date.getDate()
    return `${month} ${day}`
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
        const values = line.split(",") // GitHub CSV uses comma separators
        const highestTemp = parseFloat(values[highestTempIndex]) || 0

        // Log every 10th record for debugging
        if (index % 10 === 0) {
          console.log(`Record ${index} - Date: ${values[startDateIndex]}, Temp: ${highestTemp}°C`)
        }

        return {
          startDate: values[startDateIndex],
          endDate: values[endDateIndex],
          duration: parseInt(values[durationIndex]) || 0,
          tropicalDays: parseInt(values[tropicalDaysIndex]) || 0,
          highestTemp: highestTemp,
        }
      })

      console.log(`Loaded ${heatwaves.length} heatwave records`)

      // Log the first few records to inspect the data
      console.log("First 5 heatwave records:", heatwaves.slice(0, 5))

      // Find min and max temperatures for better scaling
      const temperatures = heatwaves.map((hw) => hw.highestTemp)
      const minDataTemp = Math.min(...temperatures)
      const maxDataTemp = Math.max(...temperatures)
      console.log(`Actual temperature range in data: ${minDataTemp}°C to ${maxDataTemp}°C`)

      // Store the temperature range globally for use in heartbeat generation
      globalMinTemp = minDataTemp
      globalMaxTemp = maxDataTemp

      // Use the original order from the CSV (first row to last row)
      console.log("Processing heartbeats in original data order (first row to last row)")

      // Generate heartbeats from all heatwave data in original order
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
    const randomFactor = Math.random() * 0.5 + 0.5 // 0.5 to 1.0

    // Extract year from the start date (format: YYYY-MM-DD)
    const dateParts = heatwave.startDate.split("-")
    // Get the year from the first part of the date
    let year = dateParts.length >= 1 ? dateParts[0] : "Unknown"

    // Use consistent bright red color for all heartbeats
    const color = "rgba(255, 50, 50, 1)" // Bright red

    // Scale R peak height based on temperature (30-42°C maps to 50-150px)
    const temp = heatwave.highestTemp
    const minTemp = 30
    const maxTemp = 42
    const minHeight = 50
    const maxHeight = 150

    // Normalize temperature and map to height range
    const normalizedTemp = Math.min(Math.max(temp - minTemp, 0), maxTemp - minTemp) / (maxTemp - minTemp)
    const rHeight = minHeight + normalizedTemp * (maxHeight - minHeight)

    return {
      pHeight: 10 + randomFactor * 15, // P wave: 10-25px
      qDepth: 5 + randomFactor * 10, // Q wave: 5-15px
      rHeight: rHeight, // R peak: 50-150px based on temperature
      sDepth: 10 + randomFactor * 15, // S wave: 10-25px
      tHeight: 15 + randomFactor * 20, // T wave: 15-35px
      color: color,
      opacity: 0.8,
      heatwaveData: heatwave,
      year: year, // Store the year for timing and display
      duration: heatwave.duration, // Store duration for wave length scaling
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
      year: "unknown"
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
    // Scale wave length based on duration (1-30 days maps to 0.6-1.2 scale factor)
    const duration = params.duration || 5 // Default to 5 days if no duration
    const minDuration = 1
    const maxDuration = 30
    const minScale = 0.6
    const maxScale = 1.2
    const normalizedDuration = Math.min(Math.max(duration - minDuration, 0), maxDuration - minDuration) / (maxDuration - minDuration)
    const lengthScale = minScale + normalizedDuration * (maxScale - minScale)
    
    // Center the heartbeat in the SVG
    const baseLineWidth = width * 0.8
    const lineWidth = baseLineWidth * lengthScale // Scale width based on duration
    const startX = (width - lineWidth) / 2 // Center the heartbeat

    // Extract parameters
    const { pHeight, qDepth, rHeight, sDepth, tHeight } = params

    // Add subtle ECG noise and variations
    const subtleNoise = () => (Math.random() - 0.5) * 0.15 + 1 // 0.925 to 1.075 multiplier for subtle variation
    const baselineNoise = () => (Math.random() - 0.5) * 1 // ±1px baseline drift
    const microNoise = () => (Math.random() - 0.5) * 0.5 // ±0.5px micro variations
    
    // Apply subtle noise to all wave components
    const noisyPHeight = pHeight * subtleNoise()
    const noisyQDepth = qDepth * subtleNoise()
    const noisyRHeight = rHeight * subtleNoise()
    const noisySDepth = sDepth * subtleNoise()
    const noisyTHeight = tHeight * subtleNoise()

    // Add subtle horizontal timing variations
    const timingNoise = () => (Math.random() - 0.5) * 0.01 // ±1% timing variation
    
    // Create baseline drift throughout the signal
    const baselineDrift1 = baselineNoise()
    const baselineDrift2 = baselineNoise()
    const baselineDrift3 = baselineNoise()
    const baselineDrift4 = baselineNoise()

    // Log the actual R peak position
    const rPeakY = baselineY - noisyRHeight
    console.log(`R peak will be drawn at Y: ${rPeakY} (baselineY: ${baselineY} - rHeight: ${noisyRHeight})`)

    // Create realistic ECG segments with noise, micro-variations, and baseline drift
    return [
      // Start with baseline (with drift)
      `M ${startX} ${baselineY + baselineDrift1 + microNoise()}`,
      `L ${startX + lineWidth * (0.15 + timingNoise())} ${baselineY + baselineDrift1 + microNoise()}`,
      `L ${startX + lineWidth * (0.2 + timingNoise())} ${baselineY + baselineDrift1 + microNoise()}`,

      // P wave (small bump with realistic shape)
      `L ${startX + lineWidth * (0.23 + timingNoise())} ${baselineY - noisyPHeight * 0.3 + baselineDrift1 + microNoise()}`, // P wave start
      `L ${startX + lineWidth * (0.25 + timingNoise())} ${baselineY - noisyPHeight + baselineDrift1 + microNoise()}`, // P peak
      `L ${startX + lineWidth * (0.27 + timingNoise())} ${baselineY - noisyPHeight * 0.3 + baselineDrift1 + microNoise()}`, // P wave end
      `L ${startX + lineWidth * (0.3 + timingNoise())} ${baselineY + baselineDrift2 + microNoise()}`,

      // PR segment (flat with slight drift)
      `L ${startX + lineWidth * (0.32 + timingNoise())} ${baselineY + baselineDrift2 + microNoise()}`,
      `L ${startX + lineWidth * (0.35 + timingNoise())} ${baselineY + baselineDrift2 + microNoise()}`,

      // QRS complex (sharp and realistic)
      // Q wave (small downward)
      `L ${startX + lineWidth * (0.37 + timingNoise())} ${baselineY + noisyQDepth + baselineDrift2 + microNoise()}`,
      
      // R wave (sharp tall spike with slight asymmetry)
      `L ${startX + lineWidth * (0.395 + timingNoise())} ${baselineY - noisyRHeight * 0.9 + baselineDrift2 + microNoise()}`, // R wave rise
      `L ${startX + lineWidth * (0.4 + timingNoise())} ${baselineY - noisyRHeight + baselineDrift2 + microNoise()}`, // R peak
      `L ${startX + lineWidth * (0.405 + timingNoise())} ${baselineY - noisyRHeight * 0.8 + baselineDrift2 + microNoise()}`, // R wave fall

      // S wave (sharp downward)
      `L ${startX + lineWidth * (0.43 + timingNoise())} ${baselineY + noisySDepth + baselineDrift3 + microNoise()}`,

      // Return to baseline with slight overshoot
      `L ${startX + lineWidth * (0.45 + timingNoise())} ${baselineY - 2 + baselineDrift3 + microNoise()}`, // Slight overshoot
      `L ${startX + lineWidth * (0.47 + timingNoise())} ${baselineY + baselineDrift3 + microNoise()}`,

      // ST segment (flat with drift and micro-variations)
      `L ${startX + lineWidth * (0.5 + timingNoise())} ${baselineY + baselineDrift3 + microNoise()}`,
      `L ${startX + lineWidth * (0.53 + timingNoise())} ${baselineY + baselineDrift3 + microNoise()}`,
      `L ${startX + lineWidth * (0.55 + timingNoise())} ${baselineY + baselineDrift3 + microNoise()}`,

      // T wave (broader, more realistic shape)
      `L ${startX + lineWidth * (0.57 + timingNoise())} ${baselineY - noisyTHeight * 0.2 + baselineDrift3 + microNoise()}`, // T wave start
      `L ${startX + lineWidth * (0.6 + timingNoise())} ${baselineY - noisyTHeight * 0.8 + baselineDrift4 + microNoise()}`, // T wave rise
      `L ${startX + lineWidth * (0.62 + timingNoise())} ${baselineY - noisyTHeight + baselineDrift4 + microNoise()}`, // T peak
      `L ${startX + lineWidth * (0.65 + timingNoise())} ${baselineY - noisyTHeight * 0.6 + baselineDrift4 + microNoise()}`, // T wave fall
      `L ${startX + lineWidth * (0.68 + timingNoise())} ${baselineY - noisyTHeight * 0.1 + baselineDrift4 + microNoise()}`, // T wave end
      `L ${startX + lineWidth * (0.7 + timingNoise())} ${baselineY + baselineDrift4 + microNoise()}`,

      // Final baseline with continued drift and noise
      `L ${startX + lineWidth * (0.8 + timingNoise())} ${baselineY + baselineDrift4 + microNoise()}`,
      `L ${startX + lineWidth * (0.9 + timingNoise())} ${baselineY + baselineDrift4 + microNoise()}`,
      `L ${startX + lineWidth} ${baselineY + baselineDrift4 + microNoise()}`,
    ].join(" ")
  }

  // Animation variables
  let currentDate = new Date(1911, 0, 1) // Start at January 1911
  let monthUpdateInterval = 20 // Update every 20ms for 5x faster progression
  let lastMonthUpdateTime = 0
  let animationFrame: number | null = null
  
  // Current heartbeat state
  let activeHeartbeat: HeartbeatParams | null = null
  let lastHeatwaveInfo: HeartbeatParams | null = null
  let heartbeatProgress = 0
  let fadeOutProgress = 0
  let isFadingOut = false
  let isShowingHeartbeat = false
  
  // Store displayed heartbeats for mini graph
  let displayedHeartbeats: { year: string; temp: number; date: string }[] = []

  // Fixed x-axis range
  const startYear = 1911 // First year in dataset
  const endYear = 2050 // End year for projection

  // D3 scale for mapping dates to x positions (with padding for labels)
  const timeScale = d3.scaleTime()
    .domain([new Date(startYear, 0, 1), new Date(endYear, 0, 1)])
    .range([40, 760])

  // Function to calculate x position based on date using D3
  function calculateXPosition(dateStr: string): number {
    try {
      // Parse the date string (format: YYYY-MM-DD)
      const parts = dateStr.split("-")
      if (parts.length !== 3) {
        console.warn(`Invalid date format: ${dateStr}, expected YYYY-MM-DD`)
        return 0
      }

      // Extract year, month, day
      const year = parseInt(parts[0])
      const month = parseInt(parts[1]) - 1 // 0-indexed months
      const day = parseInt(parts[2])

      // Validate year
      if (isNaN(year)) {
        console.warn(`Invalid year: ${year}`)
        return 0
      }
      
      // Create a date object and clamp it to the valid range
      const date = new Date(year, month, day)
      const clampedDate = new Date(
        Math.max(startYear, Math.min(endYear, year)),
        month,
        day
      )
      
      // Use D3 scale to map the date to an x position
      return Math.max(1, Math.min(799, timeScale(clampedDate)))
    } catch (error) {
      console.error(`Error calculating position for date: ${dateStr}`, error)
      return 0
    }
  }

  // Main animation loop - handles both time progression and heartbeat animation
  function mainAnimationLoop(): void {
    const now = performance.now()
    
    if (!hasStarted) return
    
    // Initialize timing
    if (lastMonthUpdateTime === 0) {
      lastMonthUpdateTime = now
    }
    
    // Update time progression
    if (now - lastMonthUpdateTime >= monthUpdateInterval) {
      // Advance one month - create new Date object for reactivity
      const newDate = new Date(currentDate)
      newDate.setMonth(newDate.getMonth() + 1)
      currentDate = newDate
      lastMonthUpdateTime = now
      
      console.log(`Current date: ${currentDate.toLocaleDateString()}`)
      
      // Check if we've reached 2050
      if (currentDate.getFullYear() >= 2050 && currentDate.getMonth() >= 11) {
        console.log("Reached December 2050 - stopping")
        hasStarted = false
        return
      }
      
      // Check for heatwave events at current date
      checkForHeatwaveAtCurrentDate()
    }
    
    // Handle heartbeat animation if active
    if (isShowingHeartbeat && activeHeartbeat) {
      if (!isFadingOut) {
        // Drawing phase
        if (heartbeatProgress < 1) {
          heartbeatProgress += 0.05 // Animation speed
        } else {
          isFadingOut = true
        }
      } else {
        // Fade out phase
        if (fadeOutProgress < 1) {
          fadeOutProgress += 0.02
        } else {
          // Heartbeat complete
          isShowingHeartbeat = false
          activeHeartbeat = null
          heartbeatProgress = 0
          fadeOutProgress = 0
          isFadingOut = false
        }
      }
    }
    
    // Continue animation loop
    if (hasStarted) {
      animationFrame = requestAnimationFrame(mainAnimationLoop)
    }
  }
  
  // Check for heatwave events at current date and trigger heartbeat
  function checkForHeatwaveAtCurrentDate(): void {
    if (!heartbeats.length) return
    
    const currentYearMonth = `${currentDate.getFullYear()}-${String(currentDate.getMonth() + 1).padStart(2, '0')}`
    
    // Find matching heatwave
    const matchingHeartbeat = heartbeats.find(hb => 
      hb.heatwaveData && hb.heatwaveData.startDate.startsWith(currentYearMonth)
    )
    
    if (matchingHeartbeat) {
      console.log(`Found heatwave for ${currentYearMonth}:`, matchingHeartbeat.heatwaveData)
      
      // Trigger heartbeat
      activeHeartbeat = matchingHeartbeat
      lastHeatwaveInfo = matchingHeartbeat // Store for persistent display
      isShowingHeartbeat = true
      heartbeatProgress = 0
      fadeOutProgress = 0
      isFadingOut = false
      
      // Add to displayed heartbeats for timeline
      if (matchingHeartbeat.heatwaveData) {
        displayedHeartbeats = [
          ...displayedHeartbeats,
          {
            year: matchingHeartbeat.year,
            temp: matchingHeartbeat.heatwaveData.highestTemp,
            date: matchingHeartbeat.heatwaveData.startDate,
          },
        ]
      }
      
      // Update timeline visualization
      updateD3Timeline()
      
      // Play sound
      playHeartbeatSound()
    }
  }
  

  // Play heartbeat sound
  function playHeartbeatSound() {
    if (heartbeatSound) {
      // Reset the sound to the beginning if it's already playing
      heartbeatSound.currentTime = 0

      // Play the sound
      heartbeatSound.play().catch((error) => {
        console.error("Error playing heartbeat sound:", error)
      })
    }
  }

  // Start the visualization
  function startVisualization() {
    if (hasStarted) return // Prevent multiple starts

    hasStarted = true
    
    // Reset all animation variables
    currentDate = new Date(1911, 0, 1)
    lastMonthUpdateTime = 0
    displayedHeartbeats = []
    
    // Reset heartbeat state
    activeHeartbeat = null
    lastHeatwaveInfo = null
    isShowingHeartbeat = false
    heartbeatProgress = 0
    fadeOutProgress = 0
    isFadingOut = false
    
    // Initialize audio if not already done
    if (!heartbeatSound) {
      heartbeatSound = new Audio("/heartbeat-loop-96879 (1).mp3")
      heartbeatSound.volume = 0.5
    }

    // Cancel any existing animation frame
    if (animationFrame) {
      cancelAnimationFrame(animationFrame)
      animationFrame = null
    }
    
    // Start main animation loop
    console.log("Starting visualization")
    animationFrame = requestAnimationFrame(mainAnimationLoop)
  }

  // No longer needed - using bars instead of path

  // Setup initial state
  async function setupAnimation() {
    // Don't automatically start animation - wait for play button
    console.log("Animation ready - waiting for play button")
    
    // Initialize D3 timeline
    initD3Timeline()
  }
  
  // Initialize D3 timeline
  function initD3Timeline() {
    if (!miniGraphContainer) return
    
    // Clear any existing content
    d3.select(miniGraphContainer).selectAll("*").remove()
    
    // Create SVG container
    const svg = d3.select(miniGraphContainer)
      .append("svg")
      .attr("width", 800)
      .attr("height", 60)
      .attr("viewBox", "0 0 800 60")
      .style("display", "block")
      .style("margin", "0 auto")
    
    // Add background
    svg.append("rect")
      .attr("x", 0)
      .attr("y", 0)
      .attr("width", 800)
      .attr("height", 60)
      .attr("fill", "rgba(0,0,0,0.3)")
    
    // Add border to clearly define timeline boundaries
    svg.append("rect")
      .attr("x", 0)
      .attr("y", 0)
      .attr("width", 800)
      .attr("height", 60)
      .attr("fill", "none")
      .attr("stroke", "rgba(255,255,255,0.2)")
      .attr("stroke-width", 1)
    
    // Create time axis
    const xAxis = d3.axisBottom(timeScale)
      .tickFormat(d3.timeFormat("%Y"))
      .ticks(10)
    
    // Add dashed center line
    svg.append("line")
      .attr("x1", 40)
      .attr("y1", 30)
      .attr("x2", 760)
      .attr("y2", 30)
      .attr("stroke", "rgba(0,255,0,0.5)")
      .attr("stroke-width", 1)
      .attr("stroke-dasharray", "4 2")
    
    // Add axis with custom styling
    svg.append("g")
      .attr("transform", "translate(0, 30)")
      .call(xAxis)
      .selectAll("text")
      .attr("fill", "rgba(0,255,0,0.9)")
      .attr("font-size", 12)
      .attr("font-family", "Courier New, monospace")
      .attr("text-anchor", "middle")
      .attr("dy", 20)
    
    // Style the axis
    svg.selectAll(".domain").remove() // Remove the axis line
    svg.selectAll(".tick line")
      .attr("stroke", "rgba(0,255,0,0.4)")
      .attr("y1", -5)
      .attr("y2", 5)
    
    // Create a group for the mini graph points
    svg.append("g")
      .attr("class", "mini-graph-points")
  }
  
  // Update D3 timeline with enhanced circles focusing on latest
  function updateD3Timeline() {
    if (!miniGraphContainer) return
    
    const svg = d3.select(miniGraphContainer).select("svg")
    
    // Log the number of displayed heartbeats for debugging
    console.log(`Updating D3 timeline with ${displayedHeartbeats.length} heartbeats`)
    
    // Enhanced mini graph circles with focus on latest
    const points = svg.select(".mini-graph-points")
      .selectAll("circle")
      .data(displayedHeartbeats, (d: any, i: any) => `${d.date}-${i}`) // Use a key function to better track data points
    
    // Remove old points
    points.exit().remove()
    
    // Add new points with enhanced styling
    points.enter()
      .append("circle")
      .merge(points)
      .attr("cx", (d: any) => {
        try {
          return timeScale(new Date(d.date.split("-")[0], d.date.split("-")[1] - 1, d.date.split("-")[2]))
        } catch (e) {
          console.error("Error calculating circle position:", e, d)
          return 0
        }
      })
      .attr("cy", 30)
      .attr("r", (d: any, i: any) => {
        const isLatest = i === displayedHeartbeats.length - 1
        return isLatest ? 6 : 4 // Latest circle is larger
      })
      .attr("fill", (d: any, i: any) => {
        const isLatest = i === displayedHeartbeats.length - 1
        return isLatest ? "rgba(255,100,100,1)" : "rgba(255,50,50,1)"
      })
      .attr("opacity", (d: any, i: any) => {
        const isLatest = i === displayedHeartbeats.length - 1
        return isLatest ? 1.0 : 0.7 // Latest is fully visible, others are clearly visible
      })
      .attr("stroke", (d: any, i: any) => {
        const isLatest = i === displayedHeartbeats.length - 1
        return isLatest ? "rgba(255,255,255,0.8)" : "none"
      })
      .attr("stroke-width", (d: any, i: any) => {
        const isLatest = i === displayedHeartbeats.length - 1
        return isLatest ? 1 : 0
      })
      .attr("filter", (d: any, i: any) => {
        const isLatest = i === displayedHeartbeats.length - 1
        return isLatest ? "drop-shadow(0 0 6px rgba(255,100,100,0.6))" : "none"
      })
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

        // Preload the heartbeat sound
        heartbeatSound = new Audio("/heartbeat-loop-96879 (1).mp3")
        heartbeatSound.volume = 0.5
        heartbeatSound.load()
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

    <!-- Title and Introduction -->
    <div class="visualization-header">
      <h1 class="main-title">Climate Heartbeat Monitor</h1>
      <p class="intro-text">Each heartbeat represents a <span class="heatwave-highlight">heatwave</span> event. Watch as climate patterns pulse through time, revealing the rhythm of our changing planet. <span class="data-source">Data: <a href="https://raw.githubusercontent.com/sophievanderhorst/data/refs/heads/main/heatwaves.csv" target="_blank">Heatwave Dataset</a></span></p>
    </div>

    <!-- Medical Monitor Style Information Display - Always show the date -->
    <div class="monitor-info">
      <!-- Year and Historic/Future title above - show only year -->
      <div class="monitor-header">
        <div class="year-display">{currentDate.getFullYear()}</div>
        <div class="time-direction">{currentDate.getFullYear() > 2023 ? "PROJECTED" : "HISTORIC"}</div>
      </div>
      
      <!-- Show last heatwave information (persists until next heatwave) -->
      {#if lastHeatwaveInfo && lastHeatwaveInfo.heatwaveData}
        <div class="monitor-data">
          <div class="data-item">
            <span class="label">DATE:</span>
            <span class="value">{formatDate(lastHeatwaveInfo.heatwaveData.startDate)} - {formatDate(lastHeatwaveInfo.heatwaveData.endDate)}</span>
          </div>
          <div class="data-separator">|</div>
          <div class="data-item">
            <span class="label">DURATION:</span>
            <span class="value">{lastHeatwaveInfo.heatwaveData.duration} DAYS</span>
          </div>
          <div class="data-separator">|</div>
          <div class="data-item">
            <span class="label">MAX TEMP:</span>
            <span class="value">{lastHeatwaveInfo.heatwaveData.highestTemp}°C</span>
          </div>
          <div class="data-separator">|</div>
          <div class="data-item">
            <span class="label">TROPICAL DAYS:</span>
            <span class="value">{lastHeatwaveInfo.heatwaveData.tropicalDays}</span>
          </div>
        </div>
      {/if}
    </div>

    <svg {width} {height} viewBox="0 0 {width} {height}" preserveAspectRatio="xMidYMid meet">
      <!-- Black background -->
      <rect width="100%" height="100%" fill="black" />
      
      <!-- Monitor-style border -->
      <rect x="2" y="2" width={width - 4} height={height - 4} fill="none" stroke="rgba(0, 255, 0, 0.3)" stroke-width="2" rx="8" />

      <!-- ECG grid pattern - fine grid -->
      {#each Array(60) as _, i}
        <line x1="0" y1={i * (height / 60)} x2={width} y2={i * (height / 60)} stroke="rgba(0, 100, 0, 0.08)" stroke-width="0.5" />
      {/each}
      {#each Array(80) as _, i}
        <line x1={i * (width / 80)} y1="0" x2={i * (width / 80)} y2={height} stroke="rgba(0, 100, 0, 0.08)" stroke-width="0.5" />
      {/each}
      
      <!-- ECG grid pattern - major grid lines -->
      {#each Array(12) as _, i}
        <line x1="0" y1={i * (height / 12)} x2={width} y2={i * (height / 12)} stroke="rgba(0, 150, 0, 0.2)" stroke-width="1" />
      {/each}
      {#each Array(16) as _, i}
        <line x1={i * (width / 16)} y1="0" x2={i * (width / 16)} y2={height} stroke="rgba(0, 150, 0, 0.2)" stroke-width="1" />
      {/each}
      
      <!-- Prominent baseline -->
      <line x1="0" y1={baselineY} x2={width} y2={baselineY} stroke="rgba(0, 255, 0, 0.4)" stroke-width="1.5" stroke-dasharray="5,5" />
      
      <!-- Monitor-style corner indicators -->
      <circle cx="20" cy="20" r="3" fill="rgba(0, 255, 0, 0.6)" />
      <circle cx={width - 20} cy="20" r="3" fill="rgba(0, 255, 0, 0.6)" />
      <circle cx="20" cy={height - 20} r="3" fill="rgba(0, 255, 0, 0.6)" />
      <circle cx={width - 20} cy={height - 20} r="3" fill="rgba(0, 255, 0, 0.6)" />
      
      <!-- Arrow Play Button (only show if not started) -->
      {#if !hasStarted}
        <g 
          class="play-button-svg" 
          on:click={startVisualization} 
          on:keydown={(e) => e.key === 'Enter' || e.key === ' ' ? startVisualization() : null}
          role="button"
          tabindex="0"
          aria-label="Start climate heartbeat visualization"
          style="cursor: pointer;"
        >
          <!-- Semi-transparent background circle -->
          <circle 
            cx={width / 2} 
            cy={height / 2} 
            r="40" 
            fill="rgba(0, 0, 0, 0.7)" 
            stroke="rgba(0, 255, 0, 0.8)" 
            stroke-width="2"
            class="play-bg-circle"
          />
          <!-- Play arrow triangle -->
          <polygon 
            points="{width / 2 - 12},{height / 2 - 15} {width / 2 - 12},{height / 2 + 15} {width / 2 + 18},{height / 2}"
            fill="rgba(0, 255, 0, 0.9)"
            class="play-arrow"
          />
        </g>
      {/if}

      <!-- Show heartbeat when active, otherwise show flatline -->
      {#if isShowingHeartbeat && activeHeartbeat}
        <g class="heartbeat-group">
          <!-- ECG trace with glow effect -->
          <g>
            <!-- Glow effect (background) -->
            <path
              d={generateHeartlinePath(activeHeartbeat)}
              stroke={activeHeartbeat.color}
              stroke-width="6"
              fill="none"
              opacity={isFadingOut
                ? activeHeartbeat.opacity * (1 - fadeOutProgress) * 0.3
                : activeHeartbeat.opacity * 0.3}
              pathLength="1000"
              style="stroke-dasharray: {heartbeatProgress < 1 ? `${1000 * heartbeatProgress} 1000` : '1000 0'}; 
                     stroke-dashoffset: 0;
                     filter: blur(2px)"
            />
            <!-- Main ECG trace -->
            <path
              d={generateHeartlinePath(activeHeartbeat)}
              stroke={activeHeartbeat.color}
              stroke-width="3"
              fill="none"
              opacity={isFadingOut
                ? activeHeartbeat.opacity * (1 - fadeOutProgress)
                : activeHeartbeat.opacity}
              class="heartline-path"
              pathLength="1000"
              style="stroke-dasharray: {heartbeatProgress < 1 ? `${1000 * heartbeatProgress} 1000` : '1000 0'}; 
                     stroke-dashoffset: 0;
                     stroke-linecap: round;
                     stroke-linejoin: round"
            />
          </g>
        </g>
      {:else if hasStarted}
        <!-- Show flatline when no heatwave is present -->
        <g class="flatline-group">
          <line 
            x1="0" 
            y1={baselineY} 
            x2={width} 
            y2={baselineY} 
            stroke="rgba(0, 255, 0, 0.7)" 
            stroke-width="2" 
            class="flatline"
          />
          
          <!-- Small blip that moves along the flatline to show activity -->
          <circle
            cx={(width * 0.2) + Math.sin(performance.now() / 1000) * (width * 0.6)}
            cy={baselineY}
            r="2"
            fill="rgba(0, 255, 0, 0.8)"
            class="flatline-blip"
          />
        </g>
      {/if}
    </svg>

    <!-- Mini circle graph with D3.js timeline from 1911 to 2050 -->
    <div class="mini-graph-container" bind:this={miniGraphContainer}>
      <!-- D3 will render the timeline here -->
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
    justify-content: flex-start;
    position: relative;
    margin: 0 auto; /* Center horizontally */
    max-width: 1200px; /* Limit maximum width */
    padding-top: 2vh; /* Reduced top padding to move content higher */
  }

  svg {
    width: 100%;
    max-width: 1000px;
    height: 55vh; /* Reduced height for more compact design */
    margin: 0.5vh auto; /* Reduced vertical margin to move higher */
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


  .legend {
    position: absolute;
    bottom: 20px;
    left: 0;
    right: 0;
    text-align: center;
    background-color: rgba(0, 10, 0, 0.85);
    padding: 15px 20px;
    border-top: 2px solid rgba(0, 255, 0, 0.4);
    backdrop-filter: blur(3px);
  }

  .legend h2 {
    margin: 0 0 12px 0;
    font-size: 1.3rem;
    color: rgba(0, 255, 0, 0.9);
    font-family: 'Courier New', monospace;
    font-weight: bold;
    text-shadow: 0 0 8px rgba(0, 255, 0, 0.5);
    letter-spacing: 1px;
  }

  .legend p {
    margin: 8px 0;
    font-size: 0.95rem;
    color: rgba(200, 200, 200, 0.9);
    line-height: 1.4;
  }

  .legend a {
    color: rgba(0, 255, 0, 0.8);
    text-decoration: none;
    font-weight: bold;
  }

  .legend a:hover {
    color: rgba(0, 255, 0, 1);
    text-decoration: underline;
    text-shadow: 0 0 5px rgba(0, 255, 0, 0.6);
  }
  
  .heatwave-highlight {
    color: rgba(255, 50, 50, 1);
    font-weight: bold;
    text-shadow: 0 0 4px rgba(255, 50, 50, 0.6);
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

  /* Arrow Play Button Styling */
  .play-button-svg {
    transition: all 0.3s ease;
  }
  
  .play-button-svg:hover .play-bg-circle {
    fill: rgba(0, 0, 0, 0.9);
    stroke: rgba(0, 255, 0, 1);
    stroke-width: 3;
  }
  
  .play-button-svg:hover .play-arrow {
    fill: rgba(0, 255, 0, 1);
  }
  
  .play-button-svg:active {
    transform: scale(0.95);
  }
  
  .play-bg-circle {
    transition: all 0.2s ease;
  }
  
  .play-arrow {
    transition: all 0.2s ease;
  }

  .mini-graph-container {
    position: absolute;
    width: 100%;
    max-width: 900px;
    height: 80px;
    bottom: 280px; /* Positioned higher, closer to heartbeat monitor */
    left: 50%;
    transform: translateX(-50%);
    z-index: 2;
    overflow: hidden;
  }

  
  /* Pulse animation for current heartbeat point */
  @keyframes pulse {
    0% {
      r: 12;
      opacity: 0.6;
    }
    50% {
      r: 16;
      opacity: 0.3;
    }
    100% {
      r: 12;
      opacity: 0.6;
    }
  }
  
  /* Visualization Header */
  .visualization-header {
    text-align: center;
    margin: 5px auto 10px auto;
    max-width: 800px;
    padding: 0 20px;
  }
  
  .main-title {
    font-size: 2.5rem;
    font-weight: bold;
    color: rgba(0, 255, 0, 1);
    text-shadow: 0 0 15px rgba(0, 255, 0, 0.8);
    margin: 0 0 15px 0;
    font-family: 'Courier New', monospace;
    letter-spacing: 2px;
  }
  
  .intro-text {
    font-size: 1.1rem;
    color: rgba(200, 200, 200, 0.9);
    line-height: 1.5; /* Slightly reduced line height */
    margin: 0 0 15px 0; /* Reduced bottom margin */
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  }

  /* Medical Monitor Information Display */
  .monitor-info {
    position: relative;
    margin: 3px auto 8px auto;
    background: rgba(0, 20, 0, 0.9);
    border: 2px solid rgba(0, 255, 0, 0.6);
    border-radius: 8px;
    padding: 12px 20px; /* Reduced padding for more compact design */
    font-family: 'Courier New', monospace;
    text-align: center;
    box-shadow: 0 0 20px rgba(0, 255, 0, 0.3);
    backdrop-filter: blur(5px);
    max-width: 800px;
  }
  
  .monitor-header {
    display: flex;
    flex-direction: column;
    align-items: center;
    margin-bottom: 12px;
    gap: 4px;
  }
  
  .year-display {
    font-size: 28px;
    font-weight: bold;
    color: rgba(0, 255, 0, 1);
    text-shadow: 0 0 8px rgba(0, 255, 0, 0.8);
    letter-spacing: 2px;
  }
  
  .month-display {
    font-size: 18px;
    color: rgba(0, 220, 0, 0.9);
    text-shadow: 0 0 5px rgba(0, 255, 0, 0.5);
    letter-spacing: 1px;
  }
  
  .time-direction {
    font-size: 12px;
    color: rgba(0, 200, 0, 0.9);
    font-weight: bold;
    letter-spacing: 1px;
  }
  
  .monitor-data {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 15px;
    flex-wrap: wrap;
  }
  
  .data-item {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 2px;
  }
  
  .data-item .label {
    font-size: 10px;
    color: rgba(0, 180, 0, 0.8);
    font-weight: bold;
    letter-spacing: 0.5px;
  }
  
  .data-item .value {
    font-size: 16px;
    color: rgba(255, 50, 50, 1);
    font-weight: bold;
    text-shadow: 0 0 4px rgba(255, 50, 50, 0.6);
  }
  
  .data-separator {
    color: rgba(0, 150, 0, 0.6);
    font-size: 16px;
    font-weight: bold;
  }
  
  
  /* Flatline animation */
  @keyframes flatline-pulse {
    0% {
      opacity: 0.5;
      stroke-width: 1.5;
    }
    50% {
      opacity: 0.7;
      stroke-width: 2;
    }
    100% {
      opacity: 0.5;
      stroke-width: 1.5;
    }
  }
  
  .flatline {
    animation: flatline-pulse 2s infinite ease-in-out;
  }
  
  @keyframes flatline-blip-pulse {
    0% {
      opacity: 0.7;
      r: 2;
    }
    50% {
      opacity: 1;
      r: 3;
    }
    100% {
      opacity: 0.7;
      r: 2;
    }
  }
  
  .flatline-blip {
    animation: flatline-blip-pulse 1s infinite ease-in-out;
  }
  
  /* Responsive adjustments */
  @media (max-width: 768px) {
    .main-title {
      font-size: 2rem;
      letter-spacing: 1px;
    }
    
    .intro-text {
      font-size: 1rem;
      padding: 0 10px;
    }
    
    .visualization-header {
      margin: 15px auto 20px auto;
    }
    
    .monitor-info {
      padding: 10px 15px;
      margin: 5px auto 5px auto;
    }
    
    .year-display {
      font-size: 22px;
    }
    
    .month-display {
      font-size: 16px;
    }
    
    .monitor-data {
      gap: 10px;
    }
    
    .data-item .value {
      font-size: 14px;
    }
  }
</style>
