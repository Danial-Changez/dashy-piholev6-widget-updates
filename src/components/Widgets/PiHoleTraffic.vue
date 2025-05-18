<template>
  <div class="pihole-traffic">
    <div class="chart-header">
      <h3>Recent Queries & Ads</h3>
    </div>
    <div class="traffic-chart-container">
      <canvas ref="trafficCanvas" class="traffic-chart"></canvas>
    </div>
  </div>
</template>

<script>
import Chart from 'chart.js/auto';

export default {
  name: 'PiHoleTraffic',
  props: {
    options: { type: Object, required: true }
  },
  data() {
    return {
      chartInstance: null,
      labels: [],        // time labels for the chart
      queriesData: [],   // data series for total queries
      blockedData: []    // data series for blocked queries
    };
  },
  computed: {
    // Preserve endpoint computed property (for reference, not directly used by new implementation).
    endpoint() {
      const base = this.options.hostname.replace(/\/+$/, '');
      // In Pi-hole 5: /admin/api.php?overTimeData10mins&auth=<apiKey>
      // Pi-hole 6 equivalent is /api/history (with optional from/until params). Here we show base path.
      return `${base}/api/history`;
    }
  },
  methods: {
    async fetchData() {
      const baseUrl = this.options.hostname.replace(/\/+$/, '');
      const password = this.options.apiKey || '';
      let sid = null;
      try {
        // Authenticate if needed
        if (password) {
          const authRes = await fetch(`${baseUrl}/api/auth`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ password: password })
          });
          const authData = await authRes.json();
          if (!authData.session || authData.session.sid === undefined) {
            throw new Error('Authentication failed: no session ID');
          }
          sid = authData.session.sid;
          if (sid === null && authData.session.valid && authData.session.message === "no password set") {
            sid = null;
          }
        }
        // Fetch history data (last 24h of queries in 10-min intervals by default)
        const historyUrl = `${baseUrl}/api/history`;
        const headers = sid ? { 'Accept': 'application/json', 'sid': sid } : { 'Accept': 'application/json' };
        const historyRes = await fetch(historyUrl, { headers });
        if (!historyRes.ok) throw new Error('Failed to fetch history data');
        const historyData = await historyRes.json();
        // Validate the response has the expected time-series arrays
        if (!historyData.domains_over_time || !historyData.ads_over_time) {
          throw new Error('Expected data was not returned from Pi-Hole');
        }
        // Pi-hole returns arrays of query counts per 10-minute interval.
        const domainsSeries = historyData.domains_over_time;
        const adsSeries = historyData.ads_over_time;
        // Ensure we have equal length arrays
        const length = Math.min(domainsSeries.length, adsSeries.length);
        const totalPoints = length;
        // Prepare labels for each 10-minute interval
        this.labels = [];
        const now = new Date();
        // Determine start of series. If array length is 144 (full 24h), start at 24h ago, else assume start of day 0:00.
        let startTime;
        if (totalPoints === 144) {
          // full 24 hours of data – assume each point is 10-minute increments ending at current time.
          startTime = new Date(now.getTime() - 24 * 60 * 60 * 1000);
        } else {
          // If not a full day, assume data starts at midnight of current day.
          startTime = new Date();
          startTime.setHours(0, 0, 0, 0);
        }
        for (let i = 0; i < totalPoints; i++) {
          const t = new Date(startTime.getTime() + i * 600000); // 600000 ms = 10 minutes
          // Format time label (e.g. "5:55 PM" or "17:55" depending on preference)
          // We'll use locale time with short format
          let hours = t.getHours();
          let minutes = t.getMinutes();
          const ampm = hours >= 12 ? 'PM' : 'AM';
          hours = hours % 12;
          if (hours === 0) hours = 12;
          const minuteStr = minutes < 10 ? '0' + minutes : '' + minutes;
          const label = `${hours}:${minuteStr} ${ampm}`;
          this.labels.push(label);
        }
        // Slice series to matched length
        this.queriesData = domainsSeries.slice(0, totalPoints);
        this.blockedData = adsSeries.slice(0, totalPoints);
        // Update the chart with new data
        this.updateChart();
      } catch (err) {
        console.error('PiHoleTraffic widget error:', err);
        // Clear data on error
        this.labels = [];
        this.queriesData = [];
        this.blockedData = [];
        throw err;
      }
    },
    updateChart() {
      const ctx = this.$refs.trafficCanvas.getContext('2d');
      const dataConfig = {
        labels: this.labels,
        datasets: [
          {
            label: 'Queries',
            data: this.queriesData,
            borderColor: '#27ae60',      // green line for total queries
            backgroundColor: 'rgba(39,174,96,0.2)', // translucent fill for area
            tension: 0.1,
            fill: true
          },
          {
            label: 'Ads Blocked',
            data: this.blockedData,
            borderColor: '#e74c3c',     // pink/red line for blocked queries
            backgroundColor: 'rgba(231,76,60,0.2)',
            tension: 0.1,
            fill: true
          }
        ]
      };
      const optionsConfig = {
        responsive: true,
        scales: {
          x: {
            ticks: {
              autoSkip: true,
              maxTicksLimit: 10  // show at most 10 time labels to avoid crowding
            }
          },
          y: {
            beginAtZero: true
          }
        },
        plugins: {
          legend: {
            display: true,
            position: 'bottom'
          },
          tooltip: {
            mode: 'index',
            intersect: false
          }
        }
      };
      if (this.chartInstance) {
        // Update existing chart instance
        this.chartInstance.data = dataConfig;
        this.chartInstance.options = optionsConfig;
        this.chartInstance.update();
      } else {
        // Create new chart
        this.chartInstance = new Chart(ctx, {
          type: 'line',
          data: dataConfig,
          options: optionsConfig
        });
      }
    }
  },
  mounted() {
    this.fetchData().catch(err => {
      // Error already logged. Could set an error message for UI here if desired.
    });
  }
};
</script>

<style scoped>
.pihole-traffic {
  width: 100%;
}
.chart-header h3 {
  margin: 0 0 0.5em 0;
  font-weight: bold;
}
.traffic-chart-container {
  width: 100%;
  overflow-x: auto;
}
/* Canvas takes full width of container */
.traffic-chart {
  width: 100% !important;
  max-height: 200px;
}
</style>
