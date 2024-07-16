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

    <div class=card v-if=settings.enabled >
      <select class=form-select v-model=settings.ca @change=selectAsset() >
        <option v-for='a in assets' :value=a.name > {{a.name}} </option>
      </select>

      <Range :rb=cr :reb=settings.rbdthd />
      <div class=card-body >
        <div class='d-flex flex-row' >
          <h5 class=card-title > Rebals </h5>
          <div class='form-check form-switch' >
            <input class=form-check-input type=checkbox role=switch v-model=settings.autorebal @change=reload() >
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
        <div class='d-flex flex-row' >
          <h5 class=card-title > Settings </h5>
          <div class='form-check form-switch' >
            <input class=form-check-input type=checkbox role=switch v-model=settings.enabled @change=onEnable() >
          </div>
        </div>

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

      assets: [],

      price: null,
      cr: {rmin: null, rmax: null, rpct: null, range: null, dmin: null, dmax: null,},
      br: {rmin: null, rmax: null, rpct: null, range: null, dmin: null},

      settings: {
        ca: null,
        enabled: true,
        autorebal: false,
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

    async readSettings() {
      let { ['vfat_automate']: value } = await chrome.storage.local.get(['vfat_automate'])
      if (value?.rebals) value.rebals = Object.values(value.rebals) 
      return value
    },
    async loadSettings() {
      this.settings = { ...this.settings, ...(await this.readSettings()) }
    },
    async saveSettings() {
      await chrome.storage.local.set({ ['vfat_automate']: this.settings })
    },

    onEnable() {
      this.status = (!this.settings.enabled) ? 'Disabled' : null
    },

    loadAssets() {
      let sel = document.querySelectorAll('.farm-container tr[data-row] td:nth-child(2)')
      this.assets = [...sel].map(s => {
        return {name: s.innerText.split('\n')[0]}
      })
    },

    selectAsset() {
      if (!this.settings.ca) this.settings.ca = this.assets[0].name
      let row = $(`.farm-container tr[data-row] td:nth-child(2):contains("${this.settings.ca}")`)
      if (row.parents('.bx--selected_row').length) return
      row.click()
    },

    extractRange(pr) {
      return {
        price: this.parseNum(pr.find('.current-price-tick').text().split('\n')[0]),
        rmin:  this.parseNum(pr.find('.min-percentage').text().split('\n')[0]),
        rmax:  this.parseNum(pr.find('.max-percentage').text().split('\n')[0]),
        rpct:  this.parseNum(pr.find('.row :contains("Width")').text().match(/(\d\.\d+)/)[1]),
      }
    },

    reload() {
      this.loadAssets()

      if (!this.settings.enabled) return
      if (this.status) return // can't change params during OP

      this.selectAsset()

      let pr = $('.price-range-indicator-container').first()
      if (!pr.length) return
      let {price,rmin,rmax,rpct} = this.extractRange(pr)
      this.cr.rmin = rmin
      this.cr.rmax = rmax
      this.cr.rpct = rpct
      this.cr.range = this.cr.rmax - this.cr.rmin
      if (price != this.price) this.onPriceChange()
      this.price = price

      this.findBestRebal()

      this.cr.dmax = this.calcDist(rmax - price)
      this.cr.dmin = this.calcDist(price - rmin)

      if (!this.settings.autorebal) return
      if (this.cr.dmin <= this.settings.rbdthd || this.cr.dmax <= this.settings.rbdthd)
        this.rebalance()
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
      let pr = $('.price-range-indicator-container').last()
      if (!pr.length) return
      let {price,rmin,rmax,rpct} = this.extractRange(pr)
      this.br.rmin = rmin
      this.br.rmax = rmax
      this.br.rpct = rpct
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

    onPriceChange() {
      this.getBaseRange()
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

    setSlippage() {
      [...document.querySelectorAll(`.bx--number__controls button[aria-label="${this.settings.slippage}"]`)].forEach(e => {
        e.click()
      })
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

    checkStatus(retry) {
      if (document.querySelector('.bx--inline-notification--error')) {
        if (retry) return // keep interval active
        else return this.clearOp()
      } else if (document.querySelector('.bx--inline-notification--success'))
        return this.clearOp()
    },

    async opTrack(cb, {retry = true} = {}) {
      if (this.opstatus) return // still running
      this.opstatus = 'Waiting for route'
      await cb()
      await this.sleep(5)
      if (this.checkStatus(retry)) return this.opTrack(cb, {retry}) //mostly slippage errors
      this.opstatus = 'Waiting for Rabby'
      await this.sleep(10)
      this.rabbyConfirm()
      this.opstatus = 'Waiting status'
      await this.sleep(10)
      if (this.checkStatus(retry)) return
    },

    async transact(status, cb) {
      if (this.status) return //busy
      this.clearOp()
      this.status = status
      await this.opTrack(cb)
      this.clearOp() // either success or error
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
      return true
    },

    async rabbyConfirm(cb) {
      if (!this.settings.autorebal) return
      this.rabby_op = 'sign'
      this.rabbyMsg(async () => {
        cb()
        //await this.sleep(5)
        //this.rabby_op = 'confirm'
        ////this.rabbyMsg(cb)
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

    calcDist(diff, range) {
      range = range || this.cr.range
      return Math.round(100 * (Math.abs(diff) / range))
    },
    distThd(dmin) {
      return Math.min(dmin - this.settings.rbdthd, 100-this.settings.rbdthd - dmin)
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
    this.onEnable()
    this.getRange()
    setInterval(this.reload, 1*1000)
  },
}

</script>

