<template>
  <div :class="$style.wrapper" id="connect-web3">

    <span class="web3-ready-info">

      <button
        v-if="showConnectAction"
        :class="[$style.button, $style.buttonPrimary]"
        @click="isDialogOpen = true"
      >
        {{ $t("app.main.connectButton") }}
      </button>

      <Loading v-if="signerId !== null && !isValidated" />

      <span v-if="signerId !== null && isValidated">
        <NetworkIndicator
          :network_id="networkId"
          :required_network="requiredNetwork"
          :tiny="true"
        />
        <AccountIndicator
          :account="account"
          :tiny="true"
        />
        <button
          :class="[$style.button, $style.buttonTransparent]"
          @click="changeProvider"
        >{{ $t("app.main.changeButton") }}</button>
      </span>
    </span>

    <Modal
      v-if="isDialogOpen && !isValidated"
      :signerId="signerId"
      :clickHandler="dialogClick"
    >

      <SelectProvider
        v-if="!signerId"
        :providers="enabledProviders"
        class="modal-content"
        :selectProvider="selectProvider"
      />

      <Metamask
        v-if="signerId === 'metamask' && !isValidated"
        class="modal-content"
      />

      <WalletConnect
        v-if="signerId === 'walletConnect' && !isValidated"
        class="modal-content"
      />

      <Ledger
        v-if="signerId === 'ledger' && !isValidated"
        class="modal-content"
      />
    </Modal>

  </div>
</template>

<script>
import Vue from 'vue'
import { mapState, mapGetters } from 'vuex'
import VTooltip from 'v-tooltip'
import store from './store'
import i18n from './conf/i18n'
import web3ApiProviderApi from './conf/web3ProviderApi'

// Components
import Loading from './components/Loading'
import AsyncLoading from './components/AsyncLoading'
import AsyncError from './components/AsyncError'

import Modal from './components/Modal'

// Vue globals.
Vue.use(VTooltip)
Object.defineProperty(Vue.prototype, 'web3ProviderApi', {
  value: web3ApiProviderApi,
})

const asyncWrap = {
  loading: AsyncLoading,
  error: AsyncError,
  delay: 200,
  timeout: 8000,
}

export default {
  name: 'App',
  store,
  i18n,
  components: {
    Loading,
    Modal,
    // SelectProvider: () => import('./components/SelectProvider'),
    SelectProvider: () => ({ ...asyncWrap, component: import('./components/SelectProvider') }),
    Metamask: () => ({ ...asyncWrap, component: import('./components/signerDialogs/Metamask') }),
    WalletConnect: () => ({ ...asyncWrap, component: import('./components/signerDialogs/WalletConnect') }),
    Ledger: () => ({ ...asyncWrap, component: import('./components/signerDialogs/Ledger') }),
    NetworkIndicator: () => ({ ...asyncWrap, component: import('./components/NetworkIndicator') }),
    AccountIndicator: () => ({ ...asyncWrap, component: import('./components/AccountIndicator') }),
  },
  data() {
    return {
      isDialogOpen: false,
      isLoading: true
    }
  },
  props: {
    // These props are defined in the web-component.
    // In development mode they are taken from example-props.js
    dappName: {
      type: String,
      required: true,
    },
    requiredNetwork: {
      type: String,
      required: true,
    },
    rpcUrl: {
      type: String,
      required: true,
    },
    providers: {
      type: String,
      default: null,
    },
  },
  computed: {
    showConnectAction() {
      if (this.$store.state.signerId === null) {
        return true
      }
      if (this.$store.state.signerId) {
        if (Vue.prototype.web3ProviderApi[this.$store.state.signerId].createAutovalidateProvider) {
          return false
        }
      }
      return !this.$store.getters.isValidated
    },
    ...mapState([
      'enabledProviders',
      'signerId',
      'account',
      'resetProvider',
      'networkId',
    ]),
    ...mapGetters([
      'isValidated',
    ]),
  },
  beforeCreate() {
    // Verify and save initial properties to store.
    this.$store.dispatch({ ...{ type: 'initStore' }, ...this.$options.propsData })

    // Validate based on last provider selected.
    if (this.$store.state.signerId) {
      if (Vue.prototype.web3ProviderApi[this.$store.state.signerId].createAutovalidateProvider) {
        this.$store.dispatch('providerAutoValidate')
      }
      else {
        this.$store.commit('signerId', null)
      }
    }
  },
  watch: {
    isValidated: {
      handler(isValidated) {
        if (isValidated) {
          this.isDialogOpen = false
        }
      }
    },
  },
  methods: {
    async selectProvider(id) {
      this.$store.commit('signerId', id)
    },
    async changeProvider() {
      this.$store.dispatch('resetProvider')
      this.isDialogOpen = true
    },
    dialogClick(event) {
      if (event.target.hash === '#back') {
        this.$store.commit('signerId', null)
        this.$store.dispatch('resetProvider')
      }
      else if (!this.isDialogContent(event)) {
        this.isDialogOpen = false
        if (!this.isValidated) {
          this.$store.commit('signerId', null)
        }
      }
      event.preventDefault()
    },
    isDialogContent(event) {
      // Clicks outside .modal-content will close dialogue.
      if (!event.path) {
        return false
      }
      for (let i = 0, len = event.path.length; i < len; i += 1) {
        if (!event.path[i].classList) {
          return false
        }
        if (event.path[i].classList.contains('modal-content')) {
          return true
        }
      }
      return false
    },
  },
}
</script>

<!--
  Import Global styles
  Allow inlineing common icons only once.

  Note: Image paths resolve as SCSS import doesn't work.
-->
<style src="./styles/global.scss" lang="scss" />

<style lang="scss" module>
  @import "./styles/_abstract";
  .wrapper{
    font-family: -apple-system, system-ui, BlinkMacSystemFont, "SF Pro Text", Roboto, Helvetica, Arial, sans-serif;
    font-size: 1rem;
    line-height: 1.4;
    * {
      box-sizing: border-box;
    }
  }

  .button {
    @extend %button;
    &Primary {
      color: $color_primary_contrast;
      background: $color_primary;
      box-shadow: $button_shadow;
    }
    &Transparent {
      font-size: .78571429rem;
      text-decoration: underline;
      background: none;
      &:hover {
        border: 1px solid #d6d6d6;
        background: rgba(133, 133, 133, 0.22);
        box-shadow: $button_shadow;
      }
    }
  }
</style>
