<template>
    <v-dialog
        v-model="bool"
        :max-width="400"
        content-class="overflow-x-hidden"
        @click:outside="closeDialog"
        @keydown.esc="closeDialog">
        <v-card>
            <div v-if="file.big_thumbnail" class="d-flex align-center justify-center" style="min-height: 200px">
                <v-img
                    :src="file.big_thumbnail"
                    :max-width="maxThumbnailWidth"
                    class="d-inline-block"
                    :style="bigThumbnailStyle" />
            </div>
            <v-card-title class="text-h5">{{ $t('Dialogs.StartPrint.Headline') }}</v-card-title>
            <v-card-text class="pb-0">
                <p class="body-2">
                    {{ question }}
                </p>
            </v-card-text>
            <!-- Mesh Procedure (radio buttons) -->
            <v-card-text class="py-0">
                <settings-row title="Mesh Procedure">
                    <v-radio-group v-model="meshProcedure" row hide-details class="mt-0 mesh-radio">
                        <v-radio label="Slow" value="slow" class="mr-4"/>
                        <v-radio label="Fast" value="fast" />
                    </v-radio-group>
                </settings-row>
            </v-card-text>



            <!-- NEW: Nozzle cleanliness slider -->
            <v-card-text class="pt-0">
                <settings-row title="Nozzles clean?">
                    <v-switch
                        v-model="nozzleCleanBool"
                        inset
                        hide-details
                        class="mt-0"
                        :label="nozzleCleanBool ? 'Clean' : 'Dirty'"
                />
                </settings-row>
            </v-card-text>
            <v-divider class="mt-2 mb-0" />
            <!-- /NEW -->


            <start-print-dialog-spoolman v-if="moonrakerComponents.includes('spoolman')" :file="file" />
            <template v-if="moonrakerComponents.includes('timelapse')">
                <v-divider v-if="!moonrakerComponents.includes('spoolman')" class="mt-3 mb-2" />
                <v-card-text class="py-0">
                    <settings-row :title="$t('Dialogs.StartPrint.Timelapse')">
                        <v-switch v-model="timelapseEnabled" hide-details class="mt-0" />
                    </settings-row>
                </v-card-text>
                <v-divider class="mt-2 mb-0" />
            </template>
            <v-card-actions>
                <v-spacer />
                <v-btn text @click="closeDialog">{{ $t('Dialogs.StartPrint.Cancel') }}</v-btn>
                <v-btn
                    color="primary"
                    text
                    :disabled="printerIsPrinting || !klipperReadyForGui"
                    @click="startPrint(file.filename)">
                    {{ $t('Dialogs.StartPrint.Print') }}
                </v-btn>
            </v-card-actions>
        </v-card>
    </v-dialog>
</template>

<script lang="ts">
import { Component, Mixins, Prop } from 'vue-property-decorator'
import BaseMixin from '@/components/mixins/base'
import { FileStateGcodefile } from '@/store/files/types'
import SettingsRow from '@/components/settings/SettingsRow.vue'
import { mdiPrinter3d } from '@mdi/js'
import { ServerSpoolmanStateSpool } from '@/store/server/spoolman/types'
import { defaultBigThumbnailBackground } from '@/store/variables'

@Component({
    components: {
        SettingsRow,
    },
})
export default class StartPrintDialog extends Mixins(BaseMixin) {
    mdiPrinter3d = mdiPrinter3d
    meshProcedure: 'slow' | 'fast' = (localStorage.getItem('meshProcedure') as any) ?? 'slow'
    // NEW: slider state (1 = Clean default so operators don’t get nagged)
    nozzleCleanBool: boolean = (localStorage.getItem('nozzleCleanBool') ?? 'true') === 'true'
    // /NEW

    @Prop({ required: true, default: false })
    declare readonly bool: boolean

    @Prop({ required: true, default: '' })
    declare readonly currentPath: string

    @Prop({ required: true })
    declare file: FileStateGcodefile

    get timelapseEnabled() {
        return this.$store.state.server.timelapse?.settings?.enabled ?? false
    }

    set timelapseEnabled(newVal) {
        this.$socket.emit(
            'machine.timelapse.post_settings',
            { enabled: newVal },
            { action: 'server/timelapse/initSettings' }
        )
    }

    get bigThumbnailBackground() {
        return this.$store.state.gui.uiSettings.bigThumbnailBackground ?? defaultBigThumbnailBackground
    }

    get bigThumbnailStyle() {
        if (defaultBigThumbnailBackground.toLowerCase() === this.bigThumbnailBackground.toLowerCase()) {
            return {}
        }

        return { backgroundColor: this.bigThumbnailBackground }
    }

    get active_spool(): ServerSpoolmanStateSpool | null {
        return this.$store.state.server.spoolman.active_spool ?? null
    }

    get filamentVendor() {
        return this.active_spool?.filament?.vendor?.name ?? 'Unknown'
    }

    get filamentName() {
        return this.active_spool?.filament.name ?? 'Unknown'
    }

    get filament() {
        return `${this.filamentVendor} - ${this.filamentName}`
    }

    get question() {
        if (this.active_spool)
            return this.$t('Dialogs.StartPrint.DoYouWantToStartFilenameFilament', {
                filename: this.file?.filename ?? 'unknown',
            })

        return this.$t('Dialogs.StartPrint.DoYouWantToStartFilename', { filename: this.file?.filename ?? 'unknown' })
    }

    get maxThumbnailWidth() {
        return this.file?.big_thumbnail_width ?? 400
    }

    // /NEW

    // MOD: make async and send macro first
// Send 0/1 to Klipper based on the toggle
async startPrint(filename = '') {
  filename = (this.currentPath + '/' + filename).substring(1)

  try {
    const cleanInt = this.nozzleCleanBool ? 1 : 0
    const macro = this.meshProcedure === 'fast' ? 'FAST_PROCEDURE' : 'STARTUP_PROCEDURE'

    // Fire your macro with the cleanliness flag
    await this.$store.dispatch('printer/sendGcode', `${macro} CLEAN=${cleanInt}`)

    // Start the print as usual
    this.closeDialog()
    this.$socket.emit('printer.print.start', { filename }, { action: 'switchToDashboard' })
  } catch (e) {
    this.$store.dispatch('ui/showSnackbar', {
      color: 'error',
      text: `Failed to start: ${e?.message || e}`
    })
  }
}

    closeDialog() {
        this.$emit('closeDialog')
    }
}
</script>
<style scoped>
/* Pierce Vuetify internals */
.mesh-radio ::v-deep .v-input--radio-group__input,
.mesh-radio ::v-deep .v-input__slot {
  display: flex;
  flex-wrap: nowrap !important;
  align-items: center;
}

/* Keep each option on one line + spacing */
.mesh-radio ::v-deep .v-radio {
  white-space: nowrap;
  margin-right: 16px;
}
</style>