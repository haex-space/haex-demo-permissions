<script setup lang="ts">
import { ref } from 'vue'
import { HaexVaultClient } from '@haex-space/vault-sdk'

interface TestResult {
	name: string
	description: string
	expected: 'success' | 'error'
	status: 'pending' | 'running' | 'success' | 'error'
	message?: string
}

const client = new HaexVaultClient()
const tests = ref<TestResult[]>([
	{
		name: 'Fetch allowed URL',
		description: 'GET https://httpbin.org/get (allowed in manifest)',
		expected: 'success',
		status: 'pending',
	},
	{
		name: 'Fetch blocked URL',
		description: 'GET https://example.com (not in manifest)',
		expected: 'error',
		status: 'pending',
	},
	{
		name: 'POST to allowed URL',
		description: 'POST https://httpbin.org/post',
		expected: 'success',
		status: 'pending',
	},
])

const runAllTests = async () => {
	for (const test of tests.value) {
		test.status = 'pending'
		test.message = undefined
	}

	// Test 1: Fetch allowed URL
	await runTest(0, async () => {
		const response = await client.web.fetchAsync('https://httpbin.org/get')
		return `Status: ${response.status}`
	})

	// Test 2: Fetch blocked URL
	await runTest(1, async () => {
		const response = await client.web.fetchAsync('https://example.com')
		return `Status: ${response.status} (unexpected!)`
	})

	// Test 3: POST to allowed URL
	await runTest(2, async () => {
		const response = await client.web.fetchAsync('https://httpbin.org/post', {
			method: 'POST',
			headers: { 'Content-Type': 'application/json' },
			body: JSON.stringify({ test: 'data' }),
		})
		return `Status: ${response.status}`
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
			<h2 class="text-xl font-semibold">HTTP Permission Tests</h2>
			<button class="btn btn-primary" @click="runAllTests">
				Run All Tests
			</button>
		</div>

		<div class="mb-4 p-3 bg-base-200 rounded-lg">
			<p class="text-sm">
				<strong>Allowed URLs:</strong>
				<code class="ml-2">https://httpbin.org/*</code>
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
