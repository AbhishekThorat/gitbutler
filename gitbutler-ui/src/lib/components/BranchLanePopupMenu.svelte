<script lang="ts">
	import Button from '$lib/components/Button.svelte';
	import Modal from '$lib/components/Modal.svelte';
	import TextBox from '$lib/components/TextBox.svelte';
	import ContextMenu from '$lib/components/contextmenu/ContextMenu.svelte';
	import ContextMenuItem from '$lib/components/contextmenu/ContextMenuItem.svelte';
	import ContextMenuSection from '$lib/components/contextmenu/ContextMenuSection.svelte';
	import { projectAiGenEnabled } from '$lib/config/config';
	import { normalizeBranchName } from '$lib/utils/branch';
	import { createEventDispatcher } from 'svelte';
	import type { BranchController } from '$lib/vbranches/branchController';
	import type { Branch } from '$lib/vbranches/types';

	export let branchController: BranchController;
	export let branch: Branch;
	export let projectId: string;
	export let visible: boolean;
	export let isUnapplied = false;

	let deleteBranchModal: Modal;
	let renameRemoteModal: Modal;
	let newRemoteName: string;

	const dispatch = createEventDispatcher<{
		action: 'expand' | 'collapse' | 'generate-branch-name';
	}>();

	const aiGenEnabled = projectAiGenEnabled(projectId);

	$: commits = branch.commits;
	$: hasIntegratedCommits =
		commits.length > 0 ? commits.some((c) => c.status == 'integrated') : false;
</script>

{#if visible}
	<ContextMenu>
		<ContextMenuSection>
			{#if !isUnapplied}
				<ContextMenuItem
					label="Unapply"
					on:click={() => {
						if (branch.id) branchController.unapplyBranch(branch.id);
						visible = false;
					}}
				/>
			{/if}

			<ContextMenuItem
				label="Delete"
				on:click={async () => {
					if (
						branch.name.toLowerCase().includes('virtual branch') &&
						commits.length == 0 &&
						branch.files?.length == 0
					) {
						await branchController.deleteBranch(branch.id);
					} else {
						deleteBranchModal.show(branch);
					}
					visible = false;
				}}
			/>

			<ContextMenuItem
				label="Generate branch name"
				on:click={() => {
					dispatch('action', 'generate-branch-name');
					visible = false;
				}}
				disabled={isUnapplied || !$aiGenEnabled || branch.files?.length == 0 || !branch.active}
			/>
		</ContextMenuSection>
		<ContextMenuSection>
			<ContextMenuItem
				label="Set branch name"
				disabled={isUnapplied || hasIntegratedCommits}
				on:click={() => {
					newRemoteName = branch.upstreamName || normalizeBranchName(branch.name) || '';
					visible = false;
					renameRemoteModal.show(branch);
				}}
			/>
		</ContextMenuSection>
		<ContextMenuSection>
			<ContextMenuItem
				label="Create branch to the left"
				on:click={() => {
					branchController.createBranch({ order: branch.order });
					visible = false;
				}}
			/>

			<ContextMenuItem
				label="Create branch to the right"
				on:click={() => {
					branchController.createBranch({ order: branch.order + 1 });
					visible = false;
				}}
			/>
		</ContextMenuSection>
	</ContextMenu>
{/if}

<Modal width="small" bind:this={renameRemoteModal}>
	<TextBox label="Remote branch name" id="newRemoteName" bind:value={newRemoteName}></TextBox>

	<svelte:fragment slot="controls" let:close>
		<Button color="neutral" kind="outlined" on:click={close}>Cancel</Button>
		<Button
			color="primary"
			on:click={() => {
				branchController.updateBranchRemoteName(branch.id, newRemoteName);
				close();
				visible = false;
			}}
		>
			Rename
		</Button>
	</svelte:fragment>
</Modal>

<Modal width="small" title="Delete branch" bind:this={deleteBranchModal} let:item={branch}>
	<div>
		Deleting <code>{branch.name}</code> cannot be undone.
	</div>
	<svelte:fragment slot="controls" let:close let:item={branch}>
		<Button kind="outlined" color="neutral" on:click={close}>Cancel</Button>
		<Button
			color="error"
			on:click={async () => {
				await branchController.deleteBranch(branch.id);
				visible = false;
			}}
		>
			Delete
		</Button>
	</svelte:fragment>
</Modal>

<style lang="postcss">
</style>
