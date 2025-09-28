<!--
 - SPDX-FileCopyrightText: 2024 Nextcloud GmbH and Nextcloud contributors
 - SPDX-License-Identifier: AGPL-3.0-or-later
 -->
<template>
	<fieldset class="ldap-wizard__users">
		{{ t('user_name', 'Listing and searching for users is constrained by these criteria:') }}

		<div class="ldap-wizard__users__line ldap-wizard__users__user-filter-object-class">
			<NcSelect v-model="ldapUserFilterObjectclass"
				:disabled="ldapUserFilterMode"
				class="ldap-wizard__users__user-filter-object-class__select"
				:options="userObjectClasses"
				:input-label="t('user_name', 'Only these object classes:')"
				:multiple="true" />
			{{ t('user_name', 'The most common object classes for users are organizationalPerson, person, user, and inetOrgPerson. If you are not sure which object class to select, please consult your directory admin.') }}
		</div>

		<div class="ldap-wizard__users__line ldap-wizard__users__user-filter-groups">
			<div>
				{{ t('user_name', 'Only from these groups:') }}
			</div>

			<NcSelect v-model="ldapUserFilterGroups"
				class="ldap-wizard__users__user-filter-groups__select"
				:disabled="ldapUserFilterMode"
				:options="userGroups"
				:input-label="t('user_name', 'Only these groups:')"
				:multiple="true" />
		</div>

		<div class="ldap-wizard__users__line ldap-wizard__users__user-filter">
			<NcCheckboxRadioSwitch :checked="ldapUserFilterMode"
				@update:checked="toggleFilterMode">
				{{ t('user_name', 'Edit LDAP Query') }}
			</NcCheckboxRadioSwitch>

			<div v-if="ldapUserFilterMode">
				<NcTextArea :value.sync="ldapConfig.ldapUserFilter"
					:placeholder="t('user_name', 'Edit LDAP Query')"
					:helper-text="t('user_name', 'The filter specifies which LDAP users shall have access to the {instanceName} instance.', { instanceName })" />
			</div>
			<div v-else>
				<label>{{ t('user_name', 'LDAP Filter:') }}</label>
				<code>{{ ldapConfig.ldapUserFilter }}</code>
			</div>
		</div>

		<div class="ldap-wizard__users__line ldap-wizard__users__user-count-check">
			<NcButton @click="countUsers">
				{{ t('user_name', 'Verify settings and count users') }}
			</NcButton>

			<span v-if="usersCount !== undefined">{{ t('user_ldap', "User count: {userCount}", { usersCount }) }}</span>
		</div>
	</fieldset>
</template>

<script lang="ts" setup>
import { ref, computed } from 'vue'
import { storeToRefs } from 'pinia'

import { t } from '@nextcloud/l10n'
import { NcButton, NcTextArea, NcCheckboxRadioSwitch, NcSelect } from '@nextcloud/vue'
import { getCapabilities } from '@nextcloud/capabilities'

import { useLDAPConfigsStore } from '../../store/configs'
import { useWizardStore } from '../../store/wizard'
import { showEnableAutomaticFilterInfo } from '../../services/ldapConfigService'

const wizardStore = useWizardStore()
const ldapConfigsStore = useLDAPConfigsStore()

const { ldapConfigs, selectedConfigId } = storeToRefs(ldapConfigsStore)
const ldapConfig = ldapConfigsStore.selectedConfig({
	ldapUserFilterObjectclass: reloadFilters,
	ldapUserFilterGroups: reloadFilters,
})

const usersCount = ref<number|undefined>(undefined)

const instanceName = (getCapabilities() as { theming: { name:string } }).theming.name

const userObjectClasses = ref([] as string[])
const userGroups = ref([] as string[])

const ldapUserFilterObjectclass = computed<string[]>({
	get() { return ldapConfig.ldapUserFilterObjectclass?.split(';').filter((item) => item !== '') ?? [] },
	set(value) { ldapConfig.ldapUserFilterObjectclass = value.join(';') },
})
const ldapUserFilterGroups = computed({
	get() { return ldapConfig.ldapUserFilterGroups.split(';').filter((item) => item !== '') },
	set(value) { ldapConfig.ldapUserFilterGroups = value.join(';') },
})
const ldapUserFilterMode = computed(() => ldapConfig.ldapUserFilterMode === '1')

async function init() {
	const response1 = await wizardStore.callWizardAction('determineUserObjectClasses')
	userObjectClasses.value = response1.options.ldap_userfilter_objectclass
	// Not using ldapConfig to avoid triggering the save logic.
	ldapConfigs.value[selectedConfigId.value].ldapUserFilterObjectclass = response1.changes.ldap_userfilter_objectclass.join(';')

	const response2 = await wizardStore.callWizardAction('determineGroupsForUsers')
	userGroups.value = response2.options.ldap_userfilter_groups
	// Not using ldapConfig to avoid triggering the save logic.
	ldapConfigs.value[selectedConfigId.value].ldapUserFilterGroups = response2.changes.ldap_userfilter_groups.join(';')
}

init()

async function reloadFilters() {
	if (ldapConfig.ldapUserFilterMode === '0') {
		const response1 = await wizardStore.callWizardAction('getUserListFilter')
		// Not using ldapConfig to avoid triggering the save logic.
		ldapConfigs.value[selectedConfigId.value].ldapUserFilter = response1.changes.ldap_userlist_filter

		const response2 = await wizardStore.callWizardAction('getUserLoginFilter')
		// Not using ldapConfig to avoid triggering the save logic.
		ldapConfigs.value[selectedConfigId.value].ldapLoginFilter = response2.changes.ldap_userlogin_filter
	}
}

async function countUsers() {
	const { changes: { ldap_test_base: ldapTestBase } } = await wizardStore.callWizardAction('countUsers')
	usersCount.value = ldapTestBase
}

async function toggleFilterMode(value: boolean) {
	if (value) {
		ldapConfig.ldapUserFilterMode = '1'
	} else {
		ldapConfig.ldapUserFilterMode = await showEnableAutomaticFilterInfo()
	}
}
</script>
<style lang="scss" scoped>
.ldap-wizard__users {
	display: flex;
	flex-direction: column;
	gap: 16px;

	&__line {
		display: flex;
		align-items: start;
	}

	&__user-filter-object-class {
		display: flex;
		gap: 16px;

		&__select {
			min-width: 50%;
			flex-grow: 1;
		}
	}

	&__user-filter-groups {
		display: flex;
		gap: 16px;

		&__select {
			min-width: 50%;
			flex-grow: 1;
		}
	}

	&__user-filter {
		display: flex;
		flex-direction: column;

		code {
			background-color: var(--color-background-dark);
			padding: 4px;
			border-radius: 4px;
		}
	}

	&__user-count-check {
		display: flex;
		align-items: center;
		gap: 16px;
	}
}
</style>
