<template>
	<k-draggable
		:list="state"
		:options="dragOptions"
		:data-layout="layout"
		element="k-dropdown"
		class="k-multiselect-input"
		@click.native="$refs.dropdown.toggle"
		@end="onInput"
	>
		<k-tag
			v-for="tag in tags"
			:ref="tag.value"
			:key="tag.value"
			:removable="true"
			@click.native.stop
			@remove="remove(tag)"
			@keydown.native.left="navigate('prev')"
			@keydown.native.right="navigate('next')"
			@keydown.native.down="$refs.dropdown.open"
		>
			<!-- eslint-disable-next-line vue/no-v-html -->
			<span v-html="tag.text" />
		</k-tag>

		<template #footer>
			<k-dropdown-content
				ref="dropdown"
				@open="onOpen"
				@close="onClose"
				@keydown.native.esc.stop="close"
			>
				<k-dropdown-item
					v-if="search"
					icon="search"
					class="k-multiselect-search"
				>
					<input
						ref="search"
						v-model="q"
						:placeholder="
							search.min
								? $t('search.min', { min: search.min })
								: $t('search') + ' …'
						"
						@keydown.esc.stop="onEscape"
					/>
				</k-dropdown-item>

				<div class="k-multiselect-options scroll-y-auto">
					<k-dropdown-item
						v-for="option in visible"
						:key="option.value"
						:icon="isSelected(option) ? 'check' : 'circle-outline'"
						:class="{
							'k-multiselect-option': true,
							selected: isSelected(option),
							disabled: !more
						}"
						@click.prevent="select(option)"
						@keydown.native.enter.prevent.stop="select(option)"
						@keydown.native.space.prevent.stop="select(option)"
					>
						<!-- eslint-disable-next-line vue/no-v-html -->
						<span v-html="option.text" />
					</k-dropdown-item>

					<k-dropdown-item
						v-if="filtered.length === 0"
						:disabled="true"
						class="k-multiselect-option"
					>
						{{ emptyLabel }}
					</k-dropdown-item>
				</div>

				<k-button
					v-if="visible.length < filtered.length"
					:text="`${$t('search.all')} (${filtered.length})`"
					class="k-multiselect-more"
					@click.stop="limit = false"
				/>
			</k-dropdown-content>
		</template>
	</k-draggable>
</template>

<script>
import { disabled, id, required } from "@/mixins/props.js";

import {
	required as validateRequired,
	minLength as validateMinLength,
	maxLength as validateMaxLength
} from "vuelidate/lib/validators";

export const props = {
	mixins: [disabled, id, required],
	props: {
		/**
		 * Maximum number for selected options
		 */
		max: Number,
		/**
		 * Minimum number for selected options
		 */
		min: Number,
		/**
		 * @values "list"
		 */
		layout: String,
		/**
		 * Available options to select
		 */
		options: Array,
		search: [Object, Boolean],
		separator: {
			type: String,
			default: ","
		},
		sort: Boolean,
		value: {
			type: Array,
			required: true,
			default() {
				return [];
			}
		}
	}
};

