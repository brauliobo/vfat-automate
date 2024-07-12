<template>
  <div data-bs-theme=dark class='card position-fixed start-0 bottom-0 m-5 z-3'>
    <div class='card-body' v-if=status >
      <b>Status: </b>
      <span>{{status}}</span>
      <span v-if=opstatus > - {{opstatus}}</span>
      <button class='btn' @click='cancel()'> Cancel </button>
    </div>

    <div class=btn-group >
      <button class='btn btn-secondary' :disabled=status @click=compound()  > Compound  </button>
      <button class='btn btn-secondary' :disabled=status @click=rebalance() > Rebalance </button>
      <button class='btn btn-secondary' :disabled=status @click=harvest() > Harvest </button>
    </div>

    <Range :rb=cr :reb=settings.rbdthd />

    <div id=settings class=card >
      <div class=card-body >
        <div class='d-flex flex-row' >
          <h5 class=card-title > Rebals </h5>
          <div class='form-check form-switch' >
            <input class=form-check-input type=checkbox role=switch v-model=settings.enabled @change=reload() >
          </div>
        </div>
        <Range v-for='(rb,i) in rebals' :rb=rb :reb=settings.rbdthd >
          <div class='form-check form-switch d-inline-block' >
            <input class=form-check-input type=checkbox role=switch v-model=settings.rebals[i].enabled >
            <label class=form-check-label> <b> {{rb.pmin}}/{{rb.pmax}} </b> </label>
          </div>
        </Range>

      </div>
    </div>

    <div id=settings class=card >
      <div class=card-body >
        <h5 class=card-title > Settings </h5>
        <div> Rebalance Trigger %:
          <input type=number min=1 max=40 v-model=settings.rbdthd @change=reload() />
          <span> current price var: {{round(cr.range * settings.rbdthd/100)}}</span>
        </div>
        <div> Rebalance Acceptance %: <input type=number :min=settings.rbdthd max=40 v-model=settings.rbdreq /> </div>
        <div> Slippage: <input type=number min=0.1 max=3 v-model=settings.slippage /> </div>
      </div>
    </div>
  </div>
</template>

<script>
import Range from '/components/Range.vue'
import '../style.css'
import $ from 'jquery' // for :constains selector
import _ from 'lodash'

const RABBY_EXTID = 'jcnpbgidgmloifpjgiodolmobcjjpkno'

