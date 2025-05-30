<script lang="ts">
	import { addWhitespaceBeforeCapitals, capitalize, classNames } from '$lib/utils'
	import Tooltip from '$lib/components/Tooltip.svelte'

	import ToggleButtonGroup from '$lib/components/common/toggleButton-v2/ToggleButtonGroup.svelte'
	import ToggleButton from '$lib/components/common/toggleButton-v2/ToggleButton.svelte'
	import { Loader2, Pen, User } from 'lucide-svelte'

	import Toggle from '$lib/components/Toggle.svelte'
	import StaticInputEditor from '../apps/editor/settingsPanel/inputEditor/StaticInputEditor.svelte'
	import { fieldTypeToTsType } from '../apps/utils'
	import type { RichConfiguration } from '../apps/types'
	import type { InputType } from '../apps/inputType'

	export let id: string
	export let componentInput: RichConfiguration
	export let key: string
	export let shouldCapitalize: boolean = true
	export let resourceOnly = false
	export let tooltip: string | undefined = undefined
	export let fieldType: InputType
	export let subFieldType: InputType | undefined
	export let format: string | undefined
	export let selectOptions: string[] | undefined
	export let placeholder: string | undefined
	export let customTitle: string | undefined = undefined
	export let displayType: boolean = false
	export let allowTypeChange: boolean = true
	export let loading: boolean = false
	export let documentationLink: string | undefined = undefined
	export let markdownTooltip: string | undefined = undefined

	$: if (componentInput == undefined) {
		//@ts-ignore
		componentInput = {
			type: 'user'
		}
	}
</script>

{#if !(resourceOnly && (fieldType !== 'object' || !format?.startsWith('resource-')))}
	<div class={classNames('flex gap-1', 'flex-col')}>
		<div class="flex justify-between items-end">
			<div class="flex flex-row gap-4 items-center">
				<div class="flex items-center">
					<span class="!text-2xs font-semibold text-ellipsis text-primary">
						{customTitle
							? customTitle
							: shouldCapitalize
								? capitalize(addWhitespaceBeforeCapitals(key))
								: key}
					</span>
					{#if loading}
						<Loader2 size={14} class="animate-spin ml-2" />
					{/if}
					{#if tooltip || markdownTooltip}
						<Tooltip small {documentationLink} {markdownTooltip}>
							{tooltip}
						</Tooltip>
					{/if}
				</div>
				{#if displayType}
					<div class="text-xs text-tertiary mr-1">
						{fieldType === 'array' && subFieldType
							? `${fieldTypeToTsType(subFieldType)}[]`
							: fieldTypeToTsType(fieldType)}
					</div>
				{/if}
			</div>

			<div class={classNames('flex gap-x-2 gap-y-1 justify-end items-center')}>
				{#if componentInput?.type && allowTypeChange !== false}
					<ToggleButtonGroup class="h-7" bind:selected={componentInput.type} let:item>
						<ToggleButton {item} value="user" icon={User} iconOnly tooltip="User Input" />
						<ToggleButton {item} value="static" icon={Pen} iconOnly tooltip="Static" />
					</ToggleButtonGroup>
				{/if}
			</div>
		</div>

		{#if componentInput?.type === 'static'}
			<div class={'w-full flex flex-row-reverse'}>
				<StaticInputEditor
					{id}
					{fieldType}
					{subFieldType}
					{selectOptions}
					{format}
					{placeholder}
					bind:componentInput
				/>
			</div>
		{:else if componentInput?.type === 'user' || componentInput?.type == undefined}
			<span class="text-2xs italic text-tertiary">Field's value is a frontend input</span>
		{/if}
		{#if componentInput?.type === 'user' && ((fieldType == 'object' && format?.startsWith('resource-') && format !== 'resource-s3_object') || fieldType == 'resource')}
			<div class="flex flex-row items-center">
				<Toggle
					size="xs"
					bind:checked={componentInput.allowUserResources}
					options={{
						left: 'static resource select only',
						right: 'resources from users allowed'
					}}
				/>
				<Tooltip
					>Apps are executed on behalf of publishers. If you want to accept resources from user, you
					need to enable this (potentially dangerous!)</Tooltip
				>
			</div>
		{/if}
	</div>
{/if}
