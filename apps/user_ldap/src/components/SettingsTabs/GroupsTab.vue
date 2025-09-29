<!--
 - SPDX-FileCopyrightText: 2024 Nextcloud GmbH and Nextcloud contributors
 - SPDX-License-Identifier: AGPL-3.0-or-later
 -->
<template>
	<fieldset class="ldap-wizard__groups">
		{{ t('user_ldap', 'Groups meeting these criteria are available in {instanceName}:', {instanceName}) }}

		<div class="ldap-wizard__groups__line ldap-wizard__groups__filter-selection">
			<NcSelect v-model="ldapGroupFilterObjectclass"
				class="ldap-wizard__groups__group-filter-groups__select"
				:options="groupObjectClasses"
				:disabled="ldapGroupFilterMode"
				:input-label="t('user_ldap', 'Only these object classes:')"
				:multiple="true" />

			<NcSelect v-model="ldapGroupFilterGroups"
				class="ldap-wizard__groups__group-filter-groups__select"
				:options="groupGroups"
				:disabled="ldapGroupFilterMode"
				:input-label="t('user_ldap', 'Only from these groups:')"
				:multiple="true" />
		</div>

		<div class="ldap-wizard__groups__line ldap-wizard__groups__groups-filter">
			<NcCheckboxRadioSwitch :checked="ldapGroupFilterMode"
				@update:checked="toggleFilterMode">
				{{ t('user_ldap', 'Edit LDAP Query') }}
			</NcCheckboxRadioSwitch>

			<div v-if="ldapGroupFilterMode">
				<NcTextArea :value.sync="ldapConfig.ldapGroupFilter"
					:placeholder="t('user_ldap', 'Edit LDAP Query')"
					:helper-text="t('user_ldap', 'The filter specifies which LDAP groups shall have access to the {instanceName} instance.', {instanceName})" />
			</div>
			<div v-else>
				<label>{{ t('user_ldap', 'LDAP Filter:') }}</label>
				<code>{{ ldapConfig.ldapGroupFilter }}</code>
			</div>
		</div>

		<div class="ldap-wizard__groups__line ldap-wizard__groups__groups-count-check">
			<NcButton @click="countGroups">
				{{ t('user_ldap', 'Verify settings and count the groups') }}
			</NcButton>

			<span v-if="groupsCountLabel !== undefined">{{ groupsCountLabel }}</span>
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
import { showEnableAutomaticFilterInfo } from '../../services/ldapConfigService'
import { useWizardStore } from '../../store/wizard'

const ldapConfigsStore = useLDAPConfigsStore()
const wizardStore = useWizardStore()

const { ldapConfigs, selectedConfigId } = storeToRefs(ldapConfigsStore)
const ldapConfig = ldapConfigsStore.selectedConfig({
	ldapGroupFilterObjectclass: getGroupFilter,
	ldapGroupFilterGroups: getGroupFilter,
})

const instanceName = (getCapabilities() as { theming: { name:string } }).theming.name

const groupsCountLabel = ref<number|undefined>(undefined)

const groupObjectClasses = ref([] as string[])
const groupGroups = ref([] as string[])

const ldapGroupFilterObjectclass = computed({
	get() { return ldapConfig.ldapGroupFilterObjectclass.split(';').filter((item) => item !== '') },
	set(value) { ldapConfig.ldapGroupFilterObjectclass = value.join(';') },
})
const ldapGroupFilterGroups = computed({
	get() { return ldapConfig.ldapGroupFilterGroups.split(';').filter((item) => item !== '') },
	set(value) { ldapConfig.ldapGroupFilterGroups = value.join(';') },
})
const ldapGroupFilterMode = computed(() => ldapConfig.ldapGroupFilterMode === '1')

async function init() {
	const response1 = await wizardStore.callWizardAction('determineGroupObjectClasses')
	groupObjectClasses.value = response1.options.ldap_groupfilter_objectclass

	const response2 = await wizardStore.callWizardAction('determineGroupsForGroups')
	groupGroups.value = response2.options.ldap_groupfilter_groups
}

init()

async function getGroupFilter() {
	const response = await wizardStore.callWizardAction('getGroupFilter')
	// Not using ldapConfig to avoid triggering the save logic.
	ldapConfigs.value[selectedConfigId.value].ldapGroupFilter = response.changes.ldap_group_filter
}

async function countGroups() {
	const { changes: { ldap_group_count: ldapGroupCount } } = await wizardStore.callWizardAction('countGroups')
	groupsCountLabel.value = ldapGroupCount
}

async function toggleFilterMode(value: boolean) {
	if (value) {
		ldapConfig.ldapGroupFilterMode = '1'
	} else {
		ldapConfig.ldapGroupFilterMode = await showEnableAutomaticFilterInfo()
	}
}
</script>
<style lang="scss" scoped>
.ldap-wizard__groups {
	display: flex;
	flex-direction: column;
	gap: 16px;

	&__line {
		display: flex;
		align-items: start;
	}

	&__filter-selection {
		flex-direction: column;
	}

	&__groups-filter {
		display: flex;
		flex-direction: column;

		code {
			background-color: var(--color-background-dark);
			padding: 4px;
			border-radius: 4px;
		}
	}

	&__groups-count-check {
		display: flex;
		align-items: center;
		gap: 16px;
	}
}
</style>
