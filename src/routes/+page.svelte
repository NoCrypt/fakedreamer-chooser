<script>
  import { onMount } from 'svelte';
  import { Button } from '$lib/components/ui/button';
  import {
    Card,
    CardContent,
    CardDescription,
    CardHeader,
    CardTitle
  } from '$lib/components/ui/card';
  import { Select, SelectContent, SelectItem, SelectTrigger } from '$lib/components/ui/select';
  import { Badge } from '$lib/components/ui/badge';
  import {
    Download,
    Github,
    Cpu,
    Battery,
    Smartphone,
    Shield,
    Settings,
    Loader,
    LoaderPinwheel,
    LoaderCircle
  } from 'lucide-svelte';
  import Label from '$lib/components/ui/label/label.svelte';
  import Checkbox from '$lib/components/ui/checkbox/checkbox.svelte';
  import * as Tooltip from '$lib/components/ui/tooltip/index.js';
  import Info from '$lib/components/info.svelte';
  import { unzip, zip } from 'fflate';
  import { toast } from 'svelte-sonner';
  import * as Accordion from '$lib/components/ui/accordion/index.js';
  import Copy from '$lib/components/copy.svelte';
  import { base } from '$app/paths';

  let releases = $state([]);
  let latestLocal = $state([]);
  let loading = $state(true);
  let variant = $state({
    device: 'alioth',
    ksu: 'ksu',
    battery: 'stk',
    ui: 'aosp',
    cpu: 'default',
    gpu: 'default'
  });

  let wantSideload = $state(false);
  let latestDate;
  let showAllVersions = $state('');
  let showCopyModal = $state(false);

  onMount(() => {
    try {
      const storedVariant = localStorage.getItem('variant');
      if (storedVariant) {
        variant = JSON.parse(storedVariant);
      }

      const localStorageShowAllVersions = localStorage.getItem('showAllVersions');
      if (localStorageShowAllVersions) {
        showAllVersions = localStorageShowAllVersions;
      }
    } catch (error) {
      toast.error('Failed to retrieve variant from localStorage:', { description: error });
    }
    fetchReleases();
    fetchLatestLocal();
  });

  $effect(() => {
    localStorage.setItem('variant', JSON.stringify(variant));
  });
  $effect(() => {
    localStorage.setItem('showAllVersions', showAllVersions);
  });

  $effect(() => {
    if (variant.device === 'apollo' && variant.battery === 'j3s') {
      variant.battery = 'stk';
    }
  });

  async function fetchReleases() {
    try {
      const response = await fetch('https://api.github.com/repos/re-noroi/kernel_sm8250/releases');
      const data = await response.json();
      releases = data;
    } catch (error) {
      console.error('Failed to fetch releases:', error);
    } finally {
      loading = false;
    }
  }
  async function fetchLatestLocal() {
    try {
      const response = await fetch(`${base}/downloads/latest.json`);
      const data = await response.json();
      latestLocal = data.latest;
    } catch (error) {
      console.error('Failed to fetch releases:', error);
    }
  }

  function generateFileName() {
    const parts = [];

    // Battery is required
    parts.push(variant.battery);

    // Add optional parts
    if (variant.ui !== 'aosp') parts.push(variant.ui);
    if (variant.cpu !== 'default') parts.push(variant.cpu);
    if (variant.gpu !== 'default') parts.push(variant.gpu);

    // parts.push('xxx');
    return parts.join('-') + '.zip';
  }

  let generatedFileName = $derived(generateFileName(variant));

  function generateSideloadFileName() {
    const parts = [];

    // Battery
    parts.push(variant.battery);

    // UI - convert "aosp" to "aosp"
    if (variant.ui === 'aosp') {
      parts.push('aosp');
    } else {
      parts.push(variant.ui);
    }

    // CPU and GPU (only if value)
    if (variant.cpu !== 'default') parts.push(variant.cpu);
    if (variant.gpu !== 'default') parts.push(variant.gpu);

    return parts.join('-') + '.dreamless';
  }

  function findMatchingAsset(release) {
    const deviceName = variant.device;
    const ksuVariant =
      variant.ksu === 'non-ksu' ? 'noksu' : variant.ksu === 'ksu' ? 'next' : variant.ksu;

    return release.assets.find(
      (asset) =>
        asset.name.toLowerCase().includes(deviceName) &&
        asset.name.toLowerCase().includes(ksuVariant)
    );
  }

  function getAvailableReleases() {
    return releases.filter((release) => {
      const hasMatchingAsset = findMatchingAsset(release);
      return hasMatchingAsset;
    });
  }

  async function handleDownload(ref, release) {
    try {
      ref.srcElement.disabled = true;
      const asset = findMatchingAsset(release);
      if (!asset) {
        return;
      }

      // const fileName = generateFileName();
      // link.href = asset.browser_download_url;
      // 1) Fetch ZIP
      const res = await fetch(`${base}/downloads/${release?.tag_name}/${asset?.name}`, {
        credentials: 'omit'
      });
      if (!res.ok) throw new Error(`HTTP ${res.status} ${res.statusText}`);
      let buf = new Uint8Array(await res.arrayBuffer());

      if (wantSideload) {
        // 2) Unzip to in-memory files
        const files = await new Promise((resolve, reject) => {
          unzip(buf, (err, data) => (err ? reject(err) : resolve(data)));
        });

        // 3) Rename dreamless file
        for (const [fname, bytes] of Object.entries(files)) {
          if (fname.includes('.dreamless')) {
            let outName = generateSideloadFileName();
            files[outName] = bytes;
            delete files[fname];
          }
        }

        // 4) Re-zip all files
        buf = await new Promise((resolve, reject) => {
          zip(files, (err, data) => (err ? reject(err) : resolve(data)));
        });
      }


      // 5) Trigger download
      const blob = new Blob([buf], { type: 'application/zip' });
      const link = document.createElement('a');
      link.href = URL.createObjectURL(blob);
      link.download = asset.name.slice(0, -4) + '-' + generateFileName();
      link.click();
      URL.revokeObjectURL(link.href);
      link.remove();
    } catch (e) {
      toast.error('Download failed', {
        description: e.message
      });
    } finally {
      ref.srcElement.disabled = false;
    }
  }

  function handleManualDownload(release) {
    const asset = findMatchingAsset(release);
    if (!asset) {
      toast.error('Asset not found.');
      return;
    }
    const link = document.createElement('a');
    link.href = asset.browser_download_url;
    link.download = release.name.slice(0, -4) + '-' + generateFileName();
    link.click();
    link.remove();
    showCopyModal = true;
  }
