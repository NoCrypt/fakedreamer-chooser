<script>
  import * as Tooltip from '$lib/components/ui/tooltip/index.js';
  import { badgeVariants } from './ui/badge';
  import { CircleQuestionMarkIcon } from 'lucide-svelte';
  import * as Drawer from '$lib/components/ui/drawer/index.js';
  import { DownloadCloudIcon, DownloadIcon } from 'lucide-svelte';
  import { MediaQuery } from 'svelte/reactivity';


  let { children } = $props();

  const isMobile = new MediaQuery('max-width: 48rem');
</script>

{#if isMobile.current}
  <Drawer.Root>
    <Drawer.Trigger><CircleQuestionMarkIcon class="h-3 w-3" /></Drawer.Trigger>
      <Drawer.Content>
        <Drawer.Header>
          <Drawer.Title class="flex items-center gap-2">
            <CircleQuestionMarkIcon class="h-5 w-5" />
            Information
          </Drawer.Title>
          <Drawer.Description class="mb-20">
            <p>{@render children()}</p>
          </Drawer.Description>
        </Drawer.Header>
      </Drawer.Content>
  </Drawer.Root>
{:else}
  <Tooltip.Provider delayDuration={0}>
    <Tooltip.Root>
      <Tooltip.Trigger><CircleQuestionMarkIcon class="h-3 w-3" /></Tooltip.Trigger>
      <Tooltip.Content>
        <p>{@render children()}</p>
      </Tooltip.Content>
    </Tooltip.Root>
  </Tooltip.Provider>
{/if}
