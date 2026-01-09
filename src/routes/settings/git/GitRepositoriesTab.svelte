<script lang="ts">
	import { onMount } from 'svelte';
	import { toast } from 'svelte-sonner';
	import { Button } from '$lib/components/ui/button';
	import * as Card from '$lib/components/ui/card';
	import { Badge } from '$lib/components/ui/badge';
	import { Plus, Trash2, Pencil, GitBranch, FolderGit2, Plug, CheckCircle, XCircle, Loader2, Github, Lock, Globe } from 'lucide-svelte';
	import ConfirmPopover from '$lib/components/ConfirmPopover.svelte';
	import { canAccess } from '$lib/stores/auth';
	import GitRepositoryModal from './GitRepositoryModal.svelte';
	import { EmptyState } from '$lib/components/ui/empty-state';
	import { _ } from '$lib/i18n';

	interface GitCredential {
		id: number;
		name: string;
		authType: string;
	}

	interface GitRepository {
		id: number;
		name: string;
		url: string;
		branch: string;
		credentialId: number | null;
		credentialName?: string;
		createdAt: string;
	}

	let repositories = $state<GitRepository[]>([]);
	let credentials = $state<GitCredential[]>([]);
	let loading = $state(true);
	let showModal = $state(false);
	let editingRepo = $state<GitRepository | null>(null);
	let confirmDeleteId = $state<number | null>(null);
	let testingId = $state<number | null>(null);
	let testResult = $state<{ id: number; success: boolean; message: string } | null>(null);

	async function fetchRepositories() {
		try {
			const response = await fetch('/api/git/repositories');
			repositories = await response.json();
		} catch (error) {
			console.error('Failed to fetch git repositories:', error);
			toast.error($_('settings.git_section.repositories.fetch_failed'));
		} finally {
			loading = false;
		}
	}

	async function fetchCredentials() {
		try {
			const response = await fetch('/api/git/credentials');
			credentials = await response.json();
		} catch (error) {
			console.error('Failed to fetch git credentials:', error);
			toast.error($_('settings.git_section.credentials.fetch_failed'));
		}
	}

	function openModal(repo?: GitRepository) {
		editingRepo = repo || null;
		showModal = true;
	}

	function closeModal() {
		showModal = false;
		editingRepo = null;
	}

	async function handleSaved() {
		await fetchRepositories();
	}

	async function deleteRepository(id: number) {
		try {
			const response = await fetch(`/api/git/repositories/${id}`, { method: 'DELETE' });
			if (response.ok) {
				await fetchRepositories();
				toast.success($_('settings.git_section.repositories.delete_success'));
			} else {
				toast.error($_('settings.git_section.repositories.delete_failed'));
			}
		} catch (error) {
			console.error('Failed to delete repository:', error);
			toast.error($_('settings.git_section.repositories.delete_failed'));
		}
	}

	async function testRepository(id: number) {
		testingId = id;
		testResult = null;
		try {
			const response = await fetch(`/api/git/repositories/${id}/test`, { method: 'POST' });
			const data = await response.json();
			if (data.success) {
				testResult = {
					id,
					success: true,
					message: $_('settings.git_section.repositories.test_success_detail', {
						values: { branch: data.branch, commit: data.lastCommit }
					})
				};
				toast.success($_('settings.git_section.repositories.test_success'));
			} else {
				testResult = {
					id,
					success: false,
					message: data.error || $_('settings.git_section.repositories.connection_failed')
				};
				toast.error($_('settings.git_section.repositories.connection_failed_detail', {
					values: { error: data.error || $_('settings.git_section.unknown_error') }
				}));
			}
			// Auto-clear after 5 seconds
			setTimeout(() => {
				if (testResult?.id === id) {
					testResult = null;
				}
			}, 5000);
		} catch (error) {
			testResult = {
				id,
				success: false,
				message: $_('settings.git_section.repositories.test_failed')
			};
			toast.error($_('settings.git_section.repositories.test_failed'));
		} finally {
			testingId = null;
		}
	}

	onMount(() => {
		fetchCredentials();
		fetchRepositories();
	});
</script>

