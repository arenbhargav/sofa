<template>
  <div class="cve-search">
    <h2>Search CVE</h2>
    <label class="quick-search-container">
      <input type="checkbox" v-model="quickSearch" class="quick-search-checkbox" />
      Quick Search
    </label>
    <div class="input-container">
      <input
        v-model="searchTerm"
        @input="quickSearch ? searchCve() : null"
        @keyup.enter="quickSearch ? null : searchCve"
        placeholder="Enter CVE ID (e.g., CVE-2023-12345 or part of it)"
      />
      <button @click="searchCve" class="action-button" :disabled="quickSearch">Search</button>
      <button @click="resetSearch" class="action-button reset-button">Reset</button>
    </div>
    <div class="button-container">
      <button @click="sortResultsByKev" class="action-button">KEV on Top</button>
      <button @click="exportToCsv" class="action-button">Export as CSV</button>
    </div>
    <div v-if="searchResults.length">
      <h3>Search Results for "{{ searchTerm }}"</h3>
      <ul>
        <li v-for="(result, index) in searchResults" :key="index" class="cve-result">
          <!-- Make CVE ID clickable -->
          <p>
            <strong>CVE ID: </strong> 
            <a :href="`/cve-details.html?cveId=${result.cveId}`" class="cve-id-link">{{ result.cveId }} (detailed info)</a>
          </p>
          <p><strong>OS Version(s):</strong> {{ formatOsVersionsWithFixes(result.osVersionDetails) }}</p>
          <p><strong>KEV:</strong> {{ result.isKev ? '🔥 Yes' : 'No' }}</p>
          <div v-if="result.urls.length">
            <p><strong>Apple security content notes:</strong></p>
            <ul class="security-notes-list">
              <li v-for="(url, idx) in result.urls" :key="idx">
                <a :href="url" target="_blank">{{ url }}</a>
              </li>
            </ul>
          </div>
          <div>
            <p><strong>Lookup Resources:</strong></p>
            <ul class="external-links-list">
              <li><a :href="`https://www.cve.org/CVERecord?id=${result.cveId}`" target="_blank">View {{ result.cveId }} on CVE.org</a></li>
              <li><a :href="`https://nvd.nist.gov/vuln/detail/${result.cveId}`" target="_blank">View {{ result.cveId }} on NVD (NIST)</a></li>
              <li v-if="result.isKev"><a :href="`https://www.cisa.gov/known-exploited-vulnerabilities-catalog?search_api_fulltext=${result.cveId}`" target="_blank">View {{ result.cveId }} on CISA KEV</a></li>
              <li><a :href="`https://cvefeed.io/vuln/detail/${result.cveId}`" target="_blank">View {{ result.cveId }} on cvefeed.io</a></li>
              <li><a :href="`https://www.opencve.io/cve/${result.cveId}`" target="_blank">View {{ result.cveId }} on OpenCVE</a></li>
            </ul>
          </div>
        </li>
      </ul>
    </div>
    <div v-else-if="searchTerm">
      <p>No results found for "{{ searchTerm }}"</p>
    </div>
  </div>
</template>

<script>
import macOSData from '@v1/macos_data_feed.json';
import iOSData from '@v1/ios_data_feed.json';

