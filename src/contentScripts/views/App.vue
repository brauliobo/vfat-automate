<script>
import Range from '/components/Range.vue'
import '../style.css'
import $ from 'jquery' // for :constains selector

export default {

  data() {
    return {
      enabled: true,
      status: null, opInt: null,
      rabby_id: 'jcnpbgidgmloifpjgiodolmobcjjpkno',

      //                       price                                 rmin   - rmax             rpct
      // 'Current pool price is 3430.3, your position price range is 3401.6 - 3470.3 and it is 2% wide.'
      price: null,
      rmin: null, rmax: null, rpct: null, range: null,
      dmin: null, dmax: null,

      bmin: null, bmax: null, brange: null, bdist: null,

      settings: { rbdmin: 20, rbdreq: 30, slippage: 0.1, },
      rebals: [ 
        {enabled: true, pmin:  0, pmax: 0, rmin: null, rmax: null, range: null, dmin: null},
        {enabled: true, pmin:  0, pmax: 1, rmin: null, rmax: null, range: null, dmin: null},
        {enabled: true, pmin: -1, pmax: 0, rmin: null, rmax: null, range: null, dmin: null},
        {enabled: true, pmin: -1, pmax: 1, rmin: null, rmax: null, range: null, dmin: null},
      ],
    }
  },

  methods: {

    calcDist(diff, range) {
      range = range || this.range
      return Math.round(100 * (Math.abs(diff) / range))
    },

    reload() {
      let el = document.querySelector('p.bx--inline-notification__title') 
      if (!el) return document.querySelector('tr:not(.bx--expandable-row) .symbol')?.click()

      let m = el.innerText.match(/is ([\d,.]+).+is ([\d,.]+) - ([\d,\.]+).*is (\d+)% wide/)
      if (!m) return
      let [v,rmin,rmax,rpct] = m.slice(1,5).map((v) => parseFloat(v.replace(/,/g, '')))
      this.rmin = rmin
      this.rmax = rmax
      this.rpct = rpct
      this.range = rmax-rmin
      if (v != this.price) this.onPriceChange()
      this.price = v
      if (this.price < this.rmin || this.price > this.rmax)
        return

      this.dmax = this.calcDist(rmax-v)
      this.dmin = this.calcDist(v-rmin)

      //this.setSlippage()

      //new Notification('test', {body: Min ${100*(1-v/rmin)}%, Max ${100*(1-rmax/v)}%})

      //if (this.dmax < this.settings.rbdmin || this.dmin < this.settings.rbdmin)
        //this.rebalance()
    },

    setSlippage() {
      [...document.querySelectorAll(`.bx--number__controls button[aria-label="${this.settings.slippage}"]`)].forEach(e => {
        e.click()
      })
    },

    onPriceChange() {
      this.getBaseRange()
    },

    async getBaseRange() {
      this.status = 'Fetching base range'

      await this.sleep(1*1000)

      //this.pmin = parseInt($('label:contains("Min price")').val())
      //this.pmax = parseInt($('label:contains("Max price")').val())

      this.brange = this.range / this.rpct
      this.bmin = this.brange * 100
      if (this.bmin > this.price) this.bmin -= this.brange
      this.bmax = this.bmin + this.brange

      this.rebals.forEach(rb => {
        rb.rmin = (this.bmin + rb.pmin*this.brange).toFixed(1)
        rb.rmax = (this.bmax + rb.pmax*this.brange).toFixed(1)
        rb.range = rb.rmax - rb.rmin
        rb.dmin = this.calcDist(this.price-rb.rmin, rb.range)
      })

      this.status = null
    },

    updRebals() {
    },

    async compound() {
      document.querySelector('.bx--content-switcher button:nth-child(4)').click()
      await this.sleep(1*1000) // wait for UI
      let btn = $('.aerodrome-row .bx--btn:contains("Compound")').last()
      while (btn.prop('disabled')) await this.sleep(1*1000)
      btn.click()
      this.confirm('Compounding')
    },

    async rebalance() {
      document.querySelector('.bx--content-switcher button:nth-child(2)').click()
      await this.sleep(1*1000) // wait for UI
      let btn = document.querySelector('.bx--expandable-row button[depositamount]')
      while (btn.prop('disabled')) await this.sleep(1*1000)
      btn.click()
      this.confirm('Rebalancing')
    },

    cancel() {
      this.rabbyMsg('cancel')
      this.clearOp()
    },

    confirm(status) {
      this.status = status
      this.opInt  = setInterval(() => this.rabbyConfirm(resp => this.clearOp), 1*1000)
    },

    flashStatus(status) {
      this.status = status
      setInterval(() => this.status = null, 3*1000)
    },

    clearOp() {
      this.status = null
      clearInterval(this.opInt)
    },

    rabbyConfirm(cb) {
      this.rabbyMsg('sign', () => this.rabbyMsg('confirm', cb))
    },
    rabbyMsg(op, cb) {
      chrome.runtime.sendMessage(this.rabby_id, {type: `rabby_${op}`})
        .then(cb)
        .catch(error => {})
    },

    sleep(ms) {
      return new Promise(res => setTimeout(res, ms))
    },
    
  },

  mounted() {
    setInterval(this.reload, 1*1000)
  },
}

</script>

<template>
  <div data-bs-theme=dark class='card position-fixed start-0 bottom-0 m-5 z-3'>
    <div class=row v-if=status >
      <b>Status:</b> {{status}}
      <button @click='cancel()'> Cancel </button>
    </div>

    <div class=btn-group >
      <button class='btn btn-secondary' :disabled=status @click='compound()'  > Compound  </button>
      <button class='btn btn-secondary' :disabled=status @click='rebalance()' > Rebalance </button>
    </div>

    <Range :min=rmin :max=rmax :dmin=dmin :reb=settings.rbdmin />

    <div id=settings class=card >
      <div class=card-body >
        <h5 class=card-title > Rebals </h5>
          <Range v-for='rb in rebals' :min=rb.rmin :max=rb.rmax :dmin=rb.dmin :reb=settings.rbdmin >
            <div class='form-check d-inline-block' >
              <input class=form-check-input type=checkbox role=switch :checked=rb.enabled >
              <label class=form-check-label> <b> {{rb.pmin}}/{{rb.pmax}} </b> </label>
            </div>
          </Range>

      </div>
    </div>

    <div id=settings class=card >
      <div class=card-body >
        <h5 class=card-title > Settings </h5>
        <div> Rebalance Trigger %: <input type=number min=1 max=40 v-model=settings.rbdmin /> </div>
        <div> Rebalance Acceptance %: <input type=number :min=settings.rbdmin max=40 v-model=settings.rbdreq /> </div>
        <div> Slippage: <input type=number min=0.1 max=3 v-model=settings.slippage /> </div>
      </div>
    </div>
  </div>
</template>

