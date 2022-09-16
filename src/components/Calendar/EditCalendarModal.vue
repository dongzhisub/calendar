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
				<input :id="calendarNameId" v-model="calendarName" type="text">
			</div>

			<div class="edit-calendar-modal__row edit-calendar-modal__row--color">
				<label :for="colorPickerId">{{ $t('calendar', 'Color') }}</label>
				<div class="edit-calendar-modal__row--color__picker">
					<ColorPicker v-model="calendarColor">
						<input v-model="calendarColor"
							class="edit-calendar-modal__row--color__picker__input"
							:style="{'background-color': calendarColor}">
					</ColorPicker>
				</div>
			</div>

			<h2>{{ $t('calendar', 'Share calendar') }}</h2>

			<div class="edit-calendar-modal__sharing">
				<CalendarListItemSharingPublishItem :calendar="calendar" />

				<Multiselect :options="usersOrGroups"
					:searchable="true"
					:internal-search="false"
					:max-height="600"
					:show-no-results="true"
					:placeholder="$t('calendar', 'Share with users or groups')"
					:user-select="true"
					open-direction="bottom"
					track-by="user"
					label="displayName"
					@search-change="findSharee"
					@change="shareCalendar">
					<span slot="noResult">{{ $t('calendar', 'No users or groups') }}</span>
				</Multiselect>
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
import Multiselect from '@nextcloud/vue/dist/Components/Multiselect'
import CalendarListItemSharingPublishItem from '../AppNavigation/CalendarList/CalendarListItemSharingPublishItem'
import { mapGetters } from 'vuex'
import { randomId } from '../../utils/randomId'
import debounce from 'debounce'
import { principalPropertySearchByDisplaynameOrEmail } from '../../services/caldavService'
import axios from '@nextcloud/axios'
import { urldecode } from '../../utils/url'
import { generateOcsUrl } from '@nextcloud/router'

