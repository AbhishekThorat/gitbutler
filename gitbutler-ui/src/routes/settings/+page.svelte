<script lang="ts">
	import { deleteAllData } from '$lib/backend/data';
	import AnalyticsSettings from '$lib/components/AnalyticsSettings.svelte';
	import Button from '$lib/components/Button.svelte';
	import ClickableCard from '$lib/components/ClickableCard.svelte';
	import GithubIntegration from '$lib/components/GithubIntegration.svelte';
	import Link from '$lib/components/Link.svelte';
	import Modal from '$lib/components/Modal.svelte';
	import SectionCard from '$lib/components/SectionCard.svelte';
	import Spacer from '$lib/components/Spacer.svelte';
	import TextBox from '$lib/components/TextBox.svelte';
	import ThemeSelector from '$lib/components/ThemeSelector.svelte';
	import Toggle from '$lib/components/Toggle.svelte';
	import WelcomeSigninAction from '$lib/components/WelcomeSigninAction.svelte';
	import ContentWrapper from '$lib/components/settings/ContentWrapper.svelte';
	import ProfileSIdebar from '$lib/components/settings/ProfileSIdebar.svelte';
	import { copyToClipboard } from '$lib/utils/clipboard';
	import * as toasts from '$lib/utils/toasts';
	import { open } from '@tauri-apps/api/shell';
	import { invoke } from '@tauri-apps/api/tauri';
	import type { PageData } from './$types';
	import { goto } from '$app/navigation';

	export let data: PageData;
	const { cloud, user$, userService } = data;

	$: saving = false;

	$: userPicture = $user$?.picture;

	const fileTypes = ['image/jpeg', 'image/png'];

	const validFileType = (file: File) => {
		return fileTypes.includes(file.type);
	};

	const onPictureChange = (e: Event) => {
		const target = e.target as HTMLInputElement;
		const file = target.files?.[0];

		if (file && validFileType(file)) {
			userPicture = URL.createObjectURL(file);
		} else {
			userPicture = $user$?.picture;
			toasts.error('Please use a valid image file');
		}
	};

	let newName = '';

	let loaded = false;
	$: if ($user$ && !loaded) {
		loaded = true;
		cloud.user.get($user$?.access_token).then((cloudUser) => {
			cloudUser.github_access_token = $user$?.github_access_token; // prevent overwriting with null
			userService.setUser(cloudUser);
		});
		newName = $user$?.name || '';
	}

	const onSubmit = async (e: SubmitEvent) => {
		if (!$user$) return;
		saving = true;

		const target = e.target as HTMLFormElement;
		const formData = new FormData(target);
		const picture = formData.get('picture') as File | undefined;

		try {
			const updatedUser = await cloud.user.update($user$.access_token, {
				name: newName,
				picture: picture
			});
			updatedUser.github_access_token = $user$?.github_access_token; // prevent overwriting with null
			userService.setUser(updatedUser);
			toasts.success('Profile updated');
		} catch (e) {
			console.error(e);
			toasts.error('Failed to update user');
		}
		saving = false;
	};

	let isDeleting = false;
	let deleteConfirmationModal: Modal;

	export function git_get_config(params: { key: string }) {
		return invoke<string>('git_get_global_config', params);
	}

	export function git_set_config(params: { key: string; value: string }) {
		return invoke<string>('git_set_global_config', params);
	}

	const setCommitterSetting = (value: boolean) => {
		annotateCommits = value;
		git_set_config({
			key: 'gitbutler.gitbutlerCommitter',
			value: value ? '1' : '0'
		});
	};

	const setSigningSetting = (value: boolean) => {
		signCommits = value;
		git_set_config({
			key: 'gitbutler.signCommits',
			value: value ? 'true' : 'false'
		});
	};

	export function get_public_key() {
		return invoke<string>('get_public_key');
	}

	let sshKey = '';
	get_public_key().then((key) => {
		sshKey = key;
	});

	$: annotateCommits = true;
	$: signCommits = false;

	git_get_config({ key: 'gitbutler.gitbutlerCommitter' }).then((value) => {
		annotateCommits = value ? value === '1' : false;
	});

	git_get_config({ key: 'gitbutler.signCommits' }).then((value) => {
		signCommits = value ? value === 'true' : false;
	});

	const onDeleteClicked = () =>
		Promise.resolve()
			.then(() => (isDeleting = true))
			.then(() => deleteAllData())
			// TODO: Delete user from observable!!!
			.then(() => userService.logout())
			.then(() => toasts.success('All data deleted'))
			.catch((e) => {
				console.error(e);
				toasts.error('Failed to delete project');
			})
			.then(() => deleteConfirmationModal.close())
			.then(() => goto('/', { replaceState: true, invalidateAll: true }))
			.finally(() => (isDeleting = false));

	let currentSection: 'profile' | 'git-stuff' | 'telemetry' | 'integrations' = 'profile';

	const toggleGBCommiter = () => setCommitterSetting(!annotateCommits);
	const toggleGBSigner = () => setSigningSetting(!signCommits);
