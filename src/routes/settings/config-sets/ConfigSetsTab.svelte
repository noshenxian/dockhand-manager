<script lang="ts">
	import { onMount } from 'svelte';
	import { fade } from 'svelte/transition';
	import { toast } from 'svelte-sonner';
	import { Button } from '$lib/components/ui/button';
	import * as Card from '$lib/components/ui/card';
	import { Badge } from '$lib/components/ui/badge';
	import { Plus, Trash2, Pencil, Layers } from 'lucide-svelte';
	import ConfirmPopover from '$lib/components/ConfirmPopover.svelte';
	import { canAccess } from '$lib/stores/auth';
	import ConfigSetModal from './ConfigSetModal.svelte';
	import { EmptyState } from '$lib/components/ui/empty-state';
	import { _ } from '$lib/i18n';

	// Config set types
	interface ConfigSet {
		id: number;
		name: string;
		description?: string;
		envVars: { key: string; value: string }[];
		labels: { key: string; value: string }[];
		ports: { hostPort: string; containerPort: string; protocol: string }[];
		volumes: { hostPath: string; containerPath: string; mode: string }[];
		networkMode: string;
		restartPolicy: string;
		createdAt: string;
		updatedAt: string;
	}

	// Config set state
	let configSets = $state<ConfigSet[]>([]);
	let cfgLoading = $state(true);
	let showCfgModal = $state(false);
	let editingCfg = $state<ConfigSet | null>(null);
	let confirmDeleteConfigSetId = $state<number | null>(null);
	function getNetworkLabel(mode: string): string {
		switch (mode) {
			case 'bridge':
				return $_('settings.config_sets_page.network_bridge');
			case 'host':
				return $_('settings.config_sets_page.network_host');
			case 'none':
				return $_('settings.config_sets_page.network_none');
			default:
				return mode;
		}
	}

	function getRestartLabel(policy: string): string {
		switch (policy) {
			case 'no':
				return $_('settings.config_sets_page.restart_no');
			case 'always':
				return $_('settings.config_sets_page.restart_always');
			case 'on-failure':
				return $_('settings.config_sets_page.restart_on_failure');
			case 'unless-stopped':
				return $_('settings.config_sets_page.restart_unless_stopped');
			default:
				return policy;
		}
	}

	async function fetchConfigSets() {
		cfgLoading = true;
		try {
			const response = await fetch('/api/config-sets');
			configSets = await response.json();
		} catch (error) {
			console.error('Failed to fetch config sets:', error);
			toast.error($_('settings.config_sets_page.fetch_failed'));
		} finally {
			cfgLoading = false;
		}
	}

	function openCfgModal(cfg?: ConfigSet) {
		editingCfg = cfg || null;
		showCfgModal = true;
	}

	async function deleteConfigSet(id: number) {
		try {
			const response = await fetch(`/api/config-sets/${id}`, {
				method: 'DELETE'
			});

			if (response.ok) {
				await fetchConfigSets();
				toast.success($_('settings.config_sets_page.delete_success'));
			} else {
				const data = await response.json();
				toast.error(data.error || $_('settings.config_sets_page.delete_failed'));
			}
		} catch (error) {
			toast.error($_('settings.config_sets_page.delete_failed'));
		}
	}

	onMount(() => {
		fetchConfigSets();
	});
</script>

