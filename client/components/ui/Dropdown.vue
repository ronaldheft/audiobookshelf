<template>
  <div class="relative w-full" v-click-outside="clickOutsideObj">
    <p class="text-sm font-semibold px-1" :class="disabled ? 'text-gray-300' : ''">{{ label }}</p>
    <button type="button" :disabled="disabled" class="relative w-full border rounded shadow-sm pl-3 pr-8 py-2 text-left focus:outline-none sm:text-sm" :class="buttonClass" aria-haspopup="listbox" aria-expanded="true" @click.stop.prevent="clickShowMenu">
      <span class="flex items-center">
        <span class="block truncate" :class="small ? 'text-sm' : ''">{{ selectedText }}</span>
      </span>
      <span class="ml-3 absolute inset-y-0 right-0 flex items-center pr-2 pointer-events-none">
        <span class="material-icons">expand_more</span>
      </span>
    </button>

    <transition name="menu">
      <ul v-show="showMenu" class="absolute z-10 -mt-px w-full bg-primary border border-black-200 shadow-lg max-h-56 rounded-b-md py-1 ring-1 ring-black ring-opacity-5 overflow-auto focus:outline-none sm:text-sm" tabindex="-1" role="listbox">
        <template v-for="item in items">
          <li :key="item.value" class="text-gray-100 select-none relative py-2 cursor-pointer hover:bg-black-400" id="listbox-option-0" role="option" @click="clickedOption(item.value)">
            <div class="flex items-center">
              <span class="font-normal ml-3 block truncate font-sans text-sm">{{ item.text }}</span>
            </div>
          </li>
        </template>
      </ul>
    </transition>
  </div>
</template>

<script>
export default {
  props: {
    value: [String, Number],
    label: {
      type: String,
      default: ''
    },
    items: {
      type: Array,
      default: () => []
    },
    disabled: Boolean,
    small: Boolean
  },
  data() {
    return {
      clickOutsideObj: {
        handler: this.clickedOutside,
        events: ['mousedown'],
        isActive: true
      },
      showMenu: false
    }
  },
  computed: {
    selected: {
      get() {
        return this.value
      },
      set(val) {
        this.$emit('input', val)
      }
    },
    selectedItem() {
      return this.items.find((i) => i.value === this.selected)
    },
    selectedText() {
      return this.selectedItem ? this.selectedItem.text : ''
    },
    buttonClass() {
      var classes = []
      if (this.small) classes.push('h-9')
      else classes.push('h-10')

      if (this.disabled) classes.push('cursor-not-allowed border-gray-600 bg-primary bg-opacity-70 border-opacity-70 text-gray-400')
      else classes.push('cursor-pointer border-gray-600 bg-primary text-gray-100')

      return classes.join(' ')
    }
  },
  methods: {
    clickShowMenu() {
      if (this.disabled) return
      this.showMenu = !this.showMenu
    },
    clickedOutside() {
      this.showMenu = false
    },
    clickedOption(itemValue) {
      this.selected = itemValue
      this.showMenu = false
    }
  },
  mounted() {}
}
</script>