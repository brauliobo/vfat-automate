<template>
  <div>
    <span> <slot /> </span>
    <input type=range disabled class='w-auto' min=0 max=100 :value=dmin list=values />
    <span v-if=dmin ><b>{{dmin}}%</b> in </span>
    <span v-if=min >{{min}} - {{max}}</span>

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
  props: ['min', 'max', 'dmin', 'reb'],
})
</script>

