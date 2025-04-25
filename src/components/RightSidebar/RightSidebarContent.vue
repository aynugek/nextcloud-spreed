<!--
  - SPDX-FileCopyrightText: 2025 Nextcloud GmbH and Nextcloud contributors
  - SPDX-License-Identifier: AGPL-3.0-or-later
-->

<script setup lang="ts">
import { computed, ref, watch } from 'vue'

import IconArrowLeft from 'vue-material-design-icons/ArrowLeft.vue'
import IconMagnify from 'vue-material-design-icons/Magnify.vue'

import { t } from '@nextcloud/l10n'
import moment from '@nextcloud/moment'

import NcActions from '@nextcloud/vue/components/NcActions'
import NcActionButton from '@nextcloud/vue/components/NcActionButton'
import NcAppSidebarHeader from '@nextcloud/vue/components/NcAppSidebarHeader'
import NcButton from '@nextcloud/vue/components/NcButton'
import { useIsDarkTheme } from '@nextcloud/vue/composables/useIsDarkTheme'

import InternalSignalingHint from './InternalSignalingHint.vue'
import LobbyStatus from './LobbyStatus.vue'
import TransitionWrapper from '../UIShared/TransitionWrapper.vue'

import { useIsInCall } from '../../composables/useIsInCall.js'
import { useStore } from '../../composables/useStore.js'
import { CONVERSATION, PARTICIPANT, WEBINAR } from '../../constants.ts'
import { getConversationAvatarOcsUrl } from '../../services/avatarService.ts'
import { hasTalkFeature } from '../../services/CapabilitiesManager.ts'
import { getUserProfile } from '../../services/coreService.ts'
import type {
	Conversation,
	UserProfileData,
} from '../../types/index.ts'

const supportsAvatar = hasTalkFeature('local', 'avatar')

const props = defineProps<{
	isUser: boolean,
	conversation: Conversation,
	state: 'default' | 'search',
	mode: 'compact' | 'preview' | 'full',
}>()

const emit = defineEmits<{
	(event: 'update:search', value: boolean): void
}>()

// FIXME cache in store until reload
let isGetProfileAllowed = true

const store = useStore()
const isInCall = useIsInCall()
const isDarkTheme = useIsDarkTheme()

const profileLoading = ref(false)

// FIXME cache in store to make less requests
const profileData = ref<UserProfileData | null>(null)
const profileActions = computed<UserProfileData['actions']>(() => {
	if (!profileData.value) {
		return []
	}
	return profileData.value.actions.filter(action => action.id !== 'talk')
})

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

const avatarUrl = computed(() => {
	if (!supportsAvatar || props.conversation.isDummyConversation) {
		return undefined
	}

	return getConversationAvatarOcsUrl(props.conversation.token, isDarkTheme.value, props.conversation.avatarVersion)
})

const profileInformation = computed(() => {
	if (!profileData.value) {
		if (isGroupConversation.value) {
			return [props.conversation.description]
		}
		return []
	}

	const fields = []

	if (profileData.value.role || profileData.value.pronouns) {
		fields.push(joinFields(profileData.value.role, profileData.value.pronouns))
	}
	if (profileData.value.organisation) {
		fields.push(profileData.value.organisation)
	}

	const currentTime = moment(new Date().setSeconds(new Date().getTimezoneOffset() * 60 + profileData.value.timezoneOffset))
	fields.push(joinFields(profileData.value.address, currentTime.format('LT')))

	return fields
})


watch(() => props.conversation.token, async () => {
	if (!isGetProfileAllowed) {
		return
	}

	if (isOneToOneConversation.value) {
		try {
			profileLoading.value = true
			const response = await getUserProfile(props.conversation.name)
			console.log(response.data.ocs.data)
			profileData.value = response.data.ocs.data
		} catch (error) {
			if (error?.response?.status === 405) {
				// Method does not exist on current server version
				// Skip further requests
				isGetProfileAllowed = false
			} else {
				console.error(error)
			}
		} finally {
			profileLoading.value = false
		}
	} else {
		profileData.value = null
	}
}, { immediate: true })

function joinFields (fieldA: string | null, fieldB: string | null): string {
	return [fieldA, fieldB].filter(Boolean).join(' · ')
}
</script>

