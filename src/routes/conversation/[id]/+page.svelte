<script lang="ts">
	import ChatWindow from '$lib/components/chat/ChatWindow.svelte';
	import { pendingMessage } from '$lib/stores/pendingMessage';
	import { onMount } from 'svelte';
	import type { PageData } from './$types';
	import { page } from '$app/stores';
	import { HfInference } from '@huggingface/inference';

	export let data: PageData;

	$: messages = data.messages;

	const hf = new HfInference();
	const model = hf.endpoint($page.url.href);

	let loading = false;

	async function getTextGenerationStream(inputs: string) {
		const response = model.textGenerationStream(
			{
				inputs,
				parameters: {
					// Taken from https://huggingface.co/spaces/huggingface/open-assistant-private-testing/blob/main/app.py#L54
					// @ts-ignore
					stop: ['<|endoftext|>'],
					max_new_tokens: 1024,
					truncate: 1024,
					typical_p: 0.2
				}
			},
			{
				use_cache: false
			}
		);

		// Regex to check if the text finishes by "<" but is not a piece of code like "`<img>`"
		const endOfTextRegex = /(?<!`)<(?!`)/;

		for await (const data of response) {
			if (!data) break;

			if (!data.token.special) {
				const lastMessage = messages.at(-1);

				if (lastMessage?.from !== 'assistant') {
					// First token has a space at the beginning, trim it
					messages = [...messages, { from: 'assistant', content: data.token.text.trimStart() }];
				} else {
					lastMessage.content += data.token.text;
					messages = [...messages];
				}
			}
		}
	}

	async function writeMessage(message: string) {
		if (!message.trim()) return;

		try {
			loading = true;

			messages = [...messages, { from: 'user', content: message }];

			await getTextGenerationStream(message);
		} finally {
			loading = false;
		}
	}

	onMount(async () => {
		if ($pendingMessage) {
			const val = $pendingMessage;
			$pendingMessage = '';

			if (messages.length === 0) {
				writeMessage(val);
			}
		}
	});
</script>

<ChatWindow disabled={loading} {messages} on:message={(message) => writeMessage(message.detail)} />