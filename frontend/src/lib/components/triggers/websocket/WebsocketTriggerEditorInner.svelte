<script lang="ts">
	import { Alert, Button } from '$lib/components/common'
	import Drawer from '$lib/components/common/drawer/Drawer.svelte'
	import DrawerContent from '$lib/components/common/drawer/DrawerContent.svelte'
	import Path from '$lib/components/Path.svelte'
	import Required from '$lib/components/Required.svelte'
	import ScriptPicker from '$lib/components/ScriptPicker.svelte'
	import {
		FlowService,
		ScriptService,
		WebsocketTriggerService,
		type Flow,
		type Script,
		type ScriptArgs,
		type WebsocketTriggerInitialMessage
	} from '$lib/gen'
	import { usedTriggerKinds, userStore, workspaceStore } from '$lib/stores'
	import { canWrite, emptySchema, emptyString, sendUserToast } from '$lib/utils'
	import { createEventDispatcher } from 'svelte'
	import Section from '$lib/components/Section.svelte'
	import { Loader2, Save, X, Plus } from 'lucide-svelte'
	import Label from '$lib/components/Label.svelte'
	import { fade } from 'svelte/transition'
	import type { Schema } from '$lib/common'
	import JsonEditor from '$lib/components/JsonEditor.svelte'
	import Toggle from '$lib/components/Toggle.svelte'
	import WebsocketEditorConfigSection from './WebsocketEditorConfigSection.svelte'

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
	let url = ''
	let dirtyUrl = false
	let enabled = false
	let filters: {
		key: string
		value: any
	}[] = []
	let initial_messages: WebsocketTriggerInitialMessage[] = []
	let url_runnable_args: Record<string, any> | undefined = {}
	let can_return_message = false
	let dirtyPath = false
	let can_write = true
	let drawerLoading = true

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
			dirtyUrl = false
			await loadTrigger()
		} catch (err) {
			sendUserToast(`Could not load websocket trigger: ${err}`, true)
		} finally {
			drawerLoading = false
		}
	}

	export async function openNew(
		nis_flow: boolean,
		fixedScriptPath_?: string,
		defaultValues?: Record<string, any>
	) {
		drawerLoading = true
		try {
			drawer?.openDrawer()
			is_flow = nis_flow
			edit = false
			itemKind = nis_flow ? 'flow' : 'script'
			url = defaultValues?.url ?? ''
			dirtyUrl = false
			initialScriptPath = ''
			fixedScriptPath = fixedScriptPath_ ?? ''
			script_path = fixedScriptPath
			path = ''
			initialPath = ''
			filters = []
			initial_messages = []
			url_runnable_args = defaultValues?.url_runnable_args ?? {}
			dirtyPath = false
			can_return_message = false
		} finally {
			drawerLoading = false
		}
	}

	async function loadTrigger(): Promise<void> {
		const s = await WebsocketTriggerService.getWebsocketTrigger({
			workspace: $workspaceStore!,
			path: initialPath
		})
		script_path = s.script_path
		initialScriptPath = s.script_path

		is_flow = s.is_flow
		path = s.path
		url = s.url
		enabled = s.enabled
		filters = s.filters
		initial_messages = s.initial_messages ?? []
		url_runnable_args = s.url_runnable_args
		can_return_message = s.can_return_message

		can_write = canWrite(s.path, s.extra_perms, $userStore)
	}

	let initialMessageRunnableSchemas: Record<string, Schema> = {}
	async function loadInitialMessageRunnableSchemas(
		initialMessageRunnables: {
			path: string
			is_flow: boolean
		}[]
	) {
		for (const { path, is_flow } of initialMessageRunnables) {
			if (!path) {
				continue
			}
			try {
				let schema: Schema | undefined = emptySchema()
				let scriptOrFlow: Script | Flow = is_flow
					? await FlowService.getFlowByPath({ workspace: $workspaceStore!, path })
					: await ScriptService.getScriptByPath({ workspace: $workspaceStore!, path })
				schema = scriptOrFlow.schema as Schema
				if (schema && schema.properties) {
					initialMessageRunnableSchemas[(is_flow ? 'flow/' : '') + path] = schema
				}
			} catch (err) {
				sendUserToast(
					`Could not query runnable schema for ${is_flow ? 'flow' : 'script'} ${path}: ${err}`,
					true
				)
			}
		}
	}
	$: initialMessageRunnables = initial_messages
		.map((v) => ('runnable_result' in v ? v.runnable_result : undefined))
		.filter((v): v is { path: string; is_flow: boolean; args: ScriptArgs } => !!v)
	$: loadInitialMessageRunnableSchemas(initialMessageRunnables)

	$: invalidInitialMessages = initial_messages.some((v) => {
		if ('runnable_result' in v) {
			return !v.runnable_result.path
		}
		return false
	})

	async function updateTrigger(): Promise<void> {
		if (edit) {
			await WebsocketTriggerService.updateWebsocketTrigger({
				workspace: $workspaceStore!,
				path: initialPath,
				requestBody: {
					path,
					script_path,
					is_flow,
					url,
					filters,
					initial_messages,
					url_runnable_args,
					can_return_message
				}
			})
			sendUserToast(`Websocket trigger ${path} updated`)
		} else {
			await WebsocketTriggerService.createWebsocketTrigger({
				workspace: $workspaceStore!,
				requestBody: {
					path,
					script_path,
					is_flow,
					url,
					enabled: true,
					filters,
					initial_messages,
					url_runnable_args,
					can_return_message
				}
			})
			sendUserToast(`Websocket trigger ${path} created`)
		}
		if (!$usedTriggerKinds.includes('ws')) {
			$usedTriggerKinds = [...$usedTriggerKinds, 'ws']
		}
		dispatch('update')
		drawer.closeDrawer()
	}

	let isValid = false
