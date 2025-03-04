<template>
  <div data-testid="evm-native-token">
    <div class="border--separator border--margin" />
    <div class="rows">
      <div class="row row--details">
        <div class="row__left">
          <div class="column--currency">
            <img class="token-logo" :src="nativeTokenImg" :alt="nativeTokenSymbol" />
            <div v-if="nativeTokenSymbol && currentNetworkName" class="column--ticker">
              <span class="text--title">{{ nativeTokenSymbol }}</span>
              <span class="text--label">{{ currentNetworkName }}</span>
            </div>
            <div v-else>
              <q-skeleton animation="fade" class="skeleton--md" />
            </div>
          </div>
        </div>
        <div class="row__right row__right--evm-native-token">
          <div class="column column--balance">
            <div class="column__box">
              <div class="text--accent">
                <token-balance :balance="String(bal)" :symbol="nativeTokenSymbol" />
              </div>
              <div class="text--label">
                <span>{{ $n(balUsd) }} {{ $t('usd') }}</span>
              </div>
            </div>
          </div>
          <div class="column--asset-buttons column--buttons--native-token">
            <router-link v-if="isZkEvm" :to="buildEthereumBridgePageLink()">
              <button class="btn btn--sm">
                {{ $t('assets.bridge') }}
              </button>
            </router-link>
            <router-link :to="buildTransferPageLink(nativeTokenSymbol)">
              <button class="btn btn--sm">
                {{ $t('assets.transfer') }}
              </button>
            </router-link>
            <!-- Only SDN is able to bridge via cBridge at this moment -->
            <a
              v-if="nativeTokenSymbol === 'SDN'"
              :href="cbridgeAppLink"
              target="_blank"
              rel="noopener noreferrer"
            >
              <button class="btn btn--sm">
                {{ $t('assets.bridge') }}
              </button>
            </a>
            <a v-if="isZkatana" :href="faucetSethLink" target="_blank" rel="noopener noreferrer">
              <button class="btn btn--sm">
                {{ $t('assets.faucet') }}
              </button>
            </a>
            <button
              v-else-if="isFaucet"
              class="btn btn--sm"
              @click="handleModalFaucet({ isOpen: true })"
            >
              {{ $t('assets.faucet') }}
            </button>
          </div>
        </div>
      </div>
    </div>
    <modal-faucet :is-modal-faucet="isModalFaucet" :handle-modal-faucet="handleModalFaucet" />
  </div>
</template>
<script lang="ts">
import { ethers } from 'ethers';
import { $web3 } from 'src/boot/api';
import { cbridgeAppLink } from 'src/c-bridge';
import ModalFaucet from 'src/components/assets/modals/ModalFaucet.vue';
import TokenBalance from 'src/components/common/TokenBalance.vue';
import { faucetBalRequirement } from 'src/config/wallets';
import { useAccount, useNetworkInfo, usePrice } from 'src/hooks';
import { getTokenImage } from 'src/modules/token';
import { buildTransferPageLink, buildEthereumBridgePageLink } from 'src/router/routes';
import { useStore } from 'src/store';
import { computed, defineComponent, ref, watchEffect } from 'vue';
import { faucetSethLink } from 'src/links';

export default defineComponent({
  components: { ModalFaucet, TokenBalance },
  setup() {
    const bal = ref<number>(0);
    const balUsd = ref<number>(0);
    const isShibuya = ref<boolean>(false);
    const isRocstar = ref<boolean>(false);
    const isFaucet = ref<boolean>(false);
    const isModalFaucet = ref<boolean>(false);

    const { currentNetworkName, nativeTokenSymbol, isZkEvm, isZkatana } = useNetworkInfo();
    const { currentAccount } = useAccount();
    const { nativeTokenUsd } = usePrice();
    const store = useStore();
    const isH160 = computed<boolean>(() => store.getters['general/isH160Formatted']);
    const isLoading = computed<boolean>(() => store.getters['general/isLoading']);

    const nativeTokenImg = computed<string>(() =>
      getTokenImage({
        isNativeToken: true,
        symbol: nativeTokenSymbol.value,
        isZkEvm: isZkEvm.value,
      })
    );

    const updateStates = async (nativeTokenUsd: number): Promise<void> => {
      if (isLoading.value || !nativeTokenSymbol.value || !isH160.value || !$web3.value) return;
      try {
        const balWei = await $web3.value!.eth.getBalance(currentAccount.value);
        bal.value = Number(ethers.utils.formatEther(balWei));
        isShibuya.value = nativeTokenSymbol.value === 'SBY';
        isRocstar.value = nativeTokenSymbol.value === 'RSTR';
        isFaucet.value = isRocstar.value
          ? false
          : isShibuya.value || faucetBalRequirement > bal.value;
        if (nativeTokenUsd) {
          balUsd.value = nativeTokenUsd * bal.value;
        }
      } catch (error: any) {
        console.error(error.message);
      }
    };

    watchEffect(async () => {
      await updateStates(nativeTokenUsd.value);
    });

    const handleModalFaucet = ({ isOpen }: { isOpen: boolean }): void => {
      isModalFaucet.value = isOpen;
    };

    return {
      nativeTokenImg,
      nativeTokenSymbol,
      currentNetworkName,
      bal,
      balUsd,
      cbridgeAppLink,
      isFaucet,
      isModalFaucet,
      isZkEvm,
      isZkatana,
      faucetSethLink,
      handleModalFaucet,
      buildTransferPageLink,
      buildEthereumBridgePageLink,
    };
  },
});
</script>

<style lang="scss" scoped>
@use 'src/components/assets/styles/asset-list.scss';
</style>
