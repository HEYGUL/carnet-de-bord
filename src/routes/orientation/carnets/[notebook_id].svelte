<script context="module" lang="ts">
	import NotebookView from '$lib/ui/views/NotebookView.svelte';
	import LoaderIndicator from '$lib/ui/utils/LoaderIndicator.svelte';
	import type { Load } from '@sveltejs/kit';
	import { GetNotebookByIdDocument } from '$lib/graphql/_gen/typed-document-nodes';
	import { operationStore, query } from '@urql/svelte';
	import { displayFullName } from '$lib/ui/format';

	export const load: Load = ({ params }) => {
		const notebookId = params.notebook_id;

		return {
			props: {
				notebookId,
			},
		};
	};
</script>

<script lang="ts">
	export let notebookId: string;

	const getNotebookResult = operationStore(GetNotebookByIdDocument, {
		id: notebookId,
	});
	query(getNotebookResult);

	$: name = $getNotebookResult.data?.notebook
		? displayFullName($getNotebookResult.data?.notebook.beneficiary)
		: '';
</script>

<svelte:head>
	<title>Carnet {name} - Carnet de bord</title>
</svelte:head>

<LoaderIndicator result={getNotebookResult}>
	<div class="my-6">
		<NotebookView notebook={$getNotebookResult.data.notebook} />
	</div>
</LoaderIndicator>
