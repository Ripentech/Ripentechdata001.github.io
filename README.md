<!DOCTYPE html>
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
      return res
