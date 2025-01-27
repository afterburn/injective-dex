<template>
  <v-modal
    :is-open="isModalOpen"
    sm
    :modal-closed:animation="handleResetBridge"
    @modal-closed="handleCloseModal"
  >
    <div slot="title">
      <h3 class="flex items-center">
        {{ bridgeTitle }}
        <v-icon-info-tooltip
          v-if="bridgeType === BridgeType.Transfer"
          class="ml-2"
          :tooltip="$t('bridge.transferTitleTooltip')"
        />
      </h3>
    </div>
    <ValidationObserver
      v-if="isUserWalletConnected"
      v-slot="{ invalid }"
      ref="form"
    >
      <div>
        <v-transfer-direction-switch
          v-if="bridgeType === BridgeType.Transfer"
          v-bind="{ transferDirection }"
          @transfer-direction:switch="handleTransferDirectionSwitch"
        />
        <v-network-select
          v-else
          v-bind="{ value: bridgingNetwork, bridgeType }"
          @bridging-network:change="handleBridgingNetworkSwitch"
        >
          <template slot="title">{{ networkSelectorTitle }}</template>
        </v-network-select>
      </div>
      <div v-if="isWithdrawToInjectiveAddress" class="mt-6">
        <ValidationProvider
          v-slot="{ errors, valid }"
          name="form.destination"
          :rules="`required|injaddress`"
        >
          <v-input
            :value="form.destinationAddress"
            :errors="errors"
            :valid="valid"
            placeholder="inj"
            :label="$t('bridge.injAddress')"
            @input="handleDestinationAddressChange"
          >
          </v-input>
        </ValidationProvider>
      </div>
      <div v-if="!isIbcTransfer" class="mt-6">
        <div v-if="hasAllowance">
          <v-balance :balance="balance" :token="form.token" class="mb-2" />
          <v-token-selector
            :amount="form.amount"
            :value="form.token"
            :origin="origin"
            :destination="destination"
            :is-ibc-transfer="isIbcTransfer"
            :balance="balance"
            @input:amount="handleAmountChange"
            @input:token="handleTokenChange"
          >
          </v-token-selector>
        </div>
        <div class="mt-8 text-center">
          <v-allowance v-if="!hasAllowance" :token-with-balance="form.token" />

          <v-button
            v-else
            lg
            primary
            class="w-full xs:w-1/2 font-bold"
            :disabled="invalid"
            @click="handleTransferNowClick"
          >
            {{ $t('bridge.transferNow') }}
          </v-button>
        </div>
      </div>
      <v-ibc-transfer-note v-else />
    </ValidationObserver>
    <v-user-wallet-connect-warning v-else />
  </v-modal>
</template>

<script lang="ts">
import Vue, { PropType } from 'vue'
import { ValidationObserver, ValidationProvider } from 'vee-validate'
import {
  BankBalanceWithToken,
  BridgingNetwork,
  SubaccountBalanceWithToken,
  Token,
  TokenWithBalanceAndPrice,
  ZERO_IN_BASE
} from '@injectivelabs/ui-common'
import { BigNumberInBase, BigNumberInWei } from '@injectivelabs/utils'
import { BridgeType, Modal, TransferDirection } from '~/types'
import VTokenSelector from '~/components/partials/portfolio/bridge/token-selector/index.vue'
import VAllowance from '~/components/elements/allowance.vue'
import VBalance from '~/components/partials/portfolio/bridge/balance.vue'
import VNetworkSelect from '~/components/partials/portfolio/bridge/network-select.vue'
import VIbcTransferNote from '~/components/partials/portfolio/bridge/ibc-transfer-note.vue'
import VTransferDirectionSwitch from '~/components/partials/portfolio/bridge/transfer-direction-switch.vue'

