<template>
	<div class="_gaps_m">
		<MkPagination :pagination="noteNotificationPagination">
			<template #empty>
				<div class="_fullinfo">
					<img src="/static-assets/aira/info.png" class="_ghost"/>
					<div>{{ i18n.ts.noUsers }}</div>
				</div>
			</template>

			<template #default="{ items }">
				<div class="_gaps_s">
					<div v-for="item in items" :key="item.id" :class="[$style.userItem, { [$style.userItemOpend]: expandedNoteNotificationItems.includes(item.id) }]">
						<div :class="$style.userItemMain">
							<MkA :class="$style.userItemMainBody" :to="userPage(item.user)">
								<MkUserCardMini :user="item.user"/>
							</MkA>
							<button class="_button" :class="$style.userToggle" @click="toggleNoteNotificationItem(item)"><i :class="$style.chevron" class="ti ti-chevron-down"></i></button>
							<button class="_button" :class="$style.remove" @click="noteUnsubscribe(item.userId, $event)"><i class="ti ti-x"></i></button>
						</div>
						<div v-if="expandedNoteNotificationItems.includes(item.id)" :class="$style.userItemSub">
							<div>
								Created at:
								<MkTime :time="item.createdAt" mode="detail"/>
							</div>
						</div>
					</div>
				</div>
			</template>
		</MkPagination>
	</div>
</template>

<script lang="ts" setup>
import { ref, computed } from 'vue';
import MkPagination from '@/components/MkPagination.vue';
import { i18n } from '@/i18n.js';
import { definePageMetadata } from '@/scripts/page-metadata.js';
import MkUserCardMini from '@/components/MkUserCardMini.vue';
import { userPage } from '@/filters/user.js';
import * as os from '@/os.js';

const noteNotificationPagination = {
	endpoint: 'note-notification/list' as const,
	limit: 10,
};

const expandedNoteNotificationItems = ref([]);

async function noteUnsubscribe(userId, ev) {
	os.popupMenu([{
		text: i18n.ts.noteUnsubscribe,
		icon: 'ti ti-x',
		action: async () => {
			await os.apiWithDialog('note-notification/delete', { userId });
		},
	}], ev.currentTarget ?? ev.target);
}

async function toggleNoteNotificationItem(item) {
	if (expandedNoteNotificationItems.value.includes(item.id)) {
		expandedNoteNotificationItems.value = expandedNoteNotificationItems.value.filter(x => x !== item.id);
	} else {
		expandedNoteNotificationItems.value.push(item.id);
	}
}

const headerActions = computed(() => []);

const headerTabs = computed(() => []);

definePageMetadata({
	title: i18n.ts.notifications,
	icon: 'ti ti-bell',
});
</script>

<style lang="scss" module>
.userItemMain {
	display: flex;
}

.userItemSub {
	padding: 6px 12px;
	font-size: 85%;
	color: var(--fgTransparentWeak);
}

.userItemMainBody {
	flex: 1;
	min-width: 0;
	margin-right: 8px;

	&:hover {
		text-decoration: none;
	}
}

.userToggle,
.remove {
	width: 32px;
	height: 32px;
	align-self: center;
}

.chevron {
	display: block;
	transition: transform 0.1s ease-out;
}

.userItem.userItemOpend {
	.chevron {
		transform: rotateX(180deg);
	}
}</style>
