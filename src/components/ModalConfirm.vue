<script setup lang="ts">
import { computed, ref, watch } from 'vue';
import { useI18n } from '@/composables/useI18n';
import { shorten, getChoiceString, explorerUrl } from '@/helpers/utils';
import { useClient } from '@/composables/useClient';
import { useIntl } from '@/composables/useIntl';
import { getPower } from '@/helpers/snapshot';
import { useWeb3 } from '@/composables/useWeb3';
import { useProposals } from '@/composables';
import { ExtendedSpace, Proposal } from '@/helpers/interfaces';
import shutterEncryptChoice from '@/helpers/shutter';

const { web3Account } = useWeb3();

const vp = ref(0);
const vpByStrategy = ref([]);
const vpLoading = ref(false);
const vpLoadingFailed = ref(false);
const vpLoaded = ref(false);
const reason = ref('');

const props = defineProps<{
  open: boolean;
  space: ExtendedSpace;
  proposal: Proposal;
  selectedChoices: number | number[] | Record<string, any> | null;
  strategies: { name: string; network: string; params: Record<string, any> }[];
}>();

const emit = defineEmits(['reload', 'close', 'openPostVoteModal']);

const { t } = useI18n();
const { send, isSending } = useClient();
const format = getChoiceString;
const { formatNumber, formatCompactNumber } = useIntl();
const { addVotedProposalId } = useProposals();

const symbols = computed(() =>
  props.strategies.map(strategy => strategy.params.symbol || '')
);

const isLoadingShutter = ref(false);

async function voteShutter() {
  isLoadingShutter.value = true;
  const encryptedChoice = await shutterEncryptChoice(
    JSON.stringify(props.selectedChoices),
    props.proposal.id
  );
  isLoadingShutter.value = false;

  if (!encryptedChoice) return null;
  return vote({
    proposal: props.proposal,
    choice: encryptedChoice,
    privacy: 'shutter',
    reason: reason.value
  });
}

async function vote(payload) {
  return send(props.space, 'vote', payload);
}

async function handleSubmit() {
  let result: { id: string } | null = null;
  if (props.proposal.privacy === 'shutter') result = await voteShutter();
  else
    result = await vote({
      proposal: props.proposal,
      choice: props.selectedChoices,
      reason: reason.value
    });

  console.log('Result', result);

  if (result?.id) {
    emit('openPostVoteModal');
    emit('reload');
    addVotedProposalId(props.proposal.id);
    emit('close');
  }
}

async function loadVotingPower() {
  vpLoading.value = true;
  vpLoadingFailed.value = false;
  try {
    const result = await getPower(
      props.space,
      web3Account.value,
      props.proposal
    );
    vp.value = result.vp;
    vpByStrategy.value = result.vp_by_strategy;
  } catch (e) {
    vpLoadingFailed.value = true;
    console.log(e);
  } finally {
    vpLoaded.value = true;
    vpLoading.value = false;
  }
}

watch(
  () => [props.open, web3Account.value],
  async () => {
    if (props.open === false) return;
    loadVotingPower();
  }
);
</script>

<template>
  <BaseModal :open="open" hide-close class="flex" @close="$emit('close')">
    <div class="flex flex-auto flex-col">
      <h4 class="m-4 mb-0 text-center">
        {{ $tc('proposal.castVote') }}
      </h4>
      <div slim class="m-4 text-skin-link">
        <div class="flex">
          <span class="mr-1 flex-auto text-skin-text" v-text="$t('choice')" />
          <span
            v-tippy="{
              content:
                format(proposal, selectedChoices).length > 30
                  ? format(proposal, selectedChoices)
                  : null
            }"
            class="ml-4 truncate text-right"
          >
            {{ format(proposal, selectedChoices) }}
          </span>
        </div>
        <div class="flex">
          <span class="mr-1 flex-auto text-skin-text" v-text="$t('snapshot')" />
          <BaseLink
            :link="explorerUrl(proposal.network, proposal.snapshot, 'block')"
            class="float-right"
          >
            {{ formatNumber(Number(proposal.snapshot)) }}
          </BaseLink>
        </div>
        <div class="flex">
          <span
            class="mr-1 flex-auto text-skin-text"
            v-text="$t('votingPower')"
          />
          <span v-if="vpLoadingFailed" class="item-center flex">
            <BaseButtonIcon class="p-0 pr-2" @click="loadVotingPower">
              <i-ho-refresh class="text-sm" />
            </BaseButtonIcon>
            <i-ho-exclamation-circle class="mt-[1px]" />
          </span>
          <span
            v-else-if="vpLoaded && !vpLoading"
            v-tippy="{
              content: vpByStrategy
                .map(
                  (score, index) =>
                    `${formatCompactNumber(score)} ${symbols[index]}`
                )
                .join(' + ')
            }"
          >
            {{ formatCompactNumber(vp) }}
            {{ shorten(proposal.symbol || space.symbol, 'symbol') }}
          </span>
          <LoadingSpinner v-else />
        </div>
        <div class="mt-3">
          <BaseMessageBlock
            v-if="!vpLoading && vpLoadingFailed"
            level="warning"
          >
            {{ t('vpError') }}
          </BaseMessageBlock>
          <BaseMessageBlock v-else-if="vp === 0 && vpLoaded" level="warning">
            {{
              $t('noVotingPower', {
                blockNumber: formatNumber(Number(proposal.snapshot))
              })
            }}
            <BaseLink
              link="https://github.com/snapshot-labs/snapshot/discussions/767"
            >
              {{ $t('learnMore') }}</BaseLink
            >
          </BaseMessageBlock>
          <div v-else-if="props.proposal.privacy !== 'shutter'" class="flex">
            <TextareaAutosize
              v-model="reason"
              :max-length="140"
              class="s-input !rounded-3xl"
              :placeholder="$t('comment.placeholder')"
            />
          </div>
        </div>
      </div>
    </div>
    <template #footer>
      <div class="float-left w-2/4 pr-2">
        <BaseButton type="button" class="w-full" @click="$emit('close')">
          {{ $t('cancel') }}
        </BaseButton>
      </div>
      <div class="float-left w-2/4 pl-2">
        <BaseButton
          :disabled="vp === 0 || isSending || isLoadingShutter"
          :loading="isSending || isLoadingShutter"
          type="submit"
          class="w-full"
          primary
          @click="handleSubmit"
        >
          {{ $t('confirm') }}
        </BaseButton>
      </div>
    </template>
  </BaseModal>
</template>
