<!--
  - @copyright Copyright (c) 2022 Richard Steinmetz <richard@steinmetz.cloud>
  -
  - @author Richard Steinmetz <richard@steinmetz.cloud>
  -
  - @license AGPL-3.0-or-later
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program. If not, see <http://www.gnu.org/licenses/>.
  -
  -->

<template>
	<NcModal v-if="!!editCalendarModal && calendar" size="small" @close="hideModal">
		<div class="edit-calendar-modal">
			<h2>{{ $t('calendar', 'Edit calendar') }}</h2>

			<div class="edit-calendar-modal__name">
				<label :for="calendarNameId">{{ $t('calendar', 'Name') }}</label>
				<input :id="calendarNameId" v-model="calendarName" type="text">
			</div>

			<div class="edit-calendar-modal__color">
				<label :for="colorPickerId">{{ $t('calendar', 'Color') }}</label>
				<div class="edit-calendar-modal__color__picker">
					<NcColorPicker v-model="calendarColor">
						<input v-model="calendarColor"
							class="edit-calendar-modal__color__picker__input"
							:style="{'background-color': calendarColor}">
					</NcColorPicker>
				</div>
			</div>

			<h2 class="edit-calendar-modal__sharing-header">
				{{ $t('calendar', 'Share calendar') }}
			</h2>

			<div class="edit-calendar-modal__sharing">
				<CalendarListItemSharingPublishItem :calendar="calendar" />
				<CalendarListItemSharingSearch :calendar="calendar" />
				<CalendarListItemSharingShareItem v-for="sharee in calendar.shares"
					:key="sharee.uri"
					:sharee="sharee"
					:calendar="calendar" />
			</div>

			<div class="edit-calendar-modal__actions">
				<NcButton type="primary" @click="saveAndClose">
					{{ $t('calendar', 'Save') }}
				</NcButton>
			</div>
		</div>
	</NcModal>
</template>

<script>
import NcModal from '@nextcloud/vue/dist/Components/NcModal'
import NcColorPicker from '@nextcloud/vue/dist/Components/NcColorPicker'
import NcButton from '@nextcloud/vue/dist/Components/NcButton'
import CalendarListItemSharingPublishItem
	from '../AppNavigation/CalendarList/CalendarListItemSharingPublishItem'
import CalendarListItemSharingSearch
	from '../AppNavigation/CalendarList/CalendarListItemSharingSearch'
import CalendarListItemSharingShareItem
	from '../AppNavigation/CalendarList/CalendarListItemSharingShareItem'
import { mapGetters } from 'vuex'
import { randomId } from '../../utils/randomId'

export default {
	name: 'EditCalendarModal',
	components: {
		NcModal,
		NcColorPicker,
		NcButton,
		CalendarListItemSharingSearch,
		CalendarListItemSharingPublishItem,
		CalendarListItemSharingShareItem,
	},
	props: {
	},
	data() {
		return {
			calendarColor: undefined,
			calendarName: undefined,
			calendarNameId: randomId(),
			colorPickerId: randomId(),
		}
	},
	computed: {
		...mapGetters(['editCalendarModal']),
		calendar() {
			const id = this.editCalendarModal?.calendarId
			if (!id) {
				return undefined
			}

			return this.$store.getters.getCalendarById(id)
		},
	},
	watch: {
		editCalendarModal(value) {
			if (!value) {
				return
			}

			this.calendarName = this.calendar.displayName
			this.calendarColor = this.calendar.color
		},
	},
	methods: {
		hideModal() {
			this.$store.commit('hideEditCalendarModal')
		},

		async saveAndClose() {
			this.hideModal()
		},
	},
}
</script>

<style lang="scss" scoped>
.edit-calendar-modal {
	padding: 20px;
	display: flex;
	flex-direction: column;

	&__name {
		display: flex;
		flex-direction: column;
		margin-bottom: 10px;

		input {
			width: 100%;
		}
	}

	&__color {
		display: flex;
		flex-direction: column;

		&__picker {
			display: flex;
			gap: 10px;

			&__input,
			::v-deep .v-popper,
			::v-deep .trigger {
				width: 100%;
			}
		}
	}

	&__sharing-header {
		margin-top: 25px;
	}

	&__sharing {
		display: flex;
		flex-direction: column;
		gap: 5px;
	}

	&__actions {
		margin-top: 15px;
	}
}
</style>
