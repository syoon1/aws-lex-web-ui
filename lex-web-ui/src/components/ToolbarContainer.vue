<template>
  <v-toolbar
    v-bind:color="toolbarColor"
    app
    dark
    fixed
    v-if="!isUiMinimized"
    v-on="toolbarClickHandler"
    v-bind:dense="this.$store.state.isRunningEmbedded && !isUiMinimized"
    v-bind:class="{ minimized: isUiMinimized }"
    aria-label="Toolbar with sound FX mute button, minimise chat window button and option chat back a step button"
  >
    <img
      v-if="toolbarLogo"
      v-bind:src="toolbarLogo"
      alt="logo"
      aria-hidden="true"
    />

    <div class="nav-buttons">
      <v-btn
        small
        icon
        :disabled="isLexProcessing"
        class="nav-button-prev"
        v-on="prevNavEventHandlers"
        v-on:click="onPrev"
        v-show="hasPrevUtterance && !isUiMinimized && shouldRenderBackButton"
        aria-label="go back to previous message"
      >
        <v-icon> arrow_back </v-icon>
      </v-btn>
      <v-tooltip
        v-model="prevNav"
        activator=".nav-button-prev"
        content-class="tooltip-custom"
        right
      >
        <span>Previous</span>
      </v-tooltip>
    </div>

    <v-toolbar-title
      class="hidden-xs-and-down"
      v-on:click.stop="toggleMinimize"
      v-show="!isUiMinimized"
    >
      <h1>{{ toolbarTitle }}</h1>
    </v-toolbar-title>

    <v-toolbar-title class="hidden-xs-and-down" v-show="!isUiMinimized">
      {{ userName }}
    </v-toolbar-title>

    <v-spacer />
    <!-- tooltip should be before btn to avoid right margin issue in mobile -->
    <v-tooltip
      v-model="shouldShowTooltip"
      content-class="tooltip-custom"
      activator=".min-max-toggle"
      left
    >
      <span id="min-max-tooltip">{{ toolTipMinimize }}</span>
    </v-tooltip>
    <v-tooltip
      v-model="shouldShowHelpTooltip"
      content-class="tooltip-custom"
      activator=".help-toggle"
      left
    >
      <span id="help-tooltip">help</span>
    </v-tooltip>
    <v-tooltip
      v-model="shouldShowMenuTooltip"
      content-class="tooltip-custom"
      activator=".menu"
      left
    >
      <span id="menu-tooltip">menu</span>
    </v-tooltip>
    <v-btn
      v-if="shouldRenderHelpButton && !isUiMinimized"
      v-on:click="sendHelp"
      v-on="tooltipHelpEventHandlers"
      v-bind:disabled="isLexProcessing"
      icon
      class="help-toggle"
    >
      <v-icon>
        help_outline
      </v-icon>
    </v-btn>

    <v-menu v-if="showToolbarMenu" offset-y>
      <v-btn 
      slot="activator" 
      v-show="!isUiMinimized"
      v-on="tooltipMenuEventHandlers"
      class="menu"
      fab
      small
      aria-label="menu options">
        <v-icon>
          menu
        </v-icon>
      </v-btn>

      <v-list>
        <v-list-tile v-if="isEnableLogin">
          <v-list-tile-title v-if="isLoggedIn" v-on:click="requestLogout" aria-label="logout">
            <v-icon>
              {{ items[1].icon }}
            </v-icon>
            {{ items[1].title }}
          </v-list-tile-title>
          <v-list-tile-title v-if="!isLoggedIn" v-on:click="requestLogin" aria-label="login">
            <v-icon>
              {{ items[0].icon }}
            </v-icon>
            {{ items[0].title }}
            </v-list-tile-title>
        </v-list-tile>
        <v-list-tile v-if="isSaveHistory">
          <v-list-tile-title v-on:click="requestResetHistory" aria-label="clear chat history">
            <v-icon>
              {{ items[2].icon }}
            </v-icon>
            {{ items[2].title }}
          </v-list-tile-title>
        </v-list-tile>
        <v-list-tile v-if="shouldRenderSfxButton && isSFXOn">
          <v-list-tile-title v-on:click="toggleSFXMute" aria-label="mute sound effects">
            <v-icon>
              {{ items[3].icon }}
            </v-icon>
            {{ items[3].title }}
          </v-list-tile-title>
        </v-list-tile>
        <v-list-tile v-if="shouldRenderSfxButton && !isSFXOn">
          <v-list-tile-title v-on:click="toggleSFXMute" aria-label="unmute sound effects">
            <v-icon>
              {{ items[4].icon }}
            </v-icon>
            {{ items[4].title }}
          </v-list-tile-title>
        </v-list-tile>
        <v-list-tile v-if="canLiveChat">
          <v-list-tile-title v-on:click="requestLiveChat" aria-label="request live chat">
            <v-icon>
              {{ items[5].icon }}
            </v-icon>
            {{ items[5].title }}
          </v-list-tile-title>
        </v-list-tile>
        <v-list-tile v-if="isLiveChat">
          <v-list-tile-title v-on:click="endLiveChat" aria-label="end live chat">
            <v-icon>
              {{ items[6].icon }}
            </v-icon>
            {{ items[6].title }}
          </v-list-tile-title>
        </v-list-tile>
      </v-list>
    </v-menu>

    <v-btn
      v-if="$store.state.isRunningEmbedded"
      v-on:click.stop="toggleMinimize"
      v-on="tooltipEventHandlers"
      class="min-max-toggle"
      icon
      v-bind:aria-label="isUiMinimized ? 'chat' : 'minimize chat window toggle'"
    >
      <v-icon>
        {{ isUiMinimized ? "chat" : "arrow_drop_down" }}
      </v-icon>
    </v-btn>
  </v-toolbar>
