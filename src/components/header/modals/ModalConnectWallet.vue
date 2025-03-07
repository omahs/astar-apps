<template>
  <astar-modal-drawer
    :show="isModalConnectWallet"
    :is-closing="isClosing"
    title="Select a Wallet"
    @close="closeModal()"
  >
    <div class="wrapper--modal--wallet">
      <div>
        <div class="title--account-type">
          <span>
            {{ $t('wallet.evmAccount') }}
          </span>
        </div>
        <div class="wrapper--wallets">
          <button
            v-for="(wallet, index) in evmWallets"
            :key="index"
            class="box__row--wallet box--hover--active"
            :class="currentWallet === wallet.source && 'border--active'"
            :wallet="wallet"
            @click="setEvmWalletModal(wallet.source)"
          >
            <div class="box--img">
              <img :src="wallet.img" />
            </div>
            <div>
              <span>
                {{ castWalletName(wallet.name) }}
              </span>
            </div>
          </button>
        </div>
      </div>
      <div>
        <div class="title--account-type">
          <span>
            {{ $t('wallet.nativeAccount') }}
          </span>
        </div>
        <div class="wrapper--wallets">
          <button
            v-for="(wallet, index) in nativeWallets"
            :key="index"
            :disabled="checkIsDisabledWallet(wallet.source) || isZkEvm"
            class="box__row--wallet box--hover--active"
            :class="currentWallet === wallet.source && 'border--active'"
            @click="!checkIsDisabledWallet(wallet.source) && setSubstrateWalletModal(wallet.source)"
          >
            <div class="box--img">
              <img :src="wallet.img" />
            </div>
            <div>
              <span>
                {{ castWalletName(wallet.name) }}
              </span>
            </div>
          </button>
        </div>
        <div v-if="selWallet && isNoExtension" class="box--no-extension">
          <div class="title--no-extension">
            <span class="text--install-title">
              {{ $t('installWallet.getWallet', { value: $t(selWallet.name) }) }}
            </span>
          </div>
          <div class="row--no-extension">
            <span class="text--install">
              {{ $t('installWallet.installWallet', { value: $t(selWallet.name) }) }}</span
            >
          </div>
          <div class="row--icon-links">
            <button>
              <a
                :href="selWallet.walletUrl"
                target="_blank"
                rel="noopener noreferrer"
                class="button--link"
              >
                <div class="icon--link">
                  <astar-icon-external-link />
                </div>
                <span class="text--install-link">
                  {{ $t('installWallet.install') }}
                </span>
              </a>
            </button>
            <button>
              <a
                :href="selWallet.guideUrl"
                target="_blank"
                rel="noopener noreferrer"
                class="button--link"
              >
                <div class="icon--link">
                  <astar-icon-external-link />
                </div>
                <span class="text--install-link">
                  {{ $t('installWallet.learn') }}
                </span>
              </a>
            </button>
          </div>
        </div>
      </div>
      <div v-if="currentNetworkIdx === endpointKey.ASTAR">
        <div class="title--account-type">
          <span>
            {{ $t('wallet.multisigAccount') }}
          </span>
        </div>
        <div class="wrapper--wallets">
          <button
            class="box__row--wallet box--hover--active"
            :class="currentWallet === SupportMultisig.Polkasafe && 'border--active'"
            @click="setPolkasafeModal()"
          >
            <div class="box--img">
              <img
                :src="require('src/assets/img/logo-polkasafe-black.svg')"
                class="img--polkasafe"
              />
            </div>
            <div>
              <span> PolkaSafe </span>
            </div>
          </button>
        </div>
      </div>
      <!-- temporarily disable until we implement account selection UI -->
      <!-- <div v-if="isAccountUnification">
        <div class="title--account-type">
          <span>
            {{ $t('wallet.accountUnification') }}
          </span>
        </div>
        <div class="wrapper--wallets">
          <div class="box__row--wallet box--hover--active" @click="setAccountUnificationModal()">
            <div class="box--img astar-account">
              <img :src="require('src/assets/img/token/astr.png')" />
            </div>
            <div>
              <span> Astar Account </span>
            </div>
          </div>
        </div>
      </div> -->
      <button :disabled="!currentAccountName" class="btn--disconnect" @click="disconnectAccount()">
        {{ $t('disconnect') }}
      </button>
    </div>
  </astar-modal-drawer>
</template>
<script lang="ts">
import { wait } from '@astar-network/astar-sdk-core';
import { initPolkadotSnap } from '@astar-network/metamask-astar-adapter';
import { get } from 'lodash-es';
import { $api } from 'src/boot/api';
import { endpointKey } from 'src/config/chainEndpoints';
import {
  SupportMultisig,
  SupportWallet,
  Wallet,
  supportAllWalletsObj,
  supportEvmWalletObj,
  supportEvmWallets,
  supportWallets,
} from 'src/config/wallets';
import { useAccount, useNetworkInfo } from 'src/hooks';
import { getInjectedExtensions, isMobileDevice } from 'src/hooks/helper/wallet';
import { useExtensions } from 'src/hooks/useExtensions';
import { initiatePolkdatodSnap } from 'src/modules/snap';
import { useStore } from 'src/store';
import { SubstrateAccount } from 'src/store/general/state';
import { PropType, computed, defineComponent, ref } from 'vue';
import { productionOrigin } from 'src/links';