export default {

  data() {
    return {
      status: null, opstatus: null,
      opInt: null, rabby_op: null,

      //                       price                                 rmin   - rmax             rpct
      // 'Current pool price is 3430.3, your position price range is 3401.6 - 3470.3 and it is 2% wide.'
      price: null,
      cr: {rmin: null, rmax: null, rpct: null, range: null, dmin: null, dmax: null,},

      br: {rmin: null, rmax: null, range: null,},

      settings: {
        enabled: false,
        cpmin: -1, cpmax: 1,
        rbdthd: 20, rbdreq: 30, slippage: 0.1,
        rebals: [ 
          {enabled: true},
          {enabled: true},
          {enabled: true},
          {enabled: true},
        ],
      },

      rebals: [
        {pmin:  0, pmax: 0},
        {pmin:  0, pmax: 1},
        {pmin: -1, pmax: 0},
        {pmin: -1, pmax: 1},
      ],
    }
  },

  watch: {
    settings: { deep: true, handler(newVal) { this.saveSettings() } },
  },

  methods: {

    async loadSettings() {
      let { ['vfat_automate']: value } = await chrome.storage.local.get(['vfat_automate'])
      if (value?.rebals) value.rebals = Object.values(value.rebals) 
      this.settings = { ...this.settings, ...value }
    },
    async saveSettings() {
      await chrome.storage.local.set({ ['vfat_automate']: this.settings })
    },

    calcDist(diff, range) {
      range = range || this.cr.range
      return Math.round(100 * (Math.abs(diff) / range))
    },
    distThd(dmin) {
      return Math.max(this.settings.rbdthd - dmin, 100-this.settings.rbdthd - dmin)
    },

    reload() {
      if (this.status) return // can't change params during OP

      let el = document.querySelector('p.bx--inline-notification__title') 
      if (!el) return document.querySelector('tr:not(.bx--expandable-row) .symbol')?.click()

      let m = el.innerText.match(/is ([\d,.]+).+is ([\d,.]+) - ([\d,\.]+).*is (\d+)% wide/)
      if (!m) return
      let [v,rmin,rmax,rpct] = m.slice(1,5).map(this.parseNum)
      this.cr.rmin = rmin
      this.cr.rmax = rmax
      this.cr.rpct = rpct
      this.cr.range = rmax-rmin
      if (v != this.price) this.onPriceChange()
      this.price = v
      if (this.price < this.cr.rmin || this.price > this.cr.rmax)
        return

      this.findBestRebal()

      this.cr.dmax = this.calcDist(rmax-v)
      this.cr.dmin = this.calcDist(v-rmin)

      if (!this.settings.enabled) return
      if (this.cr.dmin <= this.settings.rbdthd || this.cr.dmax <= this.settings.rbdthd)
        this.rebalance()
    },

    onPriceChange() {
      this.getBaseRange()
    },

    findBestRebal() {
      this.rebals.forEach(rb => rb.best = rb.style = null)
      let br = _.maxBy(this.rebals, rb => { if (this.settings.rebals[rb.i]?.enabled) return rb.dthd })
      if (!br) return
      br.style = 'outline: 1px solid #FFD700'
      br.best  = true
      return br
    },

    async getBaseRange() {
      if (this.status) return
      this.status = 'Fetching base range'

      await this.setRange(0, 0)
      await this.sleep(1)
      let txt = document.querySelector('.bx--toast-notification').innerText
      if (txt.includes('No rebalance needed')) {
        this.br.rmin = this.cr.rmin
        this.br.rmax = this.cr.rmax
      } else {
        let [brmin, brmax] = txt.match(/range ([\d,.]+) - ([\d,\.]+)\./).slice(1,3).map(this.parseNum)
        this.br.rmin = brmin
        this.br.rmax = brmax
      }
      this.br.range = this.cr.rmax / 100 // br.rmax - br.rmin is less precise

      this.rebals.forEach((rb, i) => {
        rb.i     = i
        rb.rmin  = this.round(this.br.rmin + rb.pmin*this.br.range)
        rb.rmax  = this.round(this.br.rmax + rb.pmax*this.br.range)
        rb.range = rb.rmax - rb.rmin
        rb.dmin  = this.calcDist(this.price-rb.rmin, rb.range)
        rb.dthd  = this.distThd(rb.dmin)
        rb.style = null
        rb.best  = false
      })

      await this.setRange(this.settings.cpmin, this.settings.cpmax) //revert back
      this.status = null
    },

    setSlippage() {
      [...document.querySelectorAll(`.bx--number__controls button[aria-label="${this.settings.slippage}"]`)].forEach(e => {
        e.click()
      })
    },

    async getRange() {
      await this.tabSwitch('Rebalance')
      await this.sleep(1) // wait for UI
      this.settings.cpmin = $('label:contains("Min price") + div input').last().val()
      this.settings.cpmax = $('label:contains("Max price") + div input').last().val()
    },

    async setRange(min, max) {
      await this.tabSwitch('Rebalance')
      await this.sleep(1) // wait for UI
      let pmin = $('label:contains("Min price") + div input').last()
      let pmax = $('label:contains("Max price") + div input').last()

      pmin.val(min)
      pmax.val(max)
      pmin[0].dispatchEvent(new Event('change'))
      pmax[0].dispatchEvent(new Event('change'))
    },
    async tabSwitch(name) {
      $(`.bx--content-switcher button:contains("${name}")`).click()
      await this.sleep(1) // wait for UI
    },

    async compound() {
      this.transact('Compounding', async () => {
        this.tabSwitch('Compound')
        this.setSlippage()

        let btn = $('.bx--btn:contains("Compound")').last()
        while (btn.prop('disabled')) if (this.status) await this.sleep(1); else return
        btn.click()
      })
    },

    async harvest() {
      this.transact('Harvesting', async () => {
        this.tabSwitch('Harvest')
        this.setSlippage()

        let btn = $('.bx--btn:contains("Harvest")').last()
        while (btn.prop('disabled')) if (this.status) await this.sleep(1); else return
        btn.click()
      })
    },

    async rebalance() {
      this.transact('Rebalancing', async () => {
        let brb = this.findBestRebal()
        await this.setRange(brb.pmin, brb.pmax)
        this.setSlippage()

        let btn = $('.bx--btn:contains("Rebalance")').last()
        while (btn.prop('disabled')) if (this.status) await this.sleep(1); else return
        btn.click()
      })
    },

    cancel() {
      this.rabby_op = 'cancel'
      this.rabbyMsg()
      this.clearOp()
    },

    async opTrack(cb, {retry = true} = {}) {
      this.opstatus = 'Waiting for route'
      await cb()
      this.opstatus = 'Waiting for Rabby'
      await this.sleep(14)
      this.rabbyConfirm()
      this.opstatus = 'Waiting status'
      await this.sleep(14)
      if (document.querySelector('.bx--inline-notification--error')) {
        if (retry) return // keep interval active
        else this.clearOp()
      } else if (document.querySelector('.bx--inline-notification--success'))
        this.clearOp()
    },

    transact(status, cb) {
      this.clearOp()
      this.status = status
      this.opTrack(cb)
      this.opInt = setInterval(() => this.opTrack(cb), 30*1000)
    },

    flashStatus(status) {
      this.status = status
      setInterval(() => this.status = null, 3*1000)
    },

    clearOp() {
      this.status   = null
      this.opstatus = null
      this.rabby_op = null
      clearInterval(this.opInt)
      $('.bx--modal-close').last().click()
    },

    async rabbyConfirm(cb) {
      if (!this.settings.enabled) return
      this.rabby_op = 'sign'
      this.rabbyMsg(async () => {
        await this.sleep(5)
        this.rabby_op = 'confirm'
        this.rabbyMsg(cb)
      })
    },
    rabbyMsg(cb) {
      chrome.runtime.sendMessage(RABBY_EXTID, {type: `rabby_${this.rabby_op}`})
        .then(cb)
        .catch(error => {})
    },

    sleep(s) {
      return new Promise(res => setTimeout(res, s*1000))
    },

    parseNum(n) {
      return parseFloat(n.replace(/,/g, '.'))
    },
    round(n) {
      return n.toFixed(1)
    },
    
  },

  mounted() {
    this.loadSettings()
    this.getRange()
    setInterval(this.reload, 1*1000)
  },
}

</script>

