<script setup>
import { ref, computed, onMounted } from 'vue';

var harData = ref(null);
var selectedEntry = ref(null);
var searchQuery = ref('');
var isModified = ref(false);
var selectedIds = ref(new Set());
var responseTab = ref('preview');
var activeTab = ref('request');

// --- Helpers ---
const formatTime = (dateTime) => {
  var date = new Date(dateTime);
  return date.toLocaleTimeString([], {
    hour12: false,
    hour: '2-digit',
    minute: '2-digit',
    second: '2-digit'
  });
};

const shortUrl = (url) => url ? url.split('/').pop().split('?')[0] : 'unknown';

const getUrlPath = (fullUrl) => {
  try {
    var url = new URL(fullUrl);
    return url.pathname + url.search;
  } catch { return fullUrl; }
};

const getContentType = (entry) => {
  var mimeType = entry.response.content.mimeType || '';
  var url = entry.request.url.toLowerCase();

  if (mimeType.includes('image/') || url.match(/\.(jpg|jpeg|png|gif|webp|svg|ico)$/)) {
    return 'image';
  }

  if (mimeType.includes('json') || entry.response.content.text?.trim().startsWith('{')) {
    return 'json';
  }

  return 'text';
};

const getResponseImage = (entry) => {
  var content = entry.response.content;

  if (!content.size) {
    var blankSVG = `<svg version="1.1" viewBox="0 0 1 1" xmlns="http://www.w3.org/2000/svg"></svg>`;
    return `data:image/svg+xml;utf8,${encodeURIComponent(blankSVG)}`;
  }

  if (!content.text) return '';

  var cleanText = content.text.replace(/\s/g, '');
  if (content.encoding === 'base64') {
    return `data:${content.mimeType};base64,${cleanText}`;
  }

  if (content.mimeType.includes('svg')) {
    return `data:image/svg+xml;utf8,${encodeURIComponent(content.text)}`;
  }

  return '';
};

// --- Обчислювальні властивості ---

const initiatorStack = computed(() => {
  return selectedEntry.value?._initiator?.stack?.callFrames || [];
});

const filteredEntries = computed(() => {
  if (!harData.value) return [];

  var entries = harData.value.log.entries;
  if (!searchQuery.value) return entries;

  return entries.filter(entry =>
    entry.request.url.toLowerCase().includes(searchQuery.value.toLowerCase())
  );
});

const isAllSelected = computed(() => {
  return filteredEntries.value.length > 0
    && selectedIds.value.size === filteredEntries.value.length;
});

// Statistics
const stats = computed(() => {
  if (!harData.value)
    return { count: 0, size: '0 MB', errors: 0 };

  var entries = filteredEntries.value;
  var totalSize = entries.reduce((acc, e) => acc + (e.response.content.size || 0), 0);
  var errors = entries.filter(e => e.response.status >= 400).length;

  return {
    count: entries.length,
    size: (totalSize / 1024 / 1024).toFixed(2) + ' MB',
    errors,
  };
});

// Waterfall Timeline Stats
const timelineStats = computed(() => {
  if (!harData.value?.log.entries.length) return null;

  var entries = harData.value.log.entries;
  var startTimes = entries.map(e => new Date(e.startedDateTime).getTime());
  var endTimes = entries.map(e => new Date(e.startedDateTime).getTime() + e.time);

  var minStart = Math.min(...startTimes);
  var maxEnd = Math.max(...endTimes);

  return {
    minStart,
    totalDuration: maxEnd - minStart
  };
});

const getWaterfallStyle = (entry) => {
  if (!timelineStats.value) return {};

  var startTime = new Date(entry.startedDateTime).getTime();
  var startOffset = ((startTime - timelineStats.value.minStart) / timelineStats.value.totalDuration) * 100;
  var width = (entry.time / timelineStats.value.totalDuration) * 100;

  return {
    left: `${startOffset}%`,
    width: `${Math.max(width, 0.5)}%`,
  };
};

const harMetadata = computed(() => {
  var { creator, version } = harData.value?.log ?? {
    creator: { name: 'Vue', version: '3' },
    version: '?',
  };

  return `${creator?.name} v${creator?.version} (HAR v${version})`;
});

// --- Actions ---

const handleFileUpload = (event) => {
  var file = event.target.files[0];
  if (!file) return;

  var reader = new FileReader();

  reader.onload = (e) => {
    try {

      harData.value = JSON.parse(e.target.result);
      selectedEntry.value = null;
      isModified.value = false;
      searchQuery.value = '';
      selectedIds.value.clear();

    } catch (error) {

      console.error(error);
      alert("Помилка читання файлу. Переконайтеся, що це коректний HAR.");

    }
  };

  reader.readAsText(file);
};

const deleteSelected = () => {
  if (!confirm(`Видалити обрані (${selectedIds.value.size}) записи?`)) return;

  var toDelete = new Set();
  filteredEntries.value.forEach((entry, index) => {
    if (selectedIds.value.has(index)) {
      toDelete.add(entry);
    }
  });

  harData.value.log.entries = harData.value.log.entries.filter(
    entry => !toDelete.has(entry)
  );

  selectedIds.value.clear();
  isModified.value = true;
  selectedEntry.value = null;
};

