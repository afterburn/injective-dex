<template>
  <TableRow md class="text-sm md:grid-cols-2 xl:grid-cols-12">
    <span class="font-mono text-left xl:hidden">
      {{ $t('portfolio.asset') }}
    </span>
    <span class="xl:col-span-2">
      <div class="flex items-center justify-end xl:justify-start">
        <div v-if="balance.token.logo" class="w-6 h-6">
          <img
            :src="balance.token.logo"
            :alt="balance.token.name"
            class="min-w-full h-auto rounded-full"
          />
        </div>
        <div class="ml-3">
          <span class="text-gray-200 font-bold">
            {{ balance.token.symbol }}
          </span>
        </div>
      </div>
    </span>
    <span class="font-mono text-left xl:hidden">
      {{ $t('portfolio.total') }}
    </span>
    <span class="xl:col-span-2 font-mono text-right whitespace-nowrap">
      <span v-if="hideBalance">{{ HIDDEN_BALANCE_DISPLAY }}</span>
      <span v-else>{{ bankBalanceToString }}</span>
    </span>
    <span class="font-mono text-left xl:hidden">
      {{ $t('common.value') }}
    </span>
    <span class="xl:col-span-4 font-mono whitespace-nowrap text-right">
      <span v-if="hideBalance">
        {{ HIDDEN_BALANCE_DISPLAY }}
      </span>
      <span v-else class="flex items-end justify-end flex-col">
        <span class="leading-4">{{ totalInUsdToString }} USD</span>
        <span
          v-if="totalInBtc.gt(0)"
          class="text-opacity-50 text-gray-200 text-2xs xs:ml-1 leading-4"
        >
          ≈ {{ totalInBtcToString }} BTC
        </span>
      </span>
    </span>
    <div
      class="col-span-2 xl:col-span-4 text-right text-primary-500 text-sm flex justify-around sm:justify-end items-center"
    >
      <span class="cursor-pointer" @click="handleDepositClick">
        {{ $t('common.deposit') }}
      </span>
      <span class="cursor-pointer ml-6" @click="handleWithdrawClick">{{
        $t('common.withdraw')
      }}</span>
      <span class="cursor-pointer ml-6" @click="handleTransferClick">
        {{ $t('common.transfer') }}
      </span>
    </div>
  </TableRow>
</template>

<script lang="ts">
import Vue, { PropType } from 'vue'
import { BigNumberInBase, BigNumberInWei } from '@injectivelabs/utils'
import {
  INJECTIVE_DENOM,
  ZERO_IN_BASE,
  BankBalanceWithTokenAndBalanceWithUsdBalance
} from '@injectivelabs/ui-common'
import TableRow from '~/components/elements/table-row.vue'
import {
  HIDDEN_BALANCE_DISPLAY,
  UI_DEFAULT_MIN_DISPLAY_DECIMALS,
  UI_DEFAULT_DISPLAY_DECIMALS
} from '~/app/utils/constants'

export default Vue.extend({
  components: {
    TableRow
  },

  props: {
    balance: {
      required: true,
      type: Object as PropType<BankBalanceWithTokenAndBalanceWithUsdBalance>
    },

    hideBalance: {
      type: Boolean,
      default: false
    }
  },

  data() {
    return {
      HIDDEN_BALANCE_DISPLAY,
      INJECTIVE_DENOM
    }
  },

  computed: {
    btcUsdPrice(): number {
      return this.$accessor.token.btcUsdPrice
    },

    bankBalance(): BigNumberInBase {
      const { balance } = this

      if (!balance.balance) {
        return ZERO_IN_BASE
      }

      return new BigNumberInWei(balance.balance).toBase(balance.token.decimals)
    },

    totalInBtc(): BigNumberInBase {
      const { balance, btcUsdPrice } = this

      if (!btcUsdPrice) {
        return ZERO_IN_BASE
      }

      return new BigNumberInBase(balance.balanceInUsd).dividedBy(
        new BigNumberInBase(btcUsdPrice)
      )
    },

    bankBalanceToString(): string {
      const { bankBalance } = this

      return bankBalance.toFormat(
        UI_DEFAULT_DISPLAY_DECIMALS,
        BigNumberInBase.ROUND_DOWN
      )
    },

    totalInBtcToString(): string {
      const { totalInBtc } = this

      if (totalInBtc.eq('0')) {
        return '0.00'
      }

      if (totalInBtc.lte('0.0001')) {
        return '<0.0001'
      }

      return totalInBtc.toFormat(UI_DEFAULT_DISPLAY_DECIMALS)
    },

    totalInUsdToString(): string {
      const { balance } = this

      return new BigNumberInBase(balance.balanceInUsd).toFormat(
        UI_DEFAULT_MIN_DISPLAY_DECIMALS
      )
    }
  },

  methods: {
    handleDepositClick() {
      const { balance } = this

      this.$root.$emit('bridge:deposit', balance.token)
    },

    handleWithdrawClick() {
      const { balance } = this

      this.$root.$emit('bridge:withdraw', balance.token)
    },

    handleTransferClick() {
      const { balance } = this

      this.$root.$emit('bridge:transfer', balance.token)
    }
  }
})
</script>
