<template>
	<div :class="[classes, 'select-autocomplete', isOpened ? 'opened' : false]">
		<input type="hidden" ref="hidden" :name="name" :value="value">
		<input
			type="text" ref="input" v-bind="inputAttribs"
			role="combobox" autocomplete="off" spellcheck="false" aria-autocomplete="list" aria-haspopup="listbox"
			@input="onInput" @keydown="onKeydown" @blur="onBlur"
		>
		<i class="mdi" @mousedown.prevent="onMouseDownButton"></i>
		<div class="select-autocomplete-options-cont">
			<ul
				ref="options" role="listbox" :style="optionsDirectionUp ? 'bottom: 100%' : 'top: 100%'"
				v-show="isOpened && hasVisibleOptions"
				@click="onClickOptions" @mousedown.prevent=""
			>
				<li
					v-for="(option, index) in options" :key="index" :data-index="index"
					v-show="showAllOptions || option.isVisible" :aria-selected="index === displaySelectedIndex"
				><span v-html="highlight(option.title)"></span></li>
			</ul>
		</div>
	</div>
</template>

<script>

	function escapeRegexp(value) {
		return value.replace(/[-[\]{}()*+!<=:?.\/\\^$|#\s,]/g, '\\$&');
	}

	function debounce(func, delay) {
		let timeout = null;
		return function() {
			if (!timeout) {
				timeout = setTimeout(() => {
					timeout = null;
					func();
				}, delay);
			}
		};
	}

	function matches(el, selector) {
		let matches = el.matches || el.msMatchesSelector || el.webkitMatchesSelector;
		return matches.call(el, selector);
	}

	function closest(el, selector) {
		if (el.closest) {
			return el.closest(selector);
		}
		while (el && el.nodeType === 1) {
			if (matches(el, selector)) {
				return el;
			}
			el = el.parentNode;
		}
		return null;
	}

	export default {
		name: 'SelectAutocomplete',

		props: {
			name: String,
			classes: String,
			placeholder: String,
			defaultInputAttribs: Object,
			defaultValue: [String, Number],
			defaultOptions: Array,
			defaultDisabled: Boolean,
		},

		data() {
			this.defaultOptions.forEach( option => {
				option.isVisible = true;
			});

			let selectedIndex = null;
			let inputText = '';
			if (this.defaultValue !== null) {
				let length = this.defaultOptions.length;
				for (let index = 0; index < length; index++) {
					let option = this.defaultOptions[index];
					if (option.value == this.defaultValue) {
						selectedIndex = index;
						inputText = option.title;
						break;
					}
				}
			}

			return {
				options: this.defaultOptions,
				selectedIndex: selectedIndex,
				displaySelectedIndex: selectedIndex,
				inputText: inputText,
				isDisabled: this.defaultDisabled,
				isOpened: false,
				optionsFiltered: false,
				optionsDirectionUp: false,
				showAllOptions: true,
				hasVisibleOptions: this.defaultOptions.length > 0,
				needResetScroll: true,
				needUpdateVisibleOptions: false,
				needUpdateOptionsDirection: false,
				needUpdateOptionsScroll: false,
			};
		},

		created() {
			this.inputTextChanged = debounce( () => {
				this.open();
				this.optionsFiltered = true;
				this.updateVisibleOptions();
			}, 250);
		},

		updated() {
			if (this.hasVisibleOptions && this.isOpened) {
				if (this.needUpdateOptionsDirection) {
					this.needUpdateOptionsDirection = false;

					let inputCoords = this.$refs.input.getBoundingClientRect();
					let optionsCoords = this.$refs.options.getBoundingClientRect();

					this.optionsDirectionUp =
					/* 1 */ inputCoords.bottom + optionsCoords.height > window.innerHeight &&
					/* 2 */ window.innerHeight - inputCoords.bottom < inputCoords.top &&
					/* 3 */ window.pageYOffset + inputCoords.top - optionsCoords.height > 0;
				}

				if (this.needUpdateOptionsScroll) {
					this.needUpdateOptionsScroll = false;

					let optionsEl = this.$refs.options;
					let selectedOptionEl = optionsEl.childNodes[this.displaySelectedIndex];

					if (selectedOptionEl) {
						let optionsCoords = optionsEl.getBoundingClientRect();
						let selectedCoords = selectedOptionEl.getBoundingClientRect();

						if (selectedCoords.top < optionsCoords.top) {
							optionsEl.scrollTop -= optionsCoords.top - selectedCoords.top;
						} else if (selectedCoords.bottom > optionsCoords.bottom) {
							optionsEl.scrollTop += selectedCoords.bottom - optionsCoords.bottom;
						}
					} else if (this.needResetScroll) {
						this.needResetScroll = false;
						optionsEl.scrollTop = 0;
					}
				}
			}
		},

		watch: {

			selectedIndex() {
				let selectedOption = this.selectedOption;
				this.inputText = selectedOption ? selectedOption.title : '';
				this.optionsFiltered = false;
				this.onSelected();
			},

			displaySelectedIndex(displaySelectedIndex) {
				if (displaySelectedIndex !== null) {
					this.needUpdateOptionsScroll = true;
				}
			},

			isOpened(isOpened) {
				if (isOpened) {
					this.needUpdateOptionsDirection = true;
					this.needUpdateOptionsScroll = true;
					this.updateVisibleOptions();
				}
			}

		},

		computed: {

			inputAttribs() {
				return Object.assign({
					placeholder: this.placeholder,
					disabled: this.isDisabled,
					value: this.inputText,
					'aria-expanded': this.isOpened ? 'true' : 'false',
				}, this.defaultInputAttribs);
			},

			selectedOption() {
				return this.options[this.selectedIndex] || null;
			},

			value() {
				let selectedOption = this.selectedOption;
				return selectedOption ? selectedOption.value : null;
			}

		},

		methods: {

			onInput(event) {
				this.inputText = event.target.value;
				this.inputTextChanged();
			},

			onKeydown(event) {
				if (event.ctrlKey || event.altKey) {
					return;
				}
				switch (event.key) {
					case 'Up': // IE
					case 'ArrowUp':
						this.onArrow(-1);
						event.preventDefault();
						break;
					case 'Down': // IE
					case 'ArrowDown':
						this.onArrow(1);
						event.preventDefault();
						break;
					case 'Left': // IE
					case 'ArrowLeft':
						this.open();
						break;
					case 'Right': // IE
					case 'ArrowRight':
						if (!this.isOpened) {
							this.open();
						} else if (this.displaySelectedIndex !== null) {
							this.inputText = this.options[this.displaySelectedIndex].title;
						}
						break;
					case 'Enter':
						if (!this.isOpened) {
							this.open();
							this.updateDisplaySelectedIndex();
						} else if (this.displaySelectedIndex !== null) {
							this.selectedIndex = this.displaySelectedIndex;
							this.close();
						}
						event.preventDefault();
						break;
					case 'Esc': // IE
					case 'Escape':
						this.close();
						break;
				}
			},

			onBlur() {
				if (this.inputText === '') {
					this.selectedIndex = null;
				} else {
					let displaySelectedOption = this.options[this.displaySelectedIndex];
					if (displaySelectedOption && this.inputText === displaySelectedOption.title) {
						this.selectedIndex = this.displaySelectedIndex;
						this.optionsFiltered = false;
					} else {
						let selectedOption = this.selectedOption;
						if (selectedOption) {
							this.inputText = selectedOption.title;
							this.optionsFiltered = false;
						}
					}
				}
				this.close();
			},

			onClickOptions(event) {
				let optionEl = closest(event.target, '[data-index]');
				this.selectedIndex = parseInt(optionEl.getAttribute('data-index'));
				this.inputText = this.options[this.selectedIndex].title;
				this.optionsFiltered = false;
				this.close();
			},

			onMouseDownButton() {
				if (!this.isOpened) {
					this.open();
					this.$refs.input.focus();
				} else {
					this.close();
				}
			},

			onSelected() {
				this.$emit('selected', this.value);
				// console.log('selected', this.selectedIndex);
			},

			onArrow(offset) {
				if (this.displaySelectedIndex !== null) {
					let from = this.displaySelectedIndex + offset;
					let length = this.options.length;
					if (from >= 0 && from < length) {
						let to = (offset > 0) ? length : -1;
						for (let index = from; index !== to; index += offset) {
							let option = this.options[index];
							if (this.showAllOptions || option.isVisible) {
								this.displaySelectedIndex = index;
								break;
							}
						}
					}
				}
				this.open();
				this.updateDisplaySelectedIndex();
			},


			updateDisplaySelectedIndex() {
				if (!this.hasVisibleOptions) {
					this.displaySelectedIndex = null;
					return;
				}

				if (this.displaySelectedIndex === null) {
					this.displaySelectedIndex = 0;
				}

				if (!this.showAllOptions && !this.options[this.displaySelectedIndex].isVisible) {
					let newIndex = null;
					let length = this.options.length;
					for (let index = this.displaySelectedIndex + 1; index < length; index++) {
						let option = this.options[index];
						if (option.isVisible) {
							newIndex = index;
							break;
						}
					}
					if (newIndex === null) {
						for (let index = 0; index < length; index++) {
							let option = this.options[index];
							if (option.isVisible) {
								newIndex = index;
								break;
							}
						}
					}
					this.displaySelectedIndex = newIndex;
				}
			},

			getMatchPattern(text) {
				// search from word start
				return new RegExp('\(\^\|\[\^\\w\]\)\(' + escapeRegexp(text.toLowerCase()) + '\)', 'ig');
			},

			highlight(title) {
				if (this.showAllOptions) {
					return title;
				}
				const pattern = this.getMatchPattern(this.inputText);
				return title.replace(pattern, (...matches) => matches[1] + '<span class="highlight">' + matches[2] + '</span>');
			},
			
			updateVisibleOptions() {
				if (!this.isOpened) {
					return;
				}

				if (!this.optionsFiltered || this.inputText.length <= 2) {
					this.showAllOptions = true;
					return;
				}

				const pattern = this.getMatchPattern(this.inputText);
				let hasVisibleOptions = false;
				for (let option of this.options) {
					option.isVisible = pattern.test(option.title);
					if (option.isVisible) {
						hasVisibleOptions = true;
					}
				}
				this.hasVisibleOptions = hasVisibleOptions;
				this.showAllOptions = false;
				this.updateDisplaySelectedIndex();
			},

			setSelectedIndex(index) {
				this.selectedIndex = this.options[index] ? index : null;
			},

			setValue(value) { // not tested yet
				let newIndex = null;
				let length = this.options.length;
				for (let index = 0; index < length; index++) {
					let option = this.options[index];
					if (option.value === value) {
						newIndex = index;
						break;
					}
				}
				this.selectedIndex = newIndex;
			},

			setOptions(options) { // not tested yet
				options.forEach( option => {
					option.isVisible = true;
				});
				let value = this.value;
				this.options = options;
				this.setValue(value);
			},

			open() {
				if (!this.isOpened) {
					this.displaySelectedIndex = this.selectedIndex;
					this.isOpened = true;
				}
			},

			close() {
				this.isOpened = false;
			}

		}
	}
</script>
