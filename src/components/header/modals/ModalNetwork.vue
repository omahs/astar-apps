<template>
  <astar-modal-drawer :show="isOpen" title="Network" :is-closing="isClosing" @close="closeModal">
    <div class="wrapper--modal-network">
      <div class="wrapper--select-network">
        <fieldset>
          <ul role="radiogroup" class="list--network" :style="`max-height: ${windowHeight}px`">
            <li v-for="(provider, index) in providerEndpoints" :key="index">
              <q-tooltip
                v-if="checkIsDisabledZkOption(provider)"
                anchor="top middle"
                class="box--tooltip"
              >
                <span class="text--tooltip">{{ $t('drawer.zkNetworkTip') }}</span>
              </q-tooltip>
              <button
                v-if="
                  provider.networkAlias !== 'astar-zkevm' && provider.networkAlias !== 'rocstar'
                "
                class="button--network-column"
                :disabled="checkIsDisabledZkOption(provider)"
              >
                <label
                  :class="[
                    'class-radio',
                    selNetwork === index ? 'class-radio-on' : 'class-radio-off',
                    provider.key === endpointKey.CUSTOM &&
                      isCustomNetwork &&
                      'class-radio--custom-network',
                  ]"
                >
                  <astar-radio-btn
                    class="ip--network"
                    :checked="selNetwork === index"
                    :disabled="checkIsDisabledZkOption(provider)"
                    @change="selNetwork = index"
                  />
                  <div class="wrapper--network-detail">
                    <div class="box--radio-network">
                      <img v-if="provider.defaultLogo" width="24" :src="provider.defaultLogo" />
                      <p class="box--display-name">
                        {{ provider.displayName }}
                      </p>
                    </div>
                    <div v-if="index === endpointKey.CUSTOM">
                      <input
                        v-if="selNetwork === index"
                        v-model="newEndpoint"
                        type="text"
                        placeholder="IP Address / Domain"
                        class="ip-input"
                      />
                    </div>
                    <div v-else-if="index === endpointKey.LOCAL">
                      <div />
                    </div>
                    <div v-else>
                      <div
                        v-if="
                          selNetwork === index &&
                          providerEndpoints[index].endpoints &&
                          provider.key !== endpointKey.ZKATANA &&
                          provider.key !== endpointKey.ASTAR_ZKEVM
                        "
                        class="box--endpoints"
                      >
                        <div>
                          <span class="text--md">{{ $t('drawer.endpoint') }}</span>
                        </div>
                        <div class="column--options">
                          <div
                            v-for="(endpointObj, i) in providerEndpoints[index].endpoints"
                            :key="i"
                          >
                            <div
                              v-if="checkIsDisplayEndpoint(provider, endpointObj.endpoint)"
                              class="column--network-option"
                              @click="setSelEndpoint({ endpointObj, networkIdx: index })"
                            >
                              <div class="box-input--endpoint">
                                <input
                                  name="choose_endpoint"
                                  type="radio"
                                  :checked="
                                    checkIsCheckedEndpoint({
                                      index,
                                      endpoint: endpointObj.endpoint,
                                    })
                                  "
                                  class="input--endpoint"
                                />
                              </div>
                              <span class="text--md">{{
                                $t('drawer.viaEndpoint', { value: endpointObj.name })
                              }}</span>
                            </div>
                          </div>
                        </div>
                      </div>
                    </div>
                  </div>
                </label>
              </button>
            </li>
          </ul>
        </fieldset>
      </div>
      <div v-if="isSelectLightClient" class="box--light-client-warning">
        <span class="text--accent">
          {{ $t('drawer.lightClientWarning') }}
        </span>
        <ul v-if="isLightClientExtension" class="ul--warnings">
          <li>
            <span>
              {{ $t('drawer.takeLongerTimeToConnect') }}
            </span>
          </li>
          <li>
            <span> {{ $t('drawer.takeLongerTimeToSend') }}</span>
          </li>
          <li v-if="selNetwork === endpointKey.SHIBUYA">
            <span>
              {{ $t('drawer.shibuyaTakes20mins') }}
            </span>
          </li>
        </ul>
        <div v-else>
          <ul class="ul--warnings">
            <li>
              <span> {{ $t('installWallet.installWallet', { value: 'Substrate_connect' }) }}</span>
              <a
                class="text--download"
                href="https://substrate.io/developers/substrate-connect/"
                target="_blank"
                rel="noreferrer"
              >
                {{ $t('installWallet.installExtension') }}
              </a>
            </li>
          </ul>
        </div>
      </div>
      <div class="wrapper__row--button">
        <astar-button
          class="btn--connect"
          :disabled="isDisabled"
          @click="selectNetwork(selNetwork)"
        >
          {{ $t('connect') }}
        </astar-button>
      </div>
    </div>
  </astar-modal-drawer>
