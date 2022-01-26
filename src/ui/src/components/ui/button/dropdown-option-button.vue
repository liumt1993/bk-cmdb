<template>
  <div
    :class="['dropdown-option-button', { disabled: disabled }]"
    @click="handleClick"
  >
    <slot></slot>
  </div>
</template>

<script>
  export default {
    name: 'link-button',
    props: {
      disabled: Boolean,
    },
    methods: {
      handleClick() {
        if (!this.disabled) {
          this.$emit('click')
        }
      },
    },
  }
</script>

<style lang="scss" scoped>
.dropdown-option-button {
  height: 32px;
  line-height: 32px;
  color: #63656E;
  padding-left: 12px;
  font-size: 12px;

  &:hover {
    background-color: #e1ecff;
    color: #3A84FF;
  }

  &.disabled {
    color: $textDisabledColor;
    cursor: not-allowed;
  }
}
</style>
