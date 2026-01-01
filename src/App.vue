<script setup>
import { ref, computed } from 'vue';

var harData = ref(null);
var selectedEntry = ref(null);
var searchQuery = ref('');
var isModified = ref(false);

const filteredEntries = computed(() => {
  if (!harData.value) return [];

  var entries = harData.value.log.entries;
  if (!searchQuery.value) return entries;

  return entries.filter(entry =>
    entry.request.url.toLowerCase().includes(searchQuery.value.toLowerCase())
  );
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

const handleFileUpload = (event) => {
  var file = event.target.files[0];
  if (!file) return;

  var reader = new FileReader();

  reader.onload = (e) => {
    try {

      harData.value = JSON.parse(e.target.result);
      selectedEntry.value = null;
      isModified.value = false;

    } catch (error) {

      console.log(error);
      alert("Помилка читання файлу. Переконайтеся, що це коректний HAR.");

    }
  };

  reader.readAsText(file);
};

const deleteFiltered = () => {
  if (!confirm(`Видалити ${filteredEntries.value.length} записів?`)) return;

  var urlsToDelete = new Set(filteredEntries.value.map(e => e.request.url));

  harData.value.log.entries = harData.value.log.entries.filter(
    entry => !urlsToDelete.has(entry.request.url)
  );

  isModified.value = true;
  searchQuery.value = '';
  selectedEntry.value = null;
};

const formatResponse = (entry) => {
  var text = entry.response.content.text;
  if (!text) return 'Тіло відповіді порожнє';

  try {
    return JSON.stringify(JSON.parse(text), null, 2);
  } catch (error) {
    console.error(error);
    return text;
  }
};

const exportHAR = () => {
  if (!harData.value) return;

  var dataStr = JSON.stringify(harData.value, null, 2);
  var blob = new Blob([dataStr], { type: 'application/json' });
  var url = URL.createObjectURL(blob);

  var link = document.createElement('a');
  link.href = url;
  link.download = `cleaned_${new Date().getTime()}.har`;
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
  URL.revokeObjectURL(url);
};

const getStatusClass = (s) => (s >= 400 ? 'status-error' : s >= 200 && s < 300 ? 'status-success' : 'status-warning');
const isSensitiveCookie = (name) => ['auth', 'token', 'session', 'jwt', 'sid', 'phpsessid'].some(s => name.toLowerCase().includes(s));
</script>

<template>
  <div class="app-container">

    <header class="top-bar">
      <h2>🛡️ HAR Viewer</h2>
      <div class="actions">
        <input type="file" @change="handleFileUpload" accept=".har" id="file" hidden />
        <label for="file" class="btn-upload">Завантажити файл</label>

        <button v-if="isModified" @click="exportHAR" class="btn-export animated-fade-in">💾 Зберегти зміни (.har)</button>
      </div>

      <input v-model="searchQuery" type="text" placeholder="Фільтр по URL (напр. /api/)..." class="search-input" />
    </header>

    <div class="stats-bar" v-if="harData">
      <span>Знайдено: <strong>{{ stats.count }}</strong></span>
      <span>Розмір: <strong>{{ stats.size }}</strong></span>
      <span :class="{ 'text-error': stats.errors > 0 }">Помилки: <strong>{{ stats.errors }}</strong></span>

      <button v-if="searchQuery && filteredEntries.length > 0" @click="deleteFiltered" class="btn-delete">
        Видалити знайдені ({{ filteredEntries.length }})
      </button>
    </div>

    <main class="workspace">
      <section class="sidebar">
        <table class="har-table">
          <thead>
            <tr>
              <th style="width: 60px;">Метод</th>
              <th style="width: 50px;">Код</th>
              <th>URL</th>
              <th style="width: 120px;">Waterfall</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(entry, index) in filteredEntries" :key="index"
              :class="{ 'active-row': selectedEntry === entry }" @click="selectedEntry = entry">
              <td :class="['method', entry.request.method.toLowerCase()]">{{ entry.request.method }}</td>
              <td :class="['status', getStatusClass(entry.response.status)]">{{ entry.response.status }}</td>
              <td class="url">{{ entry.request.url }}</td>
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
        <div class="details-content">
          <strong>URL:</strong> <span class="url-break">{{ selectedEntry.request.url }}</span>

          <div class="tabs-container" v-if="selectedEntry">
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
            <p v-else>Cookies відсутні</p>

            <h4>Headers</h4>
            <div class="header-grid">
              <template v-for="h in selectedEntry.request.headers" :key="h.name">
                <span class="h-name">{{ h.name }}:</span>
                <span class="h-value">{{ h.value }}</span>
              </template>
            </div>

            <h4>Response Body</h4>
            <pre class="response-pre">{{ formatResponse(selectedEntry) }}</pre>
          </div>

        </div>
      </section>
      <div v-else class="empty-state">Оберіть запит для аналізу</div>
    </main>
  </div>
</template>

<style></style>