const toggleRowSelection = (index) => {
  if (selectedIds.value.has(index)) selectedIds.value.delete(index);
  else selectedIds.value.add(index);
};

const selectAllOnPage = (event) => {
  selectedIds.value.clear();

  if (event.target.checked) {
    filteredEntries.value.forEach((_, index) => {
      selectedIds.value.add(index);
    });
  }
};

const formatJSON = (text) => {
  if (!text) return 'Тіло відповіді порожнє';

  try {
    return JSON.stringify(JSON.parse(text), null, 2);
  } catch (error) {
    console.error(error);
    return text;
  }
};

const exportHAR = () => {
  var blob = new Blob([JSON.stringify(harData.value, null, 2)], { type: 'application/json' });
  var url = URL.createObjectURL(blob);
  var a = document.createElement('a');

  a.href = url;
  a.download = `cleaned_${Date.now()}.har`;
  a.click();
};

const getStatusClass = (stat) => (stat >= 400 ? 'status-error' : stat >= 200 && stat < 300 ? 'status-success' : 'status-warning');
const isSensitiveCookie = (name) => ['auth', 'token', 'session', 'jwt', 'sid', 'phpsessid'].some(s => name.toLowerCase().includes(s));
</script>

<template>
  <div class="app-container">

    <header class="top-bar">
      <h2>HAR Viewer</h2>

      <div class="actions animated-fade-in">
        <input type="file" @change="handleFileUpload" accept=".har" id="file" hidden />
        <label for="file" class="btn-upload">
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none"
            stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <path
              d="M11.35 22H6a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h8a2.4 2.4 0 0 1 1.706.706l3.588 3.588A2.4 2.4 0 0 1 20 8v5.35" />
            <path d="M14 2v5a1 1 0 0 0 1 1h5" />
            <path d="M14 19h6" />
            <path d="M17 16v6" />
          </svg>
        </label>

        <button @click="deleteSelected" :disabled="selectedIds.size === 0" class="btn-delete-manual">
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none"
            stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6" />
            <path d="M3 6h18" />
            <path d="M8 6V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2" />
          </svg>
          <span class="btn-count">({{ selectedIds.size }})</span>
        </button>

        <button @click="exportHAR" :disabled="!isModified" class="btn-export">
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none"
            stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <path d="M12 17V3" />
            <path d="m6 11 6 6 6-6" />
            <path d="M19 21H5" />
          </svg>
        </button>
      </div>

      <div class="search-wrapper">
        <input v-model="searchQuery" type="text" placeholder="Шукати URL (напр. /api або .svg)" class="search-input" />
        <button v-if="searchQuery" @click="searchQuery = ''" class="btn-search-clear" title="Очистити пошук">✕</button>
      </div>
    </header>

    <div class="stats-bar" :aria-disabled="!harData">
      <span class="metadata">{{ harMetadata }}</span>
      <div class="separator"></div>
      <span class="label">Знайдено: <strong>{{ stats.count }}</strong></span>
      <span class="label">Розмір: <strong>{{ stats.size }}</strong></span>
      <span class="label" :class="{ 'text-error': stats.errors > 0 }">Помилки: <strong>{{
        stats.errors }}</strong></span>
    </div>

    <main class="workspace">
      <section class="sidebar">
        <table class="har-table">
          <thead>
            <tr>
              <th style="width: 30px;">
                <input type="checkbox" :checked="isAllSelected" @change="selectAllOnPage" />
              </th>
              <th style="width: 70px;">Час</th>
              <th style="width: 60px;">Метод</th>
              <th style="width: 50px;">Proto.</th>
              <th style="width: 50px;">Статус</th>
              <th>Шлях (URL)</th>
              <th style="width: 100px;">Waterfall</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(entry, index) in filteredEntries" :key="index"
              :class="{ 'active-row': selectedEntry === entry }" @click="selectedEntry = entry">
              <td @click.stop>
                <input type="checkbox" :checked="selectedIds.has(index)" @change="toggleRowSelection(index)" />
              </td>
              <td class="time-cell">{{ formatTime(entry.startedDateTime) }}</td>
              <td>
                <span :class="['method-badge', entry.request.method.toLowerCase()]">{{ entry.request.method }}</span>
              </td>
              <td class="uppercase">{{ entry.response.httpVersion || entry.request.httpVersion || '?' }}</td>
              <td>
                <span :class="['status-badge', getStatusClass(entry.response.status)]">{{
                  entry.response.status }}</span>
              </td>
              <td class="url-cell">{{ getUrlPath(entry.request.url) }}</td>
              <td>
                <div class="waterfall-container">
                  <div class="waterfall-bar" :style="getWaterfallStyle(entry)"></div>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </section>

      <section class="details-panel" v-if="selectedEntry">
        <div class="details-header">
          <nav class="main-tabs">
            <button :class="{ active: activeTab === 'request' }" @click="activeTab = 'request'">Request</button>
            <button :class="{ active: activeTab === 'payload' }" @click="activeTab = 'payload'"
              :disabled="!selectedEntry.request.postData">Payload</button>
            <button :class="{ active: activeTab === 'response' }" @click="activeTab = 'response'">Response</button>
            <button :class="{ active: activeTab === 'cookies' }" @click="activeTab = 'cookies'">Cookies</button>
            <button v-if="initiatorStack.length" :class="{ active: activeTab === 'initiator' }"
              @click="activeTab = 'initiator'">Initiator</button>
          </nav>
        </div>

        <div class="tab-content">
          <div v-if="activeTab === 'request'">
            <h4>General</h4>

            <div class="header-grid">
              <span class="h-name">Request URL:</span> <span class="h-value">{{ selectedEntry.request.url }}</span>
              <span class="h-name">Resource Type:</span> <span class="h-value">{{ selectedEntry._resourceType }}</span>
              <span class="h-name">Connection ID:</span> <span class="h-value">{{ selectedEntry._connectionId ?? '' }}</span>
            </div>

            <h4>Headers</h4>
            <div class="header-section">
              <div class="header-grid">
                <template v-for="h in selectedEntry.request.headers" :key="'rh'+h.name"><span class="h-name">{{ h.name
                }}:</span><span class="h-value">{{ h.value }}</span></template>
              </div>
            </div>
          </div>

          <div v-if="activeTab === 'payload'">
            <template v-if="selectedEntry.request.postData">
              <h4>Request Data (Payload)</h4>

              <div class="info-block payload">
                <div class="mime-type">Type: {{ selectedEntry.request.postData.mimeType }}</div>
                <pre class="payload-pre">{{ formatJSON(selectedEntry.request.postData.text) }}</pre>
              </div>
            </template>
          </div>

          <div v-if="activeTab === 'response'">
            <h4>Response Body</h4>

            <div class="response-container">
              <div class="response-tabs">
                <button :class="{ active: responseTab === 'preview' }" @click="responseTab = 'preview'">Preview</button>
                <button :class="{ active: responseTab === 'raw' }" @click="responseTab = 'raw'">Raw</button>
              </div>
              <div class="response-viewer">
                <pre v-if="responseTab === 'raw'" class="response-pre">{{ selectedEntry.response.content.text || 'No Body'
                }}</pre>

                <div v-else class="preview-content">
                  <div v-if="getContentType(selectedEntry) === 'image'" class="image-preview">
                    <img v-if="!selectedEntry.response.content.size" src="/src/assets/icons/no-body.svg" alt="No Content Body Image" />
                    <img v-else :src="getResponseImage(selectedEntry)" alt="Response Image" />
                  </div>

                  <pre v-else-if="getContentType(selectedEntry) === 'json'" class="response-pre json-formatted">{{
                    formatJSON(selectedEntry.response.content.text) }}</pre>
                  <pre v-else
                    class="response-pre">{{ selectedEntry.response.content.text || 'No preview available' }}</pre>
                </div>

              </div>
            </div>
          </div>

          <div v-if="activeTab === 'cookies'">
            <h4>Cookies</h4>
            <div v-if="selectedEntry.request.cookies.length || selectedEntry.response.cookies.length">
              <table class="cookie-table">
                <thead>
                  <tr>
                    <th>Name</th>
                    <th>Value</th>
                    <th>Source</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="c in selectedEntry.request.cookies" :key="'req' + c.name">
                    <td :class="{ 'sensitive-cookie': isSensitiveCookie(c.name) }">{{ c.name }}</td>
                    <td class="h-value">{{ c.value }}</td>
                    <td>Request</td>
                  </tr>
                  <tr v-for="c in selectedEntry.response.cookies" :key="'res' + c.name">
                    <td :class="{ 'sensitive-cookie': isSensitiveCookie(c.name) }">{{ c.name }}</td>
                    <td class="h-value">{{ c.value }}</td>
                    <td>Response</td>
                  </tr>
                </tbody>
              </table>
            </div>
            <div v-else class="empty-note">Cookies відсутні</div>
          </div>

          <div v-if="activeTab === 'initiator'" class="initiator-panel">
            <h4>Request Call Stack</h4>
            <div class="stack-trace">
              <div v-for="(frame, idx) in initiatorStack" :key="idx" class="stack-frame">
                <span class="frame-func">{{ frame.functionName || '(anonymous)' }}</span>
                <span class="frame-url" :title="frame.url">
                  {{ shortUrl(frame.url) }}:{{ frame.lineNumber + 1 }}
                </span>
                <span class="frame-id">ID: {{ frame.scriptId }}</span>
              </div>
            </div>

          </div>

        </div>
      </section>
      <div v-else class="empty-state">Оберіть запит для аналізу</div>
    </main>
  </div>
</template>

<style></style>
