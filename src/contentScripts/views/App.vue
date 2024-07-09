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

      settings: {
        enabled: false,
        rbdmin: 20, rbdreq: 30, slippage: 0.1,
      },
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

      if (this.settings.enabled && this.dmax < this.settings.rbdmin || this.dmin < this.settings.rbdmin)
        this.rebalance()
    },

    onPriceChange() {
      this.getBaseRange()
    },

    async getBaseRange() {
      this.status = 'Fetching base range'

      await this.sleep(1*1000)

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

    setSlippage() {
      [...document.querySelectorAll(`.bx--number__controls button[aria-label="${this.settings.slippage}"]`)].forEach(e => {
        e.click()
      })
    },
    async setRange(min, max) {
      await this.sleep(1*1000) // wait for UI
      let bmin = $('label:contains("Min price") + div input').last()
      let bmax = $('label:contains("Max price") + div input').last()

      bmin.val(min)
      bmax.val(max)
      bmin[0].dispatchEvent(new Event('change'))
      bmax[0].dispatchEvent(new Event('change'))
    },
    async tabSwitch(name) {
      $(`.bx--content-switcher button:contains("${name}")`).click()
      await this.sleep(1*1000) // wait for UI
    },

    async compound() {
      this.transact('Compounding', async () => {
        this.tabSwitch('Compound')
        this.setSlippage()

        let btn = $('.aerodrome-row .bx--btn:contains("Compound")').last()
        while (btn.prop('disabled')) if (this.status) await this.sleep(1*1000); else return
        btn.click()
      })
    },

    async harvest() {
      this.transact('Harvesting', async () => {
        this.tabSwitch('Harvest')
        this.setSlippage()

        let btn = $('.aerodrome-row .bx--btn:contains("Harvest")').last()
        while (btn.prop('disabled')) if (this.status) await this.sleep(1*1000); else return
        btn.click()
      })
    },

    async rebalance() {
      this.transact('Rebalancing', async () => {
        await this.tabSwitch('Rebalance')
        this.setSlippage()
        await this.setRange(-1, 1)

        let btn = $('.bx--btn:contains("Rebalance")').last()
        while (btn.prop('disabled')) if (this.status) await this.sleep(1*1000); else return
        btn.click()
      }, {retry: true})
    },

    cancel() {
      this.rabbyMsg('cancel')
      this.clearOp()
    },

    transact(status, cb, {retry}) {
      this.status = status
      cb()
      setTimeout(() => this.rabbyConfirm(resp => this.clearOp), 10*1000)

      this.opInt = setInterval(() => {
        let error = $('.bx--inline-notification__details:contains("Transaction will revert")')
        if (error.length) this.clearOp()
        if (retry) this.transact(status, cb, {retry})
      }, 1*1000)
    },

    flashStatus(status) {
      this.status = status
      setInterval(() => this.status = null, 3*1000)
    },

    clearOp() {
      this.status = null
      clearInterval(this.opInt)
    },

    async rabbyConfirm(cb) {
      if (!this.setting.enabled) return
      this.rabbyMsg('sign', async () => {
        await this.sleep(10*1000) // wait for UI
        this.rabbyMsg('confirm', cb)
      })
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
      <button class='btn btn-secondary' :disabled=status @click='harvest()' > Harvest </button>
    </div>

    <Range :min=rmin :max=rmax :dmin=dmin :reb=settings.rbdmin />

    <div id=settings class=card >
      <div class=card-body >
        <div class='d-flex flex-row' >
          <h5 class=card-title > Rebals </h5>
          <div class='form-check form-switch' >
            <input class=form-check-input type=checkbox role=switch v-model=settings.enabled >
          </div>
        </div>
        <Range v-for='rb in rebals' :min=rb.rmin :max=rb.rmax :dmin=rb.dmin :reb=settings.rbdmin >
          <div class='form-check form-switch d-inline-block' >
            <input class=form-check-input type=checkbox role=switch v-model=rb.enabled >
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

