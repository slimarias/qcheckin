<template>
  <div id="checkinComponent">
    <!--Map-->
    <open-map v-if="currentLocation" v-model="mapMarket" read-only height="200px"/>
    <!--Message map unavailable-->
    <div v-else style="height: 200px; padding: 22px 0" class="bg-grey-4 text-center">
      <div class="q-px-md">
        <q-icon color="grey-6" name="fas fa-map-marked-alt" size="80px"/>
        <div class="q-mt-md">
          <q-icon name="fas fa-exclamation-triangle" color="warning"/>
          {{ this.$tr('qcheckin.sidebar.locationAddress') }}
        </div>
      </div>
    </div>
    <!--Content-->
    <div id="chronometerContent">
      <!--Button actions chronometer-->
      <div class="text-center">
        <!--Button start chronometer-->
        <q-btn v-if="!activeShift" :loading="loading" @click="createShift" round color="green" size="23px"
               icon="fas fa-play" class="cursor-pointer q-pa-xs">
        </q-btn>
        <!--End shift-->
        <q-btn color="red" class="q-mx-xs" size="28px" round :loading="loading" v-else @click="endShift">
          <q-icon id="iconButton" name="fas fa-stop" size="25px" class="q-pa-xs q-mb-xs full-width"/>
          <label class="cursor-pointer" style="font-size: 12px">{{ this.$tr('ui.label.close') }}</label>
        </q-btn>
      </div>
      <!--Shift Information-->
      <div v-if="startChronometer">
        <div id="timer" class="q-box bg-grey-4 text-center text-blue-grey">
          <!--Date-->
          <div class="q-body-1">
            {{ activeShift.checkinAt ? $trd(activeShift.checkinAt, {type: "long"}) : "" }}
          </div>
          <!--chronometer-->
          <div id="screen" class="q-display-2 q-mb-sm">
            {{ time.text }}
          </div>
          <!--Line separator-->
          <q-separator size="1px" color="blue-grey" class="q-my-xs"/>
          <!--Address-->
          <div class="q-caption">
            <q-icon name="fas fa-map-marker-alt" class="q-mr-xs"/>
            <b>{{ locationDescription }} </b>
          </div>
        </div>
      </div>
    </div>
    <div v-if="activeShift">
      <!--Comment-->
      <div class="q-pa-md">
        <dynamic-field v-model="comment.text" :field="fields.comments" @input="sendComment"></dynamic-field>
      </div>
    </div>
    <!--Actions-->
    <div class="text-center absolute-bottom q-my-md">
      <!--My shifts-->
      <q-btn unelevated color="green" rounded no-caps :label="$tr('qcheckin.sidebar.Shifts')" icon="fas fa-user-clock"
             :to="{name : 'qcheckin.main.shifts.index'}">
      </q-btn>
    </div>
  </div>
</template>
<script>
import openMap from '@imagina/qsite/_components/master/mapLeaflet'

