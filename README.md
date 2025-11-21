<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Barossa Valley – Weight + Historic Weather Overlay</title>
  <meta name="viewport" content="width=device-width, initial-scale=1" />

  <!-- Chart.js -->
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <!-- Time adapter for Chart.js -->
  <script src="https://cdn.jsdelivr.net/npm/chartjs-adapter-date-fns"></script>

  <!-- Leaflet (map) -->
  <link
    rel="stylesheet"
    href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"
    crossorigin=""
  />
  <script
    src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"
    crossorigin=""
  ></script>

  <style>
    body {
      font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
      margin: 0;
      padding: 1rem;
      background: #f5f5f5;
    }
    h1 {
      margin-top: 0;
      font-size: 1.4rem;
    }
    .card {
      background: #ffffff;
      border-radius: 12px;
      padding: 1rem;
      margin-bottom: 1rem;
      box-shadow: 0 2px 6px rgba(0,0,0,0.08);
    }
    #chartContainer {
      position: relative;
      height: 400px;
      width: 100%;
    }
    #map {
      height: 350px;
      width: 100%;
      border-radius: 12px;
    }
    .small {
      font-size: 0.85rem;
      color: #555;
    }
  </style>
</head>
<body>
  <h1>Barossa Valley – Weight + Historic Weather Overlay</h1>

  <div class="card">
    <div class="small">
      Weight: ThingSpeak Channel <strong>2429202</strong> (field1 by default)<br>
      Weather: Historic hourly temperature &amp; rain from Open-Meteo for the same date range<br>
      Location (map): <strong>-34.464397, 139.038090</strong>
    </div>
    <div id="chartContainer">
      <canvas id="sensorWeatherChart"></canvas>
    </div>
  </div>

  <div class="card">
    <h2 style="margin-top:0;">Location Map</h2>
    <div id="map"></div>
  </div>

  <script>
    // -----------------------------
    // CONFIGURATION
    // -----------------------------
    const THINGSPEAK_CHANNEL_ID = 2429202;
    const THINGSPEAK_READ_KEY = "PUT_YOUR_READ_KEY_HERE"; // READ key if private
    const THINGSPEAK_FIELD_NAME = "field1"; // weight field
    const THINGSPEAK_RESULTS = 2000;

    const LAT = -34.464397;
    const LON = 139.038090;
    const WEATHER_MAX_DAYS = 31;

    // -----------------------------
    // UTILS
    // -----------------------------
    function formatDateYYYYMMDD(date) {
      return date.toISOString().slice(0, 10);
    }

    // Simple centred moving-average smoother
    function smoothArray(data, windowSize) {
      const result = [];
      const half = Math.floor(windowSize / 2);

      for (let i = 0; i < data.length; i++) {
        if (data[i] == null) {
          result.push(null);
          continue;
        }
        let sum = 0;
        let count = 0;
        for (let j = i - half; j <= i + half; j++) {
          if (j < 0 || j >= data.length) continue;
          const v = data[j];
          if (v == null) continue;
          sum += v;
          count++;
        }
        result.push(count ? sum / count : null);
      }
      return result;
    }

    // -----------------------------
    // FETCH DATA FROM THINGSPEAK
    // -----------------------------
    async function fetchThingSpeakData() {
      const baseUrl = `https://api.thingspeak.com/channels/${THINGSPEAK_CHANNEL_ID}/feeds.json`;
      const params = new URLSearchParams({ results: THINGSPEAK_RESULTS.toString() });

      if (THINGSPEAK_READ_KEY) {
        params.append("api_key", THINGSPEAK_READ_KEY);
      }

      const url = `${baseUrl}?${params.toString()}`;
      console.log("ThingSpeak URL:", url);

      const response = await fetch(url);
      if (!response.ok) {
        const text = await response.text().catch(() => "");
        throw new Error("ThingSpeak error " + response.status + ": " + text);
      }
      const data = await response.json();

      if (!data.feeds) {
        throw new Error("ThingSpeak response missing 'feeds' – check channel ID and key.");
      }

      const feeds = data.feeds;
      const sensorData = feeds
        .map(f => {
          const time = new Date(f.created_at);
          const valueRaw = f[THINGSPEAK_FIELD_NAME];
          const value = valueRaw != null ? parseFloat(valueRaw) : null;
          if (isNaN(value)) return null;
          return { time, value }; // grams
        })
        .filter(Boolean);

      return sensorData;
    }

    // -----------------------------
    // FETCH HISTORIC WEATHER
    // -----------------------------
    async function fetchWeatherData(sensorData) {
      if (!sensorData.length) {
        throw new Error("Cannot fetch weather: no sensor data to define date range.");
      }

      let earliest = sensorData[0].time;
      let latest = sensorData[0].time;
      for (const s of sensorData) {
        if (s.time < earliest) earliest = s.time;
        if (s.time > latest) latest = s.time;
      }

      const oneDayMs = 24 * 60 * 60 * 1000;
      let startDate = new Date(earliest.getTime() - oneDayMs);
      let endDate = new Date(latest.getTime() + oneDayMs);

      const rangeMs = endDate - startDate;
      const rangeDays = rangeMs / oneDayMs;
      if (rangeDays > WEATHER_MAX_DAYS) {
        console.warn(
          `Weather date range (${rangeDays.toFixed(
            1
          )} days) exceeds ${WEATHER_MAX_DAYS} days. Trimming to last ${WEATHER_MAX_DAYS} days of sensor data.`
        );
        endDate = new Date(latest.getTime() + oneDayMs);
        startDate = new Date(endDate.getTime() - WEATHER_MAX_DAYS * oneDayMs);
      }

      const startStr = formatDateYYYYMMDD(startDate);
      const endStr = formatDateYYYYMMDD(endDate);

      const url =
        `https://archive-api.open-meteo.com/v1/archive` +
        `?latitude=${LAT}&longitude=${LON}` +
        `&start_date=${startStr}&end_date=${endStr}` +
        `&hourly=temperature_2m,precipitation` +
        `&timezone=Australia%2FAdelaide`;

      console.log("Weather URL:", url);

      const response = await fetch(url);
      if (!response.ok) {
        const text = await response.text().catch(() => "");
        throw new Error("Weather error " + response.status + ": " + text);
      }
      const data = await response.json();

      const times = (data.hourly && data.hourly.time) || [];
      const temps = (data.hourly && data.hourly.temperature_2m) || [];
      const rain = (data.hourly && data.hourly.precipitation) || [];

      const result = [];
      for (let i = 0; i < times.length; i++) {
        const t = new Date(times[i]);
        result.push({
          time: t,
          temperature: temps[i],
          precipitation: rain[i]
        });
      }
      return result;
    }

    // -----------------------------
    // ALIGN SENSOR + WEATHER BY TIME
    // -----------------------------
    function alignData(sensorData, weatherData) {
      const labels = [];
      const weightValues = [];
      const tempValues = [];
      const rainValues = [];

      for (const s of sensorData) {
        let nearestWeather = null;
        let bestDiff = Infinity;

        for (const w of weatherData) {
          const diff = Math.abs(w.time - s.time);
          if (diff < bestDiff) {
            bestDiff = diff;
            nearestWeather = w;
          }
        }

        labels.push(s.time);
        weightValues.push(s.value);
        if (nearestWeather) {
          tempValues.push(nearestWeather.temperature);
          rainValues.push(nearestWeather.precipitation);
        } else {
          tempValues.push(null);
          rainValues.push(null);
        }
      }

      return { labels, weightValues, tempValues, rainValues };
    }

    // -----------------------------
    // RENDER CHART
    // -----------------------------
    let chartInstance = null;

    function renderChart(aligned) {
      const ctx = document.getElementById("sensorWeatherChart").getContext("2d");

      if (chartInstance) {
        chartInstance.destroy();
      }

      const smoothedWeights = smoothArray(aligned.weightValues, 7);

      chartInstance = new Chart(ctx, {
        type: "line",
        data: {
          labels: aligned.labels,
          datasets: [
            {
              label: "Weight (raw, g)",
              data: aligned.weightValues,
              borderWidth: 1,
              borderColor: "rgba(150, 150, 150, 0.5)",  // light grey
              backgroundColor: "rgba(150, 150, 150, 0.1)",
              pointRadius: 0,
              tension: 0,
              yAxisID: "yWeight",
              hidden: true
            },
            {
              label: "Weight (smoothed, g)",
              data: smoothedWeights,
              borderWidth: 2,
              borderColor: "rgba(54, 162, 235, 1)",      // blue
              backgroundColor: "rgba(54, 162, 235, 0.1)",
              pointRadius: 0,
              tension: 0.4,
              yAxisID: "yWeight"
            },
            {
              label: "Temperature (°C)",
              data: aligned.tempValues,
              borderWidth: 2,
              borderColor: "rgba(255, 99, 132, 1)",       // red/pink
              backgroundColor: "rgba(255, 99, 132, 0.1)",
              pointRadius: 0,
              borderDash: [4, 3],
              tension: 0.2,
              yAxisID: "yTemp"
            },
            {
              label: "Rain (mm/hr)",
              data: aligned.rainValues,
              borderWidth: 1.5,
              borderColor: "rgba(75, 192, 192, 1)",       // teal
              backgroundColor: "rgba(75, 192, 192, 0.15)",
              pointRadius: 0,
              borderDash: [2, 2],
              tension: 0.2,
              yAxisID: "yRain"
            }
          ]
        },
        options: {
          responsive: true,
          maintainAspectRatio: false,
          interaction: {
            mode: "index",
            intersect: false
          },
          plugins: {
            legend: {
              position: "top"
            },
            tooltip: {
              callbacks: {
                label: function (ctx) {
                  const label = ctx.dataset.label || "";
                  const value = ctx.parsed.y;
                  if (value == null) return label;
                  return `${label}: ${value.toFixed(2)}`;
                }
              }
            }
          },
          scales: {
            x: {
              type: "time",
              time: {
                unit: "day",
                tooltipFormat: "yyyy-MM-dd HH:mm"
              },
              title: {
                display: true,
                text: "Time"
              }
            },
            yWeight: {
              type: "linear",
              position: "left",
              min: 40,
              max: 130,
              title: {
                display: true,
                text: "Weight (g)"
              }
            },
            yTemp: {
              type: "linear",
              position: "right",
              title: {
                display: true,
                text: "Temperature (°C)"
              },
              grid: {
                drawOnChartArea: false
              }
            },
            yRain: {
              type: "linear",
              position: "right",
              display: false
            }
          }
        }
      });
    }

    // -----------------------------
    // INITIALISE MAP
    // -----------------------------
    function initMap() {
      const map = L.map("map").setView([LAT, LON], 13);

      L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
        maxZoom: 19,
        attribution: "© OpenStreetMap contributors"
      }).addTo(map);

      L.marker([LAT, LON]).addTo(map).bindPopup("Barossa Valley Sensor Location");
    }

    // -----------------------------
    // MAIN
    // -----------------------------
    async function main() {
      try {
        const sensorData = await fetchThingSpeakData();
        console.log("Weight points:", sensorData.length);
        if (!sensorData.length) {
          throw new Error("No weight data returned – check channel, field, and API key.");
        }

        const weatherData = await fetchWeatherData(sensorData);
        console.log("Weather points:", weatherData.length);

        const aligned = alignData(sensorData, weatherData);
        renderChart(aligned);
        initMap();
      } catch (err) {
        console.error("Main error:", err);
        alert("Error loading data: " + err.message);
      }
    }

    main();
  </script>
</body>
</html>