export default Vue.extend({
  components: {
    ValidationObserver,
    VTokenSelector,
    VNetworkSelect,
    VIbcTransferNote,
    VTransferDirectionSwitch,
    ValidationProvider,
    VAllowance,
    VBalance
  },

  props: {
    bridgeType: {
      required: true,
      type: String as PropType<BridgeType>
    },

    bridgingNetwork: {
      required: true,
      type: String as PropType<BridgingNetwork>
    },

    transferDirection: {
      required: true,
      type: String as PropType<TransferDirection>
    },

    origin: {
      required: true,
      type: String as PropType<BridgingNetwork | TransferDirection>
    },

    destination: {
      required: true,
      type: String as PropType<BridgingNetwork | TransferDirection>
    },

    isIbcTransfer: {
      required: true,
      type: Boolean
    },

    form: {
      required: true,
      type: Object as PropType<{
        token: Token
        destinationAddress: string
        amount: string
      }>
    }
  },

  data() {
    return {
      BridgeType,
      TransferDirection,
      BridgingNetwork
    }
  },

  computed: {
    isUserWalletConnected(): boolean {
      return this.$accessor.wallet.isUserWalletConnected
    },

    erc20TokensWithBalanceAndPriceFromBank(): TokenWithBalanceAndPrice[] {
      return this.$accessor.token.erc20TokensWithBalanceAndPriceFromBank
    },

    subaccountBalancesWithToken(): SubaccountBalanceWithToken[] {
      return this.$accessor.account.subaccountBalancesWithToken
    },

    bankBalancesWithToken(): BankBalanceWithToken[] {
      return this.$accessor.bank.bankErc20BalancesWithToken
    },

    ibcBankBalancesWithToken(): BankBalanceWithToken[] {
      return this.$accessor.bank.bankIbcBalancesWithToken
    },

    isWithdrawToInjectiveAddress(): boolean {
      const { bridgeType, destination } = this

      if (
        bridgeType === BridgeType.Withdraw &&
        destination === BridgingNetwork.Injective
      ) {
        return true
      }

      return false
    },

    bridgeTitle(): string {
      const { bridgeType } = this

      if (bridgeType === BridgeType.Transfer) {
        return this.$t('bridge.transferFromToTradingAccount')
      }

      if (bridgeType === BridgeType.Deposit) {
        return this.$t('bridge.depositToInjective')
      }

      // Withdraw
      return this.$t('bridge.withdrawFromInjective')
    },

    networkSelectorTitle(): string {
      const { bridgeType } = this

      if (bridgeType === BridgeType.Deposit) {
        return this.$t('bridge.selectOriginNetwork')
      }

      // Withdraw
      return this.$t('bridge.selectDestinationNetwork')
    },

    hasAllowance(): boolean {
      const { bridgeType, erc20TokensWithBalanceAndPriceFromBank, form } = this

      const token = erc20TokensWithBalanceAndPriceFromBank.find(
        ({ denom }) => denom === form.token.denom
      )

      if ([BridgeType.Transfer, BridgeType.Withdraw].includes(bridgeType)) {
        return true
      }

      if (!token) {
        return false
      }

      return new BigNumberInBase(token.allowance).gt('0')
    },

    onTransferBalance(): BigNumberInBase {
      const {
        form,
        bridgeType,
        transferDirection,
        subaccountBalancesWithToken,
        bankBalancesWithToken,
        ibcBankBalancesWithToken
      } = this

      if (bridgeType !== BridgeType.Transfer) {
        return ZERO_IN_BASE
      }

      if (transferDirection === TransferDirection.bankToTradingAccount) {
        const balances = [...bankBalancesWithToken, ...ibcBankBalancesWithToken]
        const balance = balances.find(
          (balance) => balance.token.denom === form.token.denom
        )

        if (!balance) {
          return ZERO_IN_BASE
        }

        return new BigNumberInWei(balance.balance || 0).toBase(
          balance.token.decimals
        )
      }

      const balance = subaccountBalancesWithToken.find(
        (balance) => balance.denom === form.token.denom
      )

      if (!balance) {
        return ZERO_IN_BASE
      }

      return new BigNumberInWei(balance.availableBalance || 0).toBase(
        balance.token.decimals
      )
    },

    onDepositBalance(): BigNumberInBase {
      const { bridgeType, form, erc20TokensWithBalanceAndPriceFromBank } = this

      if (bridgeType !== BridgeType.Deposit) {
        return ZERO_IN_BASE
      }

      const tokenWithBalance = erc20TokensWithBalanceAndPriceFromBank.find(
        (token) => token.denom === form.token.denom
      )

      if (!tokenWithBalance) {
        return ZERO_IN_BASE
      }

      return new BigNumberInWei(tokenWithBalance.balance).toBase(
        tokenWithBalance.decimals
      )
    },

    onWithdrawBalance(): BigNumberInBase {
      const {
        bridgeType,
        form,
        bankBalancesWithToken,
        ibcBankBalancesWithToken
      } = this

      if (bridgeType !== BridgeType.Withdraw) {
        return ZERO_IN_BASE
      }

      const balances = [...bankBalancesWithToken, ...ibcBankBalancesWithToken]
      const balance = balances.find(
        (balance) => balance.token.denom === form.token.denom
      )

      if (!balance) {
        return ZERO_IN_BASE
      }

      return new BigNumberInWei(balance.balance || 0).toBase(
        balance.token.decimals
      )
    },

    balance(): BigNumberInBase {
      const {
        bridgeType,
        onTransferBalance,
        onDepositBalance,
        onWithdrawBalance
      } = this

      if (bridgeType === BridgeType.Transfer) {
        return onTransferBalance
      }

      if (bridgeType === BridgeType.Deposit) {
        return onDepositBalance
      }

      // Withdraw
      return onWithdrawBalance
    },

    isModalOpen(): boolean {
      return this.$accessor.modal.modals[Modal.Bridge]
    },

    $form(): InstanceType<typeof ValidationObserver> {
      return this.$refs.form as InstanceType<typeof ValidationObserver>
    }
  },

  methods: {
    handleTransferNowClick() {
      this.$emit('bridge:confirm')
    },

    handleAmountChange(amount: string) {
      this.$emit('input-amount:update', amount)
    },

    handleTokenChange(token: Token) {
      this.$emit('input-token:update', token)
      this.resetForm()
    },

    handleDestinationAddressChange(address: string) {
      this.$emit('input-destinationAddress:update', address)
    },

    handleCloseModal() {
      this.$accessor.modal.closeModal(Modal.Bridge)
    },

    handleResetBridge() {
      this.$emit('bridge:reset')
    },

    handleTransferDirectionSwitch() {
      this.$emit(
        'transfer-direction:update',
        this.transferDirection === TransferDirection.bankToTradingAccount
          ? TransferDirection.tradingAccountToBank
          : TransferDirection.bankToTradingAccount
      )
      this.resetForm()
    },

    handleBridgingNetworkSwitch(bridgingNetwork: BridgingNetwork) {
      this.$emit('bridging-network:update', bridgingNetwork)
    },

    resetForm() {
      const { $form } = this

      if ($form) {
        $form.reset()
      }
    }
  }
})
</script>
