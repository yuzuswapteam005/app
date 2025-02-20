<script lang="ts" context="module">
  export const DELETE_STREAM_CONFIRM_STEP_STREAM_FRAGMENT = gql`
    fragment DeleteStreamConfirmStep on Stream {
      id
    }
  `;
</script>

<script lang="ts">
  import Button from '$lib/components/button/button.svelte';
  import StepHeader from '$lib/components/step-header/step-header.svelte';
  import StepLayout from '$lib/components/step-layout/step-layout.svelte';
  import { makeTransactPayload, type StepComponentEvents } from '$lib/components/stepper/types';
  import { createEventDispatcher } from 'svelte';
  import { getAddressDriverClient, getCallerClient } from '$lib/utils/get-drips-clients';
  import walletStore from '$lib/stores/wallet/wallet.store';
  import { buildStreamDeleteBatchTx } from '$lib/utils/streams/streams';
  import { gql } from 'graphql-request';
  import type { DeleteStreamConfirmStepFragment } from './__generated__/gql.generated';
  import assert from '$lib/utils/assert';
  import { waitForAccountMetadata } from '$lib/utils/ipfs';
  import { goto } from '$app/navigation';
  import SkullIcon from '$lib/components/icons/💀.svelte';

  const dispatch = createEventDispatcher<StepComponentEvents>();

  export let stream: DeleteStreamConfirmStepFragment;

  function startDeleting() {
    dispatch(
      'transact',
      makeTransactPayload({
        headline: 'Deleting stream',
        icon: {
          component: SkullIcon,
          props: { size: 48 },
        },
        before: async () => {
          const addressDriverClient = await getAddressDriverClient();
          const callerClient = await getCallerClient();

          const { signer } = $walletStore;
          assert(signer);

          const { batch, newHash } = await buildStreamDeleteBatchTx(
            addressDriverClient,
            signer,
            stream.id,
          );

          return {
            callerClient,
            batch,
            newHash,
          };
        },

        transactions: async ({ callerClient, batch }) => [
          {
            transaction: await callerClient.populateCallBatchedTx(batch),
            applyGasBuffer: true,
            title: 'Delete the stream',
          },
        ],

        after: async (_, { newHash }) => {
          const { dripsAccountId } = $walletStore;
          assert(dripsAccountId);

          await waitForAccountMetadata(dripsAccountId, newHash);

          await goto('/funds');
        },
      }),
    );
  }
</script>

<StepLayout>
  <StepHeader
    emoji="💀"
    headline="Delete stream"
    description="Are you sure that you want to delete this stream? It will immediately stop streaming, and be irreversibly erased."
  />
  <svelte:fragment slot="actions">
    <Button on:click={() => dispatch('conclude')} variant="ghost">Cancel</Button>
    <Button on:click={startDeleting} variant="destructive">Delete stream</Button>
  </svelte:fragment>
</StepLayout>
