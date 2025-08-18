<script>
  import * as Dialog from '$lib/components/ui/dialog/index.js';
  import { MediaQuery } from 'svelte/reactivity';
  import * as Drawer from '$lib/components/ui/drawer/index.js';
  import { onMount } from 'svelte';
  import Input from './ui/input/input.svelte';
  import { CopyIcon, DownloadCloudIcon, DownloadIcon } from 'lucide-svelte';
  import { toast } from 'svelte-sonner';

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
        <Drawer.Description class="mb-20">{@render inputs()}</Drawer.Description>
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
    Please rename the file manually.
    <br />
    <br />
    File Name:
    <button
    onclick={()=>{
      try {
        navigator.clipboard.writeText(`FD-${fileName}`).then(toast.success("Copied to clipboard", {description: `FD-${fileName}`}))
      } catch (e) {
        toast.error("Error while trying to copy", {description: e.message})
      }
    }}
      class="cursor-pointer flex h-9 w-max min-w-0 items-center rounded-md border border-input bg-background px-3 py-1 text-base shadow-xs ring-offset-background transition-[color,box-shadow] outline-none selection:bg-primary selection:text-primary-foreground placeholder:text-muted-foreground disabled:cursor-not-allowed disabled:opacity-50 md:text-sm dark:bg-input/30"
      >
      FD-{fileName}
      <CopyIcon class="ml-3 h-3 w-3" />
    </button>
    <br />
    Sideload File Name (inside ZIP):
    <button
    onclick={()=>{
            try {
        navigator.clipboard.writeText(sideloadName).then(toast.success("Copied to clipboard", {description: sideloadName}))
      } catch (e) {
        toast.error("Error while trying to copy", {description: e.message})
      }
    }}
      class="cursor-pointer flex h-9 w-max min-w-0 items-center rounded-md border border-input bg-background px-3 py-1 text-base shadow-xs ring-offset-background transition-[color,box-shadow] outline-none selection:bg-primary selection:text-primary-foreground placeholder:text-muted-foreground disabled:cursor-not-allowed disabled:opacity-50 md:text-sm dark:bg-input/30"
    >
      {sideloadName}
      <CopyIcon class="ml-3 h-3 w-3" />
    </button>
  </div>
{/snippet}