export default {
	mixins: [props],
	inheritAttrs: false,
	data() {
		return {
			// array of selected values
			state: this.value,
			// current search filter string
			q: null,
			limit: true,
			scrollTop: 0
		};
	},
	computed: {
		/**
		 * Whether tags can be manually sorted by dragging
		 * @returns {boolean}
		 */
		draggable() {
			return this.state.length > 1 && !this.sort;
		},
		/**
		 * Options for k-draggable
		 */
		dragOptions() {
			return {
				disabled: !this.draggable,
				draggable: ".k-tag",
				delay: 1
			};
		},
		/**
		 * Text to show in empty dropdown
		 * @returns {string}
		 */
		emptyLabel() {
			if (this.q) {
				return this.$t("search.results.none");
			}

			return this.$t("options.none");
		},
		/**
		 * Filtered (and highlited) list of
		 * options for dropdown
		 * @returns {Array}
		 */
		filtered() {
			if (this.q?.length >= (this.search.min || 0)) {
				return this.options
					.filter((option) => this.isFiltered(option))
					.map((option) => ({
						...option,
						text: this.toHighlightedString(option.text)
					}));
			}

			return this.options;
		},
		/**
		 * Whether more options can be selected
		 * @returns {boolean}
		 */
		more() {
			return !this.max || this.state.length < this.max;
		},
		/**
		 * Regular expression for current search term
		 * @returns {RegExp}
		 */
		regex() {
			return new RegExp(`(${RegExp.escape(this.q)})`, "ig");
		},
		/**
		 * Text-value options for the selected option
		 * to be used with `k-tag`
		 * @returns {Array}
		 */
		tags() {
			const tags = this.state.map((value) =>
				this.options.find((option) => option.value === value)
			);

			if (this.sort === false) {
				return tags;
			}

			const index = (x) => this.options.findIndex((y) => y.value === x.value);
			return tags.sort((a, b) => index(a) - index(b));
		},
		/**
		 * Options to show in dropdown (filtered and limited)
		 * @returns {Array}
		 */
		visible() {
			if (this.limit) {
				return this.filtered.slice(
					0,
					this.search.display || this.filtered.length
				);
			}

			return this.filtered;
		}
	},
	watch: {
		value(value) {
			// array of selected values
			this.state = value;
			this.onInvalid();
		}
	},
	mounted() {
		this.onInvalid();
		this.$events.$on("click", this.close);
		this.$events.$on("keydown.cmd.s", this.close);
	},
	destroyed() {
		this.$events.$off("click", this.close);
		this.$events.$off("keydown.cmd.s", this.close);
	},
	methods: {
		/**
		 * Adds new value as selected
		 * @param {object} option
		 */
		add(option) {
			if (this.more === true) {
				this.state.push(option.value);
				this.onInput();
			}
		},
		blur() {
			this.close();
		},
		close() {
			if (this.$refs.dropdown?.isOpen === true) {
				this.$refs.dropdown.close();
				this.limit = true;
			}
		},
		focus() {
			this.$refs.dropdown?.open();
		},
		/**
		 * Gets index of option in array of selected values
		 * @param {object} option
		 */
		index(option) {
			return this.state.findIndex((item) => item === option.value);
		},
		isFiltered(option) {
			return (
				String(option.text).match(this.regex) ||
				String(option.value).match(this.regex)
			);
		},
		/**
		 * Whether option exists in the currently selected items
		 * @param {object} option
		 * @returns {boolean}
		 */
		isSelected(option) {
			return this.index(option) !== -1;
		},
		navigate(direction) {
			if (direction === "prev") {
				direction = "previous";
			}

			document.activeElement?.[direction + "Sibling"]?.focus?.();
		},
		onClose() {
			if (this.$refs.dropdown?.isOpen === false) {
				if (document.activeElement === this.$parent.$el) {
					this.q = null;
				}

				this.$parent.$el.focus();
			}
		},
		onEscape() {
			if (this.q) {
				this.q = null;
				return;
			}

			this.close();
		},
		onInput() {
			this.$emit("input", this.state);
		},
		onInvalid() {
			this.$emit("invalid", this.$v.$invalid, this.$v);
		},
		onOpen() {
			this.$nextTick(() => {
				this.$refs.search?.focus?.();

				if (this.$refs.dropdown?.$el) {
					this.$refs.dropdown.$el.querySelector(
						".k-multiselect-options"
					).scrollTop = this.scrollTop;
				}
			});
		},
		/**
		 * Removes value from selected
		 * @param {object} option
		 */
		remove(option) {
			const index = this.index(option);
			this.state.splice(index, 1);
			this.onInput();
		},
		/**
		 * Toggles selection status of option
		 * @param {object} option
		 */
		select(option) {
			this.scrollTop = this.$refs.dropdown.$el.querySelector(
				".k-multiselect-options"
			).scrollTop;

			if (this.isSelected(option)) {
				this.remove(option);
			} else {
				this.add(option);
			}
		},
		toHighlightedString(string) {
			// make sure that no HTML exists before in the string
			// to avoid XSS when displaying via `v-html`
			string = this.$helper.string.stripHTML(string);
			return string.replace(this.regex, "<b>$1</b>");
		}
	},
	validations() {
		return {
			state: {
				required: this.required ? validateRequired : true,
				minLength: this.min ? validateMinLength(this.min) : true,
				maxLength: this.max ? validateMaxLength(this.max) : true
			}
		};
	}
};
</script>

<style>
.k-multiselect-input {
	display: flex;
	flex-wrap: wrap;
	position: relative;
	font-size: var(--text-sm);
	min-height: 2.25rem;
	line-height: 1;
}
.k-multiselect-input .k-sortable-ghost {
	background: var(--color-focus);
}
.k-multiselect-input .k-tag {
	border-radius: var(--rounded-sm);
}

.k-multiselect-input .k-dropdown-content {
	width: 100%;
}

.k-multiselect-search {
	margin-top: 0 !important;
	color: var(--color-white);
	background: var(--color-gray-900);
	border-bottom: 1px dashed rgba(255, 255, 255, 0.2);
}
.k-multiselect-search > .k-button-text {
	flex: 1;
	opacity: 1 !important;
}

.k-multiselect-search input {
	width: 100%;
	color: var(--color-white);
	background: none;
	border: none;
	outline: none;
	padding: 0.25rem 0;
	font: inherit;
}

.k-multiselect-options {
	position: relative;
	max-height: 275px;
	padding: 0.5rem 0;
}

.k-multiselect-option {
	position: relative;
}
.k-multiselect-option.selected {
	color: var(--color-positive-light);
}

.k-multiselect-option.disabled:not(.selected) .k-icon {
	opacity: 0;
}
.k-multiselect-option b {
	color: var(--color-focus-light);
	font-weight: 700;
}

.k-multiselect-input[data-layout="list"] .k-tag {
	width: 100%;
	margin-inline-end: 0 !important;
}

.k-multiselect-more {
	width: 100%;
	padding: 0.75rem;
	color: rgba(255, 255, 255, 0.8);
	text-align: center;
	border-top: 1px dashed rgba(255, 255, 255, 0.2);
}
.k-multiselect-more:hover {
	color: var(--color-white);
}
</style>
