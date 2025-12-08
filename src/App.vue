<script setup lang="ts">
import { ref } from 'vue'
import DatabaseTests from './components/DatabaseTests.vue'
import WebTests from './components/WebTests.vue'
import FilesystemTests from './components/FilesystemTests.vue'
import ShellTests from './components/ShellTests.vue'

const activeTab = ref('database')

const tabs = [
  { id: 'database', label: 'Database' },
  { id: 'web', label: 'Web/HTTP' },
  { id: 'filesystem', label: 'Filesystem' },
  { id: 'shell', label: 'Shell' },
]
</script>

<template>
  <div class="min-h-screen bg-base-200 p-4">
    <div class="max-w-4xl mx-auto">
      <h1 class="text-3xl font-bold text-center mb-6">
        Haex Permission Tests
      </h1>

      <p class="text-center text-base-content/70 mb-8">
        This extension tests the haex-vault permission system.
        Each tab contains tests for different permission types.
      </p>

      <!-- DaisyUI Tabs -->
      <div role="tablist" class="tabs tabs-boxed mb-6">
        <a
          v-for="tab in tabs"
          :key="tab.id"
          role="tab"
          class="tab"
          :class="{ 'tab-active': activeTab === tab.id }"
          @click="activeTab = tab.id"
        >
          {{ tab.label }}
        </a>
      </div>

      <!-- Tab Content -->
      <div class="bg-base-100 rounded-box p-6 shadow-lg">
        <DatabaseTests v-if="activeTab === 'database'" />
        <WebTests v-if="activeTab === 'web'" />
        <FilesystemTests v-if="activeTab === 'filesystem'" />
        <ShellTests v-if="activeTab === 'shell'" />
      </div>
    </div>
  </div>
</template>
