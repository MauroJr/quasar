<template>
  <div
    class="timeline-item"
    :class="{incomplete: step > stepper.currentStep}"
  >
    <div class="timeline-badge">
      <i v-show="step < stepper.currentStep">
        done
      </i>
      <span v-show="step >= stepper.currentStep">
        {{ step }}
      </span>
    </div>
    <div class="timeline-title text-bold" v-html="title"></div>
    <quasar-transition name="slide">
      <div
        class="timeline-content"
        v-show="stepper && step === stepper.currentStep"
      >
        <slot></slot>
        <div class="group" style="margin-top: 1rem; margin-left: -5px;">
          <button
            class="primary"
            :class="{disabled: !ready}"
            @click="nextStep()"
          >
            {{ stepper && step === stepper.steps ? 'Finish' : 'Continue' }}
          </button>
          <button
            class="primary clear"
            v-if="step > 1"
            @click="previousStep()"
          >
            Back
          </button>
        </div>
      </div>
    </quasar-transition>
  </div>
</template>

<script>
export default {
  props: {
    title: {
      type: String,
      required: true
    },
    ready: {
      type: Boolean,
      default: true
    }
  },
  data () {
    return {
      step: -1
    }
  },
  computed: {
    stepper () {
      return this.$parent.config
    }
  },
  methods: {
    nextStep () {
      if (this.ready) {
        this.$parent.nextStep()
      }
    },
    previousStep () {
      this.$parent.previousStep()
    },
    finish () {
      this.$parent.finish()
    }
  }
}
</script>
