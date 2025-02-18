<script lang="ts">
	import {
		type Resource,
		ResourceService,
		type ResourceType,
		VariableService
	} from '../../../src/gen'
	import { allTrue, emptySchema, sendUserToast } from '../../../src/utils'
	import { createEventDispatcher } from 'svelte'
	import type { Schema } from '../../common'
	import Modal from './Modal.svelte'
	import Path from './Path.svelte'
	import ArgInput from './ArgInput.svelte'
	import AutosizedTextarea from './AutosizedTextarea.svelte'
	import ItemPicker from './ItemPicker.svelte'
	import VariableEditor from './VariableEditor.svelte'
	import Required from './Required.svelte'

	import { workspaceStore } from '../../stores'
	import ResourceTypePicker from './ResourceTypePicker.svelte'

	let path = ''
	let initialPath = ''

	let step = 1

	let resourceToEdit: Resource | undefined

	let description: string = ''
	let DESCRIPTION_PLACEHOLDER = `You can use markdown to style your description.
A good way to make resources user friendly is to link to a default script for your resource [example](scripts/add?template=f2d1dc8df796d9e8)`
	let selectedResourceType: string | undefined
	let resourceType: ResourceType
	let resourceSchema: Schema | undefined
	let args: Record<string, any> = {}

	let error: string | undefined

	let pickForField: string | undefined
	let itemPicker: ItemPicker
	let variableEditor: VariableEditor
	let modal: Modal

	const dispatch = createEventDispatcher()

	export async function initNew() {
		selectedResourceType = undefined
		step = 1
		args = {}
		path = ''
		description = ''
		initialPath = ''
		resourceSchema = emptySchema()
		resourceToEdit = undefined
		modal.openModal()
	}

	export async function initEdit(p: string): Promise<void> {
		initialPath = p
		path = p
		step = 2
		resourceToEdit = await ResourceService.getResource({ workspace: $workspaceStore!, path: p })
		description = resourceToEdit!.description ?? ''
		selectedResourceType = resourceToEdit!.resource_type
		await loadResourceType()
		args = resourceToEdit!.value
		modal.openModal()
	}

	async function createResource(): Promise<void> {
		try {
			await ResourceService.createResource({
				workspace: $workspaceStore!,
				requestBody: { path, value: args, description, resource_type: resourceType.name }
			})
			sendUserToast(`Successfully created resource at ${path}`)

			dispatch('refresh')
			modal.closeModal()
		} catch (err) {
			sendUserToast(`${err}`, true)
		}
	}

	async function editResource(): Promise<void> {
		try {
			if (resourceToEdit) {
				await ResourceService.updateResource({
					workspace: $workspaceStore!,
					path: resourceToEdit.path,
					requestBody: { path, value: args, description }
				})
				sendUserToast(`Successfully updated resource at ${path}`)
				dispatch('refresh')
				modal.closeModal()
			} else {
				throw Error('Cannot edit undefined resourceToEdit')
			}
		} catch (err) {
			sendUserToast(`${err}`, true)
		}
	}

	async function loadResourceType(): Promise<void> {
		if (selectedResourceType) {
			resourceType = await ResourceService.getResourceType({
				workspace: $workspaceStore!,
				path: selectedResourceType
			})

			if (resourceType.schema) {
				resourceSchema = resourceType.schema as Schema
			}
		} else {
			sendUserToast(`ResourceType cannot be undefined.`, true)
		}
	}

	let inputCheck: { [id: string]: boolean } = {}

	$: isValid = allTrue(inputCheck) ?? false
</script>

<Modal
	bind:this={modal}
	on:close={() => {
		dispatch('close')
	}}
