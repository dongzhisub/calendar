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
	<NcModal v-if="!!editCalendarModal && calendar" size="normal" @close="saveAndClose">
		<div class="edit-calendar-modal">
			<h2>{{ $t('calendar', 'Edit calendar') }}</h2>

			<div class="edit-calendar-modal__name-and-color">
				<div class="edit-calendar-modal__name-and-color__color">
					<div v-if="loading"
						class="edit-calendar-modal__name-and-color__color__dot"
						:style="{'background-color': calendarColor}" />
					<NcColorPicker v-else v-model="calendarColor">
						<div class="edit-calendar-modal__name-and-color__color__dot"
							:style="{'background-color': calendarColor}" />
					</NcColorPicker>
				</div>

				<input v-model="calendarName"
					class="edit-calendar-modal__name-and-color__name"
					type="text"
					:disabled="loading">
			</div>

			<!--<h2>Additional actions</h2>-->
			<div class="edit-calendar-modal__additional-actions">
				<NcButton type="primary" @click="copyPrivateLink">
					<template #icon>
						<LinkVariantIcon :size="20" />
					</template>
					{{ $t('calendar', 'Copy private link') }}
				</NcButton>

				<NcButton type="primary" @click="exportCalendar">
					<template #icon>
						<DownloadIcon :size="20" />
					</template>
					{{ $t('calendar', 'Export') }}
				</NcButton>

				<NcButton type="error" @click="deleteCalendar">
					<template #icon>
						<DeleteIcon :size="20" />
					</template>
					{{ $t('calendar', 'Delete') }}
				</NcButton>
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

			<!--
			<div class="edit-calendar-modal__actions">
				<NcButton type="primary" @click="saveAndClose">
					{{ $t('calendar', 'Save') }}
				</NcButton>
			</div>
			-->
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
import logger from '../../utils/logger'
import debounce from 'debounce'
import LinkVariantIcon from 'vue-material-design-icons/LinkVariant.vue'
import DownloadIcon from 'vue-material-design-icons/Download.vue'
import DeleteIcon from 'vue-material-design-icons/Delete.vue'

export default {
	name: 'EditCalendarModal',
	components: {
		NcModal,
		NcColorPicker,
		NcButton,
		PublishCalendar,
		SharingSearch,
		ShareItem,
		LinkVariantIcon,
		DownloadIcon,
		DeleteIcon,
	},
	props: {
	},
	data() {
		return {
			loading: false,
			calendarColor: undefined,
			calendarColorChanged: false,
			calendarName: undefined,
			calendarNameChanged: false,
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
		async calendarColor() {
			await this.saveColor()
		},
		async calendarName() {
			await this.saveName()
		},
		editCalendarModal(value) {
			if (!value) {
				return
			}

			this.calendarName = this.calendar.displayName
			this.calendarColor = this.calendar.color
		},
	},
	methods: {
		/**
		 * Close the modal (without saving).
		 */
		closeModal() {
			this.$store.commit('hideEditCalendarModal')
		},

		/**
		 * Save the calendar color.
		 */
		async saveColor() {
			this.loading = true

			try {
				await this.$store.dispatch('changeCalendarColor', {
					calendar: this.calendar,
					newColor: this.calendarColor,
				})
			} catch (error) {
				logger.error('Failed to save calendar color', {
					calendar: this.calendar,
					newColor: this.calendarColor,
				})
			} finally {
				this.loading = false
			}
		},
		saveColorDebounced: debounce(() => this.saveColor(), 1000),

		/**
		 * Save the calendar name.
		 */
		async saveName() {
			this.loading = true

			try {
				await this.$store.dispatch('renameCalendar', {
					calendar: this.calendar,
					newName: this.calendarName,
				})
			} catch (error) {
				logger.error('Failed to save calendar name', {
					calendar: this.calendar,
					newName: this.calendarName,
				})
			} finally {
				this.loading = false
			}
		},
		saveNameDebounced: debounce(() => this.saveName(), 1000),

		/**
		 * Save unsaved changes and close the modal.
		 *
		 * @return {Promise<void>}
		 */
		async saveAndClose() {
			await this.saveColor.flush()
			await this.saveName.flush()
			this.closeModal()
		},

		/**
		 * Copy the internal/private link of this calendar to the clipboard.
		 *
		 * @return {Promise<void>}
		 */
		async copyPrivateLink() {
			// TODO
		},

		/**
		 * Export the calendar as a download.
		 *
		 * @return {Promise<void>}
		 */
		async exportCalendar() {
			// TODO
		},

		/**
		 * Delete the calendar.
		 *
		 * @return {Promise<void>}
		 */
		async deleteCalendar() {
			// TODO
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
		gap: 10px;
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

	&__additional-actions {
		display: flex;
		gap: 10px;

		button {
			flex: 1 auto;
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
