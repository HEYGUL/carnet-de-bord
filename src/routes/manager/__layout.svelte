<script context="module" lang="ts">
	import { goto } from '$app/navigation';
	import type { GetAccountByPkQuery } from '$lib/graphql/_gen/typed-document-nodes';
	import { GetAccountByPkDocument } from '$lib/graphql/_gen/typed-document-nodes';
	import type { Load } from '@sveltejs/kit';
	import type { OperationStore } from '@urql/svelte';
	import { operationStore, query } from '@urql/svelte';

	export const load: Load = async ({ session }) => {
		const accountId = session.user.id;
		const result = operationStore(GetAccountByPkDocument, { accountId });

		return {
			props: {
				result,
			},
		};
	};
</script>

<script lang="ts">
	import type { MenuItem } from '$lib/types';
	import Footer from '$lib/ui/base/Footer.svelte';
	import Header from '$lib/ui/base/Header.svelte';

	import { LayerCDB } from '$lib/ui/index';
	import { page } from '$app/stores';
	import { account } from '$lib/stores';
	import { baseUrlForRole, homeForRole } from '$lib/routes';

	export let result: OperationStore<GetAccountByPkQuery>;

	query(result);

	result.subscribe((result) => {
		if (result.data) {
			const acc = result.data.account_by_pk;
			if (acc) {
				const { username, onboardingDone, confirmed, id: accountId } = acc;
				const { id, firstname, lastname, email } = acc.manager;
				$account = {
					type: 'manager',
					accountId,
					id,
					username,
					onboardingDone,
					confirmed,
					firstname,
					lastname,
					email,
				};

				if (!onboardingDone && $page.url.pathname !== '/manager/moncompte') {
					goto('/manager/moncompte');
				}
			}
		}
	});

	const menuItems: MenuItem[] = [
		{ id: 'home', path: homeForRole('manager'), label: 'Accueil' },
		{
			id: 'beneficiaires',
			path: `${baseUrlForRole('manager')}/beneficiaires`,
			label: 'Bénéficiaires',
		},
		{ id: 'structures', path: `${baseUrlForRole('manager')}/structures`, label: 'Structures' },
		{ id: 'pro', path: `${baseUrlForRole('manager')}/professionnels`, label: 'Professionnels' },
	];
</script>

<Header {menuItems} />

<div class="fr-container fr-pb-6w fr-px-2w" style="min-height: calc(100vh - 200px)">
	<slot />
	<LayerCDB />
</div>

<Footer />