<div class="space-y-4">
	<Card.Root class="border-dashed">
		<Card.Content class="pt-4">
			<div class="flex items-start gap-3">
				<Layers class="w-5 h-5 text-muted-foreground mt-0.5" />
				<div>
					<p class="text-sm font-medium">{$_('settings.config_sets_page.title')}</p>
					<p class="text-xs text-muted-foreground mt-1">
						{$_('settings.config_sets_page.description')}
					</p>
				</div>
			</div>
		</Card.Content>
	</Card.Root>

	<div class="flex justify-between items-center">
		<div class="flex items-center gap-3">
			<Badge variant="secondary" class="text-xs">{$_('settings.config_sets_page.total', { values: { count: configSets.length } })}</Badge>
		</div>
		<div class="flex gap-2">
			{#if $canAccess('configsets', 'create')}
				<Button size="sm" onclick={() => openCfgModal()}>
					<Plus class="w-4 h-4 mr-1" />
					{$_('settings.config_sets_page.add')}
				</Button>
			{/if}
			<Button size="sm" variant="outline" onclick={fetchConfigSets}>{$_('common.refresh')}</Button>
		</div>
	</div>

	{#if cfgLoading && configSets.length === 0}
		<p class="text-muted-foreground text-sm">{$_('settings.config_sets_page.loading')}</p>
	{:else if configSets.length === 0}
		<EmptyState
			icon={Layers}
			title={$_('settings.config_sets_page.empty_title')}
			description={$_('settings.config_sets_page.empty_desc')}
		/>
	{:else}
		<div class="grid gap-4 md:grid-cols-2 lg:grid-cols-3">
			{#each configSets as cfg (cfg.id)}
				<div out:fade={{ duration: 200 }}>
				<Card.Root>
					<Card.Header class="pb-2">
						<div class="flex items-start justify-between">
							<div class="flex items-center gap-2">
								<Layers class="w-5 h-5 text-muted-foreground" />
								<Card.Title class="text-base">{cfg.name}</Card.Title>
							</div>
						</div>
					</Card.Header>
					<Card.Content class="space-y-3">
						{#if cfg.description}
							<p class="text-sm text-muted-foreground">{cfg.description}</p>
						{/if}

						<div class="flex flex-wrap gap-1.5">
							{#if cfg.envVars && cfg.envVars.length > 0}
								<Badge variant="outline" class="text-xs">{$_('settings.config_sets_page.env_vars_count', { values: { count: cfg.envVars.length } })}</Badge>
							{/if}
							{#if cfg.labels && cfg.labels.length > 0}
								<Badge variant="outline" class="text-xs">{$_('settings.config_sets_page.labels_count', { values: { count: cfg.labels.length } })}</Badge>
							{/if}
							{#if cfg.ports && cfg.ports.length > 0}
								<Badge variant="outline" class="text-xs">{$_('settings.config_sets_page.ports_count', { values: { count: cfg.ports.length } })}</Badge>
							{/if}
							{#if cfg.volumes && cfg.volumes.length > 0}
								<Badge variant="outline" class="text-xs">{$_('settings.config_sets_page.volumes_count', { values: { count: cfg.volumes.length } })}</Badge>
							{/if}
						</div>

						<div class="text-xs text-muted-foreground">
							<span>{$_('settings.config_sets_page.network_label')}: {getNetworkLabel(cfg.networkMode)}</span>
							<span class="mx-1">|</span>
							<span>{$_('settings.config_sets_page.restart_label')}: {getRestartLabel(cfg.restartPolicy)}</span>
						</div>

						<div class="flex gap-2 pt-2">
							{#if $canAccess('configsets', 'edit')}
								<Button
									variant="outline"
									size="sm"
									onclick={() => openCfgModal(cfg)}
								>
									<Pencil class="w-3 h-3 mr-1" />
									{$_('common.edit')}
								</Button>
							{/if}
							{#if $canAccess('configsets', 'delete')}
								<ConfirmPopover
									open={confirmDeleteConfigSetId === cfg.id}
									action={$_('common.delete')}
									itemType={$_('settings.config_sets_page.config_set')}
									itemName={cfg.name}
									title={$_('common.remove')}
									position="left"
									onConfirm={() => deleteConfigSet(cfg.id)}
									onOpenChange={(open) => confirmDeleteConfigSetId = open ? cfg.id : null}
								>
									{#snippet children({ open })}
										<Trash2 class="w-3 h-3 {open ? 'text-destructive' : 'text-muted-foreground hover:text-destructive'}" />
									{/snippet}
								</ConfirmPopover>
							{/if}
						</div>
					</Card.Content>
				</Card.Root>
				</div>
			{/each}
		</div>
	{/if}
</div>

<ConfigSetModal
	bind:open={showCfgModal}
	configSet={editingCfg}
	onClose={() => { showCfgModal = false; editingCfg = null; }}
	onSaved={fetchConfigSets}
/>