export default {
  data() {
    return {
      searchTerm: '',
      searchResults: [],
      quickSearch: false,
    };
  },
  methods: {
    searchCve() {
      const term = this.searchTerm.trim();
      const regex = /^[A-Za-z0-9-]+$/;

      if (!regex.test(term)) {
        console.error('Invalid search term:', term);
        this.searchResults = [];
        return;
      }

      console.log(`Searching for CVE: ${term}`);
      const searchRegex = new RegExp(term.replace(/[^a-zA-Z0-9-]/g, ''), 'i');

      if (!term) {
        this.searchResults = [];
        return;
      }

      const macOSResults = this.searchInDataFeed(macOSData, searchRegex, 'macOS');
      const iOSResults = this.searchInDataFeed(iOSData, searchRegex, 'iOS');

      this.searchResults = this.mergeResults([...macOSResults, ...iOSResults]);
      console.log('Search Results:', this.searchResults);
    },
    searchInDataFeed(dataFeed, regex, platform) {
      const results = [];
      console.log(`Searching in ${platform} data feed`);

      dataFeed.OSVersions.forEach((os) => {
        console.log(`Checking OS Version: ${os.OSVersion}`);

        os.SecurityReleases.forEach((release) => {
          console.log(`Checking Release: ${release.UpdateName}, CVEs: ${Object.keys(release.CVEs)}`);

          Object.keys(release.CVEs).forEach(cveId => {
            if (regex.test(cveId)) {
              console.log(`Found CVE: ${cveId} in ${os.OSVersion}`);
              const url = release.SecurityInfo ? [release.SecurityInfo] : [];
              results.push({
                cveId: cveId,
                osVersionDetail: `${release.ProductName} ${release.ProductVersion}`, 
                isKev: release.ActivelyExploitedCVEs.includes(cveId),
                platform: platform,
                urls: url,
              });
            }
          });
        });
      });

      return results;
    },
    mergeResults(results) {
      const mergedResults = {};

      results.forEach((result) => {
        if (mergedResults[result.cveId]) {
          mergedResults[result.cveId].osVersionDetails.push(result.osVersionDetail); 
          mergedResults[result.cveId].isKev = mergedResults[result.cveId].isKev || result.isKev;
          mergedResults[result.cveId].urls.push(...result.urls);
        } else {
          mergedResults[result.cveId] = {
            cveId: result.cveId,
            osVersionDetails: [result.osVersionDetail], 
            isKev: result.isKev,
            urls: result.urls,
          };
        }
      });

      Object.values(mergedResults).forEach(result => {
        result.urls = [...new Set(result.urls)];
        result.osVersionDetails = [...new Set(result.osVersionDetails)]; 
      });

      return Object.values(mergedResults);
    },
    sortResultsByKev() {
      this.searchResults.sort((a, b) => b.isKev - a.isKev);
    },
    resetSearch() {
      this.searchTerm = '';
      this.searchResults = [];
    },
    exportToCsv() {
      const headers = ['CVE ID', 'OS Versions', 'KEV', 'Apple security content notes'];
      const rows = this.searchResults.map(result => [
        `"${result.cveId}"`,
        `"${this.formatOsVersionsWithFixes(result.osVersionDetails)}"`,
        result.isKev ? '"Yes"' : '"No"',
        `"${result.urls.join(', ')}"`
      ]);

      const csvContent = [headers, ...rows].map(e => e.join(',')).join('\n');
      const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' });
      const link = document.createElement('a');
      const url = URL.createObjectURL(blob);

      link.setAttribute('href', url);
      link.setAttribute('download', 'cve_search_results.csv');
      link.style.visibility = 'hidden';

      document.body.appendChild(link);
      link.click();
      document.body.removeChild(link);
    },
    formatOsVersionsWithFixes(osVersionDetails) {
      return osVersionDetails.join(', ');
    }
  },
};
</script>

<style scoped>
.cve-search {
  margin: 20px 0;
}

.input-container {
  display: flex;
  align-items: center;
  gap: 10px;
}

.cve-search input {
  flex: 1;
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 5px;
  font-size: 16px;
}

.action-button,
.reset-button {
  padding: 10px 20px;
  border: 1px solid #5672cd;
  border-radius: 5px;
  background-color: #5672cd;
  color: #fff;
  cursor: pointer;
  transition: background-color 0.2s ease-in-out;
  font-size: 16px;
  box-sizing: border-box;
}

.reset-button {
  background-color: #888;
  border-color: #888;
}

.action-button:disabled {
  background-color: #ccc;
  border-color: #ccc;
  cursor: not-allowed;
}

.reset-button:hover {
  background-color: #666;
}

.quick-search-container {
  display: inline-flex;
  align-items: center;
  margin-left: auto;
  gap: 5px;
  margin-bottom: 10px;
}

.quick-search-container input {
  width: auto;
}

.button-container {
  display: flex;
  justify-content: flex-start;
  gap: 10px;
  margin-top: 20px;
  align-items: center;
}

.cve-search ul {
  list-style-type: none;
  padding: 0;
}

.cve-search li {
  padding: 10px;
  margin-bottom: 10px;
  border: 1px solid #ddd;
  border-radius: 5px;
}

.cve-search h3 {
  margin-top: 0;
  font-size: 16px;
  margin: 10px;
}

.cve-search li p {
  margin: 5px 0;
}

.cve-result ul {
  list-style: none;
  padding-left: 20px;
}

.cve-result .security-notes-list,
.cve-result .external-links-list {
  padding-left: 0;
}

.cve-result .security-notes-list li,
.cve-result .external-links-list li {
  margin-bottom: 5px;
  border: none;
}

.cve-search a {
  color: #1e90ff;
  text-decoration: none;
}

.cve-search a:hover {
  text-decoration: underline;
}

.view-details-link {
  color: #1e90ff;
  text-decoration: none;
}

.view-details-link:hover {
  text-decoration: underline;
}
</style>
