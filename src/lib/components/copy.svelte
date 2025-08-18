<script>
  import * as Dialog from '$lib/components/ui/dialog/index.js';
  import { MediaQuery } from 'svelte/reactivity';
  import * as Drawer from '$lib/components/ui/drawer/index.js';
  import { onMount } from 'svelte';
  import Input from './ui/input/input.svelte';
  import { DownloadCloudIcon, DownloadIcon } from 'lucide-svelte';

  const isMobile = new MediaQuery('max-width: 48rem');
  const title = 'Your file will be downloaded shortly';

  let { isOpen = $bindable(), generateSideloadFileName, generateFileName, variant } = $props();

  let fileName = $state('');
  let sideloadName = $state('');

  $effect(() => {
    if (variant) {
      fileName = generateFileName();
      sideloadName = generateSideloadFileName();
    }
  });
</script>

<!-- Copy Filename Dialogs and Drawers -->
{#if isMobile.current}
  <Drawer.Root bind:open={isOpen}>
    <!-- <Drawer.Trigger>Mobile</Drawer.Trigger> -->
    <Drawer.Content>
      <Drawer.Header>
        <Drawer.Title class="flex items-center gap-2">
          <DownloadCloudIcon class="h-5 w-5" />
          {title}
        </Drawer.Title>
        <Drawer.Description>{@render inputs()}</Drawer.Description>
      </Drawer.Header>
    </Drawer.Content>
  </Drawer.Root>
{:else}
  <Dialog.Root bind:open={isOpen}>
    <!-- <Dialog.Trigger>Desktop</Dialog.Trigger> -->
    <Dialog.Content>
      <Dialog.Header>
        <Dialog.Title class="flex items-center gap-2">
          <DownloadCloudIcon class="h-5 w-5" />
          {title}
        </Dialog.Title>
        <Dialog.Description>{@render inputs()}</Dialog.Description>
      </Dialog.Header>
    </Dialog.Content>
  </Dialog.Root>
{/if}

{#snippet inputs()}
  <div>
    Please rename the file manually to this:
    <br />
    <code>
      Filename: {fileName}, Sideload: {sideloadName}
    </code>
  </div>
{/snippet}