</script>

<Drawer size="800px" bind:this={drawer}>
	<DrawerContent
		title={edit
			? can_write
				? `Edit WebSocket trigger ${initialPath}`
				: `WebSocket trigger ${initialPath}`
			: 'New WebSocket trigger'}
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
								await WebsocketTriggerService.setWebsocketTriggerEnabled({
									path: initialPath,
									workspace: $workspaceStore ?? '',
									requestBody: { enabled: e.detail }
								})
								sendUserToast(
									`${e.detail ? 'enabled' : 'disabled'} websocket trigger ${initialPath}`
								)
							}}
						/>
					</div>
				{/if}
				{#if can_write}
					<Button
						startIcon={{ icon: Save }}
						disabled={pathError != '' ||
							!isValid ||
							invalidInitialMessages ||
							emptyString(script_path) ||
							!can_write}
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
					New WebSocket triggers can take up to 30 seconds to start listening.
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
							namePlaceholder="ws_trigger"
							kind="websocket_trigger"
							disabled={!can_write}
						/>
					</Label>
				</div>

				<Section label="Runnable" class="flex flex-col gap-4">
					<div>
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
									href={itemKind === 'flow' ? '/flows/add?hub=64' : '/scripts/add?hub=hub%2F19660'}
									target="_blank">Create from template</Button
								>
							{/if}
						</div>
					</div>

					<Toggle
						checked={can_return_message}
						on:change={() => {
							can_return_message = !can_return_message
						}}
						options={{
							right: 'Send runnable result',
							rightTooltip:
								'Whether the runnable result should be sent as a message to the websocket server when not null.'
						}}
					/>
				</Section>

				<WebsocketEditorConfigSection
					bind:url
					bind:url_runnable_args
					{dirtyUrl}
					{can_write}
					bind:isValid
				/>

				<Section label="Initial messages">
					<p class="text-xs mb-1 text-tertiary">
						Initial messages are sent at the beginning of the connection. They are sent in order.<br
						/>
						Raw messages and runnable results are supported.
					</p>
					<div class="flex flex-col gap-4 mt-1">
						{#each initial_messages as v, i}
							<div class="flex w-full gap-2 items-center">
								<div class="w-full flex flex-col gap-2 border p-2 rounded-md">
									<div class="flex flex-row gap-2 w-full">
										<label class="flex flex-col w-full">
											<div class="text-secondary text-sm mb-2">Type</div>
											<select
												class="w-20"
												on:change={(e) => {
													if (e.target?.['value'] === 'raw_message') {
														initial_messages[i] = {
															raw_message: '""'
														}
													} else {
														initial_messages[i] = {
															runnable_result: {
																path: '',
																args: {},
																is_flow: false
															}
														}
													}
												}}
												value={'runnable_result' in v ? 'runnable_result' : 'raw_message'}
											>
												<option value="raw_message">Raw message</option>
												<option value="runnable_result">Runnable result</option>
											</select>
										</label>
									</div>
									{#if 'raw_message' in v}
										<div class="flex flex-col w-full">
											<div class="text-secondary text-sm mb-2">
												Raw JSON message (if a string, wrapping quotes will be discarded)
											</div>
											<JsonEditor
												on:change={(ev) => {
													const { code } = ev.detail
													v = {
														raw_message: code
													}
												}}
												code={v.raw_message}
											/>
										</div>
									{:else if 'runnable_result' in v}
										<div class="flex flex-col w-full">
											<div class="text-secondary text-sm">Runnable</div>
											<ScriptPicker
												allowFlow={true}
												itemKind={v.runnable_result?.is_flow ? 'flow' : 'script'}
												initialPath={v.runnable_result?.path ?? ''}
												on:select={(ev) => {
													const { path, itemKind } = ev.detail
													v = {
														runnable_result: {
															path: path ?? '',
															args: {},
															is_flow: itemKind === 'flow'
														}
													}
												}}
											/>

											{#if v.runnable_result?.path}
												{@const schema =
													initialMessageRunnableSchemas[
														v.runnable_result.is_flow
															? 'flow/' + v.runnable_result.path
															: v.runnable_result.path
													]}
												{#if schema}
													<p class="font-semibold text-sm mt-4 mb-2">Arguments</p>
													{#await import('$lib/components/SchemaForm.svelte')}
														<Loader2 class="animate-spin mt-2" />
													{:then Module}
														<Module.default
															{schema}
															bind:args={v.runnable_result.args}
															shouldHideNoInputs
															class="text-xs"
														/>
													{/await}
													{#if schema && schema.properties && Object.keys(schema.properties).length === 0}
														<div class="text-xs texg-gray-700">This runnable takes no arguments</div
														>
													{/if}
												{:else}
													<Loader2 class="animate-spin mt-2" />
												{/if}
											{/if}
										</div>
									{:else}
										Unknown type
									{/if}
								</div>
								<button
									transition:fade|local={{ duration: 100 }}
									class="rounded-full p-1 bg-surface-secondary duration-200 hover:bg-surface-hover"
									aria-label="Clear"
									on:click={() => {
										initial_messages = initial_messages.filter((_, index) => index !== i)
									}}
								>
									<X size={14} />
								</button>
							</div>
						{/each}

						<div class="flex items-baseline">
							<Button
								variant="border"
								color="light"
								size="xs"
								btnClasses="mt-1"
								on:click={() => {
									if (initial_messages == undefined || !Array.isArray(initial_messages)) {
										initial_messages = []
									}
									initial_messages = initial_messages.concat({
										raw_message: '""'
									})
								}}
								startIcon={{ icon: Plus }}
							>
								Add item
							</Button>
						</div>
					</div>
				</Section>

				<Section label="Filters">
					<p class="text-xs mb-1 text-tertiary">
						Filters will limit the execution of the trigger to only messages that match all
						criteria.<br />
						The JSON filter checks if the value at the key is equal or a superset of the filter value.
					</p>
					<div class="flex flex-col gap-4 mt-1">
						{#each filters as v, i}
							<div class="flex w-full gap-2 items-center">
								<div class="w-full flex flex-col gap-2 border p-2 rounded-md">
									<div class="flex flex-row gap-2 w-full">
										<label class="flex flex-col w-full">
											<div class="text-secondary text-sm mb-2">Type</div>
											<select
												class="w-20"
												on:change={(e) => {
													if (e.target?.['value']) {
														filters[i] = {
															key: '',
															value: ''
														}
													}
												}}
												value={'json'}
											>
												<option value="json">JSON</option>
											</select>
										</label>
									</div>
									<label class="flex flex-col w-full">
										<div class="text-secondary text-sm mb-2">Key</div>
										<input type="text" bind:value={v.key} />
									</label>
									<!-- svelte-ignore a11y-label-has-associated-control -->
									<label class="flex flex-col w-full">
										<div class="text-secondary text-sm mb-2">Value</div>
										<JsonEditor bind:value={v.value} code={JSON.stringify(v.value)} />
									</label>
								</div>
								<button
									transition:fade|local={{ duration: 100 }}
									class="rounded-full p-1 bg-surface-secondary duration-200 hover:bg-surface-hover"
									aria-label="Clear"
									on:click={() => {
										filters = filters.filter((_, index) => index !== i)
									}}
								>
									<X size={14} />
								</button>
							</div>
						{/each}

						<div class="flex items-baseline">
							<Button
								variant="border"
								color="light"
								size="xs"
								btnClasses="mt-1"
								on:click={() => {
									if (filters == undefined || !Array.isArray(filters)) {
										filters = []
									}
									filters = filters.concat({
										key: '',
										value: ''
									})
								}}
								startIcon={{ icon: Plus }}
							>
								Add item
							</Button>
						</div>
					</div>
				</Section>
			</div>
		{/if}
	</DrawerContent>
</Drawer>