</script>

<Copy bind:isOpen={showCopyModal} {generateFileName} {generateSideloadFileName} {variant} />

<div class="min-h-screen bg-background p-4">
  <div class="mx-auto max-w-4xl space-y-6">
    <!-- Header -->
    <div class="mt-8 space-y-2 text-center">
      <h1 class="flex items-center justify-center gap-2 text-3xl font-bold">
        <Cpu class="h-8 w-8" />
        FakeDreamer Downloader
      </h1>
      <p class="text-muted-foreground">Download custom kernel variants for your device</p>
      <Button
        variant="link"
        href="https://github.com/re-noroi/kernel_sm8250/releases"
        class="text-muted-foreground">GitHub</Button
      >
      <Button variant="link" href="https://t.me/renoroihub" class="text-muted-foreground"
        >Telegram</Button
      >
    </div>

    <!-- Configuration Card -->
    <Card>
      <CardHeader>
        <CardTitle class="flex items-center gap-2">
          <Settings class="h-5 w-5" />
          Kernel Configuration
        </CardTitle>
        <CardDescription>Configure the kernel to your needs.</CardDescription>
      </CardHeader>
      <CardContent class="space-y-4">
        <div class="grid grid-cols-1 gap-4 md:grid-cols-2 lg:grid-cols-3">
          <!-- Device Selection -->
          <div class="space-y-2">
            <Label class="flex items-center gap-1 text-sm font-medium">
              <Smartphone class="h-5 w-5" />
              Device
              <Info>
                Alioth: Xiaomi Poco F3 / Xiaomi Mi 11X / Redmi K40
                <br /> Apollo: Xiaomi Mi 10T / Mi 10T Pro / Redmi K30S Ultra
              </Info>
            </Label>
            <Select type="single" bind:value={variant.device}>
              <SelectTrigger>
                <!-- <SelectValue /> -->{variant.device === 'alioth' ? 'Alioth' : 'Apollo'}
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="alioth">Alioth</SelectItem>
                <SelectItem value="apollo">Apollo</SelectItem>
              </SelectContent>
            </Select>
          </div>

          <!-- KSU Variant -->
          <div class="space-y-2">
            <Label class="flex items-center gap-1 text-sm font-medium">
              <Shield class="h-4 w-4" />
              KSU Variant
              <Info>
                Use KSU if you want root.
                <br /> Use Non-KSU if you want to use other root method.
                <br /> Use SUSFS only if you need it for banking apps.
              </Info>
            </Label>
            <Select type="single" bind:value={variant.ksu}>
              <SelectTrigger>
                <!-- <SelectValue /> -->{variant.ksu === 'ksu'
                  ? 'KSU'
                  : variant.ksu === 'non-ksu'
                    ? 'Non-KSU'
                    : 'SuSFS'}
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="ksu">KSU</SelectItem>
                <SelectItem value="non-ksu">Non-KSU</SelectItem>
                <SelectItem value="susfs">SuSFS</SelectItem>
              </SelectContent>
            </Select>
          </div>

          <!-- Battery -->
          <div class="space-y-2">
            <Label class="flex items-center gap-1 text-sm font-medium">
              <Battery class="h-4 w-4" />
              Battery
              <!-- <Badge variant="destructive" class="text-xs">Required</Badge> -->
              <Info>
                Stock Battery is 4500mAh
                <br /> If you use OEM battery or Apollo battery on Alioth
                <br /> then use the 5000mAh/j3s
              </Info>
            </Label>
            <Select type="single" bind:value={variant.battery}>
              <SelectTrigger>
                <!-- <SelectValue /> -->{variant.battery === 'stk'
                  ? 'Stock Battery (-stk)'
                  : '5000mAh Battery (-j3s)'}
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="stk">Stock Battery (-stk)</SelectItem>
                <SelectItem value="j3s">
                  5000mAh Battery (-j3s) {variant.device === 'apollo' ? '(Limited support)' : ''}
                </SelectItem>
              </SelectContent>
            </Select>
          </div>

          <!-- UI -->
          <div class="space-y-2">
            <Label class="flex items-center gap-1 text-sm font-medium">
              DTBO
              <Info>
                Most of the new AOSP ROMs are based on LineageOS.
                <br /> Please check if your ROM is using the new IR Driver or not.
                <br /> MIUI/HyperOS user should ALWAYS choose the MIUI
              </Info>
            </Label>

            <Select type="single" bind:value={variant.ui}>
              <SelectTrigger>
                <!-- <SelectValue placeholder="AOSP" /> -->{variant.ui === 'aosp'
                  ? 'AOSP ROM (default)'
                  : variant.ui === 'ir'
                    ? 'LineageOS/LOS Based (-ir)'
                    : 'MIUI/HyperOS (-miui)'}
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="aosp">AOSP ROM (default)</SelectItem>
                <SelectItem value="ir">LineageOS/LOS Based (-ir)</SelectItem>
                <SelectItem value="miui">MIUI/HyperOS (-miui)</SelectItem>
              </SelectContent>
            </Select>
          </div>

          <!-- CPU -->
          <div class="space-y-2">
            <Label class="flex items-center gap-1 text-sm font-medium">
              CPU Optimization
              <Info>
                Efficiency Mode.
                <br /> This will disables 2.6GHz - 3.2GHz CPU clock speed.
                <br /> Lower power consumption, less heat, better CPU lifespan.
                <br /> Ideal for daily use and battery life.
                <br /> <strong>Note:</strong> This may decrease performance by a bit. Barely noticable.
              </Info>
            </Label>
            <Select type="single" bind:value={variant.cpu}>
              <SelectTrigger>
                <!-- <SelectValue placeholder="Default" /> -->{variant.cpu === 'default'
                  ? 'Default'
                  : variant.cpu === 'eff'
                    ? 'Efficiency Mode (-eff)'
                    : ''}
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="default">Default</SelectItem>
                <SelectItem value="eff">Efficiency Mode (-eff)</SelectItem>
              </SelectContent>
            </Select>
          </div>

          <!-- GPU -->
          <div class="space-y-2">
            <Label class="flex items-center gap-1 text-sm font-medium">
              GPU Optimization
              <Info>
                Undervolted GPU.
                <br /> This will undervolt the GPU to reduce power consumption and heat.
                <br /> <strong>Note:</strong> Some device doesnt support undervolting GPU.
                <br /> Resulting random freezes (artifacts) and random reboots.
                <br /> <strong>If you experience this, please disable this option.</strong>
              </Info>
            </Label>
            <Select type="single" bind:value={variant.gpu}>
              <SelectTrigger>
                <!-- <SelectValue placeholder="Default" /> -->{variant.gpu === 'default'
                  ? 'Default'
                  : variant.gpu === 'uv'
                    ? 'Undervolted (-uv)'
                    : ''}
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="default">Default</SelectItem>
                <SelectItem value="uv">Undervolted (-uv)</SelectItem>
              </SelectContent>
            </Select>
          </div>
        </div>

        <div class="space-y-2">
          <Label class="flex items-center gap-1 text-sm font-medium">
            <Checkbox bind:checked={wantSideload} />
            Sideload Support
            <Info>
              Enables sideloading support.
              <br /> This will automatically extract the zip,
              <br /> rename the dreamless file, and repack the zip.
              <br /> <strong>Note:</strong> This will make the process takes a while.
            </Info>
          </Label>
        </div>
      </CardContent>
    </Card>

    <Card>
      <CardHeader>
        <CardTitle class="flex items-center gap-2">
          <Github class="h-5 w-5" />
          Latest Version
          <Info>Ask in group chat if it's not up-to-date.</Info>
        </CardTitle>
        <CardDescription>
          Files will be automatically renamed to match your configuration when downloaded.
        </CardDescription>
      </CardHeader>
      <CardContent>
        {@render releaseList(true)}
      </CardContent>
    </Card>

    <Accordion.Root type="single" bind:value={showAllVersions}>
      <Accordion.Item value="item-1">
        <Accordion.Trigger
          class="mb-4 cursor-pointer gap-6 rounded-xl border bg-card p-6 text-card-foreground shadow-sm"
          >All Versions</Accordion.Trigger
        >
        <Accordion.Content>
          <!-- Available Releases -->
          <Card>
            <CardHeader>
              <CardTitle class="flex items-center gap-2">
                <Github class="h-5 w-5" />
                All Versions
              </CardTitle>
              <CardDescription>
                Files below require manual renaming.
                <Info>
                  Only latest versions can do auto-renaming due to CORS issue
                  <br /> Latest kernels are hosted in the site to avoid this.
                  <br /> Hosting all kernel versions will make github bonk my head.
                </Info>
              </CardDescription>
            </CardHeader>
            <CardContent>
              {@render releaseList(false)}
            </CardContent>
          </Card>
        </Accordion.Content>
      </Accordion.Item>
    </Accordion.Root>
  </div>
