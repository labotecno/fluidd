<template>
  <v-dialog
    @input="$emit('input', $event)"
    :value="value"
    :loading="loading"
    hide-overlay
    fullscreen
    transition="dialog-bottom-transition"
    content-class="config-editor-overlay"
  >
    <v-card d-flex class="fill-height" style="overflow: hidden;">
      <v-toolbar
        dense
        :elevation="6"
        style="z-index: 1"
      >
        <app-btn
          icon
          color=""
          @click="emitClose()"
        >
          <v-icon>$close</v-icon>
        </app-btn>
        <v-toolbar-title>{{ filename }}</v-toolbar-title>
        <v-spacer></v-spacer>
        <v-toolbar-items>
          <app-btn
            v-if="!printerPrinting && rootProperties.showConfigRef"
            :href="$globals.DOCS_KLIPPER_CONFIG_REF"
            target="_blank">
            <v-icon small :left="!isMobile">$help</v-icon>
            <span class="d-none d-md-inline-block">{{ $t('app.general.btn.config_reference') }}</span>
          </app-btn>
          <app-btn
            v-if="!readonly && !printerPrinting && rootProperties.showSaveRestart"
            @click="emitSave(true)">
            <v-icon small :left="!isMobile">$restart</v-icon>
            <span class="d-none d-md-inline-block">{{ $t('app.general.btn.save_restart') }}</span>
          </app-btn>
          <app-btn
            v-if="!readonly"
            @click="emitSave(false)">
            <v-icon small :left="!isMobile">$save</v-icon>
            <span class="d-none d-md-inline-block">{{ $t('app.general.btn.save') }}</span>
          </app-btn>
          <app-btn
            @click="emitClose()">
            <v-icon small :left="!isMobile">$close</v-icon>
            <span class="d-none d-md-inline-block">{{ $t('app.general.btn.close') }}</span>
          </app-btn>
        </v-toolbar-items>
      </v-toolbar>

      <file-editor
        v-if="contents !== undefined"
        :value="contents"
        @input="updatedContent = $event"
        :filename="filename"
        :readonly="readonly">
      </file-editor>

    </v-card>
  </v-dialog>
</template>

<script lang="ts">
import { Component, Mixins, Prop } from 'vue-property-decorator'
import StateMixin from '@/mixins/state'

// Lazy Load the file editor.
const FileEditor = () => import(
  /* webpackPreload: true */
  /* webpackChunkName: "chunk-fileeditor" */
  './FileEditor.vue'
)
// import FileEditor from './FileEditor.vue'

@Component({
  components: {
    FileEditor
  }
})
export default class FileEditorDialog extends Mixins(StateMixin) {
  @Prop({ type: Boolean, required: true })
  public value!: boolean

  @Prop({ type: String, required: true })
  root!: string

  @Prop({ type: String, required: true })
  public filename!: string;

  @Prop({ type: String, required: true })
  public contents!: string

  @Prop({ type: Boolean, default: false })
  public loading!: boolean

  @Prop({ type: Boolean, default: false })
  public readonly!: boolean

  updatedContent = ''

  get rootProperties () {
    return this.$store.getters['files/getRootProperties'](this.root)
  }

  get isMobile () {
    return this.$vuetify.breakpoint.sm
  }

  emitClose () {
    this.$emit('input', false)
  }

  emitSave (restart: boolean) {
    this.$emit('save', this.updatedContent, restart)
    if (restart) this.$emit('input', false)
  }
}
</script>
