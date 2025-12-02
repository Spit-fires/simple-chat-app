<script lang="ts">
	import { onMount, onDestroy, tick } from 'svelte';
	import { pb } from '$lib/pocketbase';
	import { Button } from '$lib/components/ui/button';
	import { Input } from '$lib/components/ui/input';
	import * as Card from '$lib/components/ui/card';
	import { ScrollArea } from '$lib/components/ui/scroll-area';
	import * as Avatar from '$lib/components/ui/avatar';
	import { Label } from '$lib/components/ui/label';
	import type { MessagesResponse, UsersResponse } from '../../pocketbase-types';

	let currentUser = $state(pb.authStore.model);
	let messages = $state<MessagesResponse<{ user: UsersResponse }>[]>([]);
	let newMessage = $state('');
	let isLogin = $state(true);
	let email = $state('');
	let password = $state('');
	let name = $state('');
	let loading = $state(false);
	let error = $state('');
	let scrollViewport: HTMLElement | undefined = $state(undefined);
	let page = 1;
	let hasMore = true;
	let isLoadingMore = $state(false);

	onMount(async () => {

		pb.authStore.onChange(() => {
			currentUser = pb.authStore.model;
		});

		if (currentUser) {
			await loadInitialMessages();
			await subscribe();
		}
	});

	onDestroy(() => {
		pb.collection('messages').unsubscribe();
	});

	async function loadInitialMessages() {
		page = 1;
		try {
			const result = await pb.collection('messages').getList(1, 50, {
				sort: '-created',
				expand: 'user'
			});
			messages = result.items.reverse() as unknown as MessagesResponse<{ user: UsersResponse }>[];
			hasMore = result.totalPages > 1;
			scrollToBottom();
		} catch (e) {
			console.error(e);
		}
	}

	async function loadMoreMessages() {
		if (isLoadingMore || !hasMore) return;
		isLoadingMore = true;
		const oldScrollHeight = scrollViewport?.scrollHeight || 0;

		try {
			const result = await pb.collection('messages').getList(page + 1, 50, {
				sort: '-created',
				expand: 'user'
			});

			if (result.items.length > 0) {
				const newMessages = result.items.reverse() as unknown as MessagesResponse<{
					user: UsersResponse;
				}>[];
				messages = [...newMessages, ...messages];
				page++;
				hasMore = result.page < result.totalPages;

				await tick();
				if (scrollViewport) {
					scrollViewport.scrollTop = scrollViewport.scrollHeight - oldScrollHeight;
				}
			} else {
				hasMore = false;
			}
		} catch (e) {
			console.error(e);
		} finally {
			isLoadingMore = false;
		}
	}

	function handleScroll(e: Event) {
		const target = e.target as HTMLElement;
		if (target.scrollTop === 0 && hasMore) {
			loadMoreMessages();
		}
	}

	async function subscribe() {
		pb.collection('messages').subscribe('*', async ({ action, record }) => {
			if (action === 'create') {
				const userId = Array.isArray(record.user) ? record.user[0] : record.user;
				const user = await pb.collection('users').getOne(userId);
				const expandedRecord = { ...record, expand: { user } } as unknown as MessagesResponse<{
					user: UsersResponse;
				}>;
				messages = [...messages, expandedRecord];
				scrollToBottom();
			}
			if (action === 'delete') {
				messages = messages.filter((m) => m.id !== record.id);
			}
		});
	}

	async function scrollToBottom() {
		await tick();
		if (scrollViewport) {
			scrollViewport.scrollTop = scrollViewport.scrollHeight;
		}
	}

	async function handleAuth() {
		loading = true;
		error = '';
		try {
			if (isLogin) {
				await pb.collection('users').authWithPassword(email, password);
			} else {
				await pb.collection('users').create({
					email,
					password,
					passwordConfirm: password,
					name
				});
				await pb.collection('users').authWithPassword(email, password);
			}
			await loadInitialMessages();
			await subscribe();
		} catch (e: any) {
			error = e.message;
		} finally {
			loading = false;
		}
	}

	function logout() {
		pb.authStore.clear();
		messages = [];
		pb.collection('messages').unsubscribe();
	}

	async function sendMessage() {
		if (!newMessage.trim()) return;
		try {
			await pb.collection('messages').create({
				text: newMessage,
				user: currentUser?.id
			});
			newMessage = '';
		} catch (e) {
			console.error(e);
		}
	}
</script>

