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

			<div class="edit-calendar-modal__name-and-color">
				<div class="edit-calendar-modal__name-and-color__color">
					<NcColorPicker v-model="calendarColor">
						<div class="edit-calendar-modal__name-and-color__color__dot"
							:style="{'background-color': calendarColor}" />
					</NcColorPicker>
				</div>

				<input v-model="calendarName" class="edit-calendar-modal__name-and-color__name" type="text">
			</div>

			<h2 class="edit-calendar-modal__sharing-header">
				{{ $t('calendar', 'Share calendar') }}
			</h2>

			<div class="edit-calendar-modal__sharing">
				<SharingSearch :calendar="calendar" />
				<PublishCalendar :calendar="calendar" />
				<ShareItem v-for="sharee in calendar.shares"
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
import PublishCalendar from './EditCalendarModal/PublishCalendar'
import SharingSearch from './EditCalendarModal/SharingSearch'
import ShareItem from './EditCalendarModal/ShareItem'
import { mapGetters } from 'vuex'
import { randomId } from '../../utils/randomId'

export default {
	name: 'EditCalendarModal',
	components: {
		NcModal,
		NcColorPicker,
		NcButton,
		PublishCalendar,
		SharingSearch,
		ShareItem,
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

	&__name-and-color {
		display: flex;
		align-items: center;
		gap: 5px;
		margin-bottom: 10px;

		&__color {
			::v-deep &__dot {
				width: 24px;
				height: 24px;
				border-radius: 12px;
			}
		}

		&__name {
			flex: 1 auto;
		}
	}

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
		display: flex;
		justify-content: end;
		margin-top: 15px;
	}
}
</style>
