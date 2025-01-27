<template>
  <v-panel :title="$t('tradeAndEarn.pendingRewards')">
    <div v-if="pendingRewardsStartTimestamp > 0" slot="title-context">
      {{ $t('tradeAndEarn.campaignAsOf', { date: pendingRewardsCountdown }) }}
    </div>
    <VHocLoading :status="status">
      <div class="grid grid-cols-2 lg:grid-cols-12 gap-4 lg:gap-6">
        <v-item class="col-span-2 lg:col-span-4">
          <template slot="value">
            <v-emp-number
              :number="injMaxPendingCampaignRewards"
              :decimals="UI_DEFAULT_MIN_DISPLAY_DECIMALS"
            >
              <span>INJ</span>
            </v-emp-number>

            <v-emp-number
              sm
              class="text-gray-400"
              prefix="≈"
              :number="injMaxPendingCampaignRewardsInUsd"
              :decimals="UI_DEFAULT_MIN_DISPLAY_DECIMALS"
            >
              <span class="text-3xs">USD</span>
            </v-emp-number>
          </template>
          <template slot="title">
            <div class="flex items-center justify-center">
              {{ $t('tradeAndEarn.pending_max_campaign_rewards') }}
              <v-icon-info-tooltip
                class="ml-2"
                :tooltip="
                  $t('tradeAndEarn.pending_max_campaign_rewards_tooltip')
                "
              />
            </div>
          </template>
        </v-item>
        <v-item class="col-span-2 lg:col-span-4">
          <template slot="value">
            <div
              v-if="isUserWalletConnected"
              class="flex flex-wrap justify-center"
            >
              <v-emp-number :number="pendingTradeRewardPointsFactored">
                <span>{{ $t('pts') }}</span>
              </v-emp-number>
              <span class="px-2">/</span>
              <v-emp-number :number="totalPendingTradeRewardPointsFactored">
                <span>{{ $t('pts') }}</span>
              </v-emp-number>
            </div>
            <span v-else>&mdash;</span>
          </template>
          <template slot="title">
            <div class="flex items-center justify-center">
              {{ $t('tradeAndEarn.myRewardPoints') }}
              <v-icon-info-tooltip
                class="ml-2"
                :tooltip="$t('tradeAndEarn.myRewardPoints_tooltip')"
              />
            </div>
          </template>
        </v-item>
        <v-item class="col-span-2 lg:col-span-4">
          <template slot="value">
            <v-emp-number
              v-if="isUserWalletConnected"
              :number="pendingEstimatedRewardsCapped"
              :decimals="UI_DEFAULT_MIN_DISPLAY_DECIMALS"
            >
              <span>INJ</span>
            </v-emp-number>
            <span v-else>&mdash;</span>
            <v-emp-number
              v-if="isUserWalletConnected"
              sm
              class="text-gray-400"
              prefix="≈"
              :number="pendingEstimatedRewardsCappedInUsd"
              :decimals="UI_DEFAULT_MIN_DISPLAY_DECIMALS"
            >
              <span class="text-3xs">USD</span>
            </v-emp-number>
          </template>
          <template
            v-if="
              pendingEstimatedRewards.gt(0) &&
              pendingEstimatedRewardsCapped.lte(pendingEstimatedRewards)
            "
            slot="context"
          >
            <a
              v-if="isUserWalletConnected"
              :href="hubUrl"
              class="text-primary-500 flex justify-center"
              target="_blank"
            >
              {{ $t('stake_more') }}
              <v-icon-info-tooltip
                class="ml-2"
                :tooltip="
                  $t('tradeAndEarn.stake_total_to_receive_full_amount', {
                    total: pendingEstimatedRewards.toFormat(2)
                  })
                "
              />
            </a>
          </template>
          <template slot="title">
            <div class="flex items-center justify-center">
              {{ $t('tradeAndEarn.est_rewards_stake') }}
              <v-icon-info-tooltip
                class="ml-2"
                :tooltip="
                  $t('tradeAndEarn.est_rewards_stake_tooltip', {
                    maxRewards: DEFAULT_CAPPED_TRADE_AND_EARN_REWARDS
                  })
                "
              />
            </div>
          </template>
        </v-item>
      </div>
    </VHocLoading>
  </v-panel>