<div class="space-y-4">
	<div class="flex justify-between items-center">
		<div>
			<h3 class="text-lg font-medium">{$_('settings.git_section.repositories.title')}</h3>
			<p class="text-sm text-muted-foreground">{$_('settings.git_section.repositories.description')}</p>
		</div>
		{#if $canAccess('settings', 'edit')}
			<Button size="sm" onclick={() => openModal()}>
				<Plus class="w-4 h-4 mr-1" />
				{$_('settings.git_section.repositories.add')}
			</Button>
		{/if}
	</div>

	{#if loading}
		<p class="text-sm text-muted-foreground">{$_('settings.git_section.repositories.loading')}</p>
	{:else if repositories.length === 0}
		<Card.Root>
			<Card.Content>
				<EmptyState
					icon={FolderGit2}
					title={$_('settings.git_section.repositories.empty_title')}
					description={$_('settings.git_section.repositories.empty_desc')}
				/>
			</Card.Content>
		</Card.Root>
	{:else}
		<div class="space-y-1">
			{#each repositories as repo (repo.id)}
				<div class="flex items-center justify-between py-2 px-3 rounded-md border bg-card hover:bg-muted/50 transition-colors">
					<div class="flex items-center gap-2 min-w-0 flex-1">
						{#if repo.url.includes('github.com')}
							<Github class="w-4 h-4 shrink-0 text-muted-foreground" />
						{:else}
							<FolderGit2 class="w-4 h-4 shrink-0 text-muted-foreground" />
						{/if}
						<span class="font-medium text-sm truncate">{repo.name}</span>
						<span class="text-xs text-muted-foreground truncate hidden sm:inline">{repo.url}</span>
					</div>
					<div class="flex items-center gap-2 shrink-0">
						{#if testResult?.id === repo.id}
							<span class="flex items-center gap-1 text-xs px-2 py-0.5 rounded {testResult.success ? 'text-green-600 bg-green-50 dark:bg-green-950/30' : 'text-red-600 bg-red-50 dark:bg-red-950/30'}">
								{#if testResult.success}
									<CheckCircle class="w-3 h-3" />
								{:else}
									<XCircle class="w-3 h-3" />
								{/if}
								<span class="hidden sm:inline">{testResult.message}</span>
							</span>
						{/if}
						{#if repo.credentialName}
							<span class="flex items-center gap-1 text-xs text-muted-foreground" title={$_('settings.git_section.repositories.using_credential', { values: { name: repo.credentialName } })}>
								<Lock class="w-3 h-3" />
								<span class="hidden sm:inline">{repo.credentialName}</span>
							</span>
						{:else}
							<span class="flex items-center gap-1 text-xs text-muted-foreground" title={$_('settings.git_section.repositories.public_repo')}>
								<Globe class="w-3 h-3" />
								<span class="hidden sm:inline">{$_('settings.git_section.repositories.public')}</span>
							</span>
						{/if}
						<Badge variant="outline" class="text-xs flex items-center gap-1">
							<GitBranch class="w-3 h-3" />
							{repo.branch}
						</Badge>
						<Button
							variant="ghost"
							size="icon"
							class="h-7 w-7"
							onclick={() => testRepository(repo.id)}
							disabled={testingId === repo.id}
							title={$_('settings.git_section.repositories.test_connection')}
						>
							{#if testingId === repo.id}
								<Loader2 class="w-3.5 h-3.5 animate-spin" />
							{:else}
								<Plug class="w-3.5 h-3.5" />
							{/if}
						</Button>
						{#if $canAccess('settings', 'edit')}
							<Button variant="ghost" size="icon" class="h-7 w-7" onclick={() => openModal(repo)} title={$_('settings.git_section.repositories.edit')}>
								<Pencil class="w-3.5 h-3.5" />
							</Button>
							<ConfirmPopover
								open={confirmDeleteId === repo.id}
								action={$_('common.delete')}
								itemType={$_('settings.git_section.repositories.repository')}
								itemName={repo.name}
								title={$_('common.delete')}
								onConfirm={() => deleteRepository(repo.id)}
								onOpenChange={(open) => confirmDeleteId = open ? repo.id : null}
							>
								{#snippet children({ open })}
									<Trash2 class="w-3.5 h-3.5 {open ? 'text-destructive' : 'text-muted-foreground hover:text-destructive'}" />
								{/snippet}
							</ConfirmPopover>
						{/if}
					</div>
				</div>
			{/each}
		</div>
	{/if}
</div>

<GitRepositoryModal
	bind:open={showModal}
	repository={editingRepo}
	{credentials}
	onClose={closeModal}
	onSaved={handleSaved}
/>
