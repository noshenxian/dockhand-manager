<script lang="ts">
	import { toast } from 'svelte-sonner';
	import { Button } from '$lib/components/ui/button';
	import * as Dialog from '$lib/components/ui/dialog';
	import * as Select from '$lib/components/ui/select';
	import { Label } from '$lib/components/ui/label';
	import { Input } from '$lib/components/ui/input';
	import { Loader2, GitBranch, KeyRound, Lock, Key, Globe, Play, CheckCircle2 } from 'lucide-svelte';
	import { focusFirstInput } from '$lib/utils';
	import { _, locale } from '$lib/i18n';

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
	}

	interface Props {
		open: boolean;
		repository?: GitRepository | null;
		credentials: GitCredential[];
		onClose: () => void;
		onSaved: () => void;
	}

	let { open = $bindable(), repository = null, credentials, onClose, onSaved }: Props = $props();

	// Form state
	let formName = $state('');
	let formUrl = $state('');
	let formBranch = $state('main');
	let formCredentialId = $state<number | null>(null);
	let formError = $state('');
	let formErrors = $state<{ name?: string; url?: string }>({});
	let formSaving = $state(false);

	// Test state
	let testing = $state(false);
	let testResult = $state<{ success: boolean; error?: string; branch?: string; lastCommit?: string } | null>(null);

	const isEditing = $derived(repository !== null);

	function getAuthIcon(type: string) {
		switch (type) {
			case 'ssh': return KeyRound;
			case 'password': return Lock;
			default: return Key;
		}
	}

	function getAuthLabel(type: string) {
		const currentLocale = $locale;
		switch (type) {
			case 'ssh': return $_('settings.git.credentials.auth_ssh');
			case 'password': return $_('settings.git.credentials.auth_password');
			default: return $_('settings.git.credentials.auth_none');
		}
	}

	function resetForm() {
		if (repository) {
			formName = repository.name;
			formUrl = repository.url;
			formBranch = repository.branch;
			formCredentialId = repository.credentialId;
		} else {
			formName = '';
			formUrl = '';
			formBranch = 'main';
			formCredentialId = null;
		}
		formError = '';
		formErrors = {};
		testResult = null;
	}

	// Track which repository was initialized to avoid repeated resets
	let lastInitializedRepoId = $state<number | null | undefined>(undefined);

	$effect(() => {
		if (open) {
			const currentRepoId = repository?.id ?? null;
			if (lastInitializedRepoId !== currentRepoId) {
				lastInitializedRepoId = currentRepoId;
				resetForm();
			}
		} else {
			lastInitializedRepoId = undefined;
		}
	});

	async function testRepository() {
		if (!formUrl.trim()) {
			formErrors.url = $_('settings.git.repositories.url_required_test');
			return;
		}

		testing = true;
		testResult = null;

		try {
			const response = await fetch('/api/git/repositories/test', {
				method: 'POST',
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify({
					url: formUrl.trim(),
					branch: formBranch || 'main',
					credentialId: formCredentialId
				})
			});

			const data = await response.json();
			testResult = data;

			if (data.success) {
				toast.success($_('settings.git.repositories.test_success_detail', {
					values: { branch: data.branch, commit: data.lastCommit }
				}));
			} else {
				toast.error(data.error || $_('settings.git.repositories.test_failed'));
			}
		} catch (error) {
			testResult = { success: false, error: $_('settings.git.repositories.test_failed') };
			toast.error($_('settings.git.repositories.test_failed'));
		} finally {
			testing = false;
		}
	}

	async function saveRepository() {
		formErrors = {};

		if (!formName.trim()) {
			formErrors.name = $_('settings.git.repositories.name_required');
		}

		if (!formUrl.trim()) {
			formErrors.url = $_('settings.git.repositories.url_required');
		}

		if (formErrors.name || formErrors.url) {
			return;
		}

		formSaving = true;
		formError = '';

		try {
			const body = {
				name: formName.trim(),
				url: formUrl.trim(),
				branch: formBranch || 'main',
				credentialId: formCredentialId
			};

			const url = repository
				? `/api/git/repositories/${repository.id}`
				: '/api/git/repositories';
			const method = repository ? 'PUT' : 'POST';

			const response = await fetch(url, {
				method,
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify(body)
			});

			const data = await response.json();

			if (!response.ok) {
				if (data.error?.includes('already exists')) {
					formErrors.name = $_('settings.git.repositories.name_exists');
				} else {
					formError = data.error || $_('settings.git.repositories.save_failed');
				}
				toast.error(formError || $_('settings.git.repositories.save_failed'));
				return;
			}

			const wasEditing = repository !== null;
			onSaved();
			onClose();
			toast.success(wasEditing ? $_('settings.git.repositories.update_success') : $_('settings.git.repositories.add_success'));
		} catch (error) {
			formError = $_('settings.git.repositories.save_failed');
			toast.error($_('settings.git.repositories.save_failed'));
		} finally {
			formSaving = false;
		}
	}

</script>