</template>
<script lang="ts">
import {
  checkIsLightClient,
  checkIsSubstrateConnectInstalled,
} from 'src/config/api/polkadot/connectApi';
import { ChainProvider, endpointKey, providerEndpoints } from 'src/config/chainEndpoints';
import { LOCAL_STORAGE } from 'src/config/localStorage';
import { getRandomFromArray, wait } from '@astar-network/astar-sdk-core';
import { buildNetworkUrl } from 'src/router/utils';
import { computed, defineComponent, ref, watch, onUnmounted } from 'vue';
import { useNetworkInfo } from 'src/hooks';
import { useStore } from 'src/store';

export default defineComponent({
  props: {
    isOpen: {
      type: Boolean,
      required: true,
    },
    networkIdx: {
      type: Number,
      required: true,
    },
  },
  emits: ['update:select-network', 'update:is-open'],
  setup(props, { emit }) {
    const classRadioOn = 'class-radio-on';
    const classRadioOff = 'class-radio-off';

    const newEndpoint = ref('');
    const isLightClientExtension = computed<boolean>(() => checkIsSubstrateConnectInstalled());

    const { isZkEvm } = useNetworkInfo();
    const store = useStore();
    const isH160 = computed<boolean>(() => store.getters['general/isH160Formatted']);

    const checkIsDisabledZkOption = (provider: ChainProvider): boolean => {
      return !isH160.value && provider.displayName.includes('zk');
    };

    const setInitialNewEndpoint = (): string => {
      const selectedEndpointStored = String(localStorage.getItem(LOCAL_STORAGE.SELECTED_ENDPOINT));
      const selectedEndpoint = JSON.parse(selectedEndpointStored);
      const networkId = Object.keys(selectedEndpoint)[0];
      const connectedCustomEndpoint = Object.values(selectedEndpoint)[0] as string;
      return networkId === String(endpointKey.CUSTOM) ? connectedCustomEndpoint : '';
    };
    newEndpoint.value = setInitialNewEndpoint();

    const isClosing = ref<boolean>(false);

    const closeModal = async (): Promise<void> => {
      isClosing.value = true;
      const animationDuration = 500;
      await wait(animationDuration);
      isClosing.value = false;
      emit('update:is-open', false);
    };

    const { NETWORK_IDX, SELECTED_ENDPOINT } = LOCAL_STORAGE;

    const getSelectedNetwork = (networkIdx: number): string => {
      switch (networkIdx) {
        case endpointKey.ASTAR:
          return selEndpointAstar.value;
        case endpointKey.SHIDEN:
          return selEndpointShiden.value;
        case endpointKey.SHIBUYA:
          return selEndpointShibuya.value;
        case endpointKey.ZKATANA:
          return selEndpointZkatana.value;
        case endpointKey.ASTAR_ZKEVM:
          return selEndpointAstarZkevm.value;

        default:
          return selEndpointAstar.value;
      }
    };

    const selectNetwork = async (networkIdx: number): Promise<void> => {
      localStorage.setItem(NETWORK_IDX, networkIdx.toString());
      localStorage.setItem(
        SELECTED_ENDPOINT,
        JSON.stringify({
          [networkIdx]:
            newEndpoint.value && selNetwork.value === endpointKey.CUSTOM
              ? newEndpoint.value
              : getSelectedNetwork(networkIdx),
        })
      );

      const network = providerEndpoints[networkIdx].networkAlias;
      const url = buildNetworkUrl(network);

      window.open(url, '_self');

      emit('update:is-open', false);
      emit('update:select-network', networkIdx);
    };

    const setInitialSelEndpoint = (): string => {
      return JSON.parse(localStorage.getItem(SELECTED_ENDPOINT) || '{}')[props.networkIdx] || '';
    };

    const selNetwork = ref<number>(props.networkIdx);
    const selEndpointAstar = ref<string>('');
    const selEndpointShiden = ref<string>('');
    const selEndpointShibuya = ref<string>('');
    const selEndpointAstarZkevm = ref<string>('');
    const selEndpointZkatana = ref<string>('');

    const isDisabled = computed<boolean>(() => {
      if (isSelectLightClient.value) {
        return !isLightClientExtension.value;
      } else {
        return selNetwork.value === endpointKey.CUSTOM && !newEndpoint.value;
      }
    });

    const isCustomNetwork = computed<boolean>(() => selNetwork.value === endpointKey.CUSTOM);
    const isSelectLightClient = computed<boolean>(() => {
      switch (selNetwork.value) {
        case endpointKey.ASTAR:
          return checkIsLightClient(selEndpointAstar.value);
        case endpointKey.SHIDEN:
          return checkIsLightClient(selEndpointShiden.value);
        case endpointKey.SHIBUYA:
          return checkIsLightClient(selEndpointShibuya.value);
        default:
          return false;
      }
    });

    const checkIsCheckedEndpoint = ({
      index,
      endpoint,
    }: {
      index: number;
      endpoint: string;
    }): boolean => {
      return index === endpointKey.ASTAR
        ? selEndpointAstar.value === endpoint
        : index === endpointKey.SHIDEN
        ? selEndpointShiden.value === endpoint
        : index === endpointKey.SHIBUYA
        ? selEndpointShibuya.value === endpoint
        : index === endpointKey.ASTAR_ZKEVM
        ? selEndpointAstarZkevm.value === endpoint
        : selEndpointZkatana.value === endpoint;
    };

    const setSelEndpoint = ({
      endpointObj,
      networkIdx,
    }: {
      endpointObj: { name: string; endpoint: string };
      networkIdx: number;
      selEndpointRef: string;
    }): void => {
      if (networkIdx === endpointKey.ASTAR) {
        selEndpointAstar.value = endpointObj.endpoint;
      } else if (networkIdx === endpointKey.SHIDEN) {
        selEndpointShiden.value = endpointObj.endpoint;
      } else if (networkIdx === endpointKey.SHIBUYA) {
        selEndpointShibuya.value = endpointObj.endpoint;
      } else if (networkIdx === endpointKey.ASTAR_ZKEVM) {
        selEndpointAstarZkevm.value = endpointObj.endpoint;
      } else if (networkIdx === endpointKey.ZKATANA) {
        selEndpointZkatana.value = endpointObj.endpoint;
      }
    };

    const getRandomizedEndpoint = (endpointKey: endpointKey): string => {
      const endpointsWithoutLightClient = providerEndpoints[endpointKey].endpoints.filter(
        (it) => !checkIsLightClient(it.endpoint)
      );
      return getRandomFromArray(endpointsWithoutLightClient).endpoint;
    };

    const randomizedEndpoint = (networkIdx: number): void => {
      if (networkIdx === endpointKey.ASTAR) {
        selEndpointAstar.value = getRandomizedEndpoint(endpointKey.ASTAR);
      }
      if (networkIdx === endpointKey.SHIDEN) {
        selEndpointShiden.value = getRandomizedEndpoint(endpointKey.SHIDEN);
      }
      if (networkIdx === endpointKey.SHIBUYA) {
        selEndpointShibuya.value = getRandomizedEndpoint(endpointKey.SHIBUYA);
      }
      if (networkIdx === endpointKey.ASTAR_ZKEVM) {
        selEndpointAstarZkevm.value = getRandomizedEndpoint(endpointKey.ASTAR_ZKEVM);
      }
      if (networkIdx === endpointKey.ZKATANA) {
        selEndpointZkatana.value = getRandomizedEndpoint(endpointKey.ZKATANA);
      }
    };

    const setupInitialEndpointOption = (networkIdx: number) => {
      if (networkIdx === endpointKey.ASTAR) {
        selEndpointAstar.value = setInitialSelEndpoint();
        randomizedEndpoint(endpointKey.SHIDEN);
        randomizedEndpoint(endpointKey.SHIBUYA);
        randomizedEndpoint(endpointKey.ASTAR_ZKEVM);
        randomizedEndpoint(endpointKey.ZKATANA);
        return;
      }

      if (networkIdx === endpointKey.SHIDEN) {
        selEndpointShiden.value = setInitialSelEndpoint();
        randomizedEndpoint(endpointKey.ASTAR);
        randomizedEndpoint(endpointKey.SHIBUYA);
        randomizedEndpoint(endpointKey.ASTAR_ZKEVM);
        randomizedEndpoint(endpointKey.ZKATANA);
        return;
      }

      if (networkIdx === endpointKey.SHIBUYA) {
        selEndpointShibuya.value = setInitialSelEndpoint();
        randomizedEndpoint(endpointKey.ASTAR);
        randomizedEndpoint(endpointKey.SHIDEN);
        randomizedEndpoint(endpointKey.ASTAR_ZKEVM);
        randomizedEndpoint(endpointKey.ZKATANA);
        return;
      }
      if (networkIdx === endpointKey.ASTAR_ZKEVM) {
        selEndpointShibuya.value = setInitialSelEndpoint();
        randomizedEndpoint(endpointKey.ASTAR);
        randomizedEndpoint(endpointKey.SHIDEN);
        randomizedEndpoint(endpointKey.SHIBUYA);
        randomizedEndpoint(endpointKey.ZKATANA);
        return;
      }
      if (networkIdx === endpointKey.ZKATANA) {
        selEndpointShibuya.value = setInitialSelEndpoint();
        randomizedEndpoint(endpointKey.ASTAR);
        randomizedEndpoint(endpointKey.SHIDEN);
        randomizedEndpoint(endpointKey.SHIBUYA);
        randomizedEndpoint(endpointKey.ASTAR_ZKEVM);
        return;
      }
    };

    // Memo: Displays Light client option if:
    // A: the portal is opened on 'localhost' or 'staging URL'
    // B: Shibuya
    const checkIsDisplayEndpoint = (chain: ChainProvider, endpoint: string): boolean => {
      // As asked by @gluneau Light client is enabled for all networks.
      return true;
      // const origin = window.location.origin;
      // const stagingDomain = '.web.app';
      // const devPaths = [stagingDomain, stagingMainBranch, 'localhost:'];
      // const isForDeveloper = devPaths.some((it) => origin.includes(it));
      // // Memo: enables selecting light client endpoint for Shibuya on the production page
      // if (isForDeveloper || chain.key === endpointKey.SHIBUYA) {
      //   return true;
      // } else {
      //   return !checkIsLightClient(endpoint);
      // }
    };

    watch(
      [selNetwork],
      () => {
        setupInitialEndpointOption(props.networkIdx);
      },
      { immediate: true }
    );

    const windowHeight = ref<number>(window.innerHeight);
    const onHeightChange = () => {
      windowHeight.value = window.innerHeight - 300;
    };

    window.addEventListener('resize', onHeightChange);
    onHeightChange();

    onUnmounted(() => {
      window.removeEventListener('resize', onHeightChange);
    });

    return {
      newEndpoint,
      selNetwork,
      classRadioOn,
      classRadioOff,
      providerEndpoints,
      endpointKey,
      isDisabled,
      isCustomNetwork,
      isClosing,
      selEndpointAstar,
      selEndpointShiden,
      selEndpointShibuya,
      windowHeight,
      isSelectLightClient,
      isLightClientExtension,
      isZkEvm,
      isH160,
      closeModal,
      setSelEndpoint,
      checkIsCheckedEndpoint,
      selectNetwork,
      checkIsDisplayEndpoint,
      checkIsDisabledZkOption,
    };
  },
});
</script>

<style lang="scss" scoped>
@use 'src/components/header/styles/modal-network.scss';

/* Memo: somehow these classes won't work in light theme if the classes defined on modal-network.scss  */
.box--light-client-warning {
  display: none;
  @media (min-width: $sm) {
    display: flex;
    flex-direction: column;
    row-gap: 8px;
    margin-top: 24px;
  }
}

.ul--warnings {
  list-style: disc;
  text-align: left;
  padding-left: 24px;
}

.text--download {
  display: block;
  text-decoration: underline;
  color: $astar-blue;
}
</style>
