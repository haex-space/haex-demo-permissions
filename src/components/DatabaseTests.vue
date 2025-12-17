<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { HaexVaultSdk } from '@haex-space/vault-sdk'

interface TestResult {
  name: string
  description: string
  expected: 'success' | 'error'
  status: 'pending' | 'running' | 'success' | 'error'
  message?: string
}

const client = new HaexVaultSdk()
const extensionPrefix = ref<string>('')
const tests = ref<TestResult[]>([])

// Get extension prefix on mount
onMounted(async () => {
  try {
    await client.ready()
    const info = client.extensionInfo
    if (info) {
      // Format: {publicKey}__{extensionName}__
      extensionPrefix.value = `${info.publicKey}__${info.name}__`
    }
  } catch (e) {
    console.error('Failed to get extension context:', e)
    extensionPrefix.value = 'unknown__unknown__'
  }
  initTests()
})

const initTests = () => {
  tests.value = [
    // Tests that SHOULD succeed
    {
      name: 'Create own table',
      description: `CREATE TABLE ${extensionPrefix.value}test_users`,
      expected: 'success',
      status: 'pending',
    },
    {
      name: 'Insert into own table',
      description: `INSERT into own table`,
      expected: 'success',
      status: 'pending',
    },
    {
      name: 'Select from own table',
      description: `SELECT from own table`,
      expected: 'success',
      status: 'pending',
    },
    {
      name: 'Update own table',
      description: `UPDATE own table`,
      expected: 'success',
      status: 'pending',
    },
    {
      name: 'Delete from own table',
      description: `DELETE from own table`,
      expected: 'success',
      status: 'pending',
    },
    {
      name: 'Drop own table',
      description: `DROP TABLE own table`,
      expected: 'success',
      status: 'pending',
    },
    // Tests that SHOULD fail
    {
      name: 'Create table without prefix',
      description: `CREATE TABLE users_without_prefix (should fail)`,
      expected: 'error',
      status: 'pending',
    },
    {
      name: 'Create system table',
      description: `CREATE TABLE haex_malicious (should fail)`,
      expected: 'error',
      status: 'pending',
    },
    {
      name: 'Select from system table',
      description: `SELECT from haex_extensions (should fail)`,
      expected: 'error',
      status: 'pending',
    },
    {
      name: 'Drop system table',
      description: `DROP TABLE haex_extensions (should fail)`,
      expected: 'error',
      status: 'pending',
    },
  ]
}

const runAllTests = async () => {
  for (const test of tests.value) {
    test.status = 'pending'
    test.message = undefined
  }

  // Test 1: Create own table
  await runTest(0, async () => {
    const tableName = `${extensionPrefix.value}test_users`
    await client.execute(`
      CREATE TABLE IF NOT EXISTS "${tableName}" (
        id TEXT PRIMARY KEY,
        name TEXT NOT NULL,
        email TEXT,
        haex_tombstone INTEGER DEFAULT 0,
        haex_timestamp TEXT,
        haex_column_hlcs TEXT
      )
    `)
    return 'Table created successfully'
  })

  // Test 2: Insert into own table
  await runTest(1, async () => {
    const tableName = `${extensionPrefix.value}test_users`
    await client.execute(
      `INSERT INTO "${tableName}" (id, name, email) VALUES (?, ?, ?)`,
      ['user-1', 'Test User', 'test@example.com']
    )
    return 'Row inserted successfully'
  })

  // Test 3: Select from own table
  await runTest(2, async () => {
    const tableName = `${extensionPrefix.value}test_users`
    const rows = await client.query(`SELECT * FROM "${tableName}"`)
    return `Selected ${rows.length} rows`
  })

  // Test 4: Update own table
  await runTest(3, async () => {
    const tableName = `${extensionPrefix.value}test_users`
    await client.execute(
      `UPDATE "${tableName}" SET name = ? WHERE id = ?`,
      ['Updated User', 'user-1']
    )
    return 'Row updated successfully'
  })

  // Test 5: Delete from own table
  await runTest(4, async () => {
    const tableName = `${extensionPrefix.value}test_users`
    await client.execute(
      `DELETE FROM "${tableName}" WHERE id = ?`,
      ['user-1']
    )
    return 'Row deleted successfully'
  })

  // Test 6: Drop own table
  await runTest(5, async () => {
    const tableName = `${extensionPrefix.value}test_users`
    await client.execute(`DROP TABLE IF EXISTS "${tableName}"`)
    return 'Table dropped successfully'
  })

  // Test 7: Create table without prefix (should fail)
  await runTest(6, async () => {
    await client.execute(`
      CREATE TABLE "users_without_prefix" (
        id TEXT PRIMARY KEY,
        haex_tombstone INTEGER DEFAULT 0
      )
    `)
    return 'Table created (unexpected!)'
  })

  // Test 8: Create system table (should fail)
  await runTest(7, async () => {
    await client.execute(`
      CREATE TABLE "haex_malicious" (
        id TEXT PRIMARY KEY,
        haex_tombstone INTEGER DEFAULT 0
      )
    `)
    return 'Table created (unexpected!)'
  })

  // Test 9: Select from system table (should fail)
  await runTest(8, async () => {
    await client.query(`SELECT * FROM haex_extensions`)
    return 'Selected from system table (unexpected!)'
  })

  // Test 10: Drop system table (should fail)
  await runTest(9, async () => {
    await client.execute(`DROP TABLE haex_extensions`)
    return 'Dropped system table (unexpected!)'
  })
}

const runTest = async (index: number, testFn: () => Promise<string>) => {
  const test = tests.value[index]
  test.status = 'running'

  try {
    const result = await testFn()
    if (test.expected === 'success') {
      test.status = 'success'
      test.message = result
    } else {
      test.status = 'error'
      test.message = `Expected error but got success: ${result}`
    }
  } catch (e) {
    const errorMsg = e instanceof Error ? e.message : String(e)
    if (test.expected === 'error') {
      test.status = 'success'
      test.message = `Correctly blocked: ${errorMsg}`
    } else {
      test.status = 'error'
      test.message = `Unexpected error: ${errorMsg}`
    }
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
      <h2 class="text-xl font-semibold">Database Permission Tests</h2>
      <button class="btn btn-primary" @click="runAllTests">
        Run All Tests
      </button>
    </div>

    <div class="mb-4 p-3 bg-base-200 rounded-lg">
      <p class="text-sm">
        <strong>Extension Prefix:</strong>
        <code class="ml-2">{{ extensionPrefix || 'Loading...' }}</code>
      </p>
    </div>

    <div class="space-y-3">
      <div
        v-for="(test, index) in tests"
        :key="index"
        class="card bg-base-200"
      >
        <div class="card-body p-4">
          <div class="flex justify-between items-start">
            <div>
              <h3 class="font-medium">
                {{ test.name }}
                <span
                  v-if="test.expected === 'error'"
                  class="badge badge-warning badge-sm ml-2"
                >
                  Should Fail
                </span>
              </h3>
              <p class="text-sm text-base-content/70">{{ test.description }}</p>
              <p v-if="test.message" class="text-sm mt-2 font-mono">
                {{ test.message }}
              </p>
            </div>
            <span class="badge" :class="getStatusBadge(test)">
              {{ getStatusText(test) }}
            </span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
