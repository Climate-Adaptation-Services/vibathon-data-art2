<script lang="ts">
  import { onMount } from "svelte"

  // Audio for heartbeat sound
  let heartbeatSound: HTMLAudioElement
  let beepSound: HTMLAudioElement
  let isPlaying = false

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
    const monthNames = ["January", "February", "March", "April", "May", "June", "July", "August", "September", "October", "November", "December"]
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

    // Scale R peak height based on temperature (30-42°C maps to 50-150px)
    const temp = heatwave.highestTemp
    const minTemp = 30
    const maxTemp = 42
    const minHeight = 50
    const maxHeight = 150

    // Normalize temperature and map to height range
    const normalizedTemp = Math.min(Math.max(temp - minTemp, 0), maxTemp - minTemp) / (maxTemp - minTemp)
    const rHeight = minHeight + normalizedTemp * (maxHeight - minHeight)

    // Use fiery gradient colors for heartbeats based on temperature
    const tempRatio = normalizedTemp
    // Create fire-like color progression: orange → red → white-hot
    let color
    if (tempRatio < 0.3) {
      // Cool fires: deep orange
      color = `rgba(255, ${100 + Math.floor(55 * tempRatio)}, 0, 1)`
    } else if (tempRatio < 0.7) {
      // Hot fires: orange to red
      const redProgress = (tempRatio - 0.3) / 0.4
      color = `rgba(255, ${155 - Math.floor(100 * redProgress)}, ${Math.floor(20 * redProgress)}, 1)`
    } else {
      // Extreme heat: red to white-hot
      const whiteProgress = (tempRatio - 0.7) / 0.3
      const whiteComponent = Math.floor(100 * whiteProgress)
      color = `rgba(255, ${55 + whiteComponent}, ${20 + whiteComponent * 2}, 1)`
    }

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
      year: "unknown",
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

    // Use clean, stable values without random noise
    const cleanPHeight = pHeight
    const cleanQDepth = qDepth
    const cleanRHeight = rHeight
    const cleanSDepth = sDepth
    const cleanTHeight = tHeight

    // Log the actual R peak position
    const rPeakY = baselineY - cleanRHeight
    console.log(`R peak will be drawn at Y: ${rPeakY} (baselineY: ${baselineY} - rHeight: ${cleanRHeight})`)

    // Create clean, stable ECG segments
    return [
      // Start with baseline
      `M ${startX} ${baselineY}`,
      `L ${startX + lineWidth * 0.15} ${baselineY}`,
      `L ${startX + lineWidth * 0.2} ${baselineY}`,

      // P wave (small bump with realistic shape)
      `L ${startX + lineWidth * 0.23} ${baselineY - cleanPHeight * 0.3}`, // P wave start
      `L ${startX + lineWidth * 0.25} ${baselineY - cleanPHeight}`, // P peak
      `L ${startX + lineWidth * 0.27} ${baselineY - cleanPHeight * 0.3}`, // P wave end
      `L ${startX + lineWidth * 0.3} ${baselineY}`,

      // PR segment (flat)
      `L ${startX + lineWidth * 0.32} ${baselineY}`,
      `L ${startX + lineWidth * 0.35} ${baselineY}`,

      // QRS complex (sharp and clean)
      // Q wave (small downward)
      `L ${startX + lineWidth * 0.37} ${baselineY + cleanQDepth}`,

      // R wave (sharp tall spike)
      `L ${startX + lineWidth * 0.395} ${baselineY - cleanRHeight * 0.9}`, // R wave rise
      `L ${startX + lineWidth * 0.4} ${baselineY - cleanRHeight}`, // R peak
      `L ${startX + lineWidth * 0.405} ${baselineY - cleanRHeight * 0.8}`, // R wave fall

      // S wave (sharp downward)
      `L ${startX + lineWidth * 0.43} ${baselineY + cleanSDepth}`,

      // Return to baseline
      `L ${startX + lineWidth * 0.45} ${baselineY}`,
      `L ${startX + lineWidth * 0.47} ${baselineY}`,

      // ST segment (flat)
      `L ${startX + lineWidth * 0.5} ${baselineY}`,
      `L ${startX + lineWidth * 0.53} ${baselineY}`,
      `L ${startX + lineWidth * 0.55} ${baselineY}`,

      // T wave (broader, clean shape)
      `L ${startX + lineWidth * 0.57} ${baselineY - cleanTHeight * 0.2}`, // T wave start
      `L ${startX + lineWidth * 0.6} ${baselineY - cleanTHeight * 0.8}`, // T wave rise
      `L ${startX + lineWidth * 0.62} ${baselineY - cleanTHeight}`, // T peak
      `L ${startX + lineWidth * 0.65} ${baselineY - cleanTHeight * 0.6}`, // T wave fall
      `L ${startX + lineWidth * 0.68} ${baselineY - cleanTHeight * 0.1}`, // T wave end
      `L ${startX + lineWidth * 0.7} ${baselineY}`,

      // Final baseline
      `L ${startX + lineWidth * 0.8} ${baselineY}`,
      `L ${startX + lineWidth * 0.9} ${baselineY}`,
      `L ${startX + lineWidth} ${baselineY}`,
    ].join(" ")
  }

  // Animation variables
  let currentDate = new Date(1911, 0, 1) // Start at January 1911
  let isPaused = false
  // Variable speed timeline - starts fast, ends slow (less drastic slowdown)
  function getTimelineSpeed(): number {
    const currentYear = currentDate.getFullYear()
    const progress = (currentYear - startYear) / (endYear - startYear)

    // More gradual slowdown: fast at start (5ms), slow at end (30ms)
    const minSpeed = 5 // Fastest speed (early years)
    const maxSpeed = 30 // Slowest speed (recent years) - less drastic

    // Use linear curve for gentler transition
    const speedFactor = progress // Linear instead of exponential
    return minSpeed + speedFactor * (maxSpeed - minSpeed)
  }
  let lastMonthUpdateTime = 0
  let animationFrame: number | null = null

  // Current heartbeat state - now supports multiple concurrent heartbeats
  let activeHeartbeats: Array<{ id: number; params: HeartbeatParams; progress: number; fadeProgress: number; isFading: boolean }> = []
  let lastHeatwaveInfo: HeartbeatParams | null = null
  let heartbeatIdCounter = 0


  // Store displayed heartbeats for mini graph
  let displayedHeartbeats: { year: string; temp: number; date: string }[] = []

  // Store all heatwaves for table display (unlimited for full scrolling)
  let allHeatwaves: HeartbeatParams[] = []

  // Hover state for table-barchart interaction
  let hoveredHeatwaveDate: string | null = null

  // Heartbeat timing control with queuing
  let lastHeartbeatTime = 0
  const minHeartbeatInterval = 400 // 0.4 seconds minimum spacing
  let heartbeatQueue: HeartbeatParams[] = []
  let queueProcessingTimeout: number | null = null

  // New timeline system - persistent year objects that animate position only
  let yearObjects: { year: number; x: number; size: number; opacity: number; showAsDecade?: boolean }[] = []

  const startYear = 1910
  const endYear = 2050
  const timelineMargin = 40
  const timelineWidth = width - timelineMargin * 2

  // Calculate static decade position with even spacing (1910-2050)
  function calculateDecadePosition(decade: number): { x: number; size: number; opacity: number } {
    const decadesFromStart = (decade - startYear) / 10
    const totalDecades = (endYear - startYear) / 10 // 14 decades total (1910, 1920, ..., 2050)

    // Even spacing across the timeline
    const x = timelineMargin + (decadesFromStart / totalDecades) * timelineWidth

    // All decades same small size
    const size = 12 // Small consistent size for all decades

    // All decades have same opacity
    const opacity = 0.8

    return { x, size, opacity }
  }

  // Calculate current date position on timeline
  function calculateCurrentDatePosition(): number {
    const currentYear = currentDate.getFullYear()
    const currentMonth = currentDate.getMonth() // 0-11

    // Find the decade before and after current year
    const currentDecade = Math.floor(currentYear / 10) * 10
    const nextDecade = currentDecade + 10

    // Get positions of surrounding decades
    const currentDecadePos = calculateDecadePosition(currentDecade)
    const nextDecadePos = calculateDecadePosition(nextDecade)

    // Calculate progress within the decade (0-10 years)
    const yearInDecade = currentYear - currentDecade
    const monthProgress = currentMonth / 12 // 0-1 for the current year
    const decadeProgress = (yearInDecade + monthProgress) / 10 // 0-1 across the decade

    // Interpolate position between decades
    const x = currentDecadePos.x + decadeProgress * (nextDecadePos.x - currentDecadePos.x)

    return x
  }

  // Legacy function for compatibility with heatwave positioning
  function calculateYearPosition(year: number): { x: number; size: number; opacity: number; showAsDecade?: boolean } {
    const yearsFromStart = year - startYear
    const totalYears = endYear - startYear // 2025 - 1911 = 114 years

    // Calculate how "recent" this year is (0 = oldest, 1 = newest)
    const recencyRatio = yearsFromStart / totalYears

    // Position calculation with compression effect:
    // Recent years (right side) are spread out, older years (left side) are compressed
    let x
    if (recencyRatio < 0.6) {
      // Heavily compressed older years (left 40% of timeline)
      // Use quadratic compression to pack older years tightly
      const compressedPosition = Math.pow(recencyRatio / 0.6, 0.5)
      x = timelineMargin + compressedPosition * (timelineWidth * 0.4)
    } else {
      // More spread out recent years (right 60% of timeline)
      const recentPosition = (recencyRatio - 0.6) / 0.4
      x = timelineMargin + timelineWidth * 0.4 + recentPosition * (timelineWidth * 0.6)
    }

    // Size calculation - left (older) years smaller, right (newer) years bigger
    const baseSizeMin = 6 // Very small for old years
    const baseSizeMax = 22 // Large for recent years

    // Size increases with recency: older = smaller, newer = bigger
    const sizeRatio = Math.pow(recencyRatio, 0.7) // Makes older years much smaller
    const size = baseSizeMin + sizeRatio * (baseSizeMax - baseSizeMin)

    // Opacity calculation - older years dimmer, newer years brighter
    const baseOpacityMin = 0.3
    const baseOpacityMax = 0.9
    const opacity = baseOpacityMin + recencyRatio * (baseOpacityMax - baseOpacityMin)

    // Determine if this year should show as decade (for very compressed/crowded areas)
    // Show decades for older years to reduce crowding
    const showAsDecade = recencyRatio < 0.4 && year % 10 !== 0

    // For compatibility - use decade positioning
    const decade = Math.floor(year / 10) * 10
    const decadePos = calculateDecadePosition(decade)
    return { ...decadePos, showAsDecade: false }
  }

  // Initialize static timeline with decades only (1910-2050)
  function initializeTimeline() {
    yearObjects = []

    // Create only decades from 1910 to 2050
    for (let decade = startYear; decade <= endYear; decade += 10) {
      const position = calculateDecadePosition(decade)
      console.log(`Creating decade: ${decade}`)

      yearObjects.push({
        year: decade,
        x: position.x,
        size: position.size,
        opacity: position.opacity,
        showAsDecade: false, // Will show as "1910", "1920", etc.
      })
    }

    console.log(`Created ${yearObjects.length} decades (${startYear}-${endYear})`)
  }

  // Calculate timeline position for heatwave points (works for any year)
  function calculateTimelineXPosition(eventYear: number): number {
    // Find the decade before and after the event year
    const currentDecade = Math.floor(eventYear / 10) * 10
    const nextDecade = currentDecade + 10

    // Get positions of surrounding decades
    const currentDecadePos = calculateDecadePosition(currentDecade)
    const nextDecadePos = calculateDecadePosition(nextDecade)

    // Calculate progress within the decade (0-10 years)
    const yearInDecade = eventYear - currentDecade
    const decadeProgress = yearInDecade / 10 // 0-1 across the decade

    // Interpolate position between decades
    const x = currentDecadePos.x + decadeProgress * (nextDecadePos.x - currentDecadePos.x)

    return x
  }

  // Calculate bar height based on temperature for mini barchart
  function calculateBarHeight(temp: number): number {
    // Map temperature range (30-42°C) to bar height (5-40px)
    const minTemp = 30
    const maxTemp = 42
    const minHeight = 5
    const maxHeight = 40

    const normalizedTemp = Math.min(Math.max(temp - minTemp, 0), maxTemp - minTemp) / (maxTemp - minTemp)
    return minHeight + normalizedTemp * (maxHeight - minHeight)
  }

  // Main animation loop - handles both time progression and heartbeat animation
  function mainAnimationLoop(): void {
    const now = performance.now()

    if (!hasStarted || isPaused) {
      if (hasStarted && isPaused) {
        // Continue animation loop even when paused to keep checking
        animationFrame = requestAnimationFrame(mainAnimationLoop)
      }
      return
    }

    // Initialize timing
    if (lastMonthUpdateTime === 0) {
      lastMonthUpdateTime = now
    }

    // Update time progression with variable speed
    const currentSpeed = getTimelineSpeed()
    if (now - lastMonthUpdateTime >= currentSpeed) {
      // Advance one month - create new Date object for reactivity
      const newDate = new Date(currentDate)
      newDate.setMonth(newDate.getMonth() + 1)
      currentDate = newDate
      lastMonthUpdateTime = now

      // Timeline is static - no dynamic updates needed

      // Check if we've reached 2050 - restart from beginning
      if (currentDate.getFullYear() >= 2050 && currentDate.getMonth() >= 11) {
        console.log("Reached December 2050 - restarting from beginning")
        restartAnimation()
        return
      }

      // Check for heatwave events at current date
      checkForHeatwaveAtCurrentDate()
    }

    // Timeline is static - no animation needed


    // Handle multiple concurrent heartbeat animations
    if (activeHeartbeats.length > 0) {
      activeHeartbeats = activeHeartbeats
        .map((heartbeat) => {
          if (!heartbeat.isFading) {
            // Drawing phase
            if (heartbeat.progress < 1) {
              return { ...heartbeat, progress: heartbeat.progress + 0.01875 }
            } else {
              // Start fading phase
              return { ...heartbeat, isFading: true }
            }
          } else {
            // Fading phase - fade out in place
            if (heartbeat.fadeProgress < 1) {
              return { ...heartbeat, fadeProgress: heartbeat.fadeProgress + 0.01125 }
            } else {
              // Mark for removal
              return null
            }
          }
        })
        .filter((heartbeat) => heartbeat !== null)
    }

    // Continue animation loop
    if (hasStarted) {
      animationFrame = requestAnimationFrame(mainAnimationLoop)
    }
  }

  // Check for heatwave events at current date and trigger heartbeat
  function checkForHeatwaveAtCurrentDate(): void {
    if (!heartbeats.length) return

    const currentYearMonth = `${currentDate.getFullYear()}-${String(currentDate.getMonth() + 1).padStart(2, "0")}`

    // Find matching heatwave
    const matchingHeartbeat = heartbeats.find((hb) => hb.heatwaveData && hb.heatwaveData.startDate.startsWith(currentYearMonth))

    if (matchingHeartbeat) {
      console.log(`Found heatwave for ${currentYearMonth}:`, matchingHeartbeat.heatwaveData)

      // Always add to data immediately
      allHeatwaves = [matchingHeartbeat, ...allHeatwaves]
      lastHeatwaveInfo = matchingHeartbeat

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

      // Queue the heartbeat for natural timing
      queueHeartbeat(matchingHeartbeat)
    }
  }

  // Play heartbeat sound with beep
  function playHeartbeatSound() {
    // Play the original heartbeat sound
    if (heartbeatSound) {
      heartbeatSound.currentTime = 0
      heartbeatSound.play().catch((error) => {
        console.error("Error playing heartbeat sound:", error)
      })
    }

    // Play the beep sound for hospital monitor effect
    playBeepSound()
  }

  // Generate and play a beep sound
  function playBeepSound() {
    // Create AudioContext for generating beep
    const audioContext = new (window.AudioContext || window.webkitAudioContext)()
    const oscillator = audioContext.createOscillator()
    const gainNode = audioContext.createGain()

    // Connect oscillator to gain to speakers
    oscillator.connect(gainNode)
    gainNode.connect(audioContext.destination)

    // Set beep parameters
    oscillator.frequency.setValueAtTime(800, audioContext.currentTime) // 800Hz beep
    oscillator.type = "sine"

    // Set volume and fade out
    gainNode.gain.setValueAtTime(0.1, audioContext.currentTime)
    gainNode.gain.exponentialRampToValueAtTime(0.01, audioContext.currentTime + 0.1)

    // Play beep for 100ms
    oscillator.start(audioContext.currentTime)
    oscillator.stop(audioContext.currentTime + 0.1)
  }

  // Queue heartbeat for natural timing
  function queueHeartbeat(heartbeatParams: HeartbeatParams) {
    heartbeatQueue = [...heartbeatQueue, heartbeatParams]
    processHeartbeatQueue()
  }

  // Process heartbeat queue with natural timing
  function processHeartbeatQueue() {
    if (heartbeatQueue.length === 0) return

    const currentTime = performance.now()
    const timeSinceLastHeartbeat = currentTime - lastHeartbeatTime

    if (timeSinceLastHeartbeat >= minHeartbeatInterval) {
      // Enough time has passed, trigger heartbeat immediately
      triggerHeartbeat(heartbeatQueue.shift()!)
    } else {
      // Too soon, schedule for later
      const delay = minHeartbeatInterval - timeSinceLastHeartbeat

      if (queueProcessingTimeout) {
        clearTimeout(queueProcessingTimeout)
      }

      queueProcessingTimeout = setTimeout(() => {
        processHeartbeatQueue()
      }, delay)
    }
  }

  // Actually trigger the heartbeat animation and sound
  function triggerHeartbeat(heartbeatParams: HeartbeatParams) {
    const newHeartbeat = {
      id: heartbeatIdCounter++,
      params: heartbeatParams,
      progress: 0,
      fadeProgress: 0,
      isFading: false,
    }
    activeHeartbeats = [...activeHeartbeats, newHeartbeat]
    lastHeartbeatTime = performance.now()

    // Play sound
    playHeartbeatSound()

    // Continue processing queue if there are more
    if (heartbeatQueue.length > 0) {
      setTimeout(() => processHeartbeatQueue(), minHeartbeatInterval)
    }
  }

  // Toggle pause/play
  function togglePause() {
    isPaused = !isPaused
    console.log(isPaused ? "Animation paused" : "Animation resumed")
  }

  // Restart animation from beginning
  function restartAnimation() {
    console.log("Restarting animation from beginning")

    // Reset all state
    currentDate = new Date(1911, 0, 1)
    lastMonthUpdateTime = 0
    displayedHeartbeats = []
    activeHeartbeats = []
    lastHeatwaveInfo = null
    heartbeatIdCounter = 0
    allHeatwaves = []
    lastHeartbeatTime = 0
    heartbeatQueue = []
    isPaused = false

    if (queueProcessingTimeout) {
      clearTimeout(queueProcessingTimeout)
      queueProcessingTimeout = null
    }

    // Initialize timeline
    initializeTimeline()

    // Continue animation loop without canceling current frame
    console.log("Animation restarted successfully")
  }

  // Start the visualization
  function startVisualization() {
    if (hasStarted && !isPaused) return // Prevent multiple starts when already running

    hasStarted = true
    isPaused = false

    // Reset all animation variables
    currentDate = new Date(1911, 0, 1)
    lastMonthUpdateTime = 0
    displayedHeartbeats = []

    // Reset heartbeat state
    activeHeartbeats = []
    lastHeatwaveInfo = null
    heartbeatIdCounter = 0
    allHeatwaves = []
    lastHeartbeatTime = 0
    heartbeatQueue = []
    if (queueProcessingTimeout) {
      clearTimeout(queueProcessingTimeout)
      queueProcessingTimeout = null
    }


    // Initialize timeline with years
    initializeTimeline()

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
      <h1 class="main-title"><span class="heatwave-highlight">Heatwave</span> Heartbeat Monitor</h1>
      <p class="intro-text">
        Each heartbeat represents a <span class="heatwave-highlight">heatwave</span> event. Watch as climate patterns pulse through time, revealing
        the rhythm of our changing planet.
        <span class="data-source"
          >Data: <a href="https://raw.githubusercontent.com/sophievanderhorst/data/refs/heads/main/heatwaves.csv" target="_blank">Heatwave Dataset</a
          ></span
        >
      </p>
    </div>

    <!-- Medical Monitor Style Information Display - Scrollable Heatwave Table -->
    <div class="monitor-info">
      {#if allHeatwaves.length > 0}
        <div class="heatwave-table-container">
          <!-- Fixed table header -->
          <div class="table-header">
            <div class="header-cell date-col">DATE RANGE</div>
            <div class="header-cell duration-col">DURATION</div>
            <div class="header-cell temp-col">MAX TEMP</div>
            <div class="header-cell tropical-col">TROPICAL DAYS</div>
          </div>
          <!-- Scrollable table body -->
          <div class="table-body">
            {#each allHeatwaves as heatwave, i}
              {#if heatwave.heatwaveData}
                <div
                  class="table-row {i === 0 ? 'latest' : ''}"
                  on:mouseenter={() => hoveredHeatwaveDate = heatwave.heatwaveData.startDate}
                  on:mouseleave={() => hoveredHeatwaveDate = null}
                >
                  <div class="table-cell date-col">
                    {formatDate(heatwave.heatwaveData.startDate)} - {formatDate(heatwave.heatwaveData.endDate)}
                  </div>
                  <div class="table-cell duration-col">
                    {heatwave.heatwaveData.duration} DAYS
                  </div>
                  <div class="table-cell temp-col">
                    {heatwave.heatwaveData.highestTemp}°C
                  </div>
                  <div class="table-cell tropical-col">
                    {heatwave.heatwaveData.tropicalDays}
                  </div>
                </div>
              {/if}
            {/each}
          </div>
        </div>
      {:else}
        <div class="no-data">
          <span class="label">MONITORING HEATWAVE ACTIVITY...</span>
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

      <!-- Dynamic timeline baseline (expands as time progresses) -->
      {#if hasStarted}
        {@const bottomY = height - 80} <!-- Position for mini barchart and timeline - moved higher -->
        <!-- Timeline baseline moved to bottom to overlap with mini barchart -->
        <line x1="40" y1={bottomY} x2={width - 40} y2={bottomY} stroke="rgba(0, 255, 0, 0.6)" stroke-width="2" class="timeline-baseline" />

        <!-- Center line - where years shrink (keep in middle for heartbeat area) -->
        <line
          x1={width / 2}
          y1={baselineY - 40}
          x2={width / 2}
          y2={baselineY + 40}
          stroke="rgba(255, 255, 255, 0.3)"
          stroke-width="1"
          stroke-dasharray="3,3"
          class="center-line"
        />

        <!-- Timeline years with compression effect -->
        {#each yearObjects as yearData}
          {@const isFirst = yearData.year === startYear}
          {@const isDecade = yearData.showAsDecade}

          <text
            x={yearData.x}
            y={bottomY + 15}
            text-anchor="middle"
            fill="rgba(0, 255, 0, {yearData.opacity})"
            font-size={yearData.size}
            font-weight={isFirst ? "bold" : "normal"}
            font-family="Courier New, monospace"
            class="timeline-year {isFirst ? 'first-year' : ''} {isDecade ? 'decade-year' : ''}"
          >
            {isDecade ? `${yearData.year}s` : yearData.year}
          </text>
        {/each}

        <!-- Current date indicator that moves along timeline -->
        {#if hasStarted}
          {@const currentDateX = calculateCurrentDatePosition()}

          <!-- Current date line -->
          <line
            x1={currentDateX}
            y1={bottomY - 10}
            x2={currentDateX}
            y2={bottomY + 35}
            stroke="rgba(255, 50, 50, 0.8)"
            stroke-width="2"
            class="current-date-line"
          />

          <!-- Current date text -->
          <text
            x={currentDateX}
            y={bottomY + 50}
            text-anchor="middle"
            fill="rgba(255, 50, 50, 1)"
            font-size="22"
            font-weight="bold"
            font-family="Courier New, monospace"
            class="current-date-text"
          >
            {currentDate.getFullYear()}
          </text>
        {/if}

        <!-- Mini barchart for heatwave events - overlapping with timeline at bottom -->
        {#each displayedHeartbeats as heartbeat, i}
          {#if parseInt(heartbeat.year) <= currentDate.getFullYear()}
            {@const barHeight = calculateBarHeight(heartbeat.temp)}
            {@const barX = calculateTimelineXPosition(parseInt(heartbeat.year))}
            {@const isHovered = hoveredHeatwaveDate === heartbeat.date}
            {@const isLatest = i === displayedHeartbeats.length - 1}
            <rect
              x={barX - 1.5}
              y={bottomY - barHeight}
              width="3"
              height={barHeight}
              fill={isHovered ? "rgba(255,255,100,1)" : (isLatest ? "rgba(255,100,100,1)" : "rgba(255,60,60,0.8)")}
              stroke={isHovered ? "rgba(255,255,255,1)" : (isLatest ? "rgba(255,255,255,0.8)" : "none")}
              stroke-width={isHovered ? "2" : (isLatest ? "0.5" : "0")}
              opacity={isHovered ? "1.0" : (isLatest ? "1.0" : "0.7")}
              class="timeline-bar {isHovered ? 'hovered' : ''}"
            />
            <!-- Small indicator at the base -->
            <circle
              cx={barX}
              cy={bottomY}
              r={isHovered ? "2" : "1"}
              fill={isHovered ? "rgba(255,255,100,1)" : (isLatest ? "rgba(255,100,100,1)" : "rgba(255,60,60,0.6)")}
              opacity={isHovered ? "1.0" : (isLatest ? "1.0" : "0.5")}
            />
          {/if}
        {/each}


        <!-- Current year indicator (at the right end) -->
        <circle
          cx={width - 40}
          cy={bottomY}
          r="3"
          fill="rgba(0, 255, 0, 0.9)"
          stroke="rgba(255, 255, 255, 0.6)"
          stroke-width="1"
          class="current-year-indicator"
        />
      {:else}
        <!-- Static baseline when not started -->
        <line x1="0" y1={baselineY} x2={width} y2={baselineY} stroke="rgba(0, 255, 0, 0.4)" stroke-width="1.5" stroke-dasharray="5,5" />
      {/if}

      <!-- Current Date Display (top left) -->
      {#if hasStarted}
        <text
          x="25"
          y="50"
          fill="rgba(0, 255, 0, 0.9)"
          font-size="36"
          font-weight="bold"
          font-family="Courier New, monospace"
          class="current-month-year"
        >
          {currentDate.getFullYear()}
        </text>
        <!-- Historic/Predicted indicator -->
        <text
          x="25"
          y="85"
          fill={currentDate.getFullYear() < 2026 ? "rgba(200, 200, 200, 0.8)" : "rgba(255, 200, 100, 0.8)"}
          font-size="24"
          font-weight="bold"
          font-family="Courier New, monospace"
          class="time-period-indicator"
        >
          {currentDate.getFullYear() < 2026 ? "HISTORIC" : "PREDICTED"}
        </text>
      {/if}

      <!-- Play/Pause Controls (top right) -->
      {#if hasStarted}
        <g
          class="play-pause-button"
          on:click={togglePause}
          on:keydown={(e) => (e.key === "Enter" || e.key === " " ? togglePause() : null)}
          role="button"
          tabindex="0"
          aria-label={isPaused ? "Resume animation" : "Pause animation"}
          style="cursor: pointer;"
        >
          <!-- Button background -->
          <circle
            cx={width - 40}
            cy="30"
            r="20"
            fill="rgba(0, 40, 0, 0.8)"
            stroke="rgba(0, 255, 0, 0.8)"
            stroke-width="2"
            class="control-bg-circle"
          />
          {#if isPaused}
            <!-- Play arrow (when paused) -->
            <polygon
              points="{width - 48},22 {width - 48},38 {width - 28},30"
              fill="rgba(0, 255, 0, 0.9)"
              class="control-icon"
            />
          {:else}
            <!-- Pause bars (when playing) -->
            <rect
              x={width - 48}
              y="22"
              width="5"
              height="16"
              fill="rgba(0, 255, 0, 0.9)"
              class="control-icon"
            />
            <rect
              x={width - 38}
              y="22"
              width="5"
              height="16"
              fill="rgba(0, 255, 0, 0.9)"
              class="control-icon"
            />
          {/if}
        </g>
      {/if}

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
          on:keydown={(e) => (e.key === "Enter" || e.key === " " ? startVisualization() : null)}
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

      <!-- Show all active heartbeats, otherwise show flatline -->
      {#each activeHeartbeats as heartbeat (heartbeat.id)}
        {@const fadeOpacity = heartbeat.isFading ? 1 - heartbeat.fadeProgress : 1}

        <g class="heartbeat-group">
          <!-- ECG trace with fiery glow effects -->
          <g>
            <!-- Outer glow (fire-like) -->
            <path
              d={generateHeartlinePath(heartbeat.params)}
              stroke={heartbeat.params.color}
              stroke-width="12"
              fill="none"
              opacity={heartbeat.params.opacity * 0.15 * fadeOpacity}
              pathLength="1000"
              style="stroke-dasharray: {heartbeat.progress < 1 ? `${1000 * heartbeat.progress} 1000` : '1000 0'};
                     stroke-dashoffset: 0;
                     filter: blur(4px)"
            />
            <!-- Middle glow -->
            <path
              d={generateHeartlinePath(heartbeat.params)}
              stroke={heartbeat.params.color}
              stroke-width="8"
              fill="none"
              opacity={heartbeat.params.opacity * 0.25 * fadeOpacity}
              pathLength="1000"
              style="stroke-dasharray: {heartbeat.progress < 1 ? `${1000 * heartbeat.progress} 1000` : '1000 0'};
                     stroke-dashoffset: 0;
                     filter: blur(2px)"
            />
            <!-- Inner glow -->
            <path
              d={generateHeartlinePath(heartbeat.params)}
              stroke={heartbeat.params.color}
              stroke-width="5"
              fill="none"
              opacity={heartbeat.params.opacity * 0.4 * fadeOpacity}
              pathLength="1000"
              style="stroke-dasharray: {heartbeat.progress < 1 ? `${1000 * heartbeat.progress} 1000` : '1000 0'};
                     stroke-dashoffset: 0;
                     filter: blur(1px)"
            />
            <!-- Main ECG trace (bright core) -->
            <path
              d={generateHeartlinePath(heartbeat.params)}
              stroke="rgba(255, 255, 200, 1)"
              stroke-width="2"
              fill="none"
              opacity={fadeOpacity}
              class="heartline-path fiery"
              pathLength="1000"
              style="stroke-dasharray: {heartbeat.progress < 1 ? `${1000 * heartbeat.progress} 1000` : '1000 0'};
                     stroke-dashoffset: 0;
                     stroke-linecap: round;
                     stroke-linejoin: round"
            />
          </g>
        </g>
      {/each}


      {#if hasStarted && activeHeartbeats.length === 0}
        <!-- Show flatline when no heatwave is present -->
        <g class="flatline-group">
          <line x1="0" y1={baselineY} x2={width} y2={baselineY} stroke="rgba(0, 255, 0, 0.7)" stroke-width="2" class="flatline" />

          <!-- Small blip that moves along the flatline to show activity -->
          <circle
            cx={width * 0.2 + Math.sin(performance.now() / 1000) * (width * 0.6)}
            cy={baselineY}
            r="2"
            fill="rgba(0, 255, 0, 0.8)"
            class="flatline-blip"
          />
        </g>
      {/if}
    </svg>
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
    filter: drop-shadow(0 0 8px rgba(255, 100, 0, 0.8));
    will-change: stroke-dashoffset;
    transition: opacity 0.3s;
  }

  .heartline-path.fiery {
    filter: drop-shadow(0 0 6px rgba(255, 255, 200, 0.9)) drop-shadow(0 0 12px rgba(255, 150, 0, 0.6));
    animation: fireFlicker 0.1s infinite alternate;
  }

  .heartline-path:hover {
    opacity: 1 !important;
    filter: drop-shadow(0 0 15px rgba(255, 200, 0, 1)) drop-shadow(0 0 25px rgba(255, 100, 0, 0.8));
    z-index: 10;
  }

  @keyframes fireFlicker {
    0% {
      filter: drop-shadow(0 0 6px rgba(255, 255, 200, 0.9)) drop-shadow(0 0 12px rgba(255, 150, 0, 0.6));
    }
    100% {
      filter: drop-shadow(0 0 8px rgba(255, 255, 200, 1)) drop-shadow(0 0 15px rgba(255, 120, 0, 0.7));
    }
  }

  .data-source {
    font-size: 0.85rem;
    color: rgba(150, 150, 150, 0.8);
    font-style: italic;
  }

  .data-source a {
    color: rgba(0, 255, 0, 0.7);
    text-decoration: none;
  }

  .data-source a:hover {
    color: rgba(0, 255, 0, 1);
    text-decoration: underline;
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

  /* Play/Pause Control Styling */
  .play-pause-button {
    transition: all 0.3s ease;
  }

  .play-pause-button:hover .control-bg-circle {
    fill: rgba(0, 60, 0, 0.9);
    stroke: rgba(0, 255, 0, 1);
    stroke-width: 3;
  }

  .play-pause-button:hover .control-icon {
    fill: rgba(0, 255, 0, 1);
  }

  .play-pause-button:active {
    transform: scale(0.95);
  }

  .control-bg-circle {
    transition: all 0.2s ease;
  }

  .control-icon {
    transition: all 0.2s ease;
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
    font-family: "Courier New", monospace;
    letter-spacing: 2px;
  }

  .intro-text {
    font-size: 1.1rem;
    color: rgba(200, 200, 200, 0.9);
    line-height: 1.5; /* Slightly reduced line height */
    margin: 0 0 15px 0; /* Reduced bottom margin */
    font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif;
  }

  /* Medical Monitor Information Display */
  .monitor-info {
    position: relative;
    margin: 3px auto 8px auto;
    background: rgba(0, 20, 0, 0.9);
    border: 2px solid rgba(0, 255, 0, 0.6);
    border-radius: 8px;
    padding: 12px 20px;
    font-family: "Courier New", monospace;
    box-shadow: 0 0 20px rgba(0, 255, 0, 0.3);
    backdrop-filter: blur(5px);
    max-width: 800px;
  }

  /* Heatwave Table Styles */
  .heatwave-table-container {
    width: 100%;
    height: 180px;
    overflow: hidden;
  }

  .table-header {
    display: flex;
    background: rgba(0, 40, 0, 0.8);
    border-bottom: 1px solid rgba(0, 255, 0, 0.4);
    position: sticky;
    top: 0;
    z-index: 10;
  }

  .table-body {
    height: 156px;
    overflow-y: auto;
    scrollbar-width: thin;
    scrollbar-color: rgba(0, 255, 0, 0.6) rgba(0, 40, 0, 0.3);
  }

  .table-body::-webkit-scrollbar {
    width: 6px;
  }

  .table-body::-webkit-scrollbar-track {
    background: rgba(0, 40, 0, 0.3);
  }

  .table-body::-webkit-scrollbar-thumb {
    background: rgba(0, 255, 0, 0.6);
    border-radius: 3px;
  }

  .table-body::-webkit-scrollbar-thumb:hover {
    background: rgba(0, 255, 0, 0.8);
  }

  .header-cell, .table-cell {
    padding: 8px 6px;
    text-align: center;
    font-size: 13px;
  }

  .header-cell {
    font-weight: bold;
    color: rgba(0, 220, 0, 0.9);
    text-shadow: 0 0 3px rgba(0, 255, 0, 0.5);
    letter-spacing: 0.5px;
  }

  .table-row {
    display: flex;
    border-bottom: 1px solid rgba(0, 100, 0, 0.2);
    transition: background-color 0.2s ease;
    cursor: pointer;
  }

  .table-row:hover {
    background: rgba(255, 255, 100, 0.1);
    border-bottom: 1px solid rgba(255, 255, 100, 0.3);
  }

  .table-row:hover .table-cell {
    color: rgba(255, 255, 100, 1);
    text-shadow: 0 0 4px rgba(255, 255, 100, 0.6);
  }

  .table-row.latest {
    background: rgba(255, 50, 50, 0.1);
    border-bottom: 1px solid rgba(255, 50, 50, 0.3);
  }

  .table-row.latest .table-cell {
    color: rgba(255, 100, 100, 1);
    font-weight: bold;
    text-shadow: 0 0 3px rgba(255, 50, 50, 0.4);
  }

  .table-cell {
    color: rgba(0, 255, 0, 0.9);
  }

  .date-col {
    flex: 2.5;
    min-width: 160px;
  }

  .duration-col {
    flex: 1;
    min-width: 70px;
  }

  .temp-col {
    flex: 1;
    min-width: 60px;
  }

  .tropical-col {
    flex: 1.2;
    min-width: 80px;
  }

  .no-data {
    text-align: center;
    padding: 20px;
  }

  .no-data .label {
    font-size: 14px;
    color: rgba(0, 180, 0, 0.8);
    font-weight: bold;
    letter-spacing: 1px;
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


  /* Timeline animations */
  .timeline-baseline {
    transition: x2 0.1s ease-out;
  }

  .timeline-bar {
    transition: all 0.3s ease-in-out;
  }

  .timeline-bar:hover {
    opacity: 1 !important;
    filter: drop-shadow(0 0 4px rgba(255, 100, 100, 0.8));
  }

  .timeline-bar.hovered {
    filter: drop-shadow(0 0 6px rgba(255, 255, 100, 0.9));
    transform: scaleY(1.1);
    transform-origin: bottom;
  }

  .current-year-indicator {
    animation: current-year-pulse 2s infinite ease-in-out;
    filter: drop-shadow(0 0 8px rgba(0, 255, 0, 0.8));
  }

  .moving-year {
    transition: all 0.5s ease-out;
    text-shadow: 0 0 3px rgba(0, 255, 0, 0.3);
  }

  .moving-year.current-year {
    text-shadow: 0 0 10px rgba(0, 255, 0, 0.9);
    animation: current-year-glow 2s infinite ease-in-out;
  }

  .moving-year.first-year {
    text-shadow: 0 0 5px rgba(0, 255, 0, 0.6);
    opacity: 0.8;
  }

  .moving-year.decade-year {
    font-style: italic;
    color: rgba(0, 200, 0, 0.7);
  }

  .current-year-glow {
    animation: current-year-pulse-glow 2s infinite ease-in-out;
  }

  .current-date-line {
    animation: date-line-glow 2s infinite ease-in-out;
  }

  .current-date-text {
    animation: date-text-glow 2s infinite ease-in-out;
  }

  .current-month-year {
    text-shadow: 0 0 5px rgba(0, 255, 0, 0.6);
    animation: month-year-glow 3s infinite ease-in-out;
  }

  .time-period-indicator {
    text-shadow: 0 0 3px rgba(255, 255, 255, 0.3);
    letter-spacing: 1px;
    animation: indicator-glow 2s infinite ease-in-out;
  }

  @keyframes indicator-glow {
    0% {
      opacity: 0.6;
    }
    50% {
      opacity: 1.0;
    }
    100% {
      opacity: 0.6;
    }
  }

  @keyframes month-year-glow {
    0% {
      opacity: 0.8;
      text-shadow: 0 0 5px rgba(0, 255, 0, 0.6);
    }
    50% {
      opacity: 1.0;
      text-shadow: 0 0 8px rgba(0, 255, 0, 0.8);
    }
    100% {
      opacity: 0.8;
      text-shadow: 0 0 5px rgba(0, 255, 0, 0.6);
    }
  }

  @keyframes date-line-glow {
    0% {
      opacity: 0.6;
    }
    50% {
      opacity: 1;
    }
    100% {
      opacity: 0.6;
    }
  }

  @keyframes date-text-glow {
    0% {
      opacity: 0.8;
    }
    50% {
      opacity: 1;
    }
    100% {
      opacity: 0.8;
    }
  }

  @keyframes current-year-glow {
    0% {
      text-shadow: 0 0 8px rgba(0, 255, 0, 0.6);
    }
    50% {
      text-shadow: 0 0 12px rgba(0, 255, 0, 1);
    }
    100% {
      text-shadow: 0 0 8px rgba(0, 255, 0, 0.6);
    }
  }

  .center-line {
    opacity: 0.3;
  }

  @keyframes current-year-pulse {
    0% {
      opacity: 0.9;
      r: 5;
    }
    50% {
      opacity: 1;
      r: 6;
    }
    100% {
      opacity: 0.9;
      r: 5;
    }
  }

  @keyframes current-year-pulse-glow {
    0% {
      opacity: 0.3;
      r: 4;
    }
    50% {
      opacity: 0.6;
      r: 8;
    }
    100% {
      opacity: 0.3;
      r: 4;
    }
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
