<script setup>
import { computed, watch } from 'vue'

const props = defineProps({
  modelValue: {
    type: Boolean,
    default: false
  },
  position: {
    type: String,
    default: 'right',
    validator: (value) => ['left', 'right', 'top', 'bottom'].includes(value)
  },
  width: {
    type: String,
    default: '320px'
  },
  height: {
    type: String,
    default: '400px'
  },
  overlay: {
    type: Boolean,
    default: true
  },
  closeOnOverlayClick: {
    type: Boolean,
    default: true
  },
  title: {
    type: String,
    default: ''
  }
})

const emit = defineEmits(['update:modelValue', 'close'])

const isOpen = computed({
  get: () => props.modelValue,
  set: (value) => emit('update:modelValue', value)
})

const drawerClasses = computed(() => [
  'drawer',
  `drawer--${props.position}`,
  { 'drawer--open': isOpen.value, 'drawer--closed': !isOpen.value }
])

const drawerStyles = computed(() => {
  const styles = {}

  if (props.position === 'left' || props.position === 'right') {
    styles.width = props.width
    styles.height = '100%'
  } else {
    styles.height = props.height
    styles.width = '100%'
  }

  return styles
})

const closeDrawer = () => {
  isOpen.value = false
  emit('close')
}

const handleOverlayClick = () => {
  if (props.closeOnOverlayClick) {
    closeDrawer()
  }
}

// Prevent body scroll when drawer is open
watch(isOpen, (newValue) => {
  if (newValue) {
    document.body.style.overflow = 'hidden'
  } else {
    // Delay restoring scroll to allow transition to complete
    setTimeout(() => {
      document.body.style.overflow = ''
    }, 350)
  }
})
</script>

<template>
  <Teleport to="body" :disabled="false">
    <!-- Overlay with CSS transition only -->
    <div
      v-show="overlay && isOpen"
      class="drawer-overlay"
      @click="handleOverlayClick"
    />

    <!-- Drawer without Vue Transition - using CSS transitions only -->
    <div
      v-show="isOpen"
      :class="drawerClasses"
      :style="drawerStyles"
    >
        <div class="drawer__header">
          <div class="drawer__header-content" v-if="title || $slots.header">
            <slot name="header">
              <h3 class="drawer__title">{{ title }}</h3>
            </slot>
          </div>
          <button
            class="drawer__close"
            @click="closeDrawer"
            aria-label="Close drawer"
          >
            <svg width="24" height="24" viewBox="0 0 24 24" fill="none">
              <path
                d="M18 6L6 18M6 6L18 18"
                stroke="currentColor"
                stroke-width="2"
                stroke-linecap="round"
              />
            </svg>
          </button>
        </div>

        <div class="drawer__content">
          <slot />
        </div>

        <div class="drawer__footer" v-if="$slots.footer">
          <slot name="footer" />
        </div>
      </div>
  </Teleport>
</template>

<style scoped>
/* Overlay */
.drawer-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  z-index: 9998;
  transition: opacity 0.3s ease;
  opacity: 0;
}

.drawer-overlay[style*="display: block"],
.drawer-overlay:not([style*="display: none"]) {
  opacity: 1;
}

/* Drawer Base Styles */
.drawer {
  position: fixed;
  background: white;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.15);
  z-index: 9999;
  display: flex;
  flex-direction: column;
  transition: transform 0.3s ease, opacity 0.3s ease;
  opacity: 0;
  pointer-events: none;
}

.drawer--open {
  opacity: 1;
  pointer-events: auto;
}

/* Position Variants */
.drawer--left {
  top: 0;
  left: 0;
  bottom: 0;
  transform: translateX(-100%);
}

.drawer--left.drawer--open {
  transform: translateX(0);
}

.drawer--right {
  top: 0;
  right: 0;
  bottom: 0;
  transform: translateX(100%);
}

.drawer--right.drawer--open {
  transform: translateX(0);
}

.drawer--top {
  top: 0;
  left: 0;
  right: 0;
  transform: translateY(-100%);
}

.drawer--top.drawer--open {
  transform: translateY(0);
}

.drawer--bottom {
  bottom: 0;
  left: 0;
  right: 0;
  transform: translateY(100%);
}

.drawer--bottom.drawer--open {
  transform: translateY(0);
}

/* Header */
.drawer__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1rem 1.5rem;
  min-height: 60px;
  border-bottom: 1px solid #e5e7eb;
}

.drawer__header-content {
  flex: 1;
}

.drawer__title {
  margin: 0;
  font-size: 1.25rem;
  font-weight: 600;
  color: #1f2937;
}

.drawer__close {
  background: none;
  border: none;
  padding: 0.5rem;
  cursor: pointer;
  color: #6b7280;
  border-radius: 0.375rem;
  transition: all 0.2s ease;
  margin-left: auto;
}

.drawer__close:hover {
  background-color: #f3f4f6;
  color: #374151;
}

/* Content */
.drawer__content {
  flex: 1;
  padding: 1.5rem;
  overflow-y: auto;
}

/* Footer */
.drawer__footer {
  padding: 1rem 1.5rem;
  border-top: 1px solid #e5e7eb;
}

/* Overlay with CSS transition */
.drawer-overlay {
  transition: opacity 0.3s ease;
  opacity: 0;
}

.drawer-overlay:not([style*="display: none"]) {
  opacity: 1;
}

/* Responsive adjustments */
@media (max-width: 640px) {
  .drawer--left,
  .drawer--right {
    width: 100% !important;
  }

  .drawer__content {
    padding: 1rem;
  }

  .drawer__header {
    padding: 1rem;
  }

  .drawer__footer {
    padding: 1rem;
  }
}
</style>