<script lang="ts">
	import { Button } from '$lib/components/ui/button';
	import * as Dialog from '$lib/components/ui/dialog';
	import { Label } from '$lib/components/ui/label';
	import { Input } from '$lib/components/ui/input';
	import { Plus, Check, RefreshCw } from 'lucide-svelte';
	import { focusFirstInput } from '$lib/utils';
	import { _ } from '$lib/i18n';

	export interface Registry {
		id: number;
		name: string;
		url: string;
		username?: string;
		created_at: string;
	}

	interface Props {
		open: boolean;
		registry?: Registry | null;
		onClose: () => void;
		onSaved: () => void;
	}

	let { open = $bindable(), registry = null, onClose, onSaved }: Props = $props();

	const isEditing = $derived(registry !== null);

	// Form state
	let formName = $state('');
	let formUrl = $state('');
	let formUsername = $state('');
	let formPassword = $state('');
	let formError = $state('');
	let formSaving = $state(false);

	function resetForm() {
		formName = '';
		formUrl = '';
		formUsername = '';
		formPassword = '';
		formError = '';
		formSaving = false;
	}

	// Initialize form when registry changes or modal opens
	$effect(() => {
		if (open) {
			if (registry) {
				formName = registry.name;
				formUrl = registry.url;
				formUsername = registry.username || '';
				formPassword = '';
				formError = '';
			} else {
				resetForm();
			}
		}
	});

	async function save() {
		if (!formName.trim() || !formUrl.trim()) {
			formError = $_('settings.registries.error_name_url_required');
			return;
		}

		formSaving = true;
		formError = '';

		try {
			const body: Record<string, string | undefined> = {
				name: formName.trim(),
				url: formUrl.trim(),
				username: formUsername.trim() || undefined
			};

			// Only include password if provided (for edit, empty means keep existing)
			if (formPassword || !isEditing) {
				body.password = formPassword || undefined;
			}

			const url = isEditing ? `/api/registries/${registry!.id}` : '/api/registries';
			const method = isEditing ? 'PUT' : 'POST';

			const response = await fetch(url, {
				method,
				headers: { 'Content-Type': 'application/json' },
				body: JSON.stringify(body)
			});

			if (response.ok) {
				open = false;
				onSaved();
			} else {
				const data = await response.json();
				formError = data.error || (isEditing
					? $_('settings.registries.error_update_failed')
					: $_('settings.registries.error_create_failed')
				);
			}
		} catch {
			formError = isEditing
				? $_('settings.registries.error_update_failed')
				: $_('settings.registries.error_create_failed');
		} finally {
			formSaving = false;
		}
	}

	function handleClose() {
		open = false;
		onClose();
	}
</script>

<Dialog.Root bind:open onOpenChange={(o) => { if (o) { formError = ''; focusFirstInput(); } }}>
	<Dialog.Content class="max-w-md">
		<Dialog.Header>
			<Dialog.Title>{isEditing ? $_('settings.registries.edit_title') : $_('settings.registries.add_title')}</Dialog.Title>
		</Dialog.Header>
		<div class="space-y-4">
			{#if formError}
				<div class="text-sm text-red-600 dark:text-red-400">{formError}</div>
			{/if}
			<div class="space-y-2">
				<Label for="reg-name">{$_('common.name')}</Label>
				<Input id="reg-name" bind:value={formName} placeholder={$_('settings.registries.placeholder_name')} />
			</div>
			<div class="space-y-2">
				<Label for="reg-url">{$_('common.url')}</Label>
				<Input id="reg-url" bind:value={formUrl} placeholder={$_('settings.registries.placeholder_url')} />
			</div>
			<div class="space-y-4 pt-2 border-t">
				<p class="text-xs text-muted-foreground">
					{$_('settings.registries.credentials')}
					{isEditing ? ` ${$_('settings.registries.credentials_keep_password')}` : ` ${$_('settings.registries.credentials_optional')}`}
				</p>
				<div class="space-y-2">
					<Label for="reg-username">{$_('common.username')}</Label>
					<Input id="reg-username" bind:value={formUsername} placeholder={$_('settings.registries.placeholder_username')} />
				</div>
				<div class="space-y-2">
					<Label for="reg-password">{$_('settings.registries.password_token')}</Label>
					<Input
						id="reg-password"
						type="password"
						bind:value={formPassword}
						placeholder={isEditing ? $_('settings.registries.placeholder_password_keep') : $_('settings.registries.placeholder_password')}
					/>
				</div>
			</div>
		</div>
		<Dialog.Footer>
			<Button variant="outline" onclick={handleClose}>{$_('common.cancel')}</Button>
			<Button onclick={save} disabled={formSaving}>
				{#if formSaving}
					<RefreshCw class="w-4 h-4 mr-1 animate-spin" />
				{:else if isEditing}
					<Check class="w-4 h-4 mr-1" />
				{:else}
					<Plus class="w-4 h-4 mr-1" />
				{/if}
				{isEditing ? $_('common.save') : $_('common.add')}
			</Button>
		</Dialog.Footer>
	</Dialog.Content>
</Dialog.Root>
