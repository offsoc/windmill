<script lang="ts">
	import { Alert, Button } from '$lib/components/common'
	import Drawer from '$lib/components/common/drawer/Drawer.svelte'
	import DrawerContent from '$lib/components/common/drawer/DrawerContent.svelte'
	import Path from '$lib/components/Path.svelte'
	import Required from '$lib/components/Required.svelte'
	import ScriptPicker from '$lib/components/ScriptPicker.svelte'
	import { NatsTriggerService } from '$lib/gen'
	import { usedTriggerKinds, userStore, workspaceStore } from '$lib/stores'
	import { canWrite, emptyString, sendUserToast } from '$lib/utils'
	import { createEventDispatcher } from 'svelte'
	import Section from '$lib/components/Section.svelte'
	import { Loader2, Save } from 'lucide-svelte'
	import Label from '$lib/components/Label.svelte'
	import NatsTriggersConfigSection from './NatsTriggersConfigSection.svelte'
	import Toggle from '$lib/components/Toggle.svelte'

	let drawer: Drawer
	let is_flow: boolean = false
	let initialPath = ''
	let edit = true
	let itemKind: 'flow' | 'script' = 'script'
	let script_path = ''
	let initialScriptPath = ''
	let fixedScriptPath = ''
	let path: string = ''
	let pathError = ''
	let enabled = false
	let dirtyPath = false
	let can_write = true
	let drawerLoading = true
	let defaultValues: Record<string, any> | undefined = undefined
	let args: Record<string, any> = {}

	const dispatch = createEventDispatcher()

	$: is_flow = itemKind === 'flow'

	export async function openEdit(ePath: string, isFlow: boolean) {
		drawerLoading = true
		try {
			drawer?.openDrawer()
			initialPath = ePath
			itemKind = isFlow ? 'flow' : 'script'
			edit = true
			dirtyPath = false
			await loadTrigger()
		} catch (err) {
			sendUserToast(`Could not load nats trigger: ${err}`, true)
		} finally {
			drawerLoading = false
		}
	}

	export async function openNew(
		nis_flow: boolean,
		fixedScriptPath_?: string,
		nDefaultValues?: Record<string, any>
	) {
		drawerLoading = true
		try {
			drawer?.openDrawer()
			is_flow = nis_flow
			edit = false
			itemKind = nis_flow ? 'flow' : 'script'
			args.nats_resource_path = nDefaultValues?.nats_resource_path ?? ''
			args.subjects = nDefaultValues?.subjects ?? ['']
			args.use_jetstream = nDefaultValues?.use_jetstream ?? false
			args.stream_name = args.use_jetstream ? (nDefaultValues?.stream_name ?? '') : undefined
			args.consumer_name = args.use_jetstream ? (nDefaultValues?.consumer_name ?? '') : undefined
			initialScriptPath = ''
			fixedScriptPath = fixedScriptPath_ ?? ''
			script_path = fixedScriptPath
			path = ''
			initialPath = ''
			dirtyPath = false
			defaultValues = nDefaultValues
		} finally {
			drawerLoading = false
		}
	}

	async function loadTrigger(): Promise<void> {
		const s = await NatsTriggerService.getNatsTrigger({
			workspace: $workspaceStore!,
			path: initialPath
		})
		script_path = s.script_path
		initialScriptPath = s.script_path

		is_flow = s.is_flow
		path = s.path
		args.nats_resource_path = s.nats_resource_path
		args.stream_name = s.stream_name
		args.consumer_name = s.consumer_name
		args.subjects = s.subjects
		args.use_jetstream = s.use_jetstream
		enabled = s.enabled

		can_write = canWrite(s.path, s.extra_perms, $userStore)
	}

	async function updateTrigger(): Promise<void> {
		if (edit) {
			await NatsTriggerService.updateNatsTrigger({
				workspace: $workspaceStore!,
				path: initialPath,
				requestBody: {
					path,
					script_path,
					is_flow,
					nats_resource_path: args.nats_resource_path,
					stream_name: args.stream_name,
					consumer_name: args.consumer_name,
					subjects: args.subjects,
					use_jetstream: args.use_jetstream
				}
			})
			sendUserToast(`Nats trigger ${path} updated`)
		} else {
			await NatsTriggerService.createNatsTrigger({
				workspace: $workspaceStore!,
				requestBody: {
					path,
					script_path,
					is_flow,
					enabled: true,
					nats_resource_path: args.nats_resource_path,
					stream_name: args.stream_name,
					consumer_name: args.consumer_name,
					subjects: args.subjects,
					use_jetstream: args.use_jetstream
				}
			})
			sendUserToast(`Nats trigger ${path} created`)
		}
		if (!$usedTriggerKinds.includes('nats')) {
			$usedTriggerKinds = [...$usedTriggerKinds, 'nats']
		}
		dispatch('update')
		drawer.closeDrawer()
	}

	function useDefaultValues() {
		if (args.nats_resource_path && args.nats_resource_path != '') {
			return false
		}
		if (!defaultValues) {
			return false
		}
		return (
			defaultValues.servers &&
			defaultValues.servers.length > 0 &&
			defaultValues.servers.some((broker: string) => broker.trim() !== '')
		)
	}

	let isValid = false
