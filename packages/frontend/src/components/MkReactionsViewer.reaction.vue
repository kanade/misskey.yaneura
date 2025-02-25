<!--
SPDX-FileCopyrightText: syuilo and misskey-project
SPDX-License-Identifier: AGPL-3.0-only
-->

<template>
<button
	ref="buttonEl"
	v-ripple="canToggle"
	class="_button"
	:class="[$style.root, { [$style.reacted]: note.myReaction == reaction, [$style.canToggle]: (isLocal && canToggle), [$style.canToggleFallback]: (!isLocal && isAvailable), [$style.small]: defaultStore.state.reactionsDisplaySize === 'small', [$style.large]: defaultStore.state.reactionsDisplaySize === 'large' }]"
	@click="toggleReaction()"
	@contextmenu.prevent.stop="menu"
>
	<MkReactionIcon :class="defaultStore.state.limitWidthOfReaction ? $style.limitWidth : ''" :reaction="reaction" :emojiUrl="note.reactionEmojis[reaction.substring(1, reaction.length - 1)]"/>
	<span v-if="!hideReactionCount" :class="$style.count">{{ count }}</span>
</button>
</template>

<script lang="ts" setup>
import { computed, inject, onMounted, shallowRef, watch } from 'vue';
import * as Misskey from 'misskey-js';
import MkCustomEmojiDetailedDialog from './MkCustomEmojiDetailedDialog.vue';
import XDetails from '@/components/MkReactionsViewer.details.vue';
import MkReactionIcon from '@/components/MkReactionIcon.vue';
import * as os from '@/os.js';
import { misskeyApi, misskeyApiGet } from '@/scripts/misskey-api.js';
import { useTooltip } from '@/scripts/use-tooltip.js';
import { $i } from '@/account.js';
import MkReactionEffect from '@/components/MkReactionEffect.vue';
import { claimAchievement } from '@/scripts/achievements.js';
import { customEmojisMap } from '@/custom-emojis.js';
import { defaultStore } from '@/store.js';
import { i18n } from '@/i18n.js';
import * as sound from '@/scripts/sound.js';
import { checkReactionPermissions } from '@/scripts/check-reaction-permissions.js';
import { getUnicodeEmoji } from '@/scripts/emojilist.js';

const props = defineProps<{
	reaction: string;
	count: number;
	isInitial: boolean;
	note: Misskey.entities.Note;
}>();

const mock = inject<boolean>('mock', false);

const emit = defineEmits<{
	(ev: 'reactionToggled', emoji: string, newCount: number): void;
}>();

const buttonEl = shallowRef<HTMLElement>();

const emojiName = computed(() => props.reaction.replace(/:/g, '').replace(/@\./, ''));
const emoji = computed(() => customEmojisMap.get(emojiName.value) ?? getUnicodeEmoji(props.reaction));

function getReactionName(reaction: string, formated = false) {
	const r = reaction.replaceAll(':', '').replace(/@.*/, '');
	return formated ? `:${r}:` : r;
}

const isLocal = computed(() => !props.reaction.match(/@\w/));
const isAvailable = computed(() => isLocal.value ? true : customEmojisMap.has(getReactionName(props.reaction)));

const canToggle = computed(() => {
	return isAvailable.value && $i && emoji.value && checkReactionPermissions($i, props.note, emoji.value);
});
const canGetInfo = computed(() => !props.reaction.match(/@\w/) && props.reaction.includes(':'));

const plainReaction = computed(() => customEmojisMap.has(emojiName.value) ? getReactionName(props.reaction, true) : props.reaction);

const hideReactionCount = computed(() => {
	switch (defaultStore.state.hideReactionCount) {
		case 'none': return false;
		case 'all': return true;
		case 'self': return props.note.userId === $i?.id;
		case 'others': return props.note.userId !== $i?.id;
		default: return false;
	}
});