export default defineComponent({
  props: {
    isModalConnectWallet: {
      type: Boolean,
      required: true,
    },
    setCloseModal: {
      type: Function,
      required: true,
    },
    setWalletModal: {
      type: Function,
      required: true,
    },
    connectEthereumWallet: {
      type: Function,
      required: true,
    },
    openPolkasafeModal: {
      type: Function,
      required: true,
    },
    openAccountUnificationModal: {
      type: Function,
      required: true,
    },
    isNoExtension: {
      type: Boolean,
      required: true,
    },
    selectedWallet: {
      type: String as PropType<SupportWallet>,
      required: true,
    },
  },
  setup(props) {
    const store = useStore();
    const { currentAccountName, disconnectAccount, isAccountUnification, isSnapEnabled } =
      useAccount();
    const isClosing = ref<boolean>(false);
    const { currentNetworkIdx, isZkEvm } = useNetworkInfo();

    const closeModal = async (): Promise<void> => {
      isClosing.value = true;
      const animationDuration = 500;
      await wait(animationDuration);
      isClosing.value = false;
      props.setCloseModal();
    };

    const checkIsDisabledWallet = (source: SupportWallet): boolean => {
      if (source === SupportWallet.Snap && window.location.origin === productionOrigin) {
        return true;
      }
      return false;
    };

    const nativeWallets = computed(() => {
      return supportWallets
        .map((it) => {
          const { isSupportMobileApp, isSupportBrowserExtension } = it;
          if (it.source === SupportWallet.Snap) {
            return isSnapEnabled.value ? it : undefined;
          }
          if (isMobileDevice) {
            return isSupportMobileApp ? it : undefined;
          } else {
            return isSupportBrowserExtension ? it : undefined;
          }
        })
        .filter((it) => it !== undefined) as Wallet[];
    });

    const evmWallets = computed(() => {
      return supportEvmWallets
        .map((it) => {
          const { isSupportMobileApp, isSupportBrowserExtension } = it;
          if (isMobileDevice) {
            return isSupportMobileApp ? it : undefined;
          } else {
            return isSupportBrowserExtension ? it : undefined;
          }
        })
        .filter((it) => it !== undefined) as Wallet[];
    });

    const selWallet = computed(() => supportAllWalletsObj[props.selectedWallet]);
    const substrateAccounts = computed<SubstrateAccount[]>(
      () => store.getters['general/substrateAccounts']
    );

    const castWalletName = (wallet: string): string => {
      return wallet.split('(')[0].trim();
    };

    const handleExtensions = (): void => {
      if (substrateAccounts.value.length === 0) {
        useExtensions($api!!, store);
      }
    };

    const handleMetaMaskSnap = async (): Promise<void> => {
      const provider = get(window, supportEvmWalletObj[SupportWallet.MetaMask].ethExtension);
      const [address] = (await provider.request({ method: 'eth_requestAccounts' })) as string;
      if (!address) return;
      const isSnapInstalled = await initiatePolkdatodSnap();
      if (isSnapInstalled) {
        await initPolkadotSnap();
        useExtensions($api!!, store);
        const extensions = await getInjectedExtensions(true);
        const isExtensionsUpdated = extensions.some((it) => it.name === SupportWallet.Snap);
        // Memo: Sync the metamask extension for users who visit our portal first time
        !isExtensionsUpdated && (await wait(3000));
      }
    };
    const setSubstrateWalletModal = async (source: string): Promise<void> => {
      if (source === SupportWallet.Snap) {
        await handleMetaMaskSnap();
      }
      await closeModal();
      props.setWalletModal(source);
    };

    const setPolkasafeModal = async (): Promise<void> => {
      handleExtensions();
      await closeModal();
      props.openPolkasafeModal();
    };

    const setAccountUnificationModal = async (): Promise<void> => {
      props.openAccountUnificationModal();
    };

    const setEvmWalletModal = async (source: string): Promise<void> => {
      await closeModal();
      props.connectEthereumWallet(source);
    };
    const currentWallet = computed<string>(() => store.getters['general/currentWallet']);

    return {
      nativeWallets,
      evmWallets,
      isClosing,
      currentWallet,
      currentAccountName,
      selWallet,
      SupportMultisig,
      castWalletName,
      closeModal,
      setSubstrateWalletModal,
      setEvmWalletModal,
      disconnectAccount,
      setPolkasafeModal,
      checkIsDisabledWallet,
      setAccountUnificationModal,
      currentNetworkIdx,
      endpointKey,
      isZkEvm,
      isAccountUnification,
    };
  },
});
</script>

<style lang="scss" scoped>
@use 'src/components/header/styles/modal-connect-wallet.scss';
</style>
