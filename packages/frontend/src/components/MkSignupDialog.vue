<!--
SPDX-FileCopyrightText: syuilo and misskey-project
SPDX-License-Identifier: AGPL-3.0-only
-->

<template>
<MkModalWindow
	ref="dialog"
	:width="500"
	:height="600"
	@close="dialog?.close()"
	@closed="$emit('closed')"
>
	<template #header>{{ i18n.ts.signup }}</template>

	<div style="overflow-x: clip;">
		<Transition
			mode="out-in"
			:enterActiveClass="$style.transition_x_enterActive"
			:leaveActiveClass="$style.transition_x_leaveActive"
			:enterFromClass="$style.transition_x_enterFrom"
			:leaveToClass="$style.transition_x_leaveTo"
		>
			<template v-if="!isAcceptedServerRule">
				<XServerRules @done="isAcceptedServerRule = true" @cancel="dialog?.close()"/>
			</template>
			<template v-else-if="!isChoosed && instance.enableRegistrationLimit">
				<XChoice @done="doneChoice" @cancel="dialog?.close()"/>
			</template>
			<template v-else>
				<XSignup :autoSet="autoSet" :withoutInviteCode="withoutInviteCode" @back="onBack" @signup="onSignup" @signupEmailPending="onSignupEmailPending"/>
			</template>
		</Transition>
	</div>
</MkModalWindow>
</template>

<script lang="ts" setup>
import { shallowRef, ref } from 'vue';
import * as Misskey from 'misskey-js';
import XSignup from '@/components/MkSignupDialog.form.vue';
import XServerRules from '@/components/MkSignupDialog.rules.vue';
import XChoice from '@/components/MkSignupDialog.choice.vue';
import MkModalWindow from '@/components/MkModalWindow.vue';
import { instance } from '@/instance.js';
import { i18n } from '@/i18n.js';

const props = withDefaults(defineProps<{
	autoSet?: boolean;
}>(), {
	autoSet: false,
});

const emit = defineEmits<{
	(ev: 'done', res: Misskey.entities.SigninResponse): void;
	(ev: 'closed'): void;
}>();

const dialog = shallowRef<InstanceType<typeof MkModalWindow>>();

const isAcceptedServerRule = ref(false);
const isChoosed = ref(false);
const withoutInviteCode = ref(true);

function doneChoice(hasInviteCode: boolean) {
	isChoosed.value = true;
	withoutInviteCode.value = !hasInviteCode;
}

function onBack() {
	if (isChoosed.value) {
		isChoosed.value = false;
	} else {
		isAcceptedServerRule.value = false;
	}
}

function onSignup(res: Misskey.entities.SigninResponse) {
	emit('done', res);
	dialog.value?.close();
}

function onSignupEmailPending() {
	dialog.value?.close();
}
</script>

<style lang="scss" module>
.transition_x_enterActive,
.transition_x_leaveActive {
	transition: opacity 0.3s cubic-bezier(0,0,.35,1), transform 0.3s cubic-bezier(0,0,.35,1);
}
.transition_x_enterFrom {
	opacity: 0;
	transform: translateX(50px);
}
.transition_x_leaveTo {
	opacity: 0;
	transform: translateX(-50px);
}
</style>
