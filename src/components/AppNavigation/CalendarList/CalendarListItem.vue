<!--
  - @copyright Copyright (c) 2019 Georg Ehrke <oc.list@georgehrke.com>
  -
  - @author Georg Ehrke <oc.list@georgehrke.com>
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
	<AppNavigationItem :loading="calendar.loading"
		:aria-description="descriptionAppNavigationItem"
		:title="calendar.displayName || $t('calendar', 'Untitled calendar')"
		:class="{deleted: !!deleteTimeout, disabled: !calendar.enabled}"
		@click.prevent.stop="toggleEnabled">
		<template #icon>
			<CheckboxBlankCircle v-if="calendar.enabled"
				:size="20"
				:fill-color="calendar.color" />
			<CheckboxBlankCircleOutline v-else
				:size="20"
				:fill-color="calendar.color" />
		</template>

		<template #counter>
			<NcAvatar v-if="isSharedWithMe && loadedOwnerPrincipal" :user="ownerUserId" :display-name="ownerDisplayname" />
			<div v-if="isSharedWithMe && !loadedOwnerPrincipal" class="icon icon-loading" />
		</template>

		<template #actions>
			<template v-if="!deleteTimeout">
				<ActionButton @click.prevent.stop="showEditModal">
					<template #icon>
						<Pencil :size="20" decorative />
					</template>
					{{ $t('calendar', 'Edit and share calendar') }}
				</ActionButton>
				<ActionButton @click.stop.prevent="copyLink">
					<template #icon>
						<LinkVariant :size="20" decorative />
					</template>
					{{ $t('calendar', 'Copy private link') }}
				</ActionButton>
				<ActionLink target="_blank"
					:href="downloadUrl">
					<template #icon>
						<Download :size="20" decorative />
					</template>
					{{ $t('calendar', 'Export') }}
				</ActionLink>
				<ActionButton v-if="calendar.isSharedWithMe"
					@click.prevent.stop="deleteCalendar">
					<template #icon>
						<Close :size="20" decorative />
					</template>
					{{ $t('calendar', 'Unshare from me') }}
				</ActionButton>
				<ActionButton v-if="!calendar.isSharedWithMe"
					@click.prevent.stop="deleteCalendar">
					<template #icon>
						<Delete :size="20" decorative />
					</template>
					{{ $t('calendar', 'Delete') }}
				</ActionButton>
			</template>

			<template v-if="deleteTimeout">
				<ActionButton v-if="calendar.isSharedWithMe"
					@click.prevent.stop="cancelDeleteCalendar">
					<template #icon>
						<Undo :size="20" decorative />
					</template>
					{{ $n('calendar', 'Unsharing the calendar in {countdown} second', 'Unsharing the calendar in {countdown} seconds', countdown, { countdown }) }}
				</ActionButton>
				<ActionButton v-else
					@click.prevent.stop="cancelDeleteCalendar">
					<template #icon>
						<Undo :size="20" decorative />
					</template>
					{{ $n('calendar', 'Deleting the calendar in {countdown} second', 'Deleting the calendar in {countdown} seconds', countdown, { countdown }) }}
				</ActionButton>
			</template>
		</template>
	</AppNavigationItem>
</template>

<script>
import NcAvatar from '@nextcloud/vue/dist/Components/NcAvatar.js'
import ActionButton from '@nextcloud/vue/dist/Components/NcActionButton.js'
import ActionLink from '@nextcloud/vue/dist/Components/NcActionLink.js'
import AppNavigationItem from '@nextcloud/vue/dist/Components/NcAppNavigationItem.js'
import {
	showSuccess,
	showError,
} from '@nextcloud/dialogs'
import {
	generateRemoteUrl,
} from '@nextcloud/router'

import CheckboxBlankCircle from 'vue-material-design-icons/CheckboxBlankCircle.vue'
import CheckboxBlankCircleOutline from 'vue-material-design-icons/CheckboxBlankCircleOutline.vue'
import Close from 'vue-material-design-icons/Close.vue'
import Delete from 'vue-material-design-icons/Delete.vue'
import Download from 'vue-material-design-icons/Download.vue'
import LinkVariant from 'vue-material-design-icons/LinkVariant.vue'
import Pencil from 'vue-material-design-icons/Pencil.vue'
import Undo from 'vue-material-design-icons/Undo.vue'

