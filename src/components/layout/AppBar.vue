<template>
  <v-app-bar
    app
    clipped-left
  >
    <v-container fluid class="flex-nowrap constrained-width pa-0 fill-height">
      <router-link to="/">
        <app-icon
          class="shrink mr-4 mt-1 color-filter"
          width="35"
          height="40"
          :primary-color="theme.currentTheme.primary"
          :secondary-color="theme.currentTheme.primaryOffset"
        ></app-icon>
      </router-link>
      <v-toolbar-title
        class="printer-title text--secondary mr-5"
      >
        <router-link to="/" v-html="instanceName"></router-link>
      </v-toolbar-title>
      <v-spacer />

      <app-btn text to="/" color="" class="d-none d-md-flex mx-1">
        <v-icon small class="mr-md-1">$home</v-icon>
        <span>{{ $t('app.general.title.home') }}</span>
      </app-btn>
      <app-btn text to="/jobs" color="" class="d-none d-md-flex mx-1">
        <v-icon small class="mr-md-1">$files</v-icon>
        <span>{{ $t('app.general.title.jobs') }}</span>
      </app-btn>
      <app-btn text to="/tune" color="" class="d-none d-md-flex mx-1">
        <v-icon small class="mr-md-1">$tune</v-icon>
        <span>{{ $t('app.general.title.tune') }}</span>
      </app-btn>
      <app-btn text to="/configure" color="" class="d-none d-md-flex mx-1">
        <v-icon small class="mr-md-1">$cogs</v-icon>
        <span>{{ $t('app.general.title.configure') }}</span>
      </app-btn>
      <v-tooltip bottom v-if="socketConnected">
        <template v-slot:activator="{ on, attrs }">
          <app-btn
            :disabled="!klippyReady"
            icon
            color="error"
            @click="emergencyStop()"
            v-bind="attrs"
            v-on="on">
            <v-icon>$estop</v-icon>
          </app-btn>
        </template>
        {{ $t('app.general.tooltip.estop') }}
      </v-tooltip>

      <v-badge
        :value="hasUpdates"
        bordered
        dot
        color="warning"
        overlap
        offset-x="15"
        offset-y="15"
      >
        <app-btn icon color="" @click="$emit('drawer')">
          <v-icon>$menu</v-icon>
        </app-btn>
      </v-badge>

    </v-container>
  </v-app-bar>
</template>

<script lang="ts">
import { Component, Mixins } from 'vue-property-decorator'
import StateMixin from '@/mixins/state'
import { SocketActions } from '@/socketActions'

@Component({})
export default class AppBar extends Mixins(StateMixin) {
  menu = false

  get instances () {
    return this.$store.state.config.instances
  }

  get instanceName () {
    return this.$store.state.config.uiSettings.general.instanceName
  }

  get currentFile () {
    return this.$store.state.printer.printer.print_stats.filename
  }

  get hasUpdates () {
    return this.$store.getters['version/hasUpdates']
  }

  get theme () {
    return this.$store.getters['config/getTheme']
  }

  emergencyStop () {
    SocketActions.printerEmergencyStop()
  }
}
</script>

<style lang="scss" scoped>
  .fill1 {
    fill: #2E75AE;
  }

  .fill2 {
    fill: #2196F3;
  }

  .printer-title {
    font-size: 1.25rem;
    font-weight: 300;
    font-family: raleway, sans-serif;
    overflow: hidden;
    white-space: nowrap;
    text-overflow: ellipsis;
    @media #{map-get($display-breakpoints, 'sm-and-up')} {
      font-size: 2.125rem;
    }
  }

  .printer-title > a {
    color: inherit;
    text-decoration: none;
  }
</style>
