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

    <Range :min=rmin :max=rmax :dmin=dmin :reb=settings.rbdmin />

    <div id=settings class=card >
      <div class=card-body >
        <div class='d-flex flex-row' >
          <h5 class=card-title > Rebals </h5>
          <div class='form-check form-switch' >
            <input class=form-check-input type=checkbox role=switch v-model=settings.enabled @change=reload() >
          </div>
        </div>
        <Range v-for='rb in settings.rebals' :min=rb.rmin :max=rb.rmax :dmin=rb.dmin :reb=settings.rbdmin >
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
        <div> Rebalance Trigger %: <input type=number min=1 max=40 v-model=settings.rbdmin @change=reload() /> </div>
        <div> Rebalance Acceptance %: <input type=number :min=settings.rbdmin max=40 v-model=settings.rbdreq /> </div>
        <div> Slippage: <input type=number min=0.1 max=3 v-model=settings.slippage /> </div>
      </div>
    </div>
  </div>
</template>

<script>
import Range from '/components/Range.vue'
import '../style.css'
import $ from 'jquery' // for :constains selector

const RABBY_EXTID = 'jcnpbgidgmloifpjgiodolmobcjjpkno'

const ErrorsSelector = `
.bx--inline-notification--error:contains("Transaction will revert"),
.bx--inline-notification--error:contains("Transaction reverted")
`

export default {

  data() {
    return {
      status: null, opstatus: null,
      opInt: null, rabby_op: null,

      //                       price                                 rmin   - rmax             rpct
      // 'Current pool price is 3430.3, your position price range is 3401.6 - 3470.3 and it is 2% wide.'
      price: null,
      rmin: null, rmax: null, rpct: null, range: null,
      dmin: null, dmax: null,

      bmin: null, bmax: null, brange: null, bdist: null,

      settings: {
        enabled: false,
        cpmin: -1, cpmax: 1,
        rbdmin: 20, rbdreq: 30, slippage: 0.1,
        rebals: [ 
          {enabled: true, pmin:  0, pmax: 0, rmin: null, rmax: null, range: null, dmin: null},
          {enabled: true, pmin:  0, pmax: 1, rmin: null, rmax: null, range: null, dmin: null},
          {enabled: true, pmin: -1, pmax: 0, rmin: null, rmax: null, range: null, dmin: null},
          {enabled: true, pmin: -1, pmax: 1, rmin: null, rmax: null, range: null, dmin: null},
        ],
      },
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
      range = range || this.range
      return Math.round(100 * (Math.abs(diff) / range))
    },

    reload() {
      if (this.status) return // can't change params during OP

      let el = document.querySelector('p.bx--inline-notification__title') 
      if (!el) return document.querySelector('tr:not(.bx--expandable-row) .symbol')?.click()

      let m = el.innerText.match(/is ([\d,.]+).+is ([\d,.]+) - ([\d,\.]+).*is (\d+)% wide/)
      if (!m) return
      let [v,rmin,rmax,rpct] = m.slice(1,5).map(this.parseNum)
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

      if (!this.settings.enabled) return
      if (this.dmin <= this.settings.rbdmin || this.dmax <= this.settings.rbdmin)
        this.rebalance()
    },

    onPriceChange() {
      this.getBaseRange()
    },

    async getBaseRange() {
      if (this.status) return
      this.status = 'Fetching base range'

      await this.setRange(0, 0)
      await this.sleep(1)
      let txt = document.querySelector('.bx--toast-notification--info').innerText
      let [bmin, bmax] = txt.match(/range ([\d,.]+) - ([\d,\.]+)\./).slice(1,3).map(this.parseNum)
      this.bmin = bmin
      this.bmax = bmax
      this.brange = this.rmax / 100 // bmax - bmin is less precise

      this.settings.rebals.forEach(rb => {
        rb.rmin = (this.bmin + rb.pmin*this.brange).toFixed(1)
        rb.rmax = (this.bmax + rb.pmax*this.brange).toFixed(1)
        rb.range = rb.rmax - rb.rmin
        rb.dmin = this.calcDist(this.price-rb.rmin, rb.range)
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
      let bmin = $('label:contains("Min price") + div input').last()
      let bmax = $('label:contains("Max price") + div input').last()

      bmin.val(min)
      bmax.val(max)
      bmin[0].dispatchEvent(new Event('change'))
      bmax[0].dispatchEvent(new Event('change'))
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
        await this.setRange(-1, 1)
        this.setSlippage()

        let btn = $('.bx--btn:contains("Rebalance")').last()
        while (btn.prop('disabled')) if (this.status) await this.sleep(1); else return
        btn.click()
      }, {retry: true})
    },

    cancel() {
      this.rabby_op = 'cancel'
      this.rabbyMsg()
      this.clearOp()
    },

    transact(status, cb, {retry = true} = {}) {
      this.clearOp()
      this.status = status

      this.opInt = setInterval(async () => {
        await cb()
        this.opstatus = 'Waiting for Rabby'
        await this.sleep(10)
        this.rabbyConfirm()
        this.opstatus = 'Waiting status'
        await this.sleep(10)
        if (document.querySelector('.bx--inline-notification--error')) {
          if (retry) return // keep interval active
          else this.clearOp()
        } else if (document.querySelector('.bx--inline-notification--success'))
          this.clearOp()
      }, 30*1000)
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
      document.querySelector('.bx--modal-close')?.click() // close any previous error
    },

    async rabbyConfirm(cb) {
      if (!this.settings.enabled) return
      this.rabby_op = 'sign'
      this.rabbyMsg(async () => {
        this.clearOp()
        await this.sleep(10) // wait for UI
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
    
  },

  mounted() {
    this.loadSettings()
    this.getRange()
    setInterval(this.reload, 1*1000)
  },
}

</script>

