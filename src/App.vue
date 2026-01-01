<script setup>
import { ref } from 'vue';

var harData = ref(null);
var selectedEntry = ref(null);

var handleFileUpload = (event) => {
  var file = event.target.files[0];
  if (!file) return;

  var reader = new FileReader();
  reader.onload = (e) => {
    try {
      harData.value = JSON.parse(e.target.result);
    } catch (error) {
      alert("Помилка читання файлу. Переконайтеся, що це коректний HAR.");
    }
  };

  reader.readAsText(file);
};
</script>

<template>
  <div id="app">
    <h1>HAR Viewer</h1>
    
    <div class="upload-section">
      <input type="file" @change="handleFileUpload" accept=".har" />
    </div>

    <div v-if="harData" class="content">
      <table>
        <thead>
          <tr>
            <th>Метод</th>
            <th>Статус</th>
            <th>URL</th>
            <th>Час (мс)</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(entry, index) in harData.log.entries" :key="index" @click="selectedEntry = entry" class="row">
            <td>{{ entry.request.method }}</td>
            <td>{{ entry.response.status }}</td>
            <td class="url-cell">{{ entry.request.url }}</td>
            <td>{{ Math.round(entry.time) }}</td>
          </tr>
        </tbody>
      </table>

      <div v-if="selectedEntry" class="details">
        <h3>Деталі запиту</h3>
        <pre>{{ JSON.stringify(selectedEntry.request.headers, null, 2) }}</pre>
      </div>
    </div>
  </div>
</template>

<style></style>