>
	<div slot="title">{resourceToEdit ? 'Edit ' + resourceToEdit.path : 'Add a resource'}</div>
	<div slot="content">
		<!-- content -->
		{#if step === 1}
			<div class="flex flex-col gap-3	px-6 py-3 bg-gray-50 text-gray-700">
				<div>
					<span class="text-purple-500 text-2xs grow">{error ?? ''}</span>
					<span class="mb-1 font-semibold text-gray-700">Path</span>
					<Path bind:path {initialPath} namePlaceholder="my/resource">
						<div slot="ownerToolkit" class="text-gray-700 text-2xs">
							Resource permissions depend on their path. Select the group <span class="font-mono"
								>all</span
							>
							to share it, and <span class="font-mono">user</span> to keep it private.
							<a href="https://docs.windmill.dev/docs/reference/namespaces">docs</a>
						</div>
					</Path>
				</div>
				<span class=" mt-3 font-semibold text-gray-700 "
					>Description <Required required={false} />
				</span>
				<AutosizedTextarea
					bind:value={description}
					placeholder={DESCRIPTION_PLACEHOLDER}
					minRows={3}
				/>
				<div>
					<div class="mb-1 font-semibold text-gray-700">
						Resource type<Required required={true} />
					</div>
					<ResourceTypePicker
						bind:value={selectedResourceType}
						notPickable={resourceToEdit != undefined}
						on:click={() => {
							args = {}
						}}
					/>
				</div>
			</div>
		{:else}
			<div class="text-sm">
				{#if resourceSchema && resourceSchema?.properties}
					{#each Object.keys(resourceSchema.properties) as fieldName}
						<div class="flex flex-row w-full items-end justify-between">
							<ArgInput
								label={fieldName}
								description={resourceSchema.properties[fieldName]?.description}
								bind:value={args[fieldName]}
								type={resourceSchema.properties[fieldName]?.type}
								required={resourceSchema.required.includes(fieldName)}
								pattern={resourceSchema.properties[fieldName]?.pattern}
								bind:valid={inputCheck[fieldName]}
								defaultValue={resourceSchema.properties[fieldName]?.default}
								enum_={resourceSchema.properties[fieldName]?.enum}
								contentEncoding={resourceSchema.properties[fieldName]?.contentEncoding}
								itemsType={resourceSchema.properties[fieldName]?.items}
								format={resourceSchema.properties[fieldName]?.format}
							/>
							<div class="pb-3 ml-2 relative">
								<button
									class="default-button-secondary min-w-min items-center leading-4 py-0"
									on:click={() => {
										pickForField = fieldName
										itemPicker.openModal()
									}}>insert variable</button
								>
							</div>
						</div>
					{/each}
				{:else}
					<div>Invalid schema</div>
				{/if}
			</div>
		{/if}
	</div>
	<span slot="submission">
		{#if step === 1}
			{#if selectedResourceType && path != undefined && path != '' && path.split('/')[2] != ''}
				<button
					class="default-button px-4 py-2 font-semibold"
					on:click={async () => {
						await loadResourceType()
						step = 2
					}}
				>
					Next
				</button>
			{:else}
				<button class="default-button-disabled px-4 py-2 font-semibold">Next</button>
			{/if}
		{:else}
			<button
				class="default-button-secondary px-4 py-2 font-semibold"
				on:click={() => {
					step = 1
				}}
			>
				Back
			</button>
			<button
				disabled={!isValid}
				class="default-button px-4 py-2 font-semibold"
				on:click={() => {
					if (resourceToEdit) {
						editResource()
					} else {
						createResource()
					}
				}}
			>
				Save
			</button>
		{/if}
	</span>
</Modal>

<ItemPicker
	bind:this={itemPicker}
	pickCallback={(path, _) => {
		if (pickForField) {
			args[pickForField] = '$var:' + path
		}
	}}
	itemName="Variable"
	extraField="name"
	loadItems={async () =>
		(await VariableService.listVariable({ workspace: $workspaceStore ?? '' })).map((x) => ({
			name: x.path,
			...x
		}))}
>
	<div
		slot="submission"
		class="flex flex-row-reverse w-full p-5 bg-white border-t border-gray-200 rounded-bl-lg rounded-br-lg"
	>
		<button
			class="default-button-secondary"
			type="button"
			on:click={() => {
				variableEditor.initNew()
			}}
		>
			Create a new variable
		</button>
		<div class="text-xs mr-2 align-middle">
			The variable you were looking for does not exist yet?
		</div>
	</div>
</ItemPicker>

<VariableEditor bind:this={variableEditor} on:create={itemPicker.openModal} />
