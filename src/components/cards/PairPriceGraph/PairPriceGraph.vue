<script setup lang="ts">
import { format, fromUnixTime } from 'date-fns';
import {
  Dictionary,
  mapKeys,
  mapValues,
  maxBy,
  minBy,
  pickBy,
  toPairs
} from 'lodash';
import { computed, reactive, ref } from 'vue';
import { useQuery } from 'vue-query';
import { useStore } from 'vuex';

import { useTradeState } from '@/composables/trade/useTradeState';
import useBreakpoints from '@/composables/useBreakpoints';
import useTailwind from '@/composables/useTailwind';
import useTokens from '@/composables/useTokens';
import QUERY_KEYS from '@/constants/queryKeys';
import { coingeckoService } from '@/services/coingecko/coingecko.service';
import useWeb3 from '@/services/web3/useWeb3';

/**
 *
 * @param inputAsset The address of the input asset
 * @param outputAsset The address of the output asset
 * @param nativeAsset The address of the native asset on the network
 * @param wrappedNativeAsset The address of the wrapped native asset on the network
 * @param days The number of days to pull historical data for
 * @param inverse Swaps the pricing calc to be output/input rather than input/output
 */
async function getPairPriceData(
  inputAsset: string,
  outputAsset: string,
  nativeAsset: string,
  wrappedNativeAsset: string,
  days: number,
  inverse?: boolean
) {
  let _inputAsset =
    inputAsset === nativeAsset ? wrappedNativeAsset : inputAsset;
  let _outputAsset =
    outputAsset === nativeAsset ? wrappedNativeAsset : outputAsset;

  if (inverse) {
    [_inputAsset, _outputAsset] = [_outputAsset, _inputAsset];
  }
  const aggregateBy = days === 1 ? 'hour' : 'day';
  const getInputAssetData = coingeckoService.prices.getTokensHistorical(
    [_inputAsset],
    days,
    1,
    aggregateBy
  );

  const getOutputAssetData = coingeckoService.prices.getTokensHistorical(
    [_outputAsset],
    days,
    1,
    aggregateBy
  );

  const [inputAssetData, outputAssetData] = await Promise.all([
    getInputAssetData,
    getOutputAssetData
  ]);

  const calculatedPricing = mapValues(inputAssetData, (value, timestamp) => {
    if (!outputAssetData[timestamp]) return null;
    return (1 / value[0]) * outputAssetData[timestamp][0];
  });

  const calculatedPricingNoNulls = pickBy(calculatedPricing) as Dictionary<
    number
  >;

  const formatTimestamps = mapKeys(
    calculatedPricingNoNulls,
    (_, timestamp: any) =>
      format(fromUnixTime(timestamp / 1000), 'yyyy/MM/dd HH:mm')
  );

  return toPairs(formatTimestamps);
}

const chartTimespans = [
  {
    option: '1d',
    value: 1
  },
  {
    option: '1w',
    value: 7
  },
  {
    option: '1m',
    value: 30
  },
  {
    option: '1y',
    value: 365
  },
  {
    option: 'All',
    value: 4000
  }
];

type Props = {
  isModal?: boolean;
  onCloseModal?: () => void;
  toggleModal: () => void;
};

const props = defineProps<Props>();
const { upToLargeBreakpoint } = useBreakpoints();
const store = useStore();
const { getToken, wrappedNativeAsset, nativeAsset } = useTokens();
const { tokenInAddress, tokenOutAddress, initialized } = useTradeState();
const tailwind = useTailwind();
const { chainId: userNetworkId } = useWeb3();

const chartHeight = ref(
  upToLargeBreakpoint ? (props.isModal ? 250 : 75) : props.isModal ? 250 : 100
);
const activeTimespan = ref(chartTimespans[0]);
const appLoading = computed(() => store.state.app.loading);

const inputSym = computed(() => {
  if (tokenInAddress.value === '') return 'Unknown';
  return getToken(tokenInAddress.value)?.symbol;
});
const outputSym = computed(() => {
  if (tokenOutAddress.value === '') return 'Unknown';
  return getToken(tokenOutAddress.value)?.symbol;
});

const dataMin = computed(() => {
  return (minBy(priceData.value || [], v => v[1]) || [])[1] || 0;
});

const dataMax = computed(() => {
  return (maxBy(priceData.value || [], v => v[1]) || [])[1] || 0;
});

const {
  isLoading: isLoadingPriceData,
  data: priceData,
  error: failedToLoadPriceData
} = useQuery(
  QUERY_KEYS.Tokens.PairPriceData(
    tokenInAddress,
    tokenOutAddress,
    activeTimespan,
    userNetworkId,
    nativeAsset,
    wrappedNativeAsset
  ),
  () =>
    getPairPriceData(
      tokenInAddress.value,
      tokenOutAddress.value,
      nativeAsset?.address,
      wrappedNativeAsset.value?.address,
      activeTimespan.value.value,
      true
    ),
  reactive({
    enabled: initialized,
    retry: false,
    // when refetch on window focus in enabled, it causes a flash
    // in the loading state of the card which is jarring. disabling it
    refetchOnWindowFocus: false
  })
);

