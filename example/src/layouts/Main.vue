<template>
  <div class="d-flex flex-column vh-100">
    <div class="navbar flex-shrink-0 border-bottom" style="justify-content: center;">
     <h1>TiDB Visual Plan</h1>
    </div>
    <slot></slot>
  </div>
</template>

<script>
  import VLink from '../components/VLink.vue'
  export default {
    components: {
      VLink
    }
  }
</script>
