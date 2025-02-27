<!--
  - @copyright 2023 Christopher Ng <chrng8@gmail.com>
  -
  - @author Christopher Ng <chrng8@gmail.com>
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
	<div class="system-tags">
		<label for="system-tags-input">{{ t('systemtags', 'Search or create collaborative tags') }}</label>
		<NcSelectTags class="system-tags__select"
			input-id="system-tags-input"
			:placeholder="t('systemtags', 'Collaborative tags …')"
			:options="sortedTags"
			:value="selectedTags"
			:create-option="createOption"
			:taggable="true"
			:passthru="true"
			:fetch-tags="false"
			:loading="loading"
			@input="handleInput"
			@option:selected="handleSelect"
			@option:created="handleCreate"
			@option:deselected="handleDeselect">
			<template #no-options>
				{{ t('systemtags', 'No tags to select, type to create a new tag') }}
			</template>
		</NcSelectTags>
	</div>
</template>

<script lang="ts">
// FIXME Vue TypeScript ESLint errors
/* eslint-disable */
import Vue from 'vue'
import NcSelectTags from '@nextcloud/vue/dist/Components/NcSelectTags.js'

import { translate as t } from '@nextcloud/l10n'
import { showError } from '@nextcloud/dialogs'

import {
	createTag,
	deleteTag,
	fetchLastUsedTagIds,
	fetchSelectedTags,
	fetchTags,
	selectTag,
} from '../services/api.js'

import type { BaseTag, Tag, TagWithId } from '../types.js'

const defaultBaseTag: BaseTag = {
	userVisible: true,
	userAssignable: true,
	canAssign: true,
}

export default Vue.extend({
	name: 'SystemTags',

	components: {
		NcSelectTags,
	},

	props: {
		fileId: {
			type: Number,
			required: true,
		},
	},

	data() {
		return {
			sortedTags: [] as TagWithId[],
			selectedTags: [] as TagWithId[],
			loading: false,
		}
	},

	async created() {
		try {
			const tags = await fetchTags()
			const lastUsedOrder = await fetchLastUsedTagIds()

			const lastUsedTags: TagWithId[] = []
			const remainingTags: TagWithId[] = []

			for (const tag of tags) {
				if (lastUsedOrder.includes(tag.id)) {
					lastUsedTags.push(tag)
					continue
				}
				remainingTags.push(tag)
			}

			const sortByLastUsed = (a: TagWithId, b: TagWithId) => {
				return lastUsedOrder.indexOf(a.id) - lastUsedOrder.indexOf(b.id)
			}
			lastUsedTags.sort(sortByLastUsed)

			this.sortedTags = [...lastUsedTags, ...remainingTags]
		} catch (error) {
			showError(t('systemtags', 'Failed to load tags'))
		}
	},

	watch: {
		fileId: {
			immediate: true,
			async handler() {
				try {
					this.selectedTags = await fetchSelectedTags(this.fileId)
					this.$emit('has-tags', this.selectedTags.length > 0)
				} catch (error) {
					showError(t('systemtags', 'Failed to load selected tags'))
				}
			},
		},
	},

	methods: {
		t,

		createOption(newDisplayName: string): Tag {
			for (const tag of this.sortedTags) {
				const { id, displayName, ...baseTag } = tag
				if (
					displayName === newDisplayName
					&& Object.entries(baseTag)
						.every(([key, value]) => defaultBaseTag[key] === value)
				) {
					// Return existing tag to prevent vue-select from thinking the tags are different and showing duplicate options
					return tag
				}
			}
			return {
				...defaultBaseTag,
				displayName: newDisplayName,
			}
		},

		handleInput(selectedTags: Tag[]) {
			/**
			 * Filter out tags with no id to prevent duplicate selected options
			 *
			 * Created tags are added programmatically by `handleCreate()` with
			 * their respective ids returned from the server
			 */
			this.selectedTags = selectedTags.filter(selectedTag => Boolean(selectedTag.id)) as TagWithId[]
		},

		async handleSelect(tags: Tag[]) {
			const selectedTag = tags[tags.length - 1]
			if (!selectedTag.id) {
				// Ignore created tags handled by `handleCreate()`
				return
			}
			this.loading = true
			try {
				await selectTag(this.fileId, selectedTag)
				const sortToFront = (a: TagWithId, b: TagWithId) => {
					if (a.id === selectedTag.id) {
						return -1
					} else if (b.id === selectedTag.id) {
						return 1
					}
					return 0
				}
				this.sortedTags.sort(sortToFront)
			} catch (error) {
				showError(t('systemtags', 'Failed to select tag'))
			}
			this.loading = false
		},

		async handleCreate(tag: Tag) {
			this.loading = true
			try {
				const id = await createTag(this.fileId, tag)
				const createdTag = { ...tag, id }
				this.sortedTags.unshift(createdTag)
				this.selectedTags.push(createdTag)
			} catch (error) {
				showError(t('systemtags', 'Failed to create tag'))
			}
			this.loading = false
		},

		async handleDeselect(tag: Tag) {
			this.loading = true
			try {
				await deleteTag(this.fileId, tag)
			} catch (error) {
				showError(t('systemtags', 'Failed to delete tag'))
			}
			this.loading = false
		},
	},
})
</script>

<style lang="scss" scoped>
.system-tags {
	display: flex;
	flex-direction: column;

	label[for="system-tags-input"] {
		margin-bottom: 2px;
	}

	&__select {
		width: 100%;
		:deep {
			.vs__deselect {
				padding: 0;
			}
		}
	}
}
</style>
