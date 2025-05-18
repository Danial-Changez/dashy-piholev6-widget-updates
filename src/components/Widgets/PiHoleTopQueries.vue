<template>
  <div class="pihole-top-queries">
    <div class="queries-column">
      <h3>Top Ads Blocked</h3>
      <ul class="query-list">
        <li v-for="(count, domain) in data.top_ads" :key="'blocked-'+domain">
          <span class="domain">{{ domain }}</span>
          <span class="count">{{ count }}</span>
        </li>
        <li v-if="Object.keys(data.top_ads).length === 0">None</li>
      </ul>
    </div>
    <div class="queries-column">
      <h3>Top Queries</h3>
      <ul class="query-list">
        <li v-for="(count, domain) in data.top_queries" :key="'allowed-'+domain">
          <span class="domain">{{ domain }}</span>
          <span class="count">{{ count }}</span>
        </li>
        <li v-if="Object.keys(data.top_queries).length === 0">None</li>
      </ul>
    </div>
  </div>
</template>

<script>
export default {
  name: 'PiHoleTopQueries',
  props: {
    options: { type: Object, required: true }
  },
  data() {
    return {
      data: {
        top_ads: {},      // object mapping blocked domain -> count
        top_queries: {}   // object mapping allowed domain -> count
      }
    };
  },
  computed: {
    // Keep `endpoint` property for compatibility (not directly used for separate calls in v6)
    endpoint() {
      const base = this.options.hostname.replace(/\/+$/, '');
      // In Pi-hole 5, this was /admin/api.php?topItems=<count>&auth=<apiKey>
      // For reference, point to new top_domains endpoint (this is just for consistency, actual fetch uses two calls).
      const cnt = this.count;
      const authParam = this.options.apiKey ? '(authenticated via sid)' : '(no auth)';
      return `${base}/api/stats/top_domains?count=${cnt}&blocked=... ${authParam}`;
    },
    // Determine number of queries to display (defaults to 10 if not provided)
    count() {
      const c = parseInt(this.options.count);
      return (Number.isInteger(c) && c > 0) ? c : 10;
    }
  },
  methods: {
    async fetchData() {
      const baseUrl = this.options.hostname.replace(/\/+$/, '');
      const password = this.options.apiKey || '';
      let sid = null;
      try {
        // Authenticate to get sid (if password provided)
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
        // Fetch top blocked domains (ads)
        const blockedUrl = `${baseUrl}/api/stats/top_domains?blocked=true&count=${this.count}`;
        const headers = sid ? { 'Accept': 'application/json', 'sid': sid } : { 'Accept': 'application/json' };
        const blockedRes = await fetch(blockedUrl, { headers });
        if (!blockedRes.ok) throw new Error('Failed to fetch top blocked domains');
        const blockedData = await blockedRes.json();
        // Fetch top allowed domains
        const allowedUrl = `${baseUrl}/api/stats/top_domains?blocked=false&count=${this.count}`;
        const allowedRes = await fetch(allowedUrl, { headers });
        if (!allowedRes.ok) throw new Error('Failed to fetch top allowed domains');
        const allowedData = await allowedRes.json();
        // Validate that we have the expected keys
        if (blockedData === null || blockedData.top_sources || allowedData === null) {
          // Pi-hole returns an object of domain->count for these endpoints directly
          // If we got something unexpected (like keys named differently), throw error
          throw new Error('Expected data was not returned from Pi-Hole');
        }
        // Pi-hole 6's /api/stats/top_domains returns an object mapping domains to counts.
        // Use the response directly as our data (ensuring it has correct type).
        this.data.top_ads = (typeof blockedData === 'object' && !Array.isArray(blockedData)) ? blockedData : {};
        this.data.top_queries = (typeof allowedData === 'object' && !Array.isArray(allowedData)) ? allowedData : {};
      } catch (err) {
        console.error('PiHoleTopQueries widget error:', err);
        this.data.top_ads = {};
        this.data.top_queries = {};
        throw err;
      }
    }
  },
  mounted() {
    this.fetchData().catch(err => {
      // Error handling (already logged in console). 
      // Could set an error message in UI if needed.
    });
  }
};
</script>

<style scoped>
.pihole-top-queries {
  display: flex;
  justify-content: space-between;
}
.queries-column {
  width: 48%;
}
.queries-column h3 {
  margin-bottom: 0.5em;
  font-weight: bold;
}
.query-list {
  list-style: none;
  padding: 0;
  margin: 0;
}
.query-list li {
  display: flex;
  justify-content: space-between;
  padding: 2px 0;
  border-bottom: 1px dotted rgba(255,255,255,0.1);
}
.query-list li:last-child {
  border-bottom: none;
}
.domain {
  flex: 1;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.count {
  margin-left: 0.5em;
  font-weight: bold;
}
</style>