</template>

<script>
/*
Copyright 2017-2020 Amazon.com, Inc. or its affiliates. All Rights Reserved.

Licensed under the Amazon Software License (the "License"). You may not use this file
except in compliance with the License. A copy of the License is located at

http://aws.amazon.com/asl/

or in the "license" file accompanying this file. This file is distributed on an "AS IS"
BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, express or implied. See the
License for the specific language governing permissions and limitations under the License.
*/
import { chatMode, liveChatStatus } from '@/store/state';

export default {
  name: 'toolbar-container',
  data() {
    return {
      items: [
        { title: 'Login', icon: 'login' },
        { title: 'Logout', icon: 'logout' },
        { title: 'Clear Chat', icon: 'delete' },
        { title: 'Mute', icon: 'volume_up' },
        { title: 'Unmute', icon: 'volume_off' },
        { title: 'Start Live Chat', icon: 'people_alt' },
        { title: 'End Live Chat', icon: 'call_end' },
      ],
      shouldShowTooltip: false,
      shouldShowHelpTooltip: false,
      shouldShowMenuTooltip: false,
      prevNav: false,
      prevNavEventHandlers: {
        mouseenter: this.mouseOverPrev,
        mouseleave: this.mouseOverPrev,
        touchstart: this.mouseOverPrev,
        touchend: this.mouseOverPrev,
        touchcancel: this.mouseOverPrev,
      },
      tooltipHelpEventHandlers: {
        mouseenter: this.onHelpButtonHoverEnter,
        mouseleave: this.onHelpButtonHoverLeave,
        touchstart: this.onHelpButtonHoverEnter,
        touchend: this.onHelpButtonHoverLeave,
        touchcancel: this.onHelpButtonHoverLeave,
      },
      tooltipMenuEventHandlers: {
        mouseenter: this.onMenuButtonHoverEnter,
        mouseleave: this.onMenuButtonHoverLeave,
        touchstart: this.onMenuButtonHoverEnter,
        touchend: this.onMenuButtonHoverLeave,
        touchcancel: this.onMenuButtonHoverLeave,
      },
      tooltipEventHandlers: {
        mouseenter: this.onInputButtonHoverEnter,
        mouseleave: this.onInputButtonHoverLeave,
        touchstart: this.onInputButtonHoverEnter,
        touchend: this.onInputButtonHoverLeave,
        touchcancel: this.onInputButtonHoverLeave,
      },
    };
  },
  props: [
    'toolbarTitle',
    'toolbarColor',
    'toolbarLogo',
    'isUiMinimized',
    'userName',
  ],
  computed: {
    toolbarClickHandler() {
      if (this.isUiMinimized) {
        return { click: this.toggleMinimize };
      }
      return null;
    },
    toolTipMinimize() {
      return this.isUiMinimized ? 'maximize' : 'minimize';
    },
    isEnableLogin() {
      return this.$store.state.config.ui.enableLogin;
    },
    isForceLogin() {
      return this.$store.state.config.ui.forceLogin;
    },
    hasPrevUtterance() {
      return this.$store.state.utteranceStack.length > 1;
    },
    isLoggedIn() {
      return this.$store.state.isLoggedIn;
    },
    isSaveHistory() {
      return this.$store.state.config.ui.saveHistory;
    },
    canLiveChat() {
      return (this.$store.state.config.ui.enableLiveChat &&
      this.$store.state.chatMode === chatMode.BOT &&
      (this.$store.state.liveChat.status === liveChatStatus.DISCONNECTED ||
      this.$store.state.liveChat.status === liveChatStatus.ENDED)
      );
    },
    isLiveChat() {
      return (this.$store.state.config.ui.enableLiveChat &&
      this.$store.state.chatMode === chatMode.LIVECHAT);
    },
    isLexProcessing() {
      return (
        this.$store.state.isBackProcessing || this.$store.state.lex.isProcessing
      );
    },
    shouldRenderHelpButton() {
      return !!this.$store.state.config.ui.helpIntent;
    },
    shouldRenderSfxButton() {
      return (
        this.$store.state.config.ui.enableSFX &&
        this.$store.state.config.ui.messageSentSFX &&
        this.$store.state.config.ui.messageReceivedSFX
      );
    },
    shouldRenderBackButton() {
      return this.$store.state.config.ui.backButton;
    },
    isSFXOn() {
      return this.$store.state.isSFXOn;
    },
    showToolbarMenu() {
      return (
        this.$store.state.config.ui.enableLogin ||
        this.$store.state.config.ui.saveHistory ||
        this.$store.state.config.ui.shouldRenderSfxButton ||
        this.$store.state.config.ui.enableLiveChat
      );
    },
  },
  methods: {
    mouseOverPrev() {
      this.prevNav = !this.prevNav;
    },
    onInputButtonHoverEnter() {
      this.shouldShowTooltip = !this.isUiMinimized;
    },
    onInputButtonHoverLeave() {
      this.shouldShowTooltip = false;
    },
    onHelpButtonHoverEnter() {
      this.shouldShowHelpTooltip = true;
    },
    onHelpButtonHoverLeave() {
      this.shouldShowHelpTooltip = false;
    },
    onMenuButtonHoverEnter() {
      this.shouldShowMenuTooltip = true;
    },
    onMenuButtonHoverLeave() {
      this.shouldShowMenuTooltip = false;
    },
    onNavHoverEnter() {
      this.shouldShowNavToolTip = true;
    },
    onNavHoverLeave() {
      this.shouldShowNavToolTip = false;
    },
    toggleSFXMute() {
      this.onInputButtonHoverLeave();
      this.$store.dispatch('toggleIsSFXOn');
    },
    toggleMinimize() {
      if (this.$store.state.isRunningEmbedded) {
        this.onInputButtonHoverLeave();
        this.$emit('toggleMinimizeUi');
      }
    },
    sendHelp() {
      const message = {
        type: 'human',
        text: this.$store.state.config.ui.helpIntent,
      };
      this.$store.dispatch('postTextMessage', message);
      this.shouldShowHelpTooltip = false;
    },
    onPrev() {
      if (this.prevNav) {
        this.mouseOverPrev();
      }
      if (!this.$store.state.isBackProcessing) {
        this.$store.commit('popUtterance');
        const lastUtterance = this.$store.getters.lastUtterance();
        if (lastUtterance && lastUtterance.length > 0) {
          const message = {
            type: 'human',
            text: lastUtterance,
          };
          this.$store.commit('toggleBackProcessing');
          this.$store.dispatch('postTextMessage', message);
        }
      }
    },
    requestLogin() {
      this.$emit('requestLogin');
    },
    requestLogout() {
      this.$emit('requestLogout');
    },
    requestResetHistory() {
      this.$store.dispatch('resetHistory');
    },
    requestLiveChat() {
      this.$emit('requestLiveChat');
    },
    endLiveChat() {
      this.$emit('endLiveChat');
    },
    toggleIsLoggedIn() {
      this.onInputButtonHoverLeave();
      this.$emit('toggleIsLoggedIn');
    },
  },
};
</script>
<style>
.toolbar-color {
  background-color: #003da5 !important;
}

.nav-buttons {
  padding: 0;
  margin-left: 8px !important;
}

.nav-button-prev {
  padding: 0;
  margin: 0;
}

.list .icon {
  width: 20px;
  height: 20px;
  margin-right: 8px;
}

.menu__content {
  border-radius: 4px;
}
</style>
