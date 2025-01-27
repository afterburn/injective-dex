<template>
  <tr v-if="market">
    <td class="h-8 text-left cursor-pointer" @click="handleClickOnMarket">
      <div class="flex items-center justify-start">
        <div v-if="market.baseToken.logo" class="w-6 h-6">
          <img
            :src="market.baseToken.logo"
            :alt="market.baseToken.name"
            class="min-w-full h-auto rounded-full"
          />
        </div>
        <div class="ml-3">
          <span class="text-gray-200 font-semibold">
            {{ market.ticker }}
          </span>
        </div>
      </div>
    </td>

    <td class="h-8 text-left">
      <span
        class="pl-1"
        :class="{
          'text-aqua-500': orderTypeBuy,
          'text-red-500': !orderTypeBuy
        }"
      >
        {{ orderSideLocalized }}
      </span>
    </td>

    <td class="h-8 font-mono text-right">
      <v-number
        :decimals="
          market ? market.priceDecimals : UI_DEFAULT_PRICE_DISPLAY_DECIMALS
        "
        :number="price"
      />
    </td>
    <td class="h-8 text-right font-mono">
      <v-number
        :decimals="
          market ? market.quantityDecimals : UI_DEFAULT_AMOUNT_DISPLAY_DECIMALS
        "
        :number="quantity"
      />
    </td>
    <td class="h-8 text-right font-mono">
      <v-number
        :decimals="
          market ? market.quantityDecimals : UI_DEFAULT_AMOUNT_DISPLAY_DECIMALS
        "
        :number="unfilledQuantity"
      />
    </td>
    <td class="h-8 text-right">
      <v-number
        :decimals="
          market ? market.quantityDecimals : UI_DEFAULT_AMOUNT_DISPLAY_DECIMALS
        "
        :number="filledQuantity"
      />
    </td>
    <td class="h-8 font-mono text-right">
      <v-number
        :decimals="
          market ? market.priceDecimals : UI_DEFAULT_PRICE_DISPLAY_DECIMALS
        "
        :number="total"
      >
        <span slot="addon" class="text-2xs text-gray-500">
          {{ market.quoteToken.symbol }}
        </span>
      </v-number>
    </td>
    <td class="h-8 relative">
      <div class="flex items-center justify-end">
        <span
          v-if="false"
          class="cursor-pointer text-primary-500 mr-6"
          @click="handleClickOnMarket"
        >
          {{ $t('common.view') }}
        </span>
        <v-button v-if="orderFillable" :status="status" @click="onCancelOrder">
          <div
            class="flex items-center justify-center rounded-full bg-red-550 bg-opacity-10 w-8 h-8 hover:bg-red-600 text-red-550 hover:text-red-600 hover:bg-opacity-10"
          >
            <v-icon-bin />
          </div>
        </v-button>
        <span v-else class="inline-block">&mdash;</span>
      </div>
    </td>
  </tr>
</template>

<script lang="ts">
import Vue, { PropType } from 'vue'
import {
  BigNumberInBase,
  BigNumberInWei,
  Status,
  StatusType
} from '@injectivelabs/utils'
import {
  SpotOrderSide,
  UiSpotLimitOrder,
  UiSpotMarketWithToken,
  ZERO_IN_BASE
} from '@injectivelabs/ui-common'
import {
  UI_DEFAULT_AMOUNT_DISPLAY_DECIMALS,
  UI_DEFAULT_PRICE_DISPLAY_DECIMALS
} from '~/app/utils/constants'

export default Vue.extend({
  props: {
    order: {
      required: true,
      type: Object as PropType<UiSpotLimitOrder>
    }
  },

  data() {
    return {
      UI_DEFAULT_AMOUNT_DISPLAY_DECIMALS,
      UI_DEFAULT_PRICE_DISPLAY_DECIMALS,
      SpotOrderSide,
      status: new Status(),
      statusTest: new Status(StatusType.Loading)
    }
  },

  computed: {
    currentMarket(): UiSpotMarketWithToken | undefined {
      return this.$accessor.spot.market
    },

    markets(): UiSpotMarketWithToken[] {
      return this.$accessor.spot.markets
    },

    market(): UiSpotMarketWithToken | undefined {
      const { markets, order } = this

      return markets.find((m) => m.marketId === order.marketId)
    },

    orderTypeBuy(): boolean {
      const { order } = this

      return order.orderSide === SpotOrderSide.Buy
    },

    price(): BigNumberInBase {
      const { market, order } = this

      if (!market) {
        return ZERO_IN_BASE
      }

      return new BigNumberInBase(
        new BigNumberInBase(order.price).toWei(
          market.baseToken.decimals - market.quoteToken.decimals
        )
      )
    },

    quantity(): BigNumberInBase {
      const { market, order } = this

      if (!market) {
        return ZERO_IN_BASE
      }

      return new BigNumberInWei(order.quantity).toBase(
        market.baseToken.decimals
      )
    },

    unfilledQuantity(): BigNumberInBase {
      const { market, order } = this

      if (!market) {
        return ZERO_IN_BASE
      }

      return new BigNumberInWei(order.unfilledQuantity).toBase(
        market.baseToken.decimals
      )
    },

    filledQuantity(): BigNumberInBase {
      const { unfilledQuantity, quantity } = this

      return quantity.minus(unfilledQuantity)
    },

    filledQuantityPercentage(): BigNumberInBase {
      const { filledQuantity, quantity, market } = this

      if (!market) {
        return ZERO_IN_BASE
      }

      if (filledQuantity.lte(0)) {
        return ZERO_IN_BASE
      }

      return new BigNumberInBase(filledQuantity.dividedBy(quantity))
    },

    orderFullyFilled(): boolean {
      const { unfilledQuantity } = this

      return unfilledQuantity.isZero()
    },

    orderFillable(): boolean {
      const { unfilledQuantity, quantity } = this

      return unfilledQuantity.lte(quantity)
    },

    total(): BigNumberInBase {
      const { price, quantity } = this

      return quantity.multipliedBy(price)
    },

    orderSideLocalized(): string {
      const { order } = this

      return order.orderSide === SpotOrderSide.Buy
        ? this.$t('trade.buy')
        : this.$t('trade.sell')
    }
  },

  methods: {
    onCancelOrder(): void {
      this.status.setLoading()

      this.$accessor.spot
        .cancelOrder(this.order)
        .then(() => {
          this.$toast.success(this.$t('trade.order_success_canceling'))
        })
        .catch(this.$onRejected)
        .finally(() => {
          this.status.setIdle()
        })
    },

    handleClickOnMarket() {
      const { market } = this

      if (!market) {
        return
      }

      return this.$router.push({
        name: 'spot-spot',
        params: {
          marketId: market.marketId,
          spot: market.slug
        }
      })
    }
  }
})
</script>
