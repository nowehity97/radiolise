<template>
  <div class="flex cursor-pointer" @click="toggleChecked()">
    <div class="w-6.25 text-xl">
      <transition
        mode="out-in"
        enter-class="scale-x-[0.8] scale-y-[0.7]"
        leave-to-class="scale-x-[0.8] scale-y-[0.7]"
        enter-active-class="transition-transform duration-75"
        leave-active-class="transition-transform duration-75"
      >
        <FasCheckSquare v-if="checked" key="checked" />
        <FarSquare v-else key="notChecked" class="opacity-70" />
      </transition>
    </div>
    <div class="flex-1 pt-0.75">
      <strong v-if="strongHeading"><slot></slot></strong>
      <slot v-else></slot>
      <template v-if="hasDescription">
        <br /><span class="description"><slot name="description"></slot></span>
      </template>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Emit, Model, Prop, Vue } from "vue-property-decorator";

@Component
export default class RadCheck extends Vue {
  @Model("change", { type: Boolean, default: false })
  readonly checked!: boolean;

  @Prop({ type: Boolean, default: false }) readonly setting!: boolean;

  get hasDescription(): boolean {
    return this.$slots.description !== undefined;
  }

  get strongHeading(): boolean {
    return this.hasDescription || this.setting;
  }

  @Emit("change")
  toggleChecked(): boolean {
    return !this.checked;
  }
}
</script>