export default {
	name: 'EditCalendarModal',
	components: {
		Modal,
		ColorPicker,
		Button,
		Multiselect,
		CalendarListItemSharingPublishItem,
	},
	props: {
	},
	data() {
		return {
			calendarColor: undefined,
			calendarName: undefined,
			calendarNameId: randomId(),
			colorPickerId: randomId(),

			usersOrGroups: [],
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
		},
	},
	methods: {
		hideModal() {
			this.$store.commit('hideEditCalendarModal')
		},

		async saveAndClose() {
			this.hideModal()
		},

		/**
		 * Share calendar
		 *
		 * @param {object} data destructuring object
		 * @param {string} data.user the userId
		 * @param {string} data.displayName the displayName
		 * @param {string} data.uri the sharing principalScheme uri
		 * @param {boolean} data.isGroup is this a group ?
		 * @param {boolean} data.isCircle is this a circle-group ?
		 */
		async shareCalendar({ user, displayName, uri, isGroup, isCircle }) {
			await this.$store.dispatch('shareCalendar', {
				calendar: this.calendar,
				user,
				displayName,
				uri,
				isGroup,
				isCircle,
			})
		},

		/**
		 * Use the cdav client call to find matches to the query from the existing Users & Groups
		 *
		 * @param {string} query
		 */
		findSharee: debounce(async function(query) {
			const hiddenPrincipalSchemes = []
			const hiddenUrls = []
			this.calendar.shares.forEach((share) => {
				hiddenPrincipalSchemes.push(share.uri)
			})
			if (this.$store.getters.getCurrentUserPrincipal) {
				hiddenUrls.push(this.$store.getters.getCurrentUserPrincipal.url)
			}
			if (this.calendar.owner) {
				hiddenUrls.push(this.calendar.owner)
			}

			this.isLoading = true
			this.usersOrGroups = []

			if (query.length > 0) {
				const davPromise = this.findShareesFromDav(query, hiddenPrincipalSchemes, hiddenUrls)
				const ocsPromise = this.findShareesFromCircles(query, hiddenPrincipalSchemes, hiddenUrls)

				const [davResults, ocsResults] = await Promise.all([davPromise, ocsPromise])
				this.usersOrGroups = [
					...davResults,
					...ocsResults,
				]

				this.isLoading = false
				this.inputGiven = true
			} else {
				this.inputGiven = false
				this.isLoading = false
			}
		}, 500),

		/**
		 *
		 * @param {string} query The search query
		 * @param {string[]} hiddenPrincipals A list of principals to exclude from search results
		 * @param {string[]} hiddenUrls A list of urls to exclude from search results
		 * @return {Promise<object[]>}
		 */
		async findShareesFromDav(query, hiddenPrincipals, hiddenUrls) {
			let results
			try {
				results = await principalPropertySearchByDisplaynameOrEmail(query)
			} catch (error) {
				return []
			}

			return results.reduce((list, result) => {
				if (['ROOM', 'RESOURCE'].includes(result.calendarUserType)) {
					return list
				}

				const isGroup = result.calendarUserType === 'GROUP'

				// TODO: Why do we have to decode those two values?
				const user = urldecode(result[isGroup ? 'groupId' : 'userId'])
				const decodedPrincipalScheme = urldecode(result.principalScheme)

				if (hiddenPrincipals.includes(decodedPrincipalScheme)) {
					return list
				}
				if (hiddenUrls.includes(result.url)) {
					return list
				}

				// Don't show resources and rooms
				if (!['GROUP', 'INDIVIDUAL'].includes(result.calendarUserType)) {
					return list
				}

				list.push({
					user,
					displayName: result.displayname,
					icon: isGroup ? 'icon-group' : 'icon-user',
					uri: decodedPrincipalScheme,
					isGroup,
					isCircle: false,
					isNoUser: isGroup,
					search: query,
				})
				return list
			}, [])
		},

		/**
		 *
		 * @param {string} query The search query
		 * @param {string[]} hiddenPrincipals A list of principals to exclude from search results
		 * @param {string[]} hiddenUrls A list of urls to exclude from search results
		 * @return {Promise<object[]>}
		 */
		async findShareesFromCircles(query, hiddenPrincipals, hiddenUrls) {
			let results
			try {
				results = await axios.get(generateOcsUrl('apps/files_sharing/api/v1/') + 'sharees', {
					params: {
						format: 'json',
						search: query,
						perPage: 200,
						itemType: 'principals',
					},
				})
			} catch (error) {
				return []
			}

			if (results.data.ocs.meta.status === 'failure') {
				return []
			}

			let circles = []
			if (Array.isArray(results.data.ocs.data.circles)) {
				circles = circles.concat(results.data.ocs.data.circles)
			}
			if (Array.isArray(results.data.ocs.data.exact.circles)) {
				circles = circles.concat(results.data.ocs.data.exact.circles)
			}

			if (circles.length === 0) {
				return []
			}

			return circles.filter((circle) => {
				return !hiddenPrincipals.includes('principal:principals/circles/' + circle.value.shareWith)
			}).map(circle => ({
				user: circle.label,
				displayName: circle.label,
				icon: 'icon-circle',
				uri: 'principal:principals/circles/' + circle.value.shareWith,
				isGroup: false,
				isCircle: true,
				isNoUser: true,
				search: query,
			}))
		},
	},
}
</script>

<style lang="scss" scoped>
.edit-calendar-modal {
	padding: 20px;
	display: flex;
	flex-direction: column;

	div + h2 {
		margin-top: 15px;
	}

	&__row {
		display: flex;
		flex-direction: column;

		&--name {
			input {
				width: 100%;
			}
		}

		&--color {
			&__picker {
				display: flex;
				gap: 10px;

				&__input,
				::v-deep .v-popover,
				::v-deep .trigger {
					width: 100%;
				}
			}
		}
	}

	&__row + &__row {
		margin-top: 10px;
	}

	&__sharing {
		display: flex;
		flex-direction: column;
	}

	&__actions {
		margin-top: 15px;
	}
}
</style>
