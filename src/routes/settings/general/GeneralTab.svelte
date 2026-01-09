<script lang="ts">
	import * as Card from '$lib/components/ui/card';
	import * as Select from '$lib/components/ui/select';
	import { Label } from '$lib/components/ui/label';
	import { Input } from '$lib/components/ui/input';
	import { Button } from '$lib/components/ui/button';
	import { Badge } from '$lib/components/ui/badge';
	import { TogglePill, ToggleSwitch } from '$lib/components/ui/toggle-pill';
	import CronEditor from '$lib/components/cron-editor.svelte';
	import TimezoneSelector from '$lib/components/TimezoneSelector.svelte';
	import { Eye, Bell, Database, Calendar, ShieldCheck, FileText, AlertTriangle, HelpCircle, Globe } from 'lucide-svelte';
	import { appSettings, type DateFormat, type DownloadFormat } from '$lib/stores/settings';
	import { canAccess, authStore } from '$lib/stores/auth';
	import { toast } from 'svelte-sonner';
	import ThemeSelector from '$lib/components/ThemeSelector.svelte';
	import * as Tooltip from '$lib/components/ui/tooltip';
	import { _ } from '$lib/i18n';

	// General settings state - these derive from the store
	let confirmDestructive = $derived($appSettings.confirmDestructive);
	let showStoppedContainers = $derived($appSettings.showStoppedContainers);
	let highlightUpdates = $derived($appSettings.highlightUpdates);
	let timeFormat = $derived($appSettings.timeFormat);
	let dateFormat = $derived($appSettings.dateFormat);
	let downloadFormat = $derived($appSettings.downloadFormat);
	let defaultGrypeArgs = $derived($appSettings.defaultGrypeArgs);
	let defaultTrivyArgs = $derived($appSettings.defaultTrivyArgs);
	let scheduleRetentionDays = $derived($appSettings.scheduleRetentionDays);
	let eventRetentionDays = $derived($appSettings.eventRetentionDays);
	let scheduleCleanupCron = $derived($appSettings.scheduleCleanupCron);
	let eventCleanupCron = $derived($appSettings.eventCleanupCron);
	let scheduleCleanupEnabled = $derived($appSettings.scheduleCleanupEnabled);
	let eventCleanupEnabled = $derived($appSettings.eventCleanupEnabled);
	let logBufferSizeKb = $derived($appSettings.logBufferSizeKb);
	let defaultTimezone = $derived($appSettings.defaultTimezone);

	const dateFormatOptions: { value: DateFormat; label: string; example: string }[] = [
		{ value: 'DD.MM.YYYY', label: 'DD.MM.YYYY', example: '31.12.2024' },
		{ value: 'DD/MM/YYYY', label: 'DD/MM/YYYY', example: '31/12/2024' },
		{ value: 'MM/DD/YYYY', label: 'MM/DD/YYYY', example: '12/31/2024' },
		{ value: 'YYYY-MM-DD', label: 'YYYY-MM-DD', example: '2024-12-31' }
	];

	function handleScheduleRetentionChange(e: Event) {
		const value = Math.max(1, Math.min(365, parseInt((e.target as HTMLInputElement).value) || 30));
		appSettings.setScheduleRetentionDays(value);
		toast.success($_('settings.schedule_retention_updated'));
	}

	function handleEventRetentionChange(e: Event) {
		const value = Math.max(1, Math.min(365, parseInt((e.target as HTMLInputElement).value) || 30));
		appSettings.setEventRetentionDays(value);
		toast.success($_('settings.event_retention_updated'));
	}

	function handleScheduleCleanupCronChange(cron: string) {
		appSettings.setScheduleCleanupCron(cron);
		toast.success($_('settings.schedule_cleanup_cron_updated'));
	}

	function handleEventCleanupCronChange(cron: string) {
		appSettings.setEventCleanupCron(cron);
		toast.success($_('settings.event_cleanup_cron_updated'));
	}

	function handleScheduleCleanupEnabledChange() {
		appSettings.setScheduleCleanupEnabled(!scheduleCleanupEnabled);
		toast.success(scheduleCleanupEnabled ? $_('settings.schedule_cleanup_disabled') : $_('settings.schedule_cleanup_enabled'));
	}

	function handleEventCleanupEnabledChange() {
		appSettings.setEventCleanupEnabled(!eventCleanupEnabled);
		toast.success(eventCleanupEnabled ? $_('settings.event_cleanup_disabled') : $_('settings.event_cleanup_enabled'));
	}

	function handleGrypeArgsBlur(e: Event) {
		const value = (e.target as HTMLInputElement).value.trim();
		if (value !== defaultGrypeArgs) {
			appSettings.setDefaultGrypeArgs(value);
			toast.success($_('settings.grype_args_updated'));
		}
	}

	function handleTrivyArgsBlur(e: Event) {
		const value = (e.target as HTMLInputElement).value.trim();
		if (value !== defaultTrivyArgs) {
			appSettings.setDefaultTrivyArgs(value);
			toast.success($_('settings.trivy_args_updated'));
		}
	}

	function handleLogBufferSizeChange(e: Event) {
		const value = Math.max(100, Math.min(5000, parseInt((e.target as HTMLInputElement).value) || 500));
		appSettings.setLogBufferSizeKb(value);
		toast.success($_('settings.log_buffer_size_updated'));
	}
