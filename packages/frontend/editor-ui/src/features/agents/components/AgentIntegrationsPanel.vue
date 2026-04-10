<script setup lang="ts">
import { ref, onMounted } from 'vue';
import { N8nButton, N8nCard, N8nIcon, N8nText } from '@n8n/design-system';
import N8nSelect from '@n8n/design-system/components/N8nSelect';
import N8nOption from '@n8n/design-system/components/N8nOption';
import { useRootStore } from '@n8n/stores/useRootStore';
import { makeRestApiRequest } from '@n8n/rest-api-client';
import {
	connectIntegration,
	disconnectIntegration,
	getIntegrationStatus,
} from '../composables/useAgentApi';

const props = defineProps<{
	projectId: string;
	agentId: string;
}>();

const rootStore = useRootStore();

interface CredentialOption {
	id: string;
	name: string;
}

interface IntegrationConfig {
	type: string;
	label: string;
	icon: string;
	description: string;
	connectedDescription: string;
	credentialTypes: string[];
	noCredentialsMessage: string;
}

const integrationConfigs: IntegrationConfig[] = [
	{
		type: 'slack',
		label: 'Slack',
		icon: 'hashtag',
		description:
			'Connect a Slack bot credential to allow this agent to receive and respond to Slack messages.',
		connectedDescription: 'Your agent is connected to Slack and can receive messages.',
		credentialTypes: ['slackApi', 'slackOAuth2Api'],
		noCredentialsMessage:
			'No Slack API credentials found. Create a Slack API or Slack OAuth2 API credential in the Credentials page.',
	},
	{
		type: 'telegram',
		label: 'Telegram',
		icon: 'paper-plane',
		description:
			'Connect a Telegram bot credential to allow this agent to receive and respond to Telegram messages.',
		connectedDescription: 'Your agent is connected to Telegram and can receive messages.',
		credentialTypes: ['telegramApi'],
		noCredentialsMessage:
			'No Telegram API credentials found. Create a Telegram API credential in the Credentials page.',
	},
];

// Per-integration state
const statuses = ref<Record<string, string>>({});
const connectedCredentials = ref<Record<string, string>>({});
const selectedCredentials = ref<Record<string, string>>({});
const credentialsByType = ref<Record<string, CredentialOption[]>>({});
const loadingMap = ref<Record<string, boolean>>({});
const credentialsLoading = ref(false);

function isConnected(type: string): boolean {
	return statuses.value[type] === 'connected';
}

function isLoading(type: string): boolean {
	return loadingMap.value[type] ?? false;
}

async function fetchStatus() {
	try {
		const result = await getIntegrationStatus(
			rootStore.restApiContext,
			props.projectId,
			props.agentId,
		);
		for (const config of integrationConfigs) {
			statuses.value[config.type] = 'disconnected';
			connectedCredentials.value[config.type] = '';
		}
		for (const integration of result.integrations ?? []) {
			statuses.value[integration.type] = 'connected';
			connectedCredentials.value[integration.type] = integration.credentialId;
		}
	} catch {
		for (const config of integrationConfigs) {
			statuses.value[config.type] = 'disconnected';
			connectedCredentials.value[config.type] = '';
		}
	}
}

async function fetchCredentials() {
	credentialsLoading.value = true;
	try {
		const allCredentials = await makeRestApiRequest<
			Array<{ id: string; name: string; type: string }>
		>(rootStore.restApiContext, 'GET', '/credentials');

		for (const config of integrationConfigs) {
			credentialsByType.value[config.type] = allCredentials
				.filter((c) => config.credentialTypes.includes(c.type))
				.map((c) => ({ id: c.id, name: c.name }));
		}
	} catch {
		for (const config of integrationConfigs) {
			credentialsByType.value[config.type] = [];
		}
	} finally {
		credentialsLoading.value = false;
	}
}

async function onConnect(type: string) {
	const credId = selectedCredentials.value[type];
	if (!credId) return;
	loadingMap.value[type] = true;
	try {
		await connectIntegration(
			rootStore.restApiContext,
			props.projectId,
			props.agentId,
			type,
			credId,
		);
		await fetchStatus();
	} finally {
		loadingMap.value[type] = false;
	}
}

