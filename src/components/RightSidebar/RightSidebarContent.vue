<!--
  - SPDX-FileCopyrightText: 2025 Nextcloud GmbH and Nextcloud contributors
  - SPDX-License-Identifier: AGPL-3.0-or-later
-->

<script setup lang="ts">
import { computed, ref } from 'vue'

import IconArrowLeft from 'vue-material-design-icons/ArrowLeft.vue'
import IconMagnify from 'vue-material-design-icons/Magnify.vue'

import { t } from '@nextcloud/l10n'

import NcButton from '@nextcloud/vue/components/NcButton'

import InternalSignalingHint from './InternalSignalingHint.vue'
import LobbyStatus from './LobbyStatus.vue'

import { useIsInCall } from '../../composables/useIsInCall.js'
import { useStore } from '../../composables/useStore.js'
import { CONVERSATION, PARTICIPANT, WEBINAR } from '../../constants.ts'
import type { Conversation } from '../../types/index.ts'

const props = defineProps<{
	isUser: boolean,
	conversation: Conversation,
	state: 'default' | 'search',
	mode: 'compact' | 'preview' | 'full',
}>()

const emit = defineEmits<{
	(event: 'update:search', value: boolean): void
}>()

const store = useStore()
const isInCall = useIsInCall()

const isOneToOneConversation = computed(() => [CONVERSATION.TYPE.ONE_TO_ONE, CONVERSATION.TYPE.ONE_TO_ONE_FORMER].includes(props.conversation.type))
const isGroupConversation = computed(() => [CONVERSATION.TYPE.GROUP, CONVERSATION.TYPE.PUBLIC].includes(props.conversation.type))
const hasLobbyEnabled = computed(() => props.conversation.lobbyState === WEBINAR.LOBBY.NON_MODERATORS)
const canFullModerate = computed(() => [PARTICIPANT.TYPE.OWNER, PARTICIPANT.TYPE.MODERATOR].includes(props.conversation.participantType))

const sidebarTitle = computed(() => {
	if (props.state === 'search') {
		return t('spreed', 'Search in {name}', { name: props.conversation.displayName }, {
			escape: false,
			sanitize: false,
		})
	}
	return props.conversation.displayName
})
</script>

<template>
	<div class="content">
		<template v-if="state === 'default'">
			<!-- search in messages button-->
			<NcButton v-if="isUser"
				type="tertiary"
				class="content__actions"
				:title="t('spreed', 'Search messages')"
				:aria-label="t('spreed', 'Search messages')"
				@click="emit('update:search', true)">
				<template #icon>
					<IconMagnify :size="20" />
				</template>
			</NcButton>

			<div class="content__header">
				<h2 :aria-label="sidebarTitle"
					:title="sidebarTitle"
					class="content__name">
					{{ sidebarTitle }}
				</h2>
			</div>

			<div class="content__description">
				<InternalSignalingHint />
				<LobbyStatus v-if="isUser && canFullModerate && hasLobbyEnabled" :token="token" />
			</div>
		</template>
		<template v-else-if="isUser && state === 'search'">
			<div class="content__header">
				<NcButton type="tertiary"
					:title="t('spreed', 'Back')"
					:aria-label="t('spreed', 'Back')"
					@click="emit('update:search', false)">
					<template #icon>
						<IconArrowLeft class="bidirectional-icon" :size="20" />
					</template>
				</NcButton>

				<h2 :aria-label="sidebarTitle"
					:title="sidebarTitle"
					class="content__name">
					{{ sidebarTitle }}
				</h2>
			</div>
		</template>
	</div>
</template>

<style lang="scss" scoped>
.content {
	&__header {
		display: flex;
		gap: var(--default-grid-baseline);
		padding-block: calc(2 * var(--default-grid-baseline)) var(--default-grid-baseline);
		padding-inline-start: var(--default-grid-baseline);
		padding-inline-end: calc(2 * var(--default-grid-baseline) + var(--default-clickable-area) + var(--app-sidebar-close-button-offset));
	}

	&__name {
		margin: 0;
		padding: 0;
		font-size: 20px;
		line-height: var(--default-line-height);
		overflow: hidden;
		white-space: nowrap;
		text-overflow: ellipsis;
	}

	&__actions {
		position: absolute !important;
		z-index: 1;
		top: calc(2 * var(--default-grid-baseline));
		inset-inline-end: calc(var(--default-grid-baseline) + var(--app-sidebar-close-button-offset));
	}

	&__description {
		display: flex;
		flex-direction: column;
		align-items: center;
		margin: 0 10px;
	}
}
</style>