<template>
	<div :class="`content content--${mode}`">
		<template v-if="state === 'default'">
			<!-- search in messages button-->
			<div class="content__actions">
				<NcActions v-if="profileActions.length" force-menu>
					<NcActionButton v-for="action in profileActions"
						:key="action.id"
						:href="action.target">
						{{ action.title }}
					</NcActionButton>
				</NcActions>
				<NcButton v-if="isUser"
					type="tertiary"
					:title="t('spreed', 'Search messages')"
					:aria-label="t('spreed', 'Search messages')"
					@click="emit('update:search', true)">
					<template #icon>
						<IconMagnify :size="20" />
					</template>
				</NcButton>
			</div>

			<div class="content__scroller animated">
				<!-- User / conversation avatar image -->
				<div class="content__image-wrapper animated">
					<img class="content__image animated" :src="avatarUrl" :alt="conversation.displayName">
				</div>
				<!-- User / conversation profile information -->
				<div class="content__header">
					<NcAppSidebarHeader :name="sidebarTitle"
						:title="sidebarTitle"
						class="content__name"
						:class="{ 'content__name--has-actions': profileActions.length }">
						{{ sidebarTitle }}
					</NcAppSidebarHeader>
					<TransitionWrapper name="fade">
						<div v-if="mode !== 'compact'" class="content__info">
							<p v-for="row in profileInformation" class="content__info-row">
								{{ row }}
							</p>
						</div>
					</TransitionWrapper>
				</div>
			</div>

			<div class="content__description">
				<InternalSignalingHint />
				<LobbyStatus v-if="isUser && canFullModerate && hasLobbyEnabled" :token="token" />
			</div>
		</template>
		<template v-else-if="isUser && state === 'search'">
			<div class="content__header content__header--row">
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
	&--compact {
		// default
		.content__scroller {
		}
		.content__image-wrapper {
			width: 0;
			height: 0;
			padding: 0;
		}
	}

	&--preview {
		// avatar on the left
		.content__scroller {
			flex-wrap: wrap;
		}
		.content__header {
			width: 75%;
		}
		.content__image-wrapper {
			width: 25%;
			height: 25%;
			padding: var(--default-grid-baseline);
		}
	}

	&--full {
		// avatar in full size
		.content__scroller {
			flex-direction: column;
			align-items: start;
		}
		.content__header {
			width: 100%;
			padding-inline: calc(2 * var(--default-grid-baseline));
		}
		.content__image-wrapper {
			width: 100%;
			height: 300px;
			padding: 0;
			&::after {
				opacity: 1 !important;
			}
			.content__image {
				border-radius: 0;
			}
		}
		// Overwrite NcButton styles
		& :deep(.button-vue--icon-only),
		& ~ :deep(.button-vue--icon-only) {
			filter: invert(1);
		}
	}

	&__scroller {
		display: flex;
	}

	.content__image-wrapper {
		position: relative;
		flex-shrink: 0;

		&::after {
			position: absolute;
			inset: 0;
			content: '';
			z-index: 1;
			width: 100%;
			height: 100%;
			background: linear-gradient(180deg, rgba(0, 0, 0, 0.9) 0%, rgba(0, 0, 0, 0) 30%);
			opacity: 0;
			transition: opacity ease-in-out var(--animation-slow);
		}
	}

	.content__image {
		max-width: 100%;
		max-height: 100%;
		width: 100%;
		height: 100%;
		border-radius: 50%;
		object-fit: cover;
		object-position: top;
	}

	&__header {
		flex-grow: 1;
		display: flex;
		flex-direction: column;
		align-items: start;
		gap: var(--default-grid-baseline);
		padding-block: calc(2 * var(--default-grid-baseline)) var(--default-grid-baseline);
		padding-inline-start: var(--default-grid-baseline);

		&--row {
			flex-direction: row;
			align-items: center;
		}

		.content__name {
			width: 100%;
			margin: 0;
			padding-inline-end: calc(var(--default-grid-baseline) + var(--default-clickable-area) + var(--app-sidebar-close-button-offset));
			font-size: 20px;
			line-height: var(--default-clickable-area);
			overflow: hidden;
			white-space: nowrap;
			text-overflow: ellipsis;
			&--has-actions {
				padding-inline-end: calc(2 * (var(--default-grid-baseline) + var(--default-clickable-area)) + var(--app-sidebar-close-button-offset));
			}
		}

		.content__info {
		}
	}

	&__actions {
		position: absolute !important;
		z-index: 2;
		top: calc(2 * var(--default-grid-baseline));
		inset-inline-end: calc(var(--default-grid-baseline) + var(--app-sidebar-close-button-offset));
		display: flex;
		gap: 4px;
	}

	&__description {
		display: flex;
		flex-direction: column;
		align-items: center;
		margin: 0 10px;
	}
}

.animated {
	transition-property: padding, width, height, border-radius;
	transition-duration: var(--animation-slow);
	transition-timing-function: ease-in-out;
}
</style>
