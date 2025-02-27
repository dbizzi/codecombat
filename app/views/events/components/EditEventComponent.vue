<script>
import { mapGetters, mapMutations, mapActions } from 'vuex'
import _ from 'lodash'
import moment from 'moment'
import momentTimezone from 'moment-timezone'
import { HTML5_FMT_DATE_LOCAL, HTML5_FMT_TIME_LOCAL } from '../../../core/constants'
import { RRuleGenerator, rruleGeneratorModule } from 'vue2-rrule-generator'
import VueTimepicker from 'vue2-timepicker'
import MembersComponent from './MembersComponent'
import UserSearchComponent from './UserSearchComponent'
import TimeZonePicker from './TimeZonePicker'
import gcApiHandler from '../../../core/social-handlers/GoogleCalendarHandler'
import convertDateMixin from '../mixins/convertDateMixin'

export default {
  name: 'EditEventComponent',
  props: {
    editType: {
      type: String,
      default: 'new'
    }
  },
  components: {
    'rrule-generator': RRuleGenerator,
    'user-search': UserSearchComponent,
    'time-picker': VueTimepicker,
    'timezone-picker': TimeZonePicker
  },
  data () {
    return {
      isSuccess: false,
      inProgress: false,
      previewing: false,
      errorMessage: '',
      timeZone: 'America/New_York',
      tzOffset: momentTimezone.tz('America/New_York').format('Z'),
      event: {},
      resetRRule: true, // signal
      startDate: null,
      endDate: null,
      startTime: null,
      endTime: null
    }
  },
  mixins: [
    convertDateMixin
  ],
  methods: {
    ...mapMutations('events', [
      'setEvent',
      'cancelPreviewEvent'
    ]),
    ...mapActions('events', [
      'saveEvent',
      'editEvent'
    ]),
    selectTimeZone (tz) {
      this.timeZone = tz
      this.tzOffset = momentTimezone.tz(this.timeZone).format('Z')
    },
    selectOwner (u) {
      Vue.set(this.event, 'owner', u._id)
      Vue.set(this.event, 'ownerName', u.name)
    },
    cancelPreviewOnCalendar () {
      this.cancelPreviewEvent()
      this.previewing = false
    },
    previewOnCalendar () {
      if (this.previewing) {
        this.cancelPreviewOnCalendar()
      } else {
        const tempEvent = _.cloneDeep(this.event)
        tempEvent._id = 'temp-event'
        tempEvent.name = 'preview-' + tempEvent.name
        tempEvent.instances = this.makeInstances(tempEvent)
        this.setEvent(tempEvent)
        this.previewing = true
      }
    },
    makeInstances (event) {
      const instance = {
        _id: 'temp-event-' + Math.random(),
        title: 'temp-' + event.name,
        owner: event.owner,
      }
      return this.rrule.all().map(d => {
        return Object.assign({}, instance, {
          startDate: d,
          endDate: new Date((+event.endDate) - (+event.startDate) + (+d))
        })
      })
    },
    addMember (m) {
      this.event.members.add(m)
    },
    async syncToGoogleCalendar () {
      try {
        const res = await gcApiHandler.syncEventsToGC(this.event, { timeZone: this.timeZone })
        if (!this.event.googleEventId) {
          const googleEventId = res.id
          this.event.googleEventId = googleEventId
          await this.editEvent({ _id: this.event._id, googleEventId })
        }
        noty({ text: 'Synced to Google Calendar', type: 'success', layout: 'center', timeout: 3000 })
      } catch (err) {
        this.editEvent({ _id: this.event._id, syncedToGC: false })
        console.log('Error syncing to GC:', err)
        noty({ text: 'Error syncing to Google Calendar', type: 'error', timeout: 5000 })
      }
    },
    async onFormSubmit () {
      this.inProgress = true
      this.cancelPreviewOnCalendar()
      this.event.type = 'online-classes'
      this.event.rrule = this.rrule.toString()
      if (!this.event.owner) {
        this.errorMessage = 'Must set an Owner'
        this.inProgress = false
        return
      }
      const { errMsg, startDate, endDate } = this.validateDates({
        initialStartDate: this.event.startDate,
        initialEndDate: this.event.endDate
      })
      if (errMsg) {
        this.errorMessage = errMsg
        this.inProgress = false
        return
      }
      this.event.startDate = startDate
      this.event.endDate = endDate

      if (this.editType === 'new') {
        try {
          const res = await this.saveEvent(this.event)
          this.event._id = res._id
          if (this.event.syncedToGC) {
            await this.syncToGoogleCalendar()
          }
          this.$emit('save', res._id)
          this.inProgress = false
        } catch (err) {
          this.errorMessage = err.message
          setTimeout(() => {
            this.inProgress = false
          }, 3000)
        }
      } else {
        try {
          await this.editEvent(this.event)
          if (this.event.syncedToGC) {
            await this.syncToGoogleCalendar()
          }
          this.$emit('save', this.event._id)
          this.inProgress = false
        } catch (err) {
          this.errorMessage = err.message
        }
      }
    },
    eventUpdate () {
      const now = new Date()
      let date;
      if (this.clickedDate) {
        date = new Date(this.clickedDate).setHours(now.getHours())
      }
      const sDate = moment(date || now).set('minutes', 0).set('seconds', 0)
      if (this.editType === 'new') {
        this.event = {
          members: [],
          startDate: sDate.toDate(),
          endDate: sDate.clone().add(1, 'hours').toDate(),
          instances: [],
          syncedToGC: true // keeping it true by default so that on first save, we sync to google by default
        }
      } else {
        this.event = _.cloneDeep(this.propsEvent)
      }
      this.resetRRule = !this.resetRRule
      this.convertTimesForUI({ startDate: this.event.startDate, endDate: this.event.endDate, timeZone: this.timeZone })
    }
  },
  computed: {
    ...mapGetters({
      propsEvent: 'events/eventPanelEvent',
      clickedDate: 'events/eventPanelDate',
      rrule: 'rruleGenerator/rule'
    }),
    me () {
      return me
    },
    myTimeZone () {
      return momentTimezone.tz.guess()
    },
    _endTimeHourRange () {
      return [[0, 23]]
    },
    _gcEmails: {
      get () {
        return (this.event.gcEmails || []).join('\n')
      },
      set (val) {
        this.$set(this.event, 'gcEmails', val.split(/\n|,/).map(e => e.trim()))
      }
    },
    rruleStart () {
      if (this.event.startDate) {
        return new Date(this.event.startDate)
      } else {
        return new Date()
      }
    },
    rulePreviewTop6 () {
      return this.rrule.all((date, i) => i < 6).map(d => moment(d).format('ll'))
    }
  },
  created () {
    if (!this.$store.hasModule('rruleGenerator')) {
      this.$store.registerModule('rruleGenerator', rruleGeneratorModule)
    }
  },
  mounted () {
    this.eventUpdate()
  },
  watch: {
    propsEvent () {
      this.eventUpdate()
    }
  }
}
</script>

