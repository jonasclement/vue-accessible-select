<template lang="pug">
.v-select(:class="className")
  span.v-select__label(
    v-if="hasSlot('label') || label"
    :id="labelId"
    )
    slot(name="label") {{ label }}:
  .v-select__inner
    button.v-select__btn(
      :id="buttonId"
      ref="button"
      :disabled="disabled"
      :aria-expanded="ariaExpanded"
      :aria-labelledby="`${labelId ? labelId : ''} ${buttonId}`"
      type="button"
      aria-haspopup="listbox"
      @mousedown="toggle"
      @keydown="keydownHandler"
      @blur="buttonBlurHandler"
      )
      span.v-select__prepend(v-if="hasSlot('prepend')")
        slot(name="prepend")
      span.v-select__placeholder(v-if="(placeholder || hasSlot('placeholder')) && value === undefined || !optionsHasValue")
        slot(
          name="placeholder"
          :placeholder="placeholder"
          ) {{ placeholder }}
      span.v-select__selected
        slot(
          v-if="value !== undefined && optionsHasValue"
          :value="value"
          :option="currentOption"
          name="selected"
          ) {{ currentOption ? currentOption.label : value }}
      span.v-select__arrow
        slot(name="arrow")
          svg(viewBox="0 0 255 255")
            path(d="M0 64l128 127L255 64z")
    transition(
      :name="transition ? transition.name : ''"
      :mode="transition ? transition.mode : ''"
      )
      .v-select__menu(v-if="open")
        ul.v-select__list(
          v-if="options && options.length"
          ref="list"
          :aria-activedescendant="value && getOptionId(value)"
          :aria-labelledby="labelId"
          role="listbox"
          tabindex="-1"
          @keydown="keydownHandler"
          @keyup.end="setLastSelected"
          @keyup.home="setFirstSelected"
          @keyup.esc="escapeHandler"
          @blur="menuBlurHandler"
          )
          li.v-select__option(
            v-for="(option, index) in options" :key="index"
            :id="getOptionId(option)" role="option"
            ref="options"
            :class="{ 'v-select__option--selected': isSelected(option) }"
            @mousedown="clickHandler(option)" :aria-selected="isSelected(option) ? 'true': 'false'"
            )
            slot(
              name="option"
              :option="option"
              :value="value"
              )
              span {{ option.label }}
        template(v-else)
          .v-select__no-options
            slot(name="no-optioms")
              span No options provided
</template>

<script>
import {
  KEY_RETURN,
  KEY_SPACE,
  KEY_LEFT,
  KEY_UP,
  KEY_RIGHT,
  KEY_DOWN,
  KEY_ESCAPE,
} from 'keycode-js'

import config from '../../config'
import { getCurrentInstance } from 'vue'