</div>

{#snippet releaseList(isLatest = false)}
  {#if loading}
    <div class="py-8 text-center">
      <!-- <div class="mx-auto h-8 w-8 animate-spin rounded-full border-b-2 border-primary"></div> -->
      <LoaderCircle class="mx-auto h-8 w-8 animate-spin" />
      <p class="mt-2 text-muted-foreground">Loading releases...</p>
    </div>
  {:else}
    <div class="space-y-3">
      {#each getAvailableReleases() as release (release.tag_name)}
        {@const asset = findMatchingAsset(release)}
        {@const match = latestLocal.some((term) => asset?.name.includes(term))}
        {#if isLatest ? match : !match}
          {@const isWVariant = release.tag_name.includes('-W')}
          {@const isLegacyVariant = release.tag_name.includes('legacy')}
          {@const isHotfix = release.name.includes('Hotfix')}
          {@const isReleaseCandidate = release.name.includes('-rc')}
          <div class="flex items-center justify-between rounded-lg border p-4">
            <div class="space-y-1">
              <div class="flex items-center gap-2">
                <h3 class="font-medium">{release.tag_name}</h3>
                {#if isWVariant}
                  <Badge variant="secondary" class="bg-blue-700">For HyperOS A16</Badge>
                {/if}
                {#if isLegacyVariant}
                  <Badge variant="secondary" class="bg-yellow-700">Legacy</Badge>
                {/if}
                {#if isHotfix}
                  <Badge variant="secondary" class="bg-green-700">Hotfix</Badge>
                {/if}
                {#if isReleaseCandidate}
                  <Badge variant="secondary" class="bg-pink-700">RC</Badge>
                {/if}
              </div>

              <p class="text-sm text-muted-foreground">
                <!-- {asset?.name.slice(0, -4)}-{generatedFileName} -->
                {asset?.name}
              </p>
            </div>
            <Button
              onclick={(e) =>
                isLatest ? handleDownload(e, release) : handleManualDownload(release)}
              disabled={!asset}
              class="flex cursor-pointer items-center gap-2"
            >
              <Download class="h-4 w-4 in-disabled:hidden" />
              <LoaderCircle class="hidden h-4 w-4 animate-spin in-disabled:block" />
              <span class="pointer-events-none hidden md:block"> Download </span>
            </Button>
          </div>
        {/if}
      {/each}

      {#if getAvailableReleases().length === 0}
        <div class="py-8 text-center text-muted-foreground">
          No releases found for the value configuration
        </div>
      {/if}
    </div>
  {/if}
{/snippet}