<template>
  <div>
    <div class="tab-label">
      {{ $t('events.edit_event_tab_desc') }}
    </div>
    <form
      class="edit-event-form"
      @submit.prevent="onFormSubmit"
    >
      <div class="form-group">
        <label for="name"> {{ $t('events.name') }}</label>
        <input
          v-model="event.name"
          name="name"
          class="form-control"
          type="text"
        >
      </div>
      <div class="form-group">
        <label for="description"> {{ $t('events.description') }}</label>
        <input
          v-model="event.description"
          name="description"
          class="form-control"
          type="text"
        >
      </div>
      <div class="form-group">
        <label for="owner"> {{ $t('events.owner') }}</label>
        <user-search
          :permissions="'onlineTeacher'"
          :value="event.ownerName"
          :no-results="'No online teachers found'"
          :placeholder="'Search email or username for an online-teacher'"
          @select="selectOwner"
        />
      </div>

      <div class="form-group">
        <label for="timeZone"> {{ $t('events.time_zone') }}</label>
        <div class="input-lable">
          {{ $t('events.timezone_tips') + myTimeZone }}
        </div>
        <timezone-picker
          :tz="timeZone"
          @select="selectTimeZone"
        />
      </div>

      <div class="form-group">
        <label for="startDate"> {{ $t('events.start_date') }}</label>
        <input
          v-model="startDate"
          type="date"
          class="form-control"
          name="startDate"
          id="startDate"
        >
      </div>
      <div class="form-group">
        <label for="timeRange"> {{ $t('events.time_range') }}</label>
        <div>
          <time-picker format="hh:mm A" :minute-interval="10" v-model="startTime" />
        </div>
      </div>

      <div class="form-group">
        <label for="endDate"> {{ $t('events.end_date') }}</label>
        <input
          v-model="endDate"
          type="date"
          class="form-control"
          name="endDate"
          id="endDate"
        >
      </div>
      <div class="form-group">
        <label for="timeRange"> {{ $t('events.time_range') }}</label>
        <div>
          <time-picker format="hh:mm A" :minute-interval="10" v-model="endTime" />
        </div>
      </div>

      <rrule-generator
        v-if="editType === 'new' || event.rrule"
        :start="rruleStart"
        :option="{showStart: false}"
        :rrule="event.rrule"
        :resetRRule="resetRRule"
      />

      <div class="form-group">
        <label for="meetingLink"> {{ $t('events.meetingLink') }}</label>
        <div>
          <input
            v-model="event.meetingLink"
            type="text"
            class="form-control"
            name="meetingLink"
            placeholder="input your zoom link here"
          >
        </div>
      </div>
      <div class="form-group" v-if="true || me.useGoogleCalendar()">
        <label for="importedToGC"> {{ $t(`events.sync${propsEvent?.syncedToGC ? 'ed' : ''}_to_google`) }}</label>
        <div class="input-label">
          {{ $t('events.sync_to_google_desc') }}
        </div>
        <input
          v-model="event.syncedToGC"
          type="checkbox"
          class="form-control"
          name="sync"
          :disabled="propsEvent?.syncedToGC"
        >
      </div>
      <div class="form-group" v-if="event.syncedToGC">
        <label for="gcEmails">{{$t('events.google_calendar_attendees')}} </label>
        <textarea
          v-model="_gcEmails"
          class="form-control gcEmails"
          name="gcEmails"
          placeholder="List emails here to get notification by google calendar, split by newline or comma"
        />
      </div>
      <div class="form-group pull-right">
        <span
          v-if="isSuccess"
          class="success-msg"
        >
          {{ $t('teacher.success') }}
        </span>
        <span
          v-if="errorMessage"
          class="error-msg"
        >
          {{ errorMessage }}
        </span>
        <button
          class="btn btn-success btn-lg"
          type="submit"
          :disabled="inProgress"
        >
          {{ $t('common.submit') }}
        </button>
      </div>
    </form>
    <div class="preview">
      <input
        class="btn btn-success btn-lg"
        type="button"
        :value="previewing ? 'Cancel' : 'Preview'"
        @click="previewOnCalendar"
      >
      <div
        v-for="date in rulePreviewTop6"
        :key="date"
      >
        {{ date }}
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
.gcEmails {
  height: 10em;
}
.error-msg {
  color: red;
}
.tab-label, .input-label {
  font-size: 15px;
  color: rgba(128, 128, 128, 0.7);
}
</style>