export default {
  beforeDestroy() {
    this.$root.$off('qcheckin.checkout')
  },
  props: {},
  components: {openMap},
  watch: {},
  mounted() {
    this.$nextTick(function () {
      this.init()
    })
  },
  beforeRouteLeave(to, from, next) {
    if (this.control) clearInterval(this.control);
    next()
  },
  data() {
    return {
      loading: false,
      currentLocation: false,
      mapMarket: false,
      activeShift: false,
      time: {
        text: "00:00:00",
        hh: 0,
        mm: 0,
        ss: 0
      },
      control: false,
      comment: {
        loading: false,
        text: null
      }
    }
  },
  computed: {
    fields() {
      return {
        comments: {
          value: '',
          type: 'input',
          props: {
            label: `${this.$tr('ui.form.comment')}`,
            type: 'textarea',
            rows: "3",
            icon: "fas fa-comment-dots",
            debounce: 1500
          },
        },
      }
    },
    locationDescription() {
      if (this.currentLocation)
        return `Lat: ${this.currentLocation.latitude}, Lng: ${this.currentLocation.longitude}`
      else return this.$tr('qcheckin.sidebar.noLocation')
    }
  },
  methods: {
    //Init
    init() {
      this.getActiveShift()
      this.getCurrentLocation()
      //Listen checkout
      this.$root.$on('qcheckin.checkout', (item) => {
        if (item.id == this.activeShift.id) this.endShift()
      })
    },
    async getCurrentLocation() {
      let currentLocation = await this.$helper.getCurrentPosition()
      if (currentLocation) {
        //Set location to shift
        this.currentLocation = {
          latitude: currentLocation.coords.latitude,
          longitude: currentLocation.coords.longitude
        }
        //Set location map
        this.mapMarket = {
          lat: currentLocation.coords.latitude,
          lng: currentLocation.coords.longitude
        }
      }
    },
    //Get active shift
    getActiveShift() {
      return new Promise((resolve, reject) => {
        this.loading = true
        //Request params
        let requestParams = {
          refresh: true,
          params: {filter: {noCheckout: true, field: "checkin_by"}}
        }
        let userId = this.$store.state.quserAuth.userId
        //Request
        this.$crud.show("apiRoutes.qcheckin.shifts", userId, requestParams).then(response => {
          if (typeof response.data == "object") {
            this.activeShift = {
              ...this.$clone(response.data),
              checkinAt: this.$moment().format(response.data.checkinAt)
            }
            this.comment.text = response.data.options.comment || null
            this.startChronometer()//Start chromometer
          }
          this.loading = false
          resolve(response.data)
        }).catch(error => {
          this.loading = false
          reject(error)
        })
      })
    },
    //Create new shift
    async createShift() {
      this.loading = true
      let requestData = {options: {}, geoLocation: {}}

      //Set current location to checkin
      if (this.currentLocation) requestData.geoLocation = {checkin: this.$clone(this.currentLocation)}
      //Request
      this.$crud.create('apiRoutes.qcheckin.checkin', requestData).then(response => {
        this.activeShift = this.$clone(response.data)
        this.startChronometer()
        this.loading = false
      }).catch(error => {
        this.loading = false
      })
    },
    //Start chronometer
    startChronometer() {
      if (this.activeShift) {
        this.time.ss = this.$clone(this.activeShift.diff.s)
        this.time.mm = this.$clone(this.activeShift.diff.i)
        this.time.hh = this.$clone(this.activeShift.diff.h)

        this.time.text = (this.time.hh == 0 ? '00' : this.time.hh < 10 ? '0' + this.time.hh : this.time.hh)
            + ":" +
            (this.time.mm == 0 ? '00' : this.time.mm < 10 ? '0' + this.time.mm : this.time.mm)
            + ":" +
            (this.time.ss == 0 ? '00' : this.time.ss < 10 ? '0' + this.time.ss : this.time.ss)
        this.control = setInterval(this.timer, 1000);
        //Emit start chronometer
        this.$emit('checkin')
      }
    },
    //Interval Timer
    timer() {
      let ss, mm, hh
      this.activeShift.diff.s++;

      if (this.activeShift.diff.s == 60) {
        this.activeShift.diff.s = 0
        this.activeShift.diff.i++;
      }
      if (this.activeShift.diff.i == 60) {
        this.activeShift.diff.i = 0
        this.activeShift.diff.h++
      }
      ss = this.$clone(this.activeShift.diff.s)
      mm = this.$clone(this.activeShift.diff.i)
      hh = this.$clone(this.activeShift.diff.h)

      if (this.activeShift.diff.s < 10) {
        ss = "0" + ss;
      }
      if (mm < 10) {
        mm = "0" + mm;
      }
      if (hh < 10) {
        hh = "0" + hh;
      }
      this.time.text = hh + ":" + mm + ":" + ss
    },
    //Send comment
    sendComment() {
      if (this.activeShift) {
        this.loading = true
        //Request data
        let requestData = {options: {...this.activeShift.options, comment: this.comment.text || 'No comments'}}
        //Request
        this.$crud.update('apiRoutes.qcheckin.shifts', this.activeShift.id, requestData).then(response => {
          this.loading = false
        }).catch(error => this.loading = false)
      }
    },
    //End shift
    async endShift() {
      this.loading = true
      //Request Params
      let requestParams = {params: {filter: {checkout: true}}}
      //Request Data
      let requestData = this.$clone(this.activeShift)
      //Set current location to checkout
      if (this.currentLocation) requestData.geoLocation.checkout = this.$clone(this.currentLocation)

      //Request
      this.$crud.update('apiRoutes.qcheckin.checkout', requestData.id, requestData, requestParams).then(response => {
        this.activeShift = false
        clearInterval(this.control);
        this.$emit('checkout')
        this.comment.text = null
        this.loading = false
      }).catch(error => this.loading = false)
    },
  }
}
</script>
<style lang="stylus">
#checkinComponent
  #chronometerContent
    background-color $grey-2
    position relative

    &:before
      content ''
      background-color $grey-2
      width: 115px;
      height: 115px;
      position absolute
      top -15px
      left 0
      right 0
      margin auto
      border-radius 50%
      z-index: 5000;

    button
      z-index: 5000;

    #iconButton
      top 15px

    #timer
      font-size 15px
      padding 20px

      #screen
        font-size 33px

    #mapChecking
      height 250px
      width 100%
      border none

</style>
