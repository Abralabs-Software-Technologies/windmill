<script lang="ts">
	import { goto } from '$app/navigation'

	import { UserService, WorkspaceService } from '../../gen'
	import { sendUserToast } from '../../utils'
	import { page } from '$app/stores'
	import { usersWorkspaceStore, workspaceStore } from '../../stores'
	import CenteredModal from './CenteredModal.svelte'

	let workspace_id = $page.url.searchParams.get('workspace') ?? ''
	let username = ''
	let errorUsername = ''

	$: validateName(username)

	async function acceptInvite(): Promise<void> {
		await UserService.acceptInvite({
			requestBody: {
				username,
				workspace_id
			}
		})
		sendUserToast(`Invitation to ${workspace_id} accepted as ${username}`)
		usersWorkspaceStore.set(await WorkspaceService.listUserWorkspaces())
		workspaceStore.set(workspace_id)
		goto('/')
	}

	async function validateName(username: string): Promise<void> {
		try {
			await WorkspaceService.validateUsername({ requestBody: { id: workspace_id, username } })
			errorUsername = ''
		} catch {
			errorUsername = 'username already exists'
		}
	}

	function handleKey(event: KeyboardEvent) {
		const key = event.key || event.keyCode
		if (key === 13 || key === 'Enter') {
			event.preventDefault()
			acceptInvite()
		}
	}

	UserService.globalWhoami().then((x) => {
		if (x.name) {
			username = x.name.split(' ')[0]
		} else {
			username = x.email.split('@')[0]
		}
		username = username.toLowerCase()
	})
</script>

<!-- Enable submit form on enter -->

<CenteredModal title="Invitation to join {workspace_id}">
	<label class="block pb-2">
		<span class="text-gray-700">Your username in workspace {workspace_id}:</span>
		{#if errorUsername}
			<span class="text-red-500 text-xs">{errorUsername}</span>
		{/if}
		<input
			on:keyup={handleKey}
			bind:value={username}
			class="default-input"
			class:input-error={errorUsername != ''}
		/>
	</label>
	<div class="flex flex-row justify-between pt-4">
		<a href="/user/workspaces">&leftarrow; workspaces</a>

		<button
			disabled={errorUsername != '' || !username}
			class="place-items-end bg-blue-500 hover:bg-blue-700 text-white font-bold py-1 px-2 border rounded"
			type="button"
			on:click={acceptInvite}
		>
			Accept invite
		</button>
	</div>
</CenteredModal>