</script>

<div class="flex-1 min-h-0 overflow-y-auto">
	<div class="grid grid-cols-1 xl:grid-cols-2 gap-4">
		<!-- Left column -->
		<div class="space-y-4">
			<Card.Root>
				<Card.Header>
					<Card.Title class="text-sm font-medium flex items-center gap-2">
						<Eye class="w-4 h-4" />
						{$_('settings.appearance')}
						{#if !$authStore.authEnabled}
							<Tooltip.Provider delayDuration={100}>
								<Tooltip.Root>
									<Tooltip.Trigger>
										<HelpCircle class="w-4 h-4 text-muted-foreground cursor-help" />
									</Tooltip.Trigger>
									<Tooltip.Portal>
										<Tooltip.Content side="right" sideOffset={8} class="!w-80">
											{$_('settings.theme_auth_tooltip')}
										</Tooltip.Content>
									</Tooltip.Portal>
								</Tooltip.Root>
							</Tooltip.Provider>
						{/if}
					</Card.Title>
				</Card.Header>
				<Card.Content>
					<div class="grid grid-cols-1 md:grid-cols-2 gap-x-6 gap-y-4">
						<!-- Left column -->
						<div class="space-y-4">
							<div class="space-y-1">
								<div class="flex items-center gap-3">
									<Label>{$_('settings.show_stopped_containers')}</Label>
									<TogglePill
										checked={showStoppedContainers}
										onchange={() => {
											appSettings.setShowStoppedContainers(!showStoppedContainers);
											toast.success(showStoppedContainers ? $_('settings.stopped_containers_hidden') : $_('settings.stopped_containers_shown'));
										}}
										disabled={!$canAccess('settings', 'edit')}
									/>
								</div>
								<p class="text-xs text-muted-foreground">{$_('settings.show_stopped_containers_desc')}</p>
							</div>
							<div class="space-y-1">
								<div class="flex items-center gap-3">
									<Label>{$_('settings.highlight_updates')}</Label>
									<TogglePill
										checked={highlightUpdates}
										onchange={() => {
											appSettings.setHighlightUpdates(!highlightUpdates);
											toast.success(highlightUpdates ? $_('settings.update_highlighting_disabled') : $_('settings.update_highlighting_enabled'));
										}}
										disabled={!$canAccess('settings', 'edit')}
									/>
								</div>
								<p class="text-xs text-muted-foreground">{$_('settings.highlight_updates_desc')}</p>
							</div>
							<div class="space-y-1">
								<div class="flex items-center gap-3">
									<Label>{$_('settings.time_format')}</Label>
									<ToggleSwitch
										value={timeFormat}
										leftValue="24h"
										rightValue="12h"
										onchange={(newFormat) => {
											appSettings.setTimeFormat(newFormat as '12h' | '24h');
											toast.success(newFormat === '12h' ? $_('settings.time_format_set') : $_('settings.time_format_set_24h'));
										}}
										disabled={!$canAccess('settings', 'edit')}
									/>
								</div>
								<p class="text-xs text-muted-foreground">{$_('settings.time_format_desc')}</p>
							</div>
							<div class="space-y-1">
								<div class="flex items-center gap-3">
									<Label>{$_('settings.date_format')}</Label>
									<Select.Root
										type="single"
										value={dateFormat}
										onValueChange={(value) => {
											if (value) {
												appSettings.setDateFormat(value as DateFormat);
												toast.success($_('settings.date_format_set', { value: value }));
											}
										}}
										disabled={!$canAccess('settings', 'edit')}
									>
										<Select.Trigger class="w-[180px]">
											<Calendar class="w-4 h-4 mr-2" />
											<span>{dateFormat}</span>
										</Select.Trigger>
										<Select.Content>
											{#each dateFormatOptions as option}
												<Select.Item value={option.value}>
													<div class="flex items-center justify-between w-full gap-4">
														<span>{option.label}</span>
														<span class="text-xs text-muted-foreground">{option.example}</span>
													</div>
												</Select.Item>
											{/each}
										</Select.Content>
									</Select.Root>
								</div>
								<p class="text-xs text-muted-foreground">{$_('settings.date_format_desc')}</p>
							</div>
						</div>
						<!-- Right column: Theme settings (only when auth disabled) -->
						{#if !$authStore.authEnabled}
							<div class="space-y-4">
								<ThemeSelector />
							</div>
						{:else}
							<div class="text-xs text-muted-foreground flex items-start gap-1.5">
								<HelpCircle class="w-3.5 h-3.5 shrink-0 mt-0.5" />
								<div>
									<p>{$_('settings.appearance_personal_note')}</p>
									<a href="/profile" class="text-primary hover:underline">{$_('settings.configure_in_profile')}</a>
								</div>
							</div>
						{/if}
					</div>
				</Card.Content>
			</Card.Root>

			<Card.Root>
				<Card.Header>
					<Card.Title class="text-sm font-medium flex items-center gap-2">
						<Globe class="w-4 h-4" />
						{$_('settings.scheduling')}
					</Card.Title>
				</Card.Header>
				<Card.Content class="space-y-4">
					<div class="space-y-2">
						<Label>{$_('settings.default_timezone')}</Label>
						<TimezoneSelector
							value={defaultTimezone}
							onchange={(value) => {
								appSettings.setDefaultTimezone(value);
								toast.success($_('settings.default_timezone_set', { value: value }));
							}}
							class="w-[320px]"
						/>
						<p class="text-xs text-muted-foreground">{$_('settings.default_timezone_desc')}</p>
					</div>
				</Card.Content>
			</Card.Root>

			<Card.Root>
				<Card.Header>
					<Card.Title class="text-sm font-medium flex items-center gap-2">
						<Bell class="w-4 h-4" />
						{$_('settings.confirmations')}
					</Card.Title>
				</Card.Header>
				<Card.Content class="space-y-4">
					<div class="space-y-1">
						<div class="flex items-center gap-3">
							<Label>{$_('settings.confirm_destructive')}</Label>
							<TogglePill
								checked={confirmDestructive}
								onchange={() => {
									appSettings.setConfirmDestructive(!confirmDestructive);
									toast.success(confirmDestructive ? $_('settings.confirmations_disabled') : $_('settings.confirmations_enabled'));
								}}
								disabled={!$canAccess('settings', 'edit')}
							/>
						</div>
						<p class="text-xs text-muted-foreground">{$_('settings.confirm_destructive_desc')}</p>
					</div>
				</Card.Content>
			</Card.Root>

			<Card.Root>
				<Card.Header>
					<Card.Title class="text-sm font-medium flex items-center gap-2">
						<FileText class="w-4 h-4" />
						{$_('settings.logs_files')}
					</Card.Title>
				</Card.Header>
				<Card.Content class="space-y-4">
					<div class="space-y-2">
						<Label for="log-buffer-size">{$_('settings.log_buffer_size')}</Label>
						<div class="flex items-center gap-2">
							<Input
								id="log-buffer-size"
								type="number"
								min="100"
								max="5000"
								value={logBufferSizeKb}
								onchange={handleLogBufferSizeChange}
								disabled={!$canAccess('settings', 'edit')}
								class="w-24"
							/>
							<span class="text-sm text-muted-foreground">{$_('settings.kb')}</span>
						</div>
						<p class="text-xs text-muted-foreground">{$_('settings.log_buffer_size_desc')}</p>
						{#if logBufferSizeKb > 1000}
							<div class="flex items-start gap-2 p-2 rounded-md bg-amber-500/10 border border-amber-500/20">
								<AlertTriangle class="w-4 h-4 text-amber-500 shrink-0 mt-0.5" />
								<p class="text-xs text-amber-600 dark:text-amber-400">{$_('settings.log_buffer_size_warning')}</p>
							</div>
						{/if}
					</div>
					<div class="space-y-1">
						<div class="flex items-center gap-3">
							<Label>{$_('settings.download_format')}</Label>
							<ToggleSwitch
								value={downloadFormat}
								leftValue="tar"
								rightValue="tar.gz"
								onchange={(newFormat) => {
									appSettings.setDownloadFormat(newFormat as DownloadFormat);
									toast.success($_('settings.download_format_set', { value: newFormat }));
								}}
								disabled={!$canAccess('settings', 'edit')}
							/>
						</div>
						<p class="text-xs text-muted-foreground">{$_('settings.download_format_desc')}</p>
					</div>
				</Card.Content>
			</Card.Root>

		</div>

		<!-- Right column -->
		<div class="space-y-4">
			<Card.Root>
				<Card.Header>
					<Card.Title class="text-sm font-medium flex items-center gap-2">
						<ShieldCheck class="w-4 h-4" />
						{$_('settings.vulnerability_scanners')}
					</Card.Title>
				</Card.Header>
				<Card.Content class="space-y-4">
					<div class="space-y-2">
						<Label for="grype-args">{$_('settings.default_grype_args')}</Label>
						<Input
							id="grype-args"
							value={defaultGrypeArgs}
							onblur={handleGrypeArgsBlur}
							disabled={!$canAccess('settings', 'edit')}
							placeholder={"-o json -v {image}"}
						/>
						<p class="text-xs text-muted-foreground">{$_('settings.image_placeholder_desc')}</p>
					</div>
					<div class="space-y-2">
						<Label for="trivy-args">{$_('settings.default_trivy_args')}</Label>
						<Input
							id="trivy-args"
							value={defaultTrivyArgs}
							onblur={handleTrivyArgsBlur}
							disabled={!$canAccess('settings', 'edit')}
							placeholder={"image --format json {image}"}
						/>
						<p class="text-xs text-muted-foreground">{$_('settings.image_placeholder_desc')}</p>
					</div>
				</Card.Content>
			</Card.Root>

			<Card.Root>
				<Card.Header>
					<Card.Title class="text-sm font-medium flex items-center gap-2">
						<Database class="w-4 h-4" />
						{$_('settings.system_jobs')}
					</Card.Title>
				</Card.Header>
				<Card.Content class="space-y-4">
					<div class="space-y-1">
						<div class="flex items-center gap-3">
							<Label for="schedule-retention">{$_('settings.schedule_cleanup')}</Label>
							<TogglePill
								checked={scheduleCleanupEnabled}
								onchange={handleScheduleCleanupEnabledChange}
								disabled={!$canAccess('settings', 'edit')}
							/>
						</div>
						<p class="text-xs text-muted-foreground">{$_('settings.schedule_cleanup_desc')}</p>
						<div class="flex items-center gap-2 mt-2">
							<Input
								id="schedule-retention"
								type="number"
								min="1"
								max="365"
								value={scheduleRetentionDays}
								onchange={handleScheduleRetentionChange}
								disabled={!$canAccess('settings', 'edit') || !scheduleCleanupEnabled}
								class="w-20"
							/>
							<span class="text-sm text-muted-foreground">{$_('settings.days')}</span>
							<div class="ml-auto">
								<CronEditor
									value={scheduleCleanupCron}
									onchange={handleScheduleCleanupCronChange}
									disabled={!$canAccess('settings', 'edit') || !scheduleCleanupEnabled}
								/>
							</div>
						</div>
					</div>
					<div class="space-y-1">
						<div class="flex items-center gap-3">
							<Label for="event-retention">{$_('settings.event_cleanup')}</Label>
							<TogglePill
								checked={eventCleanupEnabled}
								onchange={handleEventCleanupEnabledChange}
								disabled={!$canAccess('settings', 'edit')}
							/>
						</div>
						<p class="text-xs text-muted-foreground">{$_('settings.event_cleanup_desc')}</p>
						<div class="flex items-center gap-2 mt-2">
							<Input
								id="event-retention"
								type="number"
								min="1"
								max="365"
								value={eventRetentionDays}
								onchange={handleEventRetentionChange}
								disabled={!$canAccess('settings', 'edit') || !eventCleanupEnabled}
								class="w-20"
							/>
							<span class="text-sm text-muted-foreground">{$_('settings.days')}</span>
							<div class="ml-auto">
								<CronEditor
									value={eventCleanupCron}
									onchange={handleEventCleanupCronChange}
									disabled={!$canAccess('settings', 'edit') || !eventCleanupEnabled}
								/>
							</div>
						</div>
					</div>
					<div class="space-y-1 pt-2 border-t">
						<div class="flex items-center gap-3">
							<Label>{$_('settings.volume_helper_cleanup')}</Label>
							<Badge variant="secondary" class="text-xs">{$_('settings.always_enabled')}</Badge>
						</div>
						<p class="text-xs text-muted-foreground">
							{$_('settings.volume_helper_cleanup_desc')}
						</p>
					</div>
				</Card.Content>
			</Card.Root>
		</div>
	</div>
</div>