<Dialog.Root bind:open onOpenChange={(o) => { if (o) focusFirstInput(); else onClose(); }}>
	<Dialog.Content class="max-w-lg">
		<Dialog.Header>
			<Dialog.Title class="flex items-center gap-2">
				<GitBranch class="w-5 h-5" />
				{isEditing ? $_('settings.git.repositories.edit_title') : $_('settings.git.repositories.add_title')}
			</Dialog.Title>
			<Dialog.Description>
				{isEditing ? $_('settings.git.repositories.edit_desc') : $_('settings.git.repositories.add_desc')}
			</Dialog.Description>
		</Dialog.Header>

		<form onsubmit={(e) => { e.preventDefault(); saveRepository(); }} class="space-y-4">
			<div class="space-y-2">
				<Label for="repo-name">{$_('common.name')}</Label>
				<Input
					id="repo-name"
					bind:value={formName}
					placeholder={$_('settings.git.repositories.placeholder_name')}
					class={formErrors.name ? 'border-destructive focus-visible:ring-destructive' : ''}
					oninput={() => formErrors.name = undefined}
				/>
				{#if formErrors.name}
					<p class="text-xs text-destructive">{formErrors.name}</p>
				{:else if !isEditing}
					<p class="text-xs text-muted-foreground">{$_('settings.git.repositories.name_hint')}</p>
				{/if}
			</div>

			<div class="space-y-2">
				<Label for="repo-url">{$_('settings.git.repositories.url')}</Label>
				<Input
					id="repo-url"
					bind:value={formUrl}
					placeholder={$_('settings.git.repositories.placeholder_url')}
					class={formErrors.url ? 'border-destructive focus-visible:ring-destructive' : ''}
					oninput={() => { formErrors.url = undefined; testResult = null; }}
				/>
				{#if formErrors.url}
					<p class="text-xs text-destructive">{formErrors.url}</p>
				{/if}
			</div>

			<div class="space-y-2">
				<Label for="repo-branch">{$_('settings.git.repositories.branch')}</Label>
				<Input id="repo-branch" bind:value={formBranch} placeholder={$_('settings.git.repositories.placeholder_branch')} oninput={() => testResult = null} />
			</div>

			<div class="space-y-2">
				<Label for="repo-credential">{$_('settings.git.repositories.credential_optional')}</Label>
				<Select.Root
					type="single"
					value={formCredentialId?.toString() ?? 'none'}
					onValueChange={(v) => { formCredentialId = v === 'none' ? null : parseInt(v); testResult = null; }}
				>
					<Select.Trigger class="w-full">
						{@const selectedCred = credentials.find(c => c.id === formCredentialId)}
						{#if selectedCred}
							{@const Icon = getAuthIcon(selectedCred.authType)}
							<span class="flex items-center gap-2">
								<Icon class="w-4 h-4 text-muted-foreground" />
								{selectedCred.name} ({getAuthLabel(selectedCred.authType)})
							</span>
						{:else}
							<span class="flex items-center gap-2">
								<Globe class="w-4 h-4 text-muted-foreground" />
								{$_('settings.git.repositories.none_public')}
							</span>
						{/if}
					</Select.Trigger>
					<Select.Content>
						<Select.Item value="none">
							<span class="flex items-center gap-2">
								<Globe class="w-4 h-4 text-muted-foreground" />
								{$_('settings.git.repositories.none_public')}
							</span>
						</Select.Item>
						{#each credentials as cred}
							<Select.Item value={cred.id.toString()}>
								<span class="flex items-center gap-2">
									{#if cred.authType === 'ssh'}
										<KeyRound class="w-4 h-4 text-muted-foreground" />
									{:else if cred.authType === 'password'}
										<Lock class="w-4 h-4 text-muted-foreground" />
									{:else}
										<Key class="w-4 h-4 text-muted-foreground" />
									{/if}
									{cred.name} ({getAuthLabel(cred.authType)})
								</span>
							</Select.Item>
						{/each}
					</Select.Content>
				</Select.Root>
				{#if credentials.length === 0 && !isEditing}
					<p class="text-xs text-muted-foreground">
						<a href="/settings?tab=git&subtab=credentials" class="text-primary hover:underline">{$_('settings.git.repositories.add_credentials')}</a> {$_('settings.git.repositories.add_credentials_suffix')}
					</p>
				{/if}
			</div>

			{#if formError}
				<p class="text-sm text-destructive">{formError}</p>
			{/if}

			<Dialog.Footer>
				<Button variant="outline" type="button" onclick={onClose}>{$_('common.cancel')}</Button>
				<Button
					type="button"
					variant="outline"
					onclick={testRepository}
					disabled={testing || !formUrl.trim()}
					class={testResult?.success ? 'border-green-500 text-green-600 dark:border-green-500 dark:text-green-400' : ''}
				>
					{#if testing}
						<Loader2 class="w-4 h-4 mr-1.5 animate-spin" />
					{:else if testResult?.success}
						<CheckCircle2 class="w-4 h-4 mr-1.5 text-green-500" />
					{:else}
						<Play class="w-4 h-4 mr-1.5" />
					{/if}
					{$_('settings.git.repositories.test')}
				</Button>
				<Button type="submit" disabled={formSaving}>
					{#if formSaving}
						<Loader2 class="w-4 h-4 mr-1 animate-spin" />
						{$_('settings.git.repositories.saving')}
					{:else}
						{isEditing ? $_('settings.git.repositories.save_changes') : $_('settings.git.repositories.add')}
					{/if}
				</Button>
			</Dialog.Footer>
		</form>
	</Dialog.Content>
</Dialog.Root>
