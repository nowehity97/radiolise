<template>
  <RadDrawer>
    <h3>
      <FasFileImport class="w-fixed" />
      {{ $t("general.importBackup") }}
    </h3>
    <p class="description mb-3.75 py-2.5">
      {{
        $t("importWizard.description", [
          $t(`importWizard.${type === "list" ? "stationLists" : "settings"}`),
        ])
      }}
    </p>
    <div class="text-left">
      <div>
        <h4 class="mt-0">
          <template v-if="type === 'list'">
            {{ $t("importWizard.step", [1, $t("importWizard.chooseFile")]) }}
          </template>
          <template v-else>
            {{ $t("importWizard.chooseFile") }}
          </template>
        </h4>
        <RadDropZone ref="dropZone" @change="setBackup" @error="handleError()" />
      </div>
      <template v-if="backup">
        <br />
        <template v-if="type === 'list'">
          <br />
          <div>
            <h4 class="mt-0">
              {{ $t("importWizard.step", [2, $t("importWizard.setName")]) }}
            </h4>
            <div class="inline-block">
              <RadInput
                v-model="listName"
                autocomplete="off"
                :placeholder="$t('general.newName')"
                select-on-focus
              />
            </div>
          </div>
          <br /><br />
          <div>
            <h4 class="mt-0">
              {{ $t("importWizard.step", [3, $t("importWizard.selectStations")]) }}
            </h4>
            <div>
              <RadResult
                v-for="(station, index) in stationList"
                :key="index"
                v-model="station.selected"
              >
                {{ station.name }}
                <template #tags>
                  <RadTags
                    :labels="[station.country, station.state, ...station.tags.split(',')]"
                  ></RadTags>
                </template>
              </RadResult>
            </div>
          </div>
          <br />
        </template>
        <div class="py-2.5 text-right">
          <RadLink v-slot="{ navigate }" :to="type === 'list' ? null : 'settings'">
            <RadButton @click="navigate"><FasBan /> {{ $t("general.cancel") }}</RadButton>
          </RadLink>
          <RadButton @click="importItems()"><FasArrowRight /> {{ $t("general.apply") }}</RadButton>
        </div>
      </template>
    </div>
  </RadDrawer>
</template>

<script lang="ts">
import { Component, Prop, Ref, Vue } from "vue-property-decorator";
import { Action } from "vuex-class";
import { type ModalOptions, ModalType } from "@/store";
import { navigate } from "@/common/routing";

import type RadDropZone from "@/components/RadDropZone.vue";

type BackupKind = Record<string, Station[]> | Settings;
type Backup = Record<string, string | BackupKind>;

@Component
export default class RadImportWizard extends Vue {
  appTitle = __APP_TITLE__;
  backup: SelectableStation[] | Settings | null = null;
  listName?: string;

  @Prop({ type: String, required: true }) readonly type!: "list" | "settings";

  @Ref() readonly dropZone!: RadDropZone;

  @Action applySettings!: (settings: Settings) => Promise<void>;
  @Action changeList!: (index: number) => Promise<void>;
  @Action createList!: (list: StationList) => Promise<void>;
  @Action showMessage!: (options: ModalOptions) => Promise<number>;

  @Action updateList!: (payload: { name?: string; content: Station[] }) => Promise<void>;

  get stationList() {
    return this.type === "list" ? (this.backup as SelectableStation[]) : null;
  }

  setBackup(rawBackup: Backup): void {
    if (rawBackup.version !== "2" || this.type !== rawBackup.type) {
      this.handleError();
      return;
    }

    const backup = rawBackup.data;

    if (this.type === "settings") {
      this.backup = backup as Settings;
      return;
    }

    const entries = Object.entries(backup)[0] as [string, Station[]];
    [this.listName] = entries;

    this.backup = entries[1].map((item: Station) => ({
      ...item,
      selected: true,
    }));
  }

  handleError(): void {
    this.showMessage({
      type: ModalType.ERROR,
      buttons: [this.$t("general.ok") as string],
      title: this.$t("general.error.fileNotSupported.title") as string,
      message: this.$t("general.error.fileNotSupported.description", [this.appTitle]) as string,
    });

    if (this.backup === null) {
      this.dropZone.reset();
    }
  }

  async importItems(): Promise<void> {
    if (this.type === "settings") {
      this.applySettings(this.backup as Settings);
      window.history.back();
      return;
    }

    const name = this.listName as string;

    const removeSelected = ({ selected, ...station }: SelectableStation): Station => station;

    const content = (this.backup as SelectableStation[])
      .filter((station: SelectableStation) => station.selected)
      .map((item: SelectableStation) => removeSelected(item));

    try {
      this.createList({ name, content });
      await this.$nextTick();
      this.changeList(-1);
      navigate(null);
    } catch (error: any) {
      if (error.message.includes("empty")) {
        this.showMessage({
          type: ModalType.WARNING,
          buttons: [this.$t("general.ok") as string],
          title: this.$t("importWizard.emptyName.title") as string,
          message: this.$t("importWizard.emptyName.description") as string,
        });
      } else {
        const buttonId = await this.showMessage({
          type: ModalType.QUESTION,
          title: this.$t("importWizard.nameTaken.title") as string,
          message: this.$t("importWizard.nameTaken.description") as string,
          buttons: [this.$t("general.yes") as string, this.$t("general.no") as string],
          closeable: false,
        });

        if (buttonId === 0) {
          navigate(null);
          this.updateList({ name, content });
        }
      }
    }
  }
}
</script>
