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
	<Modal v-if="!!editCalendarModal && calendar" size="small" @close="hideModal">
		<div class="edit-calendar-modal">
			<h2>{{ $t('calendar', 'Edit calendar') }}</h2>

			<div class="edit-calendar-modal__row edit-calendar-modal__row--name">
				<label :for="calendarNameId">{{ $t('calendar', 'Name') }}</label>
				<input v-model="calendarName" type="text" :id="calendarNameId">
			</div>

			<div class="edit-calendar-modal__row edit-calendar-modal__row--color">
				<label :for="colorPickerId">{{ $t('calendar', 'Color') }}</label>
				<div class="edit-calendar-modal__row--color__picker">
					<div
						class="edit-calendar-modal__row--color__picker__indicator"
						:style="{'background-color': calendarColor}" />
					<ColorPicker v-model="calendarColor">
						<Button :id=colorPickerId>{{ $t('calendar', 'Change color') }}</Button>
					</ColorPicker>
				</div>
			</div>

			<div class="edit-calendar-modal__actions">
				<Button type="primary" @click="saveAndClose">
					{{ $t('calendar', 'Save') }}
				</Button>
			</div>
		</div>
	</Modal>
</template>

<script>
import Modal from '@nextcloud/vue/dist/Components/Modal'
import ColorPicker from '@nextcloud/vue/dist/Components/ColorPicker'
import Button from '@nextcloud/vue/dist/Components/Button'
import { mapGetters } from 'vuex'
import { randomId } from '../../utils/randomId'

export default {
	name: 'EditCalendarModal',
	components: {
		Modal,
		ColorPicker,
		Button,
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
			console.log('calendar', this.editCalendarModal)
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
		}
	},
	methods: {
		hideModal() {
			this.$store.commit('hideEditCalendarModal')
		},
		async saveAndClose() {
			this.hideModal()
		},
	}
}
</script>

<style lang="scss" scoped>
.edit-calendar-modal {
	padding: 20px;
	display: flex;
	flex-direction: column;

	&__row {
		&--name {
			display: flex;
			flex-direction: column;

			input {
				width: 100%;
			}
		}

		&--color {
			display: flex;
			flex-direction: column;

			&__picker {
				display: flex;
				gap: 10px;

				&__indicator {
					flex: 1 auto;
					border-radius: var(--border-radius-large);
				}
			}
		}
	}

	&__row + &__row {
		margin-top: 10px;
	}

	&__actions {
		margin-top: 15px;
	}
}
</style>
