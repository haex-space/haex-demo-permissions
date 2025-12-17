<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { createHaexVaultSdk } from '@haex-space/vault-sdk'

interface TestResult {
  name: string
  description: string
  expected: 'prompt' | 'success' | 'error'
  status: 'pending' | 'running' | 'success' | 'error'
  message?: string
  instructions?: string
}

const client = createHaexVaultSdk()
const extensionPrefix = ref<string>('')

// Get extension prefix on mount
onMounted(async () => {
  try {
    await client.ready()
    const info = client.extensionInfo
    if (info) {
      extensionPrefix.value = `${info.publicKey}__${info.name}__`
    }
  } catch (e) {
    console.error('Failed to get extension context:', e)
    extensionPrefix.value = 'unknown__unknown__'
  }
})

const tests = ref<TestResult[]>([
  // Database Permission Prompt Tests
  {
    name: 'DB: Access other extension table (Allow)',
    description: 'Try to SELECT from another extension\'s table - click "Allow" when prompted',
    expected: 'prompt',
    status: 'pending',
    instructions: 'When the permission dialog appears, click "Allow" to grant permanent access.',
  },
  {
    name: 'DB: Access other extension table (Deny)',
    description: 'Try to SELECT from another extension\'s table - click "Deny" when prompted',
    expected: 'prompt',
    status: 'pending',
    instructions: 'When the permission dialog appears, click "Deny" to block access permanently.',
  },
  {
    name: 'DB: Access other extension table (Allow Once)',
    description: 'Try to SELECT from another extension\'s table - click "Allow Once" when prompted',
    expected: 'prompt',
    status: 'pending',
    instructions: 'When the permission dialog appears, click "Allow Once" for one-time access.',
  },
  // Web Permission Prompt Tests
  {
    name: 'Web: Fetch unlisted URL (Allow)',
    description: 'Try to fetch https://api.github.com - click "Allow" when prompted',
    expected: 'prompt',
    status: 'pending',
    instructions: 'When the permission dialog appears, click "Allow" to grant permanent access.',
  },
  {
    name: 'Web: Fetch unlisted URL (Deny)',
    description: 'Try to fetch https://jsonplaceholder.typicode.com - click "Deny" when prompted',
    expected: 'prompt',
    status: 'pending',
    instructions: 'When the permission dialog appears, click "Deny" to block access.',
  },
  {
    name: 'Web: Fetch unlisted URL (Allow Once)',
    description: 'Try to fetch https://dummyjson.com - click "Allow Once" when prompted',
    expected: 'prompt',
    status: 'pending',
    instructions: 'When the permission dialog appears, click "Allow Once" for one-time access.',
  },
])

// Individual test runners
const runDbAllowTest = async () => {
  const test = tests.value[0]
  test.status = 'running'
  test.message = 'Waiting for permission prompt...'

  try {
    // Try to access a table that doesn't belong to this extension
    // This should trigger a permission prompt
    const result = await client.query(
      `SELECT name FROM sqlite_master WHERE type='table' LIMIT 1`
    )
    test.status = 'success'
    test.message = `Access granted! Found ${result.length} tables.`
  } catch (e) {
    const errorMsg = e instanceof Error ? e.message : String(e)
    if (errorMsg.includes('Permission denied') || errorMsg.includes('denied')) {
      test.status = 'error'
      test.message = `Access denied (user clicked Deny or error): ${errorMsg}`
    } else {
      test.status = 'error'
      test.message = `Unexpected error: ${errorMsg}`
    }
  }
}

const runDbDenyTest = async () => {
  const test = tests.value[1]
  test.status = 'running'
  test.message = 'Waiting for permission prompt...'

  try {
    // Try to access haex_settings table
    await client.query(`SELECT * FROM haex_settings LIMIT 1`)
    test.status = 'error'
    test.message = 'Access was granted but expected denial'
  } catch (e) {
    const errorMsg = e instanceof Error ? e.message : String(e)
    if (errorMsg.includes('Permission denied') || errorMsg.includes('denied')) {
      test.status = 'success'
      test.message = `Correctly denied: ${errorMsg}`
    } else {
      test.status = 'error'
      test.message = `Unexpected error: ${errorMsg}`
    }
  }
}

const runDbAllowOnceTest = async () => {
  const test = tests.value[2]
  test.status = 'running'
  test.message = 'Waiting for permission prompt...'

  try {
    // Try to access haex_extensions table
    const result = await client.query(`SELECT id FROM haex_extensions LIMIT 1`)
    test.status = 'success'
    test.message = `One-time access granted! Found ${result.length} extensions.`
  } catch (e) {
    const errorMsg = e instanceof Error ? e.message : String(e)
    if (errorMsg.includes('Permission denied') || errorMsg.includes('denied')) {
      test.status = 'error'
      test.message = `Access denied: ${errorMsg}`
    } else {
      test.status = 'error'
      test.message = `Unexpected error: ${errorMsg}`
    }
  }
}