export default {
  name: 'VueAccessibleSelect',
  props: {
    options: {
      type: Array,
      required: true,
    },
    value: {
      required: true,
    },
    transition: {
      type: Object,
      default: () => config.transition || {},
    },
    label: String,
    placeholder: String,
    disabled: Boolean,
  },
  emits: ['change', 'open', 'close', 'input'],
  data() {
    const _vm = getCurrentInstance()
    return {
      open: false,
      timeout: null,
      printedText: '',
      localId_: _vm.uid,
      _vm,
    }
  },
  computed: {
    labelId() {
      return this.label ? `v-select-label-${this.localId_}` : false
    },
    buttonId() {
      return `v-select-button-${this.localId_}`
    },
    ariaExpanded() {
      return this.open ? 'true' : 'false'
    },
    className() {
      return {
        'v-select--opened': this.open,
        'v-select--disabled': this.disabled,
      }
    },
    currentOption() {
      return (
        Array.isArray(this.options) &&
        this.options.find((option) => option.value === this.value)
      )
    },
    currentOptionIndex() {
      return this.options.findIndex((option) => option === this.currentOption)
    },
    optionsHasValue() {
      return (
        this.options.findIndex((option) => option.value === this.value) !== -1
      )
    },
  },
  watch: {
    open(val) {
      if (val) {
        setTimeout(() => {
          document.addEventListener('click', this.clickListener)

          // if list is present in DOM
          const { list } = this.$refs

          if (list) {
            list.focus()

            // * if option is selected, then scroll to it
            const { value } = this
            if (value || value === 0) {
              this.scrollToSelected()
            }
          }

          this.$emit('open')
        }, 0)
      } else {
        document.removeEventListener('click', this.clickListener)
        this.$emit('close')
      }
    },
    value(val) {
      // * if option is selected, then scroll to it
      if (val || val == 0) {
        this.scrollToSelected()
      }
    },
  },
  /*
   * SSR Safe Client Side ID attribute generation
   * id's can only be generated client side, after mount.
   * this.uid is not synched between server and client.
   */
  mounted() {
    this.$nextTick(() => {
      // Update dom with auto ID after dom loaded to prevent
      // SSR hydration errors.
      this.localId_ = this._vm.uid
    })
  },
  methods: {
    toggle() {
      this.open = !this.open
    },
    emit(val) {
      this.$emit('input', val)
      this.$emit('change', val)
    },
    clickListener(e) {
      const { target } = e
      const closestBtn = target.closest('.v-select__btn')
      const closestList = target.closest('.v-select__list')

      if (closestList === this.$refs.list || closestBtn === this.$refs.button) {
      } else {
        this.open = false
      }
    },
    isSelected(option) {
      return this.value === option.value
    },
    clickHandler(option) {
      this.emit(option.value)
      this.open = false
    },
    async keydownHandler(e) {
      // ESC is handled by escapeHandler()
      if (e.keyCode === KEY_ESCAPE) {
        return
      }

      // prevent from default scrolling
      if (
        [KEY_SPACE, KEY_LEFT, KEY_UP, KEY_RIGHT, KEY_DOWN].indexOf(e.keyCode) >
        -1
      ) {
        e.preventDefault()
      }

      const { currentOptionIndex } = this

      switch (e.keyCode) {
        case KEY_SPACE:
          this.open = true
          break
        case KEY_UP:
          if (currentOptionIndex !== 0)
            this.emit(this.options[currentOptionIndex - 1].value)
          return
        case KEY_DOWN:
          if (currentOptionIndex !== this.options.length - 1)
            this.emit(this.options[currentOptionIndex + 1].value)
          return
        case KEY_RETURN:
          this.toggle()
          await this.$nextTick()
          if (!this.open) this.$refs.button.focus()
      }

      this.printHandler(e)
    },
    getOptionId(option) {
      return `v-select-option-${this.options.indexOf(option)}_${this.localId_}`
    },
    setFirstSelected() {
      this.emit(this.options[0].value)
    },
    setLastSelected() {
      this.emit(this.options[this.options.length - 1].value)
    },
    escapeHandler() {
      this.open = false
      this.$refs.button.focus()
    },
    printHandler(e) {
      const newChar = e.key ? e.key : String.fromCharCode(e.keyCode)
      if (newChar.length > 1) return

      this.printedText += newChar.toUpperCase()

      this.selectByText(this.printedText)

      clearTimeout(this.timeout)
      this.timeout = null

      this.timeout = setTimeout(() => {
        this.printedText = ''
      }, 500)
    },
    selectByText(text) {
      for (let option of this.shiftOptions(this.currentOptionIndex)) {
        if (!String(option.label).toUpperCase().startsWith(text)) continue

        if (!this.currentOption) {
          this.emit(option.value)
          return
        }

        if (this.currentOption.value !== option.value) {
          this.emit(option.value)
          return
        }
      }
    },
    scrollToSelected() {
      // * get current option DOM node
      if (Array.isArray(this.$refs.options)) {
        const currentOption = this.$refs.options[this.currentOptionIndex]

        if (currentOption) {
          const { offsetTop, clientHeight } = currentOption

          const { list } = this.$refs

          const currentVisibleArea = list.scrollTop + list.clientHeight

          if (offsetTop < list.scrollTop) {
            list.scrollTop = offsetTop
          } else if (offsetTop + clientHeight > currentVisibleArea) {
            list.scrollTop = offsetTop - list.clientHeight + clientHeight
          }
        }
      }
    },
    buttonBlurHandler(e) {
      let target = e.relatedTarget
      if (target === null) {
        target = document.activeElement
      }
      if (target !== this.$refs.list && this.open) {
        this.open = false
      }
    },
    menuBlurHandler(e) {
      let target = e.relatedTarget
      if (target === null) {
        target = document.activeElement
      }
      if (target !== this.$refs.button) {
        this.open = false
      }
    },
    hasSlot(name) {
      return Boolean(this.$slots[name])
    },
    shiftOptions(selectedIndex) {
      const { options } = this

      if (selectedIndex === -1 || this.timeout) {
        return options
      }

      const optionsBeforeSelected = options.slice(0, selectedIndex - 1)
      const optionsAfterSelected = options.slice(
        selectedIndex + 1,
        options.length
      )

      return [...optionsAfterSelected, ...optionsBeforeSelected]
    },
  },
}
</script>
