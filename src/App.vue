<template>
  <div
    id="app"
    ref="app"
    :class="[
      'min-h-screen',
      fullscreen
        ? 'absolute inset-y-0 -right-scrollbar left-0 overflow-y-auto bg-surface'
        : 'bg-default transition-[background-color] duration-2000 mobile:bg-surface mobile:transition-none',
      { 'dialog': dialog, 'cursor-grabbing': dragging, 'cursor-none': relaxed },
    ]"
    :style="{
      '--rad-color-accent': accentColor,
      '--rad-color-background': backgroundColor,
    }"
    @scroll="handleFullscreenScroll()"
  >
    <template v-if="ready">
      <RadPage />
      <RadDialogLayer />
      <RadRelaxCaption v-if="relaxed" />
      <RadVisualization v-if="visualizationActive" />
    </template>
    <RadStartup v-else />
  </div>
</template>

<script lang="ts">
import { Component, Ref, Watch, Mixins } from "vue-property-decorator";
import { State, Getter, Action } from "vuex-class";
import { formatDuration } from "date-fns";

import { ModalType } from "./store";
import { navigate } from "./common/routing";

import ColorChanger from "./mixins/ColorChanger";
import Hotkeys from "./mixins/Hotkeys";
import LikeHelper from "./mixins/LikeHelper";

const HelperMixins = Mixins(ColorChanger, Hotkeys, LikeHelper);

@Component
export default class App extends HelperMixins {
  inputEventTypes: Array<keyof GlobalEventHandlersEventMap> = [
    "mousemove",
    "mousedown",
    "keydown",
    "touchstart",
    "wheel",
  ];

  @Ref() readonly app!: HTMLDivElement;

  @State readonly currentDialog!: DialogState | null;
  @State readonly darkMode!: boolean;
  @State readonly fellAsleep!: boolean;
  @State readonly relaxed!: boolean;

  @Getter readonly colorScheme!: string | null;
  @Getter readonly dragging!: boolean;
  @Getter readonly language!: string;
  @Getter readonly lists!: StationList[];
  @Getter readonly ready!: boolean;
  @Getter readonly visualizationActive!: boolean;

  @Action confirmSleepTimer!: () => Promise<void>;
  @Action createList!: (list: StationList) => Promise<void>;
  @Action determineDateFnsLocale!: (locale: string) => Promise<Locale | undefined>;
  @Action setDarkMode!: (darkMode: boolean) => Promise<void>;
  @Action setRelaxTimer!: () => Promise<void>;
  @Action toggleNavbar!: (navbarShown: boolean) => Promise<void>;
  @Action unsetDateFnsLocale!: () => Promise<void>;

  get darkSchemeQuery(): MediaQueryList {
    return window.matchMedia("(prefers-color-scheme: dark)");
  }

  get dialog(): boolean {
    return this.currentDialog !== null;
  }

  get noOverflow(): boolean {
    return this.dialog || this.relaxed;
  }

  get relaxModeAllowed(): boolean {
    return this.settings.relax && this.playing && !this.hasVideo;
  }

  async created(): Promise<void> {
    await this.$nextTick();

    if (this.lists.length === 0) {
      this.createList({
        name: this.$t("general.defaultListName") as string,
        content: [],
      });
    }
  }

  @Watch("darkMode")
  onDarkModeToggled(dark: boolean) {
    document.documentElement.classList.toggle("dark", dark);
  }

  @Watch("noOverflow", { immediate: true })
  setOverflowAllowed(noOverflow: boolean): void {
    document.body.classList.toggle("overflow-y-hidden", noOverflow);
  }

  @Watch("relaxed", { immediate: true })
  onRelaxedChanged(): void {
    document.body.classList.toggle("lg:overflow-y-visible", !this.relaxed);
  }

  @Watch("fellAsleep")
  async handleFallenAsleep(fellAsleep: boolean): Promise<void> {
    if (fellAsleep) {
      const minutes = this.settings.sleepTimeout;
      const locale = await this.determineDateFnsLocale(this.$i18n.locale);

      let formattedDuration = formatDuration({ minutes }, { locale });
      formattedDuration = formattedDuration.charAt(0).toUpperCase() + formattedDuration.slice(1);

      await this.showMessage({
        buttons: [this.$t("general.ok") as string],
        type: ModalType.INFO,
        title: this.$t("sleepTimer.name") as string,
        message: this.$t("sleepTimer.streamStopped", {
          timePassed: this.$tc("sleepTimer.timePassed", minutes, {
            n: formattedDuration,
          }),
        }) as string,
      });

      this.confirmSleepTimer();
    }
  }

  @Watch("fullscreen")
  handleFullscreenChanged(fullscreen: boolean): void {
    if (fullscreen) {
      this.toggleNavbar(false);

      if (this.dialog) {
        navigate(null);
      }

      document.documentElement.classList.add("overflow-y-scroll");
      const scrollbarWidth = window.innerWidth - document.body.offsetWidth;
      document.documentElement.style.setProperty("--rad-scrollbar-width", `${scrollbarWidth}px`);
      document.documentElement.classList.remove("overflow-y-scroll");
    } else {
      this.toggleNavbar(true);
      document.documentElement.style.removeProperty("--rad-scrollbar-width");
      this.app.scrollTop = 0;
    }

    document.body.classList.toggle("overflow-hidden", fullscreen);
    this.setBackgroundColor();
  }

  @Watch("colorScheme")
  handleColorSchemeChanged(colorScheme: string): void {
    if (colorScheme === "auto") {
      this.darkSchemeQuery.addListener(this.applyColorScheme);
      this.applyColorScheme();
      return;
    }

    this.darkSchemeQuery.removeListener(this.applyColorScheme);
    this.setDarkMode(colorScheme === "dark");
  }

  @Watch("language", { immediate: true })
  handleLanguageChanged(locale: string): void {
    if (locale === "auto") {
      locale = this.detectLocale();
    }

    this.$i18n.locale = locale;
    document.documentElement.lang = locale;
    this.unsetDateFnsLocale();
  }

  @Watch("relaxModeAllowed")
  onRelaxModeAllowedChanged(relaxModeAllowed: boolean): void {
    if (relaxModeAllowed) {
      this.addInputListeners(this.setRelaxTimer);
      return;
    }
    this.removeInputListeners(this.setRelaxTimer);
  }

  detectLocale(): string {
    const preferredLocales = [
      ...new Set(
        [navigator.language, ...navigator.languages].map((language) => language.slice(0, 2))
      ),
    ];

    const detectedLocale = preferredLocales.find((locale) => {
      return Object.keys(this.$i18n.messages).includes(locale);
    });

    return detectedLocale ?? "en";
  }

  addInputListeners(listener: EventListener): void {
    for (const type of this.inputEventTypes) {
      window.addEventListener(type, listener);
    }
  }

  removeInputListeners(listener: EventListener): void {
    for (const type of this.inputEventTypes) {
      window.removeEventListener(type, listener);
    }
  }

  applyColorScheme(): void {
    this.setDarkMode(this.darkSchemeQuery.matches);
  }

  handleFullscreenScroll(): void {
    if (this.fullscreen) {
      this.toggleNavbar(this.app.scrollTop > 0);
    }
  }

  @Watch("settings.theme", { immediate: true })
  updateTheme(): void {
    document.documentElement.dataset.theme = this.settings.theme;
  }
}
</script>