</script>

<Drawer size="800px" bind:this={drawer}>
	<DrawerContent
		title={edit
			? can_write
				? `Edit NATS trigger ${initialPath}`
				: `NATS trigger ${initialPath}`
			: 'New NATS trigger'}
		on:close={drawer.closeDrawer}
	>
		<svelte:fragment slot="actions">
			{#if !drawerLoading}
				{#if edit}
					<div class="mr-8 center-center -mt-1">
						<Toggle
							disabled={!can_write}
							checked={enabled}
							options={{ right: 'enable', left: 'disable' }}
							on:change={async (e) => {
								await NatsTriggerService.setNatsTriggerEnabled({
									path: initialPath,
									workspace: $workspaceStore ?? '',
									requestBody: { enabled: e.detail }
								})
								sendUserToast(`${e.detail ? 'enabled' : 'disabled'} NATS trigger ${initialPath}`)
							}}
						/>
					</div>
				{/if}
				{#if can_write}
					<Button
						startIcon={{ icon: Save }}
						disabled={pathError != '' || emptyString(script_path) || !can_write || !isValid}
						on:click={updateTrigger}
					>
						Save
					</Button>
				{/if}
			{/if}
		</svelte:fragment>
		{#if drawerLoading}
			<Loader2 class="animate-spin" />
		{:else}
			<Alert title="Info" type="info">
				{#if edit}
					Changes can take up to 30 seconds to take effect.
				{:else}
					NATS consumers can take up to 30 seconds to start.
				{/if}
			</Alert>
			<div class="flex flex-col gap-12 mt-6">
				<div class="flex flex-col gap-4">
					<Label label="Path">
						<Path
							bind:dirty={dirtyPath}
							bind:error={pathError}
							bind:path
							{initialPath}
							checkInitialPathExistence={!edit}
							namePlaceholder="nats_trigger"
							kind="nats_trigger"
							disabled={!can_write}
						/>
					</Label>
				</div>
				<Section label="Runnable">
					<p class="text-xs mb-1 text-tertiary">
						Pick a script or flow to be triggered<Required required={true} />
					</p>
					<div class="flex flex-row mb-2">
						<ScriptPicker
							disabled={fixedScriptPath != '' || !can_write}
							initialPath={fixedScriptPath || initialScriptPath}
							kinds={['script']}
							allowFlow={true}
							bind:itemKind
							bind:scriptPath={script_path}
							allowRefresh={can_write}
							allowEdit={!$userStore?.operator}
						/>
						{#if emptyString(script_path)}
							<Button
								btnClasses="ml-4 mt-2"
								color="dark"
								size="xs"
								href={itemKind === 'flow' ? '/flows/add?hub=66' : '/scripts/add?hub=hub%2F19663'}
								target="_blank">Create from template</Button
							>
						{/if}
					</div>
				</Section>

				<NatsTriggersConfigSection
					{path}
					bind:args
					bind:isValid
					defaultValues={useDefaultValues() ? defaultValues : undefined}
				/>
			</div>
		{/if}
	</DrawerContent>
</Drawer>
