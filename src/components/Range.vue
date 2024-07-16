<template>
  <div v-if=rb :style=rb.style >
    <span> <slot /> </span>
    <input type=range disabled class='w-auto' min=0 max=100 :value=rb.dmin list=values />
    <span v-if=rb.dmin ><b>{{rb.dmin}}%</b> in </span>
    <span v-if=rb.rmin >{{rb.rmin}} - {{rb.rmax}}</span>
    <span v-if=rb.rpct >&nbsp;(<b>{{rb.rpct}}%</b>) </span>

    <datalist id=values>
      <option :value=reb       label='rebal down' />
      <option :value='100-reb' label='rebal up' />
    </datalist>
  </div>
</template>

<script>
import { defineComponent } from 'vue'
export default defineComponent({
  name: 'Range',
  props: ['rb', 'reb'],
})
</script>