</script>

<section class="profile-page">
	<ProfileSIdebar {userService} bind:currentSection showIntegrations={!!$user$} />
	{#if currentSection === 'profile'}
		<ContentWrapper title="Profile">
			{#if $user$}
				<SectionCard>
					<form on:submit={onSubmit} class="profile-form">
						<label id="profile-picture" class="focus-state profile-pic-wrapper" for="picture">
							<input
								on:change={onPictureChange}
								type="file"
								id="picture"
								name="picture"
								accept={fileTypes.join('')}
								class="hidden-input"
							/>

							{#if $user$.picture}
								<img class="profile-pic" src={userPicture} alt="" />
							{/if}

							<span class="profile-pic__edit-label text-base-11 text-semibold">Edit</span>
						</label>

						<div id="contact-info" class="contact-info">
							<div class="contact-info__fields">
								<TextBox label="Full name" bind:value={newName} required />
								<TextBox label="Email" bind:value={$user$.email} readonly />
							</div>

							<Button loading={saving} color="primary">Update profile</Button>
						</div>
					</form>
				</SectionCard>
			{:else}
				<WelcomeSigninAction {userService} />
				<Spacer />
			{/if}

			<SectionCard>
				<svelte:fragment slot="title">Appearance</svelte:fragment>
				<ThemeSelector />
			</SectionCard>

			<Spacer />

			<SectionCard>
				<svelte:fragment slot="title">Remove all projects</svelte:fragment>
				<svelte:fragment slot="body">
					You can delete all projects from the GitButler app. Your code remains safe.
					<br />
					it only clears the configuration.
				</svelte:fragment>

				<Button
					color="error"
					kind="outlined"
					grow
					align="flex-start"
					on:click={() => deleteConfirmationModal.show()}
				>
					Remove all projects…
				</Button>

				<Modal bind:this={deleteConfirmationModal} title="Remove all projects">
					<p>Are you sure you want to remove all GitButler projects?</p>

					<svelte:fragment slot="controls" let:close>
						<Button kind="outlined" color="error" loading={isDeleting} on:click={onDeleteClicked}
							>Remove</Button
						>
						<Button on:click={close}>Cancel</Button>
					</svelte:fragment>
				</Modal>
			</SectionCard>
		</ContentWrapper>
	{:else if currentSection === 'git-stuff'}
		<ContentWrapper title="Git stuff">
			<ClickableCard on:click={toggleGBCommiter}>
				<svelte:fragment slot="title">Credit GitButler as the Committer</svelte:fragment>
				<svelte:fragment slot="body">
					By default, everything in the GitButler client is free to use. You can opt in to crediting
					us as the committer in your virtual branch commits to help spread the word.
					<Link
						target="_blank"
						rel="noreferrer"
						href="https://docs.gitbutler.com/features/virtual-branches/committer-mark"
					>
						Learn more
					</Link>
				</svelte:fragment>
				<svelte:fragment slot="actions">
					<Toggle checked={annotateCommits} on:change={toggleGBCommiter} />
				</svelte:fragment>
			</ClickableCard>

			<Spacer />

			<SectionCard>
				<svelte:fragment slot="title">SSH Key</svelte:fragment>
				<svelte:fragment slot="body">
					GitButler uses SSH keys to authenticate with your Git provider. Add the following public
					key to your Git provider to enable GitButler to push code.
				</svelte:fragment>

				<TextBox readonly selectall bind:value={sshKey} />
				<div class="row-buttons">
					<Button
						kind="filled"
						color="primary"
						icon="copy"
						on:click={() => copyToClipboard(sshKey)}
					>
						Copy to Clipboard
					</Button>
					<Button
						kind="outlined"
						color="neutral"
						icon="open-link"
						on:click={() => {
							open('https://github.com/settings/ssh/new');
						}}
					>
						Add key to GitHub
					</Button>
				</div>
			</SectionCard>

			<ClickableCard on:click={toggleGBSigner}>
				<svelte:fragment slot="title">Sign Commits with the above SSH Key</svelte:fragment>
				<svelte:fragment slot="body">
					If you want GitButler to sign your commits with the SSH key we generated, then you can add
					that key to GitHub as a signing key to have those commits verified.
					<Link
						target="_blank"
						rel="noreferrer"
						href="https://docs.gitbutler.com/features/virtual-branches/verifying-commits"
					>
						Learn more
					</Link>
				</svelte:fragment>
				<svelte:fragment slot="actions">
					<Toggle checked={signCommits} on:change={toggleGBSigner} />
				</svelte:fragment>
			</ClickableCard>
		</ContentWrapper>
	{:else if currentSection === 'telemetry'}
		<ContentWrapper title="Telemetry">
			<AnalyticsSettings />
		</ContentWrapper>
	{:else if currentSection === 'integrations'}
		<ContentWrapper title="Integrations">
			{#if $user$}
				<GithubIntegration {userService} />
			{/if}
		</ContentWrapper>
	{/if}
</section>

<style lang="postcss">
	.profile-page {
		display: flex;
		width: 100%;
	}

	.profile-form {
		display: flex;
		gap: var(--space-24);
	}

	.hidden-input {
		z-index: 1;
		position: absolute;
		background-color: red;
		width: 100%;
		height: 100%;
		opacity: 0;
	}

	.profile-pic-wrapper {
		position: relative;
		width: 100px;
		height: 100px;
		border-radius: var(--radius-m);
		overflow: hidden;
		background-color: var(--clr-theme-scale-pop-70);
		transition: opacity var(--transition-medium);

		&:hover,
		&:focus-within {
			& .profile-pic__edit-label {
				opacity: 1;
			}

			& .profile-pic {
				opacity: 0.8;
			}
		}
	}

	.profile-pic {
		width: 100%;
		height: 100%;

		object-fit: cover;
		background-color: var(--clr-theme-scale-pop-70);
	}

	.profile-pic__edit-label {
		position: absolute;
		bottom: var(--space-8);
		left: var(--space-8);
		color: var(--clr-core-ntrl-100);
		background-color: color-mix(in srgb, var(--clr-core-ntrl-0), transparent 30%);
		padding: var(--space-4) var(--space-6);
		border-radius: var(--radius-m);
		opacity: 0;
		transition: opacity var(--transition-medium);
	}

	.contact-info {
		flex: 1;
		display: flex;
		flex-direction: column;
		gap: var(--space-20);
		align-items: flex-end;
	}

	.contact-info__fields {
		width: 100%;
		display: flex;
		flex-direction: column;
		gap: var(--space-12);
	}

	.row-buttons {
		display: flex;
		justify-content: flex-end;
		gap: var(--space-8);
	}
</style>