async function onDisconnect(type: string) {
	const credId = connectedCredentials.value[type] || selectedCredentials.value[type];
	if (!credId) return;
	loadingMap.value[type] = true;
	try {
		await disconnectIntegration(
			rootStore.restApiContext,
			props.projectId,
			props.agentId,
			type,
			credId,
		);
		await fetchStatus();
		selectedCredentials.value[type] = '';
	} finally {
		loadingMap.value[type] = false;
	}
}

onMounted(async () => {
	await Promise.all([fetchStatus(), fetchCredentials()]);
});
</script>

<template>
	<div :class="$style.panel">
		<N8nText :class="$style.heading" tag="h3" bold>Integrations</N8nText>

		<N8nCard v-for="config in integrationConfigs" :key="config.type" :class="$style.card">
			<template #header>
				<div :class="$style.cardHeader">
					<div :class="$style.statusRow">
						<span
							:class="[
								$style.statusDot,
								isConnected(config.type) ? $style.statusConnected : $style.statusDisconnected,
							]"
						/>
						<N8nText bold>{{ config.label }}</N8nText>
						<N8nText :class="$style.statusLabel" size="small">
							{{ isConnected(config.type) ? 'Connected' : 'Disconnected' }}
						</N8nText>
					</div>
				</div>
			</template>

			<div :class="$style.cardBody">
				<N8nText :class="$style.description" size="small">
					{{ config.description }}
				</N8nText>

				<div v-if="!isConnected(config.type)" :class="$style.connectForm">
					<label :class="$style.label">
						<N8nText size="small" bold>{{ config.label }} Credential</N8nText>
					</label>
					<N8nSelect
						v-model="selectedCredentials[config.type]"
						:class="$style.select"
						placeholder="Select a credential..."
						:loading="credentialsLoading"
						:disabled="isLoading(config.type)"
						size="medium"
						:data-testid="`${config.type}-credential-select`"
					>
						<N8nOption
							v-for="cred in credentialsByType[config.type] ?? []"
							:key="cred.id"
							:value="cred.id"
							:label="cred.name"
						/>
					</N8nSelect>
					<N8nText
						v-if="(credentialsByType[config.type] ?? []).length === 0 && !credentialsLoading"
						size="small"
					>
						{{ config.noCredentialsMessage }}
					</N8nText>
					<N8nButton
						:class="$style.actionButton"
						:disabled="!selectedCredentials[config.type] || isLoading(config.type)"
						:loading="isLoading(config.type)"
						size="small"
						:data-testid="`${config.type}-connect-button`"
						@click="onConnect(config.type)"
					>
						<N8nIcon icon="plug" :size="14" />
						Connect
					</N8nButton>
				</div>

				<div v-else :class="$style.disconnectSection">
					<N8nText size="small">
						{{ config.connectedDescription }}
					</N8nText>
					<N8nButton
						:class="$style.actionButton"
						type="tertiary"
						:loading="isLoading(config.type)"
						size="small"
						:data-testid="`${config.type}-disconnect-button`"
						@click="onDisconnect(config.type)"
					>
						<N8nIcon icon="unlink" :size="14" />
						Disconnect
					</N8nButton>
				</div>
			</div>
		</N8nCard>
	</div>
</template>

<style module>
.panel {
	padding: var(--spacing--lg);
	overflow-y: auto;
	height: 100%;
	display: flex;
	flex-direction: column;
	gap: var(--spacing--sm);
}

.heading {
	margin-bottom: 0;
}

.card {
	max-width: 480px;
}

.cardHeader {
	display: flex;
	align-items: center;
	justify-content: space-between;
}

.statusRow {
	display: flex;
	align-items: center;
	gap: var(--spacing--2xs);
}

.statusDot {
	width: 8px;
	height: 8px;
	border-radius: 50%;
	flex-shrink: 0;
}

.statusConnected {
	background-color: var(--color--success);
}

.statusDisconnected {
	background-color: var(--color--foreground--shade-1);
}

.statusLabel {
	color: var(--color--text--tint-2);
}

.cardBody {
	display: flex;
	flex-direction: column;
	gap: var(--spacing--sm);
}

.description {
	color: var(--color--text--tint-1);
}

.connectForm {
	display: flex;
	flex-direction: column;
	gap: var(--spacing--xs);
}

.label {
	display: block;
}

.select {
	width: 100%;
}

.disconnectSection {
	display: flex;
	flex-direction: column;
	gap: var(--spacing--xs);
}

.actionButton {
	align-self: flex-start;
}
</style>