</template>

<script lang="ts">
import {
  BigNumberInBase,
  BigNumberInWei,
  Status,
  StatusType
} from '@injectivelabs/utils'
import { format } from 'date-fns'
import Vue from 'vue'
import { ZERO_IN_BASE, cosmosSdkDecToBigNumber } from '@injectivelabs/ui-common'
import {
  UI_DEFAULT_MIN_DISPLAY_DECIMALS,
  DEFAULT_CAPPED_TRADE_AND_EARN_REWARDS
} from '~/app/utils/constants'
import VItem from '~/components/partials/common/stats/item.vue'
import {
  FeeDiscountAccountInfo,
  TradingRewardsCampaign,
  ExchangeParams
} from '~/app/services/exchange'

export default Vue.extend({
  components: {
    VItem
  },

  data() {
    return {
      status: new Status(StatusType.Loading),

      DEFAULT_CAPPED_TRADE_AND_EARN_REWARDS,
      UI_DEFAULT_MIN_DISPLAY_DECIMALS,
      now: Math.floor(Date.now() / 1000)
    }
  },

  computed: {
    isUserWalletConnected(): boolean {
      return this.$accessor.wallet.isUserWalletConnected
    },

    exchangeParams(): ExchangeParams | undefined {
      return this.$accessor.exchange.params
    },

    feeDiscountAccountInfo(): FeeDiscountAccountInfo | undefined {
      return this.$accessor.exchange.feeDiscountAccountInfo
    },

    tradingRewardsCampaign(): TradingRewardsCampaign | undefined {
      return this.$accessor.exchange.tradingRewardsCampaign
    },

    stakedAmount(): BigNumberInBase {
      const { feeDiscountAccountInfo } = this

      if (!feeDiscountAccountInfo) {
        return ZERO_IN_BASE
      }

      if (!feeDiscountAccountInfo.accountInfo) {
        return ZERO_IN_BASE
      }

      return new BigNumberInBase(
        cosmosSdkDecToBigNumber(feeDiscountAccountInfo.accountInfo.stakedAmount)
      )
    },

    pendingTradeRewardsPoints(): string[] {
      return this.$accessor.exchange.pendingTradeRewardsPoints
    },

    injUsdPrice(): number {
      return this.$accessor.token.injUsdPrice
    },

    vestingDurationInSeconds(): number {
      const { exchangeParams } = this

      if (!exchangeParams) {
        return 0
      }

      if (!exchangeParams.tradingRewardsVestingDuration) {
        return 0
      }

      return new BigNumberInBase(
        exchangeParams.tradingRewardsVestingDuration || 0
      ).toNumber()
    },

    currentEpochStartTimestamp(): number {
      const { tradingRewardsCampaign } = this

      if (!tradingRewardsCampaign) {
        return 0
      }

      if (!tradingRewardsCampaign.tradingRewardCampaignInfo) {
        return 0
      }

      const [
        schedule
      ] = tradingRewardsCampaign.pendingTradingRewardPoolCampaignScheduleList

      if (!schedule) {
        return 0
      }

      return new BigNumberInBase(schedule.startTimestamp).toNumber()
    },

    pendingRewardsStartTimestamp(): number {
      const { currentEpochStartTimestamp, vestingDurationInSeconds } = this

      if (currentEpochStartTimestamp === 0) {
        return 0
      }

      return new BigNumberInBase(currentEpochStartTimestamp)
        .minus(vestingDurationInSeconds)
        .toNumber()
    },

    pendingRewardsCountdown(): string {
      const { pendingRewardsStartTimestamp, vestingDurationInSeconds } = this

      return format(
        (pendingRewardsStartTimestamp + vestingDurationInSeconds) * 1000,
        'dd MMM HH:mm:ss'
      )
    },

    injMaxPendingCampaignRewards(): BigNumberInBase {
      const { tradingRewardsCampaign } = this

      if (!tradingRewardsCampaign) {
        return ZERO_IN_BASE
      }

      const schedules =
        tradingRewardsCampaign.pendingTradingRewardPoolCampaignScheduleList

      if (schedules.length === 0) {
        return ZERO_IN_BASE
      }

      return schedules.reduce((total, schedule) => {
        const [inj] = schedule.maxCampaignRewardsList

        if (!inj) {
          return total
        }

        return total.plus(
          new BigNumberInBase(cosmosSdkDecToBigNumber(inj.amount || 0))
        )
      }, ZERO_IN_BASE)
    },

    injMaxPendingCampaignRewardsInUsd(): BigNumberInBase {
      const { injMaxPendingCampaignRewards, injUsdPrice } = this

      return injMaxPendingCampaignRewards.multipliedBy(
        new BigNumberInBase(injUsdPrice)
      )
    },

    pendingTradeRewardPoints(): BigNumberInBase {
      const { pendingTradeRewardsPoints } = this

      if (!pendingTradeRewardsPoints.length) {
        return ZERO_IN_BASE
      }

      if (pendingTradeRewardsPoints.length === 0) {
        return ZERO_IN_BASE
      }

      return pendingTradeRewardsPoints.reduce((total, points) => {
        if (!points) {
          return total
        }

        return total.plus(new BigNumberInBase(cosmosSdkDecToBigNumber(points)))
      }, ZERO_IN_BASE)
    },

    pendingTradeRewardPointsFactored(): BigNumberInBase {
      const { pendingTradeRewardPoints } = this

      return new BigNumberInWei(pendingTradeRewardPoints).toBase(
        6 /* Default factor for points, USDT decimals */
      )
    },

    totalPendingTradeRewardPoints(): BigNumberInBase {
      const { tradingRewardsCampaign } = this

      if (!tradingRewardsCampaign) {
        return ZERO_IN_BASE
      }

      const pointsList =
        tradingRewardsCampaign.pendingTotalTradeRewardPointsList

      if (pointsList.length === 0) {
        return ZERO_IN_BASE
      }

      return pointsList.reduce((total, pendingTotalTradeRewardPoints) => {
        if (!pendingTotalTradeRewardPoints) {
          return total
        }

        return total.plus(
          new BigNumberInBase(
            cosmosSdkDecToBigNumber(pendingTotalTradeRewardPoints || 0)
          )
        )
      }, ZERO_IN_BASE)
    },

    totalPendingTradeRewardPointsFactored(): BigNumberInBase {
      const { totalPendingTradeRewardPoints } = this

      return new BigNumberInWei(totalPendingTradeRewardPoints).toBase(
        6 /* Default factor for points, USDT decimals */
      )
    },

    pendingEstimatedRewards(): BigNumberInBase {
      const {
        pendingTradeRewardPoints,
        totalPendingTradeRewardPoints,
        injMaxPendingCampaignRewards
      } = this

      if (totalPendingTradeRewardPoints.lte(0)) {
        return ZERO_IN_BASE
      }

      if (pendingTradeRewardPoints.lte(0)) {
        return ZERO_IN_BASE
      }

      return pendingTradeRewardPoints
        .dividedBy(totalPendingTradeRewardPoints)
        .times(injMaxPendingCampaignRewards)
    },

    pendingEstimatedRewardsCapped(): BigNumberInBase {
      const { pendingEstimatedRewards, stakedAmount } = this

      if (pendingEstimatedRewards.lte(DEFAULT_CAPPED_TRADE_AND_EARN_REWARDS)) {
        return pendingEstimatedRewards
      }

      if (stakedAmount.lte(DEFAULT_CAPPED_TRADE_AND_EARN_REWARDS)) {
        return new BigNumberInBase(DEFAULT_CAPPED_TRADE_AND_EARN_REWARDS)
      }

      return pendingEstimatedRewards.gte(stakedAmount)
        ? stakedAmount
        : pendingEstimatedRewards
    },

    pendingEstimatedRewardsCappedInUsd(): BigNumberInBase {
      const { pendingEstimatedRewardsCapped, injUsdPrice } = this

      return pendingEstimatedRewardsCapped.multipliedBy(
        new BigNumberInBase(injUsdPrice)
      )
    },

    hubUrl(): string {
      return 'https://hub.injective.network/staking'
    }
  },

  mounted() {
    Promise.all([this.$accessor.exchange.fetchPendingTradeRewardPoints()])
      .then(() => {
        //
      })
      .catch(this.$onError)
      .finally(() => {
        this.status.setIdle()
      })
  }
})
</script>