async function toggleReaction() {
	if (!canToggle.value) return;

	const reaction = getReactionName(props.reaction, true);
	const oldReaction = props.note.myReaction ? getReactionName(props.note.myReaction, true) : null;
	if (oldReaction) {
		const confirm = await os.confirm({
			type: 'warning',
			text: oldReaction !== reaction ? i18n.ts.changeReactionConfirm : i18n.ts.cancelReactionConfirm,
		});
		if (confirm.canceled) return;

		if (oldReaction !== reaction) {
			sound.playMisskeySfx('reaction');
		}

		if (mock) {
			emit('reactionToggled', reaction, (props.count - 1));
			return;
		}

		misskeyApi('notes/reactions/delete', {
			noteId: props.note.id,
		}).then(() => {
			if (oldReaction !== reaction) {
				misskeyApi('notes/reactions/create', {
					noteId: props.note.id,
					reaction: reaction,
				});
			}
		});
	} else {
		sound.playMisskeySfx('reaction');

		if (mock) {
			emit('reactionToggled', reaction, (props.count + 1));
			return;
		}

		misskeyApi('notes/reactions/create', {
			noteId: props.note.id,
			reaction: reaction,
		});

		if (props.note.text && props.note.text.length > 100 && (Date.now() - new Date(props.note.createdAt).getTime() < 1000 * 3)) {
			claimAchievement('reactWithoutRead');
		}
	}
}

async function menu(ev) {
	os.popupMenu([...(canGetInfo.value ? [{
		text: i18n.ts.info,
		icon: 'ti ti-info-circle',
		action: async () => {
			os.popup(MkCustomEmojiDetailedDialog, {
				emoji: await misskeyApiGet('emoji', {
					name: props.reaction.replace(/:/g, '').replace(/@\./, ''),
				}),
			});
		},
	}] : []), ...(isAvailable.value && !defaultStore.state.reactions.includes(plainReaction.value) ? [{
		text: i18n.ts.addToEmojiPicker,
		icon: 'ti ti-plus',
		action: async () => {
			defaultStore.set('reactions', [...defaultStore.state.reactions, plainReaction.value]);
		},
	}] : [])], ev.currentTarget ?? ev.target);
}

function anime() {
	if (document.hidden || !defaultStore.state.animation || buttonEl.value == null) return;

	const rect = buttonEl.value.getBoundingClientRect();
	const x = rect.left + 16;
	const y = rect.top + (buttonEl.value.offsetHeight / 2);
	os.popup(MkReactionEffect, { reaction: props.reaction, x, y }, {}, 'end');
}

watch(() => props.count, (newCount, oldCount) => {
	if (oldCount < newCount) anime();
});

onMounted(() => {
	if (!props.isInitial) anime();
});

if (!mock) {
	useTooltip(buttonEl, async (showing) => {
		const reactions = !defaultStore.state.hideReactionUsers ? await misskeyApiGet('notes/reactions', {
			noteId: props.note.id,
			type: props.reaction,
			limit: 10,
			_cacheKey_: props.count,
		}) : [];

		const users = reactions.map(x => x.user);

		os.popup(XDetails, {
			showing,
			reaction: props.reaction,
			users,
			count: props.count,
			targetElement: buttonEl.value,
		}, {}, 'closed');
	}, 100);
}
</script>

<style lang="scss" module>
.root {
	display: inline-flex;
	height: 42px;
	margin: 2px;
	padding: 0 6px;
	font-size: 1.5em;
	border-radius: 6px;
	align-items: center;
	justify-content: center;

	&.canToggle {
		background: var(--buttonBg);

		&:hover {
			background: rgba(0, 0, 0, 0.1);
		}
	}

	&.canToggleFallback:not(.canToggle):not(.reacted) {
		box-sizing: border-box;
		border: 2px dashed var(--switchBg);

		&.small {
			border-width: 1px;
			border-color: var(--buttonBgSub);
		}

		&:hover {
			background: rgba(0, 0, 0, 0.1);
		}
	}

	&:not(.canToggle):not(.canToggleFallback) {
		cursor: default;
	}

	&.small {
		height: 32px;
		font-size: 1em;
		border-radius: 4px;

		> .count {
			font-size: 0.9em;
			line-height: 32px;
		}
	}

	&.large {
		height: 52px;
		font-size: 2em;
		border-radius: 8px;

		> .count {
			font-size: 0.6em;
			line-height: 52px;
		}
	}

	&.reacted, &.reacted:hover {
		background: var(--accentedBg);
		color: var(--accent);
		box-shadow: 0 0 0 1px var(--accent) inset;

		> .count {
			color: var(--accent);
		}

		> .icon {
			filter: drop-shadow(0 0 2px rgba(0, 0, 0, 0.5));
		}
	}
}

.limitWidth {
	max-width: 70px;
	object-fit: contain;
}

.count {
	font-size: 0.7em;
	line-height: 42px;
	margin: 0 0 0 4px;
}
</style>