export default {
	name: 'CalendarListItem',
	components: {
		NcAvatar,
		ActionButton,
		ActionLink,
		AppNavigationItem,
		CheckboxBlankCircle,
		CheckboxBlankCircleOutline,
		Close,
		Delete,
		Download,
		LinkVariant,
		Pencil,
		Undo,
	},
	props: {
		calendar: {
			type: Object,
			required: true,
		},
	},
	data() {
		return {
			// share menu
			shareMenuOpen: false,
			// Deleting
			deleteInterval: null,
			deleteTimeout: null,
			countdown: 7,
		}
	},
	computed: {
		/**
		 * Download url of the calendar
		 *
		 * @return {string}
		 */
		downloadUrl() {
			return this.calendar.url + '?export'
		},
		/**
		 * Whether or not to display the sharing icon.
		 * It will only be displayed when the calendar is either sharable or publishable
		 *
		 * @return {boolean}
		 */
		showSharingIcon() {
			return this.calendar.canBeShared || this.calendar.canBePublished
		},
		/**
		 * Whether or not the calendar is either shared or published
		 * This is used to figure out whether or not to display the Shared label
		 *
		 * @return {boolean}
		 */
		isSharedOrPublished() {
			return this.isShared || this.isPublished
		},
		/**
		 * Is the calendar shared?
		 *
		 * @return {boolean}
		 */
		isShared() {
			return !!this.calendar.shares.length
		},
		/**
		 * Is the calendar shared with me?
		 *
		 * @return {boolean}
		 */
		isSharedWithMe() {
			return this.calendar.isSharedWithMe
		},
		/**
		 * Is the calendar published
		 *
		 * @return {boolean}
		 */
		isPublished() {
			return !!this.calendar.publishURL
		},
		/**
		 * TODO: this should use principals and principal.userId
		 *
		 * @return {string}
		 */
		owner() {
			if (this.calendar.owner.indexOf('principal:principals/users/') === '0') {
				console.debug(this.calendar.owner.substr(27))
				return this.calendar.owner.substr(27)
			}

			return ''
		},
		/**
		 * Whether or not the information about the owner principal was loaded
		 *
		 * @return {boolean}
		 */
		loadedOwnerPrincipal() {
			return this.$store.getters.getPrincipalByUrl(this.calendar.owner) !== undefined
		},
		ownerUserId() {
			const principal = this.$store.getters.getPrincipalByUrl(this.calendar.owner)
			if (principal) {
				return principal.userId
			}

			return ''
		},
		ownerDisplayname() {
			const principal = this.$store.getters.getPrincipalByUrl(this.calendar.owner)
			if (principal) {
				return principal.displayname
			}

			return ''
		},
		/**
		 * compute aria-description for AppNavigationItem link
		 *
		 * @return {string}
		 */
		descriptionAppNavigationItem() {
			if (this.calendar.enabled && this.calendar.displayName) {
				return t('calendar', 'Disable calendar "{calendar}"', { calendar: this.calendar.displayName })
			} else if (this.calendar.enabled && !this.calendar.displayName) {
				return t('calendar', 'Disable untitled calendar')
			} else if (!this.calendar.enabled && this.calendar.displayName) {
				return t('calendar', 'Enable calendar "{calendar}"', { calendar: this.calendar.displayName })
			} else {
				return t('calendar', 'Enable untitled calendar')
			}
		},
	},
	methods: {
		/**
		 * Toggles the enabled state of this calendar
		 */
		toggleEnabled() {
			this.$store.dispatch('toggleCalendarEnabled', { calendar: this.calendar })
				.catch((error) => {
					showError(this.$t('calendar', 'An error occurred, unable to change visibility of the calendar.'))
					console.error(error)
				})
		},

		/**
		 * Deletes or unshares the calendar
		 */
		deleteCalendar() {
			this.deleteInterval = setInterval(() => {
				this.countdown--

				if (this.countdown < 0) {
					this.countdown = 0
				}
			}, 1000)
			this.deleteTimeout = setTimeout(async () => {
				try {
					await this.$store.dispatch('deleteCalendar', { calendar: this.calendar })
				} catch (error) {
					showError(this.$t('calendar', 'An error occurred, unable to delete the calendar.'))
					console.error(error)
				} finally {
					clearInterval(this.deleteInterval)
					this.deleteTimeout = null
					this.deleteInterval = null
					this.countdown = 7
				}
			}, 7000)
		},

		/**
		 * Cancels the deletion of a calendar
		 */
		cancelDeleteCalendar() {
			clearTimeout(this.deleteTimeout)
			clearInterval(this.deleteInterval)
			this.deleteTimeout = null
			this.deleteInterval = null
			this.countdown = 7
		},

		/**
		 * Copies the private calendar link
		 * to be used with clients like Thunderbird
		 */
		async copyLink() {
			const rootUrl = generateRemoteUrl('dav')
			const url = new URL(this.calendar.url, rootUrl)

			// TODO - use menuOpen to keep it open instead of toast

			try {
				await this.$copyText(url)
				showSuccess(this.$t('calendar', 'Calendar link copied to clipboard.'))
			} catch (error) {
				console.debug(error)
				showError(this.$t('calendar', 'Calendar link could not be copied to clipboard.'))
			}
		},

		/**
		 * Open the calendar modal for this calendar item.
		 */
		showEditModal() {
			this.$store.commit('showEditCalendarModal', {
				calendarId: this.calendar.id,
			})
		},
	},
}
</script>

<style lang="scss" scoped>
	.app-navigation-entry__counter-wrapper .action-item.sharing .material-design-icon.share {
		opacity: .3;
	}
</style>
