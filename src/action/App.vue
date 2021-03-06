<template>
  <div id="app" v-show="dataLoaded">
    <div class="header">
      <div class="title">{{ getText('extensionName') }}</div>
      <div class="header-buttons">
        <v-icon-button
          class="settings-button"
          @click="showActionSettings = !showActionSettings"
        >
          <img
            class="mdc-icon-button__icon"
            :src="`/src/icons/misc/${
              showActionSettings ? 'linkOn' : 'link'
            }.svg`"
          />
        </v-icon-button>

        <v-icon-button
          class="contribute-button"
          src="/src/contribute/assets/heart.svg"
          @click="showContribute"
        >
        </v-icon-button>

        <v-icon-button
          class="menu-button"
          src="/src/icons/misc/more.svg"
          @click="showActionMenu"
        >
        </v-icon-button>
      </div>

      <v-menu
        ref="actionMenu"
        class="action-menu"
        :ripple="true"
        :items="listItems.actionMenu"
        @selected="onActionMenuSelect"
      ></v-menu>
    </div>

    <transition
      name="settings"
      v-if="dataLoaded"
      @after-enter="settingsAfterEnter"
      @after-leave="settingsAfterLeave"
    >
      <div class="settings" v-show="showActionSettings">
        <v-textfield
          ref="pageUrlInput"
          v-model.trim="pageUrl"
          :placeholder="getText('inputPlaceholder_pageURL')"
          :fullwidth="true"
        >
        </v-textfield>
      </div>
    </transition>

    <div class="list-padding-top"></div>
    <ul class="mdc-list list list-bulk-button" v-if="searchAllEngines">
      <li class="mdc-list-item list-item" @click="selectItem('allEngines')">
        <img
          class="mdc-list-item__graphic list-item-icon"
          :src="getEngineIcon('allEngines')"
        />
        {{ getText('engineName_allEngines_full') }}
      </li>
    </ul>
    <ul
      class="mdc-list list list-separator"
      v-if="searchAllEngines || hasScrollBar"
    >
      <li role="separator" class="mdc-list-divider"></li>
    </ul>
    <div class="list-items-wrap" ref="items" :class="listClasses">
      <resize-observer @notify="handleSizeChange"></resize-observer>
      <ul class="mdc-list list list-items">
        <li
          class="mdc-list-item list-item"
          v-for="engine in engines"
          :key="engine.id"
          @click="selectItem(engine)"
        >
          <img
            class="mdc-list-item__graphic list-item-icon"
            :src="getEngineIcon(engine)"
          />
          {{ getText(`engineName_${engine}_short`) }}
        </li>
      </ul>
    </div>
  </div>
</template>

<script>
import browser from 'webextension-polyfill';
import {ResizeObserver} from 'vue-resize';
import {MDCList} from '@material/list';
import {MDCRipple} from '@material/ripple';
import {IconButton, TextField, Menu} from 'ext-components';

import storage from 'storage/storage';
import {
  getEnabledEngines,
  showNotification,
  validateUrl,
  getListItems,
  showContributePage,
  showProjectPage
} from 'utils/app';
import {getText, isAndroid} from 'utils/common';
import {targetEnv} from 'utils/config';

export default {
  components: {
    [IconButton.name]: IconButton,
    [TextField.name]: TextField,
    [Menu.name]: Menu,
    [ResizeObserver.name]: ResizeObserver
  },

  data: function () {
    return {
      dataLoaded: false,

      listItems: getListItems(
        {actionMenu: ['options', 'website']},
        {scope: 'actionMenu'}
      ),

      showActionSettings: false,
      hasScrollBar: false,
      isPopup: false,
      tabId: null,

      engines: [],
      searchAllEngines: false,
      pageUrl: ''
    };
  },

  computed: {
    listClasses: function () {
      return {
        'list-items-max-height': this.isPopup
      };
    }
  },

  methods: {
    getText,

    getEngineIcon: function (engine) {
      if (engine === 'googleText') {
        engine = 'google';
      } else if (engine === 'archiveOrgAll') {
        engine = 'archiveOrg';
      } else if (engine === 'archiveIsAll') {
        engine = 'archiveIs';
      }

      let ext = 'svg';
      if (['gigablast', 'megalodon'].includes(engine)) {
        ext = 'png';
      }

      return `/src/icons/engines/${engine}.${ext}`;
    },

    selectItem: async function (engine) {
      if (this.showActionSettings) {
        if (!validateUrl(this.pageUrl)) {
          this.focusPageUrlInput();
          showNotification({messageId: 'error_invalidUrl'});
          return;
        }
      }
      browser.runtime.sendMessage({
        id: 'actionPopupSubmit',
        pageUrl: this.pageUrl,
        engine
      });

      this.closeAction();
    },

    showContribute: async function () {
      await showContributePage();
      this.closeAction();
    },

    showOptions: async function () {
      await browser.runtime.openOptionsPage();
      this.closeAction();
    },

    showWebsite: async function () {
      await showProjectPage();
      this.closeAction();
    },

    showActionMenu: function () {
      this.$refs.actionMenu.$emit('open');
    },

    onActionMenuSelect: async function (item) {
      if (item === 'options') {
        await this.showOptions();
      } else if (item === 'website') {
        await this.showWebsite();
      }
    },

    closeAction: async function () {
      if (this.tabId) {
        browser.tabs.remove(this.tabId);
      } else {
        window.close();
      }
    },

    focusPageUrlInput: function () {
      this.$refs.pageUrlInput.$refs.input.focus();
    },

    handleSizeChange: function () {
      const items = this.$refs.items;
      this.hasScrollBar = items.scrollHeight > items.clientHeight;
    },

    settingsAfterEnter: function () {
      this.handleSizeChange();
      this.focusPageUrlInput();
    },

    settingsAfterLeave: function () {
      this.handleSizeChange();
      this.pageUrl = '';
    }
  },

  created: async function () {
    const currentTab = await browser.tabs.getCurrent();
    if (currentTab) {
      this.tabId = currentTab.id;
    }
    this.isPopup = !this.tabId && !this.$isFenix;
    if (!this.isPopup) {
      document.documentElement.style.height = '100%';
      document.body.style.minWidth = 'initial';
    }

    const options = await storage.get(
      ['engines', 'disabledEngines', 'searchAllEnginesAction'],
      'sync'
    );

    const enEngines = await getEnabledEngines(options);

    if (
      targetEnv === 'firefox' &&
      (await isAndroid()) &&
      (enEngines.length <= 1 || options.searchAllEnginesAction === 'main')
    ) {
      // Removing the action popup has no effect on Firefox for Android
      showNotification({messageId: 'error_optionsNotApplied'});
      return;
    }

    this.searchAllEngines = options.searchAllEnginesAction === 'sub';
    this.engines = enEngines;

    this.dataLoaded = true;
  },

  mounted: function () {
    window.setTimeout(() => {
      for (const listEl of document.querySelectorAll(
        '.list-bulk-button, .list-items'
      )) {
        const list = new MDCList(listEl);
        for (const el of list.listElements) {
          MDCRipple.attachTo(el);
        }
      }
    }, 500);
  }
};
</script>

<style lang="scss">
$mdc-theme-primary: #1abc9c;

@import '@material/icon-button/mdc-icon-button';
@import '@material/list/mdc-list';
@import '@material/icon-button/mixins';
@import '@material/theme/mixins';
@import '@material/textfield/mixins';
@import '@material/typography/mixins';

@import 'vue-resize/dist/vue-resize';

body,
#app {
  height: 100%;
}

#app {
  display: flex;
  flex-direction: column;
}

body {
  margin: 0;
  min-width: 326px;
  overflow: hidden;
  @include mdc-typography-base;
  font-size: 100%;
  background-color: #ffffff;
}

.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  white-space: nowrap;
  padding-top: 16px;
  padding-left: 16px;
  padding-right: 4px;
}

.title {
  overflow: hidden;
  text-overflow: ellipsis;
  @include mdc-typography(headline6);
  @include mdc-theme-prop(color, text-primary-on-light);
}

.header-buttons {
  display: flex;
  align-items: center;
  height: 24px;
  margin-left: 56px;
  @media (max-width: 325px) {
    margin-left: 32px;
  }
}

.contribute-button,
.settings-button,
.menu-button {
  @include mdc-icon-button-icon-size(24px, 24px, 6px);

  &::before {
    --mdc-ripple-fg-size: 20px;
    --mdc-ripple-fg-scale: 1.8;
    --mdc-ripple-left: 8px;
    --mdc-ripple-top: 8px;
  }
}

.contribute-button {
  margin-right: 4px;
}

.settings-button {
  margin-right: 12px;
}

.action-menu {
  left: auto !important;
  top: 56px !important;
  right: 16px;
  transform-origin: top right !important;
}

.settings {
  padding: 16px;
}

.settings-enter-active,
.settings-leave-active {
  max-height: 100px;
  padding-top: 16px;
  padding-bottom: 16px;
  transition: max-height 0.3s ease, padding-top 0.3s ease,
    padding-bottom 0.3s ease, opacity 0.2s ease;
}

.settings-enter,
.settings-leave-to {
  max-height: 0;
  padding-top: 0;
  padding-bottom: 0;
  opacity: 0;
}

.list {
  padding: 0 !important;
}

.list-padding-top {
  margin-bottom: 8px;
}

.list-bulk-button {
  position: relative;
  height: 48px;
}

.list-separator {
  position: relative;
  height: 1px;
}

.list-items-wrap {
  overflow-y: auto;
}

.list-items-max-height {
  max-height: 392px;
}

.list-items {
  padding-bottom: 8px !important;
}

.list-item {
  padding-left: 16px;
  padding-right: 48px;
  cursor: pointer;
}

.list-item-icon {
  margin-right: 16px !important;
}

html.fenix {
  height: 100%;
}
.fenix {
  & .title {
    visibility: hidden;
  }

  & .mdc-list-item {
    @include mdc-theme-prop(color, #20123a);
  }

  & .mdc-text-field {
    @include mdc-text-field-ink-color(#20123a);
    @include mdc-text-field-caret-color(#312a65);
    @include mdc-text-field-bottom-line-color(#20123a);
    @include mdc-text-field-line-ripple-color(#312a65);
  }

  & .settings-button img,
  & .menu-button img {
    filter: brightness(0) saturate(100%) invert(10%) sepia(43%) saturate(1233%)
      hue-rotate(225deg) brightness(97%) contrast(105%);
  }
}
</style>