const toggle = () => {
  props.toggleModal();
};

const chartData = computed(() => [
  {
    name: `${outputSym.value}/${inputSym.value}`,
    values: priceData.value || []
  }
]);

const isNegativeTrend = computed(() => {
  const _priceData = priceData.value || [];
  if (_priceData.length > 2) {
    if (
      _priceData[_priceData.length - 1][1] <
      _priceData[_priceData.length - 2][1]
    ) {
      return true;
    }
  }
  return false;
});

const chartColors = computed(() => {
  let color = tailwind.theme.colors.green['400'];
  if (isNegativeTrend.value) color = tailwind.theme.colors.red['400'];
  return [color];
});

const chartGrid = computed(() => {
  return {
    left: '2.5%',
    right: '0',
    top: '10%',
    bottom: '15%',
    containLabel: false
  };
});
</script>

<template>
  <div
    :class="[
      '',
      {
        'h-40 lg:h-56': !isModal,
        'h-full lg:h-full': isModal
      }
    ]"
  >
    <BalLoadingBlock
      v-if="isLoadingPriceData"
      :class="{
        'h-64': !isModal,
        'h-112': isModal
      }"
    />
    <BalCard
      :square="upToLargeBreakpoint"
      shadow="none"
      hFull
      growContent
      noPad
      :noBorder="upToLargeBreakpoint || isModal"
      v-else
    >
      <div class="relative h-full bg-transparent p-4">
        <button
          v-if="!failedToLoadPriceData && !(isLoadingPriceData || appLoading)"
          @click="toggle"
          class="maximise m-4 p-2 flex justify-center items-center shadow-lg rounded-full"
        >
          <BalIcon v-if="!isModal" name="maximize-2" class="text-gray-500" />
          <BalIcon v-if="isModal" name="x" class="text-gray-500" />
        </button>
        <div
          v-if="!failedToLoadPriceData && !(isLoadingPriceData || appLoading)"
          class="flex"
        >
          <h6 class="font-medium">{{ outputSym }}/{{ inputSym }}</h6>
          <BalTooltip class="ml-2" :text="$t('coingeckoPricingTooltip')">
            <template v-slot:activator>
              <img class="h-5" src="@/assets/images/icons/coingecko.svg" />
            </template>
          </BalTooltip>
        </div>
        <div
          v-if="failedToLoadPriceData && tokenOutAddress"
          class="h-full w-full flex justify-center items-center"
        >
          <span class="text-sm text-gray-400">{{
            $t('insufficientData')
          }}</span>
        </div>
        <div
          v-if="failedToLoadPriceData && !tokenOutAddress"
          class="h-full w-full flex justify-center items-center"
        >
          <span class="text-sm text-gray-400 text-center">{{
            $t('chooseAPair')
          }}</span>
        </div>
        <div
          v-if="!failedToLoadPriceData && !isLoadingPriceData"
          class="flex-col"
        >
          <BalLineChart
            :data="chartData"
            :height="chartHeight"
            :show-legend="false"
            :color="chartColors"
            :custom-grid="chartGrid"
            :axis-label-formatter="{ yAxis: '0.000000' }"
            :wrapper-class="[
              'flex flex-row lg:flex-col',
              {
                'flex-row': !isModal,
                'flex-col': isModal
              }
            ]"
            :show-tooltip="!upToLargeBreakpoint || isModal"
            hide-y-axis
            hide-x-axis
            show-header
            use-min-max
          />
          <div
            :class="[
              'w-full flex justify-between mt-6',
              {
                'flex-col': isModal
              }
            ]"
            v-if="isModal"
          >
            <div>
              <button
                v-for="timespan in chartTimespans"
                @click="activeTimespan = timespan"
                :key="timespan.value"
                :class="[
                  'py-1 px-2 text-sm rounded-lg mr-2',
                  {
                    'text-white': activeTimespan.value === timespan.value,
                    'text-gray-500': activeTimespan.value !== timespan.value,
                    'bg-green-400':
                      !isNegativeTrend &&
                      activeTimespan.value === timespan.value,
                    'bg-red-400':
                      isNegativeTrend &&
                      activeTimespan.value === timespan.value,
                    'hover:bg-red-200': isNegativeTrend,
                    'hover:bg-green-200': !isNegativeTrend
                  }
                ]"
              >
                {{ timespan.option }}
              </button>
            </div>
            <div :class="{ 'mt-4': isModal }">
              <span class="text-sm text-gray-500 mr-4"
                >Low: {{ dataMin.toPrecision(6) }}</span
              >
              <span class="text-sm text-gray-500"
                >High: {{ dataMax.toPrecision(6) }}</span
              >
            </div>
          </div>
          <div class="-mt-2 lg:mt-2" v-else>
            <span class="text-sm text-gray-500 w-full flex justify-end">{{
              activeTimespan.option
            }}</span>
          </div>
        </div>
      </div>
    </BalCard>
  </div>
</template>

<style scoped>
.maximise {
  @apply absolute;
  right: 0;
  top: 0;
}
</style>
