<script lang="ts" setup>
import type { PropType } from 'vue';
import type { ExternalSecretsProvider } from '@/Interface';
import { useExternalSecretsStore } from '@/stores';
import { useI18n, useLoadingService, useToast } from '@/composables';
import { computed, onMounted, ref } from 'vue';
import type { EventBus } from 'n8n-design-system/utils';

const emit = defineEmits<{
	(e: 'change', value: boolean): void;
}>();

const props = defineProps({
	provider: {
		type: Object as PropType<ExternalSecretsProvider>,
		required: true,
	},
	eventBus: {
		type: Object as PropType<EventBus>,
		default: undefined,
	},
	disabled: {
		type: Boolean,
		default: false,
	},
	beforeUpdate: {
		type: Function,
		default: undefined,
	},
});

const loadingService = useLoadingService();
const externalSecretsStore = useExternalSecretsStore();
const i18n = useI18n();
const toast = useToast();

const saving = ref(false);

const connectedTextColor = computed(() => {
	return props.provider.connected ? 'success' : 'text-light';
});

onMounted(() => {
	if (props.eventBus) {
		props.eventBus.on('connect', onUpdateConnected);
	}
});

async function onUpdateConnected(value: boolean) {
	try {
		saving.value = true;

		if (props.beforeUpdate) {
			const result = await props.beforeUpdate(value);
			if (result === false) {
				saving.value = false;
				return;
			}
		}

		await externalSecretsStore.updateProviderConnected(props.provider.name, value);

		emit('change', value);
	} catch (error) {
		toast.showError(error, 'Error');
	} finally {
		saving.value = false;
	}
}
</script>

<template>
	<div class="connection-switch" v-loading="saving" element-loading-spinner="el-icon-loading">
		<n8n-icon
			v-if="provider.state === 'error'"
			color="danger"
			icon="exclamation-triangle"
			class="mr-2xs"
		/>
		<n8n-text :color="connectedTextColor" bold class="mr-2xs">
			{{
				i18n.baseText(
					`settings.externalSecrets.card.${provider.connected ? 'connected' : 'disconnected'}`,
				)
			}}
		</n8n-text>
		<el-switch
			:modelValue="provider.connected"
			:title="
				i18n.baseText('settings.externalSecrets.card.connectedSwitch.title', {
					interpolate: { provider: provider.displayName },
				})
			"
			:disabled="disabled"
			data-test-id="settings-external-secrets-connected-switch"
			@update:modelValue="onUpdateConnected"
		>
		</el-switch>
	</div>
</template>

<style lang="scss" scoped>
.connection-switch {
	position: relative;
	display: flex;
	flex-direction: row;
	align-items: center;

	&.error {
		:deep(.el-switch.is-checked .el-switch__core) {
			background-color: #ff4027;
			border-color: #ff4027;
		}
	}
}
</style>