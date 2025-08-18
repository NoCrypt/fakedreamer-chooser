<script>
  import * as Tooltip from '$lib/components/ui/tooltip/index.js';
  import { badgeVariants } from './ui/badge';
  import { CircleQuestionMarkIcon } from 'lucide-svelte';
  let { children } = $props();
</script>

<Tooltip.Provider delayDuration={0}>
  <Tooltip.Root>
    <Tooltip.Trigger><CircleQuestionMarkIcon class="h-3 w-3" /></Tooltip.Trigger>
    <Tooltip.Content>
      <p>{@render children()}</p>
    </Tooltip.Content>
  </Tooltip.Root>
</Tooltip.Provider>
