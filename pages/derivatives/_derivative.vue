<template>
  <VHocLoading :key="$route.fullPath" :status="status">
    <div
      v-if="market"
      class="flex flex-col flex-wrap min-h-screen-excluding-header"
    >
      <div class="w-full px-1">
        <v-market />
      </div>
      <div class="flex-1 grid grid-cols-6 lg:grid-cols-12 gap-1 p-1">
        <div class="col-span-6 lg:col-span-3 4xl:col-span-3">
          <div class="flex flex-col flex-wrap h-full w-full">
            <v-balances />
            <v-trading class="mt-1 flex-1" />
          </div>
        </div>
        <div class="col-span-6 lg:col-span-9 4xl:col-span-9">
          <div class="flex flex-wrap flex-col w-full h-full">
            <div class="w-full">
              <v-card tight>
                <div class="grid grid-cols-6 lg:grid-cols-12">
                  <div class="col-span-6 lg:col-span-8 4xl:col-span-9">
                    <v-market-chart :market="market" />
                  </div>
                  <div class="col-span-6 lg:col-span-4 4xl:col-span-3">
                    <v-orderbook class="p-2 lg:p-3" />
                  </div>
                </div>
              </v-card>
            </div>
            <div class="w-full flex-1 mt-1">
              <v-orders />
            </div>
          </div>
        </div>
      </div>
      <v-modal-add-margin />
      <v-modal-market-beta v-if="marketIsBeta" />
    </div>
  </VHocLoading>
</template>

<script lang="ts">
import Vue from 'vue'
import { Status, StatusType } from '@injectivelabs/utils'
import { UiDerivativeMarketWithToken } from '@injectivelabs/ui-common'
import VModalAddMargin from '~/components/partials/modals/add-margin/index.vue'
import VModalMarketBeta from '~/components/partials/modals/market-beta.vue'
import VBalances from '~/components/partials/common/balances/index.vue'
import VTrading from '~/components/partials/derivatives/trading/index.vue'
import VMarketChart from '~/components/partials/common/market/chart.vue'
import VMarket from '~/components/partials/derivatives/market.vue'
import VOrders from '~/components/partials/derivatives/orders.vue'
import VOrderbook from '~/components/partials/derivatives/orderbook.vue'
import { Modal } from '~/types'
import { ORDERBOOK_POLLING_ENABLED } from '~/app/utils/constants'
import { betaMarketSlugs } from '~/app/data/market'

export default Vue.extend({
  components: {
    VModalAddMargin,
    VModalMarketBeta,
    VTrading,
    VBalances,
    VOrders,
    VOrderbook,
    VMarketChart,
    VMarket
  },

  data() {
    return {
      status: new Status(StatusType.Loading),
      interval: 0 as any
    }
  },

  computed: {
    market(): UiDerivativeMarketWithToken | undefined {
      return this.$accessor.derivatives.market
    },

    marketIsBeta(): boolean {
      const { params } = this.$route

      return betaMarketSlugs.includes(params.derivative)
    },

    slugFromRoute(): string {
      const { params } = this.$route

      return params.derivative
    }
  },

  mounted() {
    this.$accessor.derivatives
      .initMarket(this.slugFromRoute)
      .then(() => {
        this.setOrderbookPolling()
        this.$accessor.derivatives.initMarketStreams()
      })
      .catch(this.$onRejected)
      .finally(() => {
        this.status.setIdle()

        if (this.marketIsBeta) {
          this.$accessor.modal.openModal(Modal.MarketBeta)
        }
      })

    Promise.all([
      this.$accessor.exchange.fetchTradingRewardsCampaign(),
      this.$accessor.exchange.fetchFeeDiscountAccountInfo()
    ])
      .then(() => {
        //
      })
      .catch(this.$onRejected)
      .finally(() => {
        //
      })
  },

  beforeDestroy() {
    this.$accessor.derivatives.reset()
    this.$accessor.modal.reset()
    clearInterval(this.interval)
  },

  methods: {
    setOrderbookPolling() {
      if (ORDERBOOK_POLLING_ENABLED) {
        this.interval = setInterval(async () => {
          await this.$accessor.derivatives.pollOrderbook()
        }, 2000)
      }
    }
  }
})
</script>
