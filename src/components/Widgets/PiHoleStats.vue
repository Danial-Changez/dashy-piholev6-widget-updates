<template>
  <div class="pihole-stats">
    <!-- Status Indicator / Text -->
    <div v-if="!options.hideStatus" class="status-row">
      <span class="status-label">Pi-hole Status:</span>
      <span class="status-value" :class="{'enabled': data.status === 'enabled', 'disabled': data.status === 'disabled'}">
        {{ data.status === 'enabled' ? 'Enabled' : 'Disabled' }}
      </span>
    </div>
    <!-- Donut Chart for Blocked vs Allowed (if not hidden) -->
    <div v-if="!options.hideChart" class="chart-container">
      <canvas ref="chartCanvas" class="stats-chart"></canvas>
    </div>
    <!-- Info Stats (if not hidden) -->
    <div v-if="!options.hideInfo" class="info-container">
      <div class="info-item">
        <span class="info-label">Total Queries Today:</span>
        <span class="info-value">{{ data.dns_queries_today }}</span>
      </div>
      <div class="info-item">
        <span class="info-label">Ads Blocked Today:</span>
        <span class="info-value">{{ data.ads_blocked_today }}</span>
      </div>
      <div class="info-item">
        <span class="info-label">Percent Blocked:</span>
        <span class="info-value">{{ data.ads_percentage_today }}%</span>
      </div>
      <div class="info-item">
        <span class="info-label">Domains on Blocklist:</span>
        <span class="info-value">{{ data.domains_being_blocked }}</span>
      </div>
    </div>
  </div>
</template>

<script>
import Chart from 'chart.js/auto';