<div class="flex h-screen w-full items-center justify-center bg-slate-50 p-4 dark:bg-slate-950">
	{#if !currentUser}
		<Card.Root class="w-full max-w-md">
			<Card.Header>
				<Card.Title>{isLogin ? 'Login' : 'Sign Up'}</Card.Title>
				<Card.Description>
					{isLogin ? 'Welcome back!' : 'Create an account to join the chat.'}
				</Card.Description>
			</Card.Header>
			<Card.Content>
				<form
					onsubmit={(e) => {
						e.preventDefault();
						handleAuth();
					}}
					class="space-y-4"
				>
					{#if !isLogin}
						<div class="space-y-2">
							<Label for="name">Name</Label>
							<Input id="name" bind:value={name} placeholder="John Doe" required />
						</div>
					{/if}
					<div class="space-y-2">
						<Label for="email">Email</Label>
						<Input id="email" type="email" bind:value={email} placeholder="m@example.com" required />
					</div>
					<div class="space-y-2">
						<Label for="password">Password</Label>
						<Input id="password" type="password" bind:value={password} required />
					</div>
					{#if error}
						<p class="text-sm text-red-500">{error}</p>
					{/if}
					<Button type="submit" class="w-full" disabled={loading}>
						{loading ? 'Loading...' : isLogin ? 'Login' : 'Sign Up'}
					</Button>
				</form>
			</Card.Content>
			<Card.Footer>
				<Button variant="link" onclick={() => (isLogin = !isLogin)} class="w-full">
					{isLogin ? "Don't have an account? Sign up" : 'Already have an account? Login'}
				</Button>
			</Card.Footer>
		</Card.Root>
	{:else}
		<Card.Root class="flex h-[80vh] w-full max-w-3xl flex-col overflow-hidden">
			<Card.Header class="border-b px-6 py-4">
				<div class="flex items-center justify-between">
					<div class="flex items-center gap-3">
						<Avatar.Root>
							<Avatar.Image
								src={currentUser.avatar
									? pb.files.getUrl(currentUser, currentUser.avatar)
									: undefined}
								alt={currentUser.name}
							/>
							<Avatar.Fallback>{currentUser.name?.slice(0, 2).toUpperCase() || 'U'}</Avatar.Fallback>
						</Avatar.Root>
						<div>
							<Card.Title>Global Chat</Card.Title>
							<Card.Description>Logged in as {currentUser.name || currentUser.email}</Card.Description>
						</div>
					</div>
					<Button variant="outline" size="sm" onclick={logout}>Logout</Button>
				</div>
			</Card.Header>
			<Card.Content class="flex-1 overflow-hidden p-0">
				<div class="h-full overflow-y-auto p-4" bind:this={scrollViewport} onscroll={handleScroll}>
					<div class="flex flex-col gap-4">
						{#if isLoadingMore}
							<div class="flex justify-center py-2">
								<span class="text-xs text-muted-foreground">Loading history...</span>
							</div>
						{/if}
						{#each messages as message (message.id)}
							{@const isMe = message.user.includes(currentUser.id)}
							<div class={`flex ${isMe ? 'justify-end' : 'justify-start'}`}>
								<div class={`flex max-w-[80%] gap-2 ${isMe ? 'flex-row-reverse' : 'flex-row'}`}>
									<Avatar.Root class="h-8 w-8">
										<Avatar.Image
											src={message.expand?.user?.avatar
												? pb.files.getUrl(message.expand.user, message.expand.user.avatar)
												: undefined}
											alt={message.expand?.user?.name}
										/>
										<Avatar.Fallback>
											{message.expand?.user?.name?.slice(0, 2).toUpperCase() || 'U'}
										</Avatar.Fallback>
									</Avatar.Root>
									<div
										class={`rounded-lg p-3 ${
											isMe ? 'bg-primary text-primary-foreground' : 'bg-muted'
										}`}
									>
										{#if !isMe}
											<p class="mb-1 text-xs font-medium opacity-70">
												{message.expand?.user?.name || 'Unknown'}
											</p>
										{/if}
										<p class="text-sm">{message.text}</p>
										<p class="mt-1 text-[10px] opacity-50">
											{new Date(message.created).toLocaleTimeString([], {
												hour: '2-digit',
												minute: '2-digit'
											})}
										</p>
									</div>
								</div>
							</div>
						{/each}
					</div>
				</div>
			</Card.Content>
			<Card.Footer class="border-t p-4">
				<form
					class="flex w-full gap-2"
					onsubmit={(e) => {
						e.preventDefault();
						sendMessage();
					}}
				>
					<Input bind:value={newMessage} placeholder="Type a message..." class="flex-1" />
					<Button type="submit" disabled={!newMessage.trim()}>Send</Button>
				</form>
			</Card.Footer>
		</Card.Root>
	{/if}
</div>