const runWebAllowTest = async () => {
  const test = tests.value[3]
  test.status = 'running'
  test.message = 'Waiting for permission prompt...'

  try {
    const response = await client.web.fetchAsync('https://api.github.com')
    test.status = 'success'
    test.message = `Access granted! Status: ${response.status}`
  } catch (e) {
    const errorMsg = e instanceof Error ? e.message : String(e)
    if (errorMsg.includes('Permission denied') || errorMsg.includes('denied')) {
      test.status = 'error'
      test.message = `Access denied (user clicked Deny): ${errorMsg}`
    } else {
      test.status = 'error'
      test.message = `Unexpected error: ${errorMsg}`
    }
  }
}

const runWebDenyTest = async () => {
  const test = tests.value[4]
  test.status = 'running'
  test.message = 'Waiting for permission prompt...'

  try {
    const response = await client.web.fetchAsync('https://jsonplaceholder.typicode.com/todos/1')
    test.status = 'error'
    test.message = `Access was granted but expected denial. Status: ${response.status}`
  } catch (e) {
    const errorMsg = e instanceof Error ? e.message : String(e)
    if (errorMsg.includes('Permission denied') || errorMsg.includes('denied')) {
      test.status = 'success'
      test.message = `Correctly denied: ${errorMsg}`
    } else {
      test.status = 'error'
      test.message = `Unexpected error: ${errorMsg}`
    }
  }
}

const runWebAllowOnceTest = async () => {
  const test = tests.value[5]
  test.status = 'running'
  test.message = 'Waiting for permission prompt...'

  try {
    const response = await client.web.fetchAsync('https://dummyjson.com/products/1')
    test.status = 'success'
    test.message = `One-time access granted! Status: ${response.status}`
  } catch (e) {
    const errorMsg = e instanceof Error ? e.message : String(e)
    if (errorMsg.includes('Permission denied') || errorMsg.includes('denied')) {
      test.status = 'error'
      test.message = `Access denied: ${errorMsg}`
    } else {
      test.status = 'error'
      test.message = `Unexpected error: ${errorMsg}`
    }
  }
}

const testRunners = [
  runDbAllowTest,
  runDbDenyTest,
  runDbAllowOnceTest,
  runWebAllowTest,
  runWebDenyTest,
  runWebAllowOnceTest,
]

const runTest = async (index: number) => {
  await testRunners[index]()
}

const resetTests = () => {
  for (const test of tests.value) {
    test.status = 'pending'
    test.message = undefined
  }
}

const getStatusBadge = (test: TestResult) => {
  if (test.status === 'pending') return 'badge-ghost'
  if (test.status === 'running') return 'badge-info'
  if (test.status === 'success') return 'badge-success'
  return 'badge-error'
}

const getStatusText = (test: TestResult) => {
  if (test.status === 'pending') return 'Pending'
  if (test.status === 'running') return 'Running...'
  if (test.status === 'success') return 'Pass'
  return 'Fail'
}
</script>

<template>
  <div>
    <div class="flex justify-between items-center mb-4">
      <h2 class="text-xl font-semibold">Permission Prompt Tests</h2>
      <button class="btn btn-secondary" @click="resetTests">
        Reset All
      </button>
    </div>

    <div class="mb-4 p-3 bg-base-200 rounded-lg">
      <p class="text-sm mb-2">
        <strong>About:</strong> These tests verify the runtime permission prompt system.
      </p>
      <p class="text-sm text-base-content/70">
        Each test will trigger a permission dialog. Follow the instructions to test different responses.
        Run tests <strong>one at a time</strong> and respond to the permission dialog as instructed.
      </p>
    </div>

    <div class="alert alert-info mb-4">
      <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" class="stroke-current shrink-0 w-6 h-6">
        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"></path>
      </svg>
      <div>
        <p class="font-semibold">Interactive Tests</p>
        <p class="text-sm">Click "Run" on each test and respond to the permission prompt as instructed.</p>
      </div>
    </div>

    <div class="space-y-3">
      <div
        v-for="(test, index) in tests"
        :key="index"
        class="card bg-base-200"
      >
        <div class="card-body p-4">
          <div class="flex justify-between items-start">
            <div class="flex-1">
              <h3 class="font-medium">
                {{ test.name }}
                <span class="badge badge-primary badge-sm ml-2">
                  Interactive
                </span>
              </h3>
              <p class="text-sm text-base-content/70">{{ test.description }}</p>
              <p v-if="test.instructions" class="text-sm text-warning mt-1">
                {{ test.instructions }}
              </p>
              <p v-if="test.message" class="text-sm mt-2 font-mono">
                {{ test.message }}
              </p>
            </div>
            <div class="flex items-center gap-2">
              <span class="badge" :class="getStatusBadge(test)">
                {{ getStatusText(test) }}
              </span>
              <button
                class="btn btn-sm btn-primary"
                :disabled="test.status === 'running'"
                @click="runTest(index)"
              >
                Run
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="divider"></div>

    <div class="prose prose-sm">
      <h3>How Permission Prompts Work</h3>
      <ul>
        <li><strong>Allow:</strong> Grants permanent access (status = granted)</li>
        <li><strong>Deny:</strong> Permanently blocks access (status = denied)</li>
        <li><strong>Allow Once:</strong> Allows this request only, prompts again next time (status = ask)</li>
      </ul>
      <p class="text-base-content/70">
        After running tests, check the extension's permissions in Vault settings to see how decisions were saved.
      </p>
    </div>
  </div>
</template>
