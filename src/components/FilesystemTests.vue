<script setup lang="ts">
import { ref } from 'vue'

interface TestResult {
	name: string
	description: string
	expected: 'success' | 'error'
	status: 'pending' | 'running' | 'success' | 'error'
	message?: string
}

const tests = ref<TestResult[]>([
	{
		name: 'Read extension data',
		description: 'Read from extension data directory',
		expected: 'success',
		status: 'pending',
	},
	{
		name: 'Write extension data',
		description: 'Write to extension data directory',
		expected: 'success',
		status: 'pending',
	},
	{
		name: 'Read system files',
		description: 'Read /etc/passwd (should fail)',
		expected: 'error',
		status: 'pending',
	},
])

const runAllTests = async () => {
	for (const test of tests.value) {
		test.status = 'pending'
		test.message = 'Filesystem permissions not yet implemented in SDK'
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
			<h2 class="text-xl font-semibold">Filesystem Permission Tests</h2>
			<button class="btn btn-primary" @click="runAllTests">
				Run All Tests
			</button>
		</div>

		<div class="mb-4 p-3 bg-base-200 rounded-lg">
			<p class="text-sm text-warning">
				Filesystem permissions are not yet implemented in the SDK.
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