export default {
  name: 'PiHoleStats',
  props: {
    // The widget options (expects { hostname: string, apiKey: string, hideStatus: bool, hideChart: bool, hideInfo: bool })
    options: { type: Object, required: true }
  },
  data() {
    return {
      data: {},        // will hold the processed stats data (including status and summary fields)
      chartInstance: null  // Chart.js instance for the donut chart
    };
  },
  computed: {
    // Preserve the `endpoint` property (not directly used for fetch in Pi-hole 6, but kept for compatibility)
    endpoint() {
      // In Pi-hole 5, this pointed to the summary API endpoint. We keep it for reference.
      const base = this.options.hostname.replace(/\/+$/, '');
      // Use new Pi-hole 6 summary endpoint as the computed endpoint (though we authenticate separately).
      return `${base}/api/stats/summary`;
    }
  },
  methods: {
    async fetchData() {
      const baseUrl = this.options.hostname.replace(/\/+$/, '');
      const password = this.options.apiKey || '';  // Pi-hole app password (empty if none)
      let sid = null;
      try {
        // Step 1: Authenticate to get SID (if password is set)
        if (password) {
          const authResponse = await fetch(`${baseUrl}/api/auth`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ password: password })
          });
          const authData = await authResponse.json();
          if (!authData.session || authData.session.sid === undefined) {
            throw new Error('Authentication failed: no session ID returned');
          }
          sid = authData.session.sid;
          // If Pi-hole has no password set, sid will be null but valid=true; handle accordingly (no auth needed for subsequent calls)
          if (sid === null && authData.session.valid && authData.session.message === "no password set") {
            sid = null;
          }
        }
        // Step 2: Fetch summary stats
        let summaryUrl = `${baseUrl}/api/stats/summary`;
        let summaryHeaders = { 'Accept': 'application/json' };
        if (sid) summaryHeaders['sid'] = sid;
        const summaryRes = await fetch(summaryUrl, { headers: summaryHeaders });
        if (!summaryRes.ok) throw new Error('Failed to fetch summary data');
        const summaryJson = await summaryRes.json();
        // Step 3: Fetch Pi-hole status (blocking enabled/disabled)
        let statusUrl = `${baseUrl}/api/dns/blocking`;
        let statusHeaders = { 'Accept': 'application/json' };
        if (sid) statusHeaders['sid'] = sid;
        const statusRes = await fetch(statusUrl, { headers: statusHeaders });
        if (!statusRes.ok) throw new Error('Failed to fetch status');
        const statusJson = await statusRes.json();
        // Step 4: Validate expected fields in summary data
        if (summaryJson.domains_being_blocked === undefined || 
            summaryJson.dns_queries_today === undefined || 
            summaryJson.ads_blocked_today === undefined || 
            summaryJson.ads_percentage_today === undefined) {
          throw new Error('Expected data was not returned from Pi-Hole');
        }
        // Step 5: Process and combine data
        const statusEnabled = statusJson.blocking !== undefined ? statusJson.blocking : true;
        // Pi-hole returns `blocking: true` when enabled (actively blocking ads).
        // Set status as "enabled"/"disabled" to match Pi-hole 5 API.
        summaryJson.status = statusEnabled ? 'enabled' : 'disabled';
        // Ensure percentage is formatted (Pi-hole 6 might return numeric or string)
        if (typeof summaryJson.ads_percentage_today === 'number') {
          summaryJson.ads_percentage_today = summaryJson.ads_percentage_today.toFixed(1);
        }
        this.data = summaryJson;
        // Step 6: Update or create the donut chart if needed
        if (!this.options.hideChart) {
          this.updateChart();
        }
      } catch (err) {
        console.error('PiHoleStats widget error:', err);
        // Handle errors (e.g., display a message in the UI)
        // If an error occurs, we set data fields to indicate failure (so UI can show "Unable to fetch data")
        this.data = {};
        // Optionally, you could set an error message property and display it in template.
        // For now, we simply propagate or log the error.
        throw err;
      }
    },
    updateChart() {
      // Prepare data for donut chart: blocked vs allowed queries today
      const blocked = Number(this.data.ads_blocked_today) || 0;
      const total = Number(this.data.dns_queries_today) || 0;
      const allowed = total - blocked;
      const chartData = {
        labels: ['Allowed', 'Blocked'],
        datasets: [{
          data: [allowed, blocked],
          backgroundColor: ['#2ecc71', '#e74c3c'],  // green for allowed, red for blocked (matching original colors)
          hoverBackgroundColor: ['#29b765', '#c0392b']
        }]
      };
      const chartOptions = {
        responsive: true,
        cutout: '50%',  // donut hole size (50% for a donut chart appearance)
        plugins: {
          legend: {
            display: false  // we can hide legend since labels are self-explanatory or could show if desired
          },
          tooltip: {
            callbacks: {
              label: (context) => {
                // Custom tooltip: show label and value with percentage
                const label = context.label;
                const value = context.parsed;
                const percent = total > 0 ? ((value/total)*100).toFixed(1) : '0.0';
                return `${label}: ${value} (${percent}%)`;
              }
            }
          }
        }
      };
      // Initialize or update the Chart.js donut chart
      if (this.chartInstance) {
        // Update existing chart
        this.chartInstance.data = chartData;
        this.chartInstance.options = chartOptions;
        this.chartInstance.update();
      } else {
        // Create new chart
        const ctx = this.$refs.chartCanvas.getContext('2d');
        this.chartInstance = new Chart(ctx, {
          type: 'doughnut',
          data: chartData,
          options: chartOptions
        });
      }
    }
  },
  mounted() {
    // Fetch data when component mounts (the WidgetBase or parent should also call this on an interval if updateInterval is set)
    this.fetchData().catch(err => {
      // If fetchData throws, log error (already logged in console above). 
      // You could also set a user-facing error message here if desired.
    });
  }
};
</script>

<style scoped>
.pihole-stats {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
}
/* Status row styles */
.status-row {
  margin-bottom: 0.5em;
}
.status-label {
  font-weight: bold;
  margin-right: 0.5em;
}
.status-value.enabled {
  color: #2ecc71;  /* green for enabled */
}
.status-value.disabled {
  color: #e74c3c;  /* red for disabled */
}
/* Chart container */
.chart-container {
  width: 100%;
  max-width: 300px;
  margin: 1em 0;
}
/* Info stats styles */
.info-container {
  display: flex;
  flex-direction: column;
  gap: 0.25em;
}
.info-item {
  display: flex;
  justify-content: space-between;
}
.info-label {
  font-weight: bold;
}
.info-value {
  margin-left: 0.5em;
}
</style>