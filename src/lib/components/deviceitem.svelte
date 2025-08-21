<script>
  import Badge from "./ui/badge/badge.svelte";

  let { img = "", title, desc, isNew = false } = $props();

</script>

<div class="flex gap-3 items-center">
  <img src={img} alt="phone" class="object-contain w-10 rounded-xs"/>
  <div>
    <h4 class="text-sm font-bold">
      {title}
      {#if isNew}
        <Badge variant="secondary">New</Badge>
      {/if}
    </h4>
    <span class="text-xs text-muted-foreground">
      {desc}
    </span>
  </div>
</div>