<template>
  <div
    :contenteditable="editable !== false"
    :disabled="editable === false"
    @keydown="onKeyDown"
    @input="changed = true"
    @focus="onFocus"
    @blur="onBlur"
    @click="onClick"
    @wheel="onWheel"
  ></div>
</template>

<script lang="ts">
import { Component, Vue, Watch } from 'vue-property-decorator'

import { countDecimals } from '../../utils/helpers'
@Component({
  name: 'Editable',
  props: ['content', 'step', 'min', 'max', 'editable', 'disabled']
})
export default class extends Vue {
  private content: string
  private min: number
  private max: number
  private disabled: boolean
  private clickAt: number
  private changed = false

  mounted() {
    const el = this.$el as HTMLElement

    el.innerText = this.content
  }

  @Watch('content')
  onContentChange() {
    if ((this.$el as HTMLElement).innerText !== this.content) {
      ;(this.$el as HTMLElement).innerText = this.content
    }
  }

  selectAll() {
    window.setTimeout(() => {
      let sel: Selection
      let range: Range

      if (window.getSelection && document.createRange) {
        range = document.createRange()
        range.selectNodeContents(this.$el)
        sel = window.getSelection()
        sel.removeAllRanges()
        sel.addRange(range)
      } else if ((document.body as any).createTextRange) {
        range = (document.body as any).createTextRange()
        ;(range as any).moveToElementText(this.$el)
        ;(range as any).select()
      }
    }, 1)
  }

  onBlur(event) {
    if (event.which === 13 && !isNaN(event.target.innerText)) {
      event.preventDefault()
      return
    }

    if (this.changed) {
      event.target.innerHTML = event.target.innerText
      this.$emit('output', event.target.innerText)
    }

    if (window.getSelection) {
      window.getSelection().removeAllRanges()
    } else if ((document as any).selection) {
      ;(document as any).selection.empty()
    }
  }

  onKeyDown(event) {
    if (this.disabled || event.which === 13) {
      event.preventDefault()
      ;(this.$el as HTMLInputElement).blur()

      event.target.innerText = this.content

      return
    }

    if (event.which === 38 || event.which === 40) {
      this.increment((event.which === 40 ? 1 : -1) * (event.shiftKey ? 10 : 1))
    }
  }

  onFocus() {
    this.changed = false
  }

  onClick() {
    const now = Date.now()

    if (this.clickAt && now - this.clickAt < 150) {
      this.selectAll()
    }

    this.clickAt = now
  }

  increment(direction: number) {
    const text = (this.$el as HTMLElement).innerText.replace(/[^0-9-.]/g, '')

    if (isNaN(text as any)) {
      return
    }

    const max = typeof this.max === 'undefined' ? Infinity : this.max
    const min = typeof this.min === 'undefined' ? -Infinity : this.min
    const precision = countDecimals(text)
    const step = 1 / Math.pow(10, precision)
    const change = step * direction * -1

    this.$emit('output', Math.max(min, Math.min(max, +text + change)).toFixed(precision))
  }

  onWheel(event) {
    const focusedElement = document.activeElement as HTMLElement

    if (focusedElement !== event.target || !focusedElement.isContentEditable) {
      return
    }

    event.preventDefault()

    this.increment(Math.sign(event.deltaY) * (event.shiftKey ? 10 : 1))
  }
}
</script>
