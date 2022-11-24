<script setup lang="ts">
import { toRefs, computed, ref, watch } from 'vue';
import { getInjected } from '@snapshot-labs/lock/src/utils';
import connectors from '@/helpers/connectors.json';
import { getIpfsUrl } from '@/helpers/utils';

const props = defineProps<{
  open: boolean;
}>();

defineEmits(['login', 'close']);

const { open } = toRefs(props);

const isShowingAllConnectors = ref(false);

const injected = computed(() => getInjected());

const filteredConnectors = computed(() => {
  const baseConnectors = ['injected', 'walletconnect', 'walletlink'];
  if (isShowingAllConnectors.value) return Object.keys(connectors);
  return Object.keys(connectors).filter(cId => baseConnectors.includes(cId));
});

watch(open, () => {
  isShowingAllConnectors.value = false;
});
</script>

<template>
  <BaseModal :open="open" @close="$emit('close')">
    <template #header>
      <h3>Create your Grant</h3>
    </template>
    <div>
      <div class="m-4 space-y-2">
        <BaseInput :title="`Grant name`" :max-length="128" focus-on-mount />
        <BaseInput :title="`Grant name`" :max-length="128" focus-on-mount />
        <BaseButton :primary="true" class="w-full"> Create Grant </BaseButton>
      </div>
    </div>
  </BaseModal>
</template>
