<script context="module" lang="ts">
	import { ImportStructureDocument } from '$lib/graphql/_gen/typed-document-nodes';
	import type {
		AdminStructureInput,
		ImportStructureMutation,
		ImportStructureMutationVariables,
		StructureInput,
	} from '$lib/graphql/_gen/typed-document-nodes';
	import { operationStore, mutation } from '@urql/svelte';
	import type { OperationStore } from '@urql/svelte';
</script>

<script lang="ts">
	import Dropzone from 'svelte-file-dropzone';
	import { Checkbox, GroupCheckbox } from '$lib/ui/base';
	import { Text, ImportParserError } from '$lib/ui/utils';
	import { Alert, Button } from '$lib/ui/base';
	import { pluralize } from '$lib/helpers';
	import { parseEntities } from '$lib/utils/importFileParser';

	type StructureWithAdminInput = StructureInput & AdminStructureInput;

	type StructureImport = StructureWithAdminInput & {
		valid: boolean;
		uid: string;
	};

	let forceUpdate = false;
	let sendAccountEmail = false;

	let files = [];
	let structures: StructureImport[] = [];

	let parseErrors = [];

	$: structuresToImport = structures.filter(({ uid }) => toImport.includes(uid));

	let toImport = [];

	function handleFilesSelect(event: CustomEvent<{ acceptedFiles: Buffer[] }>): void {
		files = event.detail.acceptedFiles;
		for (let i = 0; i < files.length; i++) {
			parseEntities(
				files[i],
				'StructureImport',
				headers,
				({ entities, idToImport }: Record<string, unknown>, errors: string[]): void => {
					structures = entities as StructureImport[];
					toImport = idToImport as string[];
					parseErrors = errors;
				}
			);
		}
	}

	const insertStore: OperationStore<
		ImportStructureMutation,
		ImportStructureMutationVariables,
		StructureWithAdminInput
	> = operationStore(ImportStructureDocument, null, { additionalTypenames: ['structure'] });
	const inserter = mutation(insertStore);
	let insertInProgress = false;

	let insertResult: {
		struct: StructureWithAdminInput;
		error: string | null;
	}[];

	async function handleSubmit() {
		insertInProgress = true;
		insertResult = [];
		for (const structure of structuresToImport) {
			const { uid, valid, ...struct } = structure;
			const result = await inserter({ ...struct, forceUpdate, sendAccountEmail });
			let errorMessage = "Une erreur s'est produite, la structure n'a pas été importée.";
			if (/admin_structure_structure relation failed/i.test(result.error?.message)) {
				errorMessage =
					"Une erreur s'est produite, le rattachement de l’admin de structure à échoué.";
			}
			if (/uniqueness/i.test(result.error?.message)) {
				errorMessage = 'Cette structure existe déjà.';
			}

			insertResult = [
				...insertResult,
				{
					struct,
					...(result.error && { error: errorMessage }),
				},
			];
		}
		insertInProgress = false;
	}

	const headers = [
		{ label: 'Nom*', key: 'name' },
		{ label: 'Description', key: 'shortDesc' },
		{ label: 'Téléphone', key: 'phone' },
		{ label: 'Adresse', key: 'address1' },
		{ label: 'Adresse (complément)', key: 'address2' },
		{ label: 'Code postal* ', key: 'postalCode' },
		{ label: 'Ville*', key: 'city' },
		{ label: 'Site web', key: 'website' },
		{ label: 'Courriel', key: 'email' },
		{ label: 'SIRET', key: 'siret' },
		{ label: "Courriel de l'administrateur*", key: 'adminEmail' },
		{ label: 'Prénom', key: 'firstname' },
		{ label: 'Nom', key: 'lastname' },
		{ label: 'Fonction', key: 'position' },
		{ label: 'Numéros de téléphone', key: 'phoneNumbers' },
	];

	function backToFileSelect() {
		structures = [];
		parseErrors = [];
	}

	$: successfulImports = (insertResult || []).filter(({ error }) => !error).length;
</script>

<div class="flex flex-col gap-6">
	{#if insertResult === undefined}
		{#if structures.length > 0}
			<div class="border-b border-gray-200 shadow" style="overflow-x: auto;">
				<table class="w-full divide-y divide-gray-300">
					<thead class="px-2 py-2">
						<th />
						{#each headers as { label } (label)}
							<th>{label}</th>
						{/each}
					</thead>
					<tbody class="bg-white divide-y divide-gray-300">
						{#each structures as structure}
							<tr>
								<td class="align-middle">
									{#if structure.valid}
										<GroupCheckbox
											classNames="bottom-3 left-2"
											bind:selectedOptions={toImport}
											groupId={'toImport'}
											option={{ name: structure.uid, label: '' }}
											disabled={!structure.valid}
											title={`${
												toImport.includes(structure.uid) ? 'Ne pas importer' : 'Importer'
											} la structure`}
										/>
									{:else}
										<i
											class="ri-alert-line text-error relative left-3"
											title="La structure ne contient pas les informations obligatoires (marquées d'un astérisque)"
										/>
									{/if}
								</td>
								<td class="px-2 py-2">
									<Text value={structure.name} />
								</td>
								<td class="px-2 py-2">
									<Text value={structure.city} />
								</td>
								<td class="px-2 py-2">
									<Text value={structure.postalCode} />
								</td>
								<td class="px-2 py-2">
									<Text value={structure.address1} />
								</td>
								<td class="px-2 py-2">
									<Text value={structure.address2} />
								</td>
								<td class="px-2 py-2">
									<Text value={structure.phone} />
								</td>
								<td class="px-2 py-2">
									<Text value={structure.email} />
								</td>
								<td class="px-2 py-2">
									<Text value={structure.website} />
								</td>
								<td class="px-2 py-2">
									<Text value={structure.siret} />
								</td>
								<td class="px-2 py-2">
									<Text value={structure.shortDesc} />
								</td>
								<td class="px-2 py-2">
									<Text value={structure.adminEmail} />
								</td>
								<td class="px-2 py-2">
									<Text value={structure.firstname} />
								</td>
								<td class="px-2 py-2">
									<Text value={structure.lastname} />
								</td>
								<td class="px-2 py-2">
									<Text value={structure.position} />
								</td>
								<td class="px-2 py-2">
									<Text value={structure.phoneNumbers} />
								</td>
							</tr>
						{/each}
					</tbody>
				</table>
			</div>
			<ImportParserError {parseErrors} />
			<p>
				Vous allez importer les structures suivantes. Veuillez vérifier que les données sont
				correctes et confirmer. Il est possible de mettre plusieurs fois la même structure pour
				ajouter plusieurs administrateurs à une structure (et une seule structure sera crée).
			</p>
			<p>
				<Checkbox
					name="forceUpdate"
					label="Écraser les informations des structures existantes. Dans le cas ou il y aurait plusieurs fois la même structure, seules les informations de la dernière ligne seront conservées."
					bind:checked={forceUpdate}
				/>
				<Checkbox
					name="sendConfirmEmail"
					label="Envoyer un email de creation de compte aux nouveaux gestionnaires de structure"
					bind:checked={sendAccountEmail}
				/>
			</p>
			<div class="mt-6 flex justify-end flex-row gap-4">
				<span>
					{toImport.length || 'Aucune'}
					{pluralize('structure', toImport.length)}
					{pluralize('sélectionnée', toImport.length)}
					sur {structures.length}
				</span>
			</div>
			<div class="mt-6 flex justify-end flex-row gap-4">
				<Button on:click={backToFileSelect} outline={true}>Retour</Button>
				<Button on:click={handleSubmit} disabled={toImport.length < 1}>Confirmer</Button>
			</div>
		{:else}
			<div>
				Veuillez fournir un fichier au format EXCEL ou CSV.
				<br />Vous pouvez
				<a href="/fichiers/import_structures.csv" download>télécharger un modèle</a>
				et
				<a href="https://pad.incubateur.net/s/y-ZW1qQOw#" target="_blank" rel="noopener noreferrer"
					>consulter la notice de remplissage</a
				>.
			</div>
			<Dropzone on:drop={handleFilesSelect} multiple={false} accept=".csv,.xls,.xlsx">
				Déposez votre fichier ou cliquez pour le rechercher sur votre ordinateur.
			</Dropzone>
			<ImportParserError {parseErrors} />
		{/if}
	{:else}
		<div class="flex flex-col gap-4">
			{#if insertInProgress}
				<Alert
					type="info"
					title={`Ajout ${pluralize('de la', toImport.length, 'des')} ${pluralize(
						'structure',
						toImport.length
					)} en cours... ${insertResult.length}/${toImport.length}`}
				/>
			{:else}
				<Alert
					type={successfulImports ? 'success' : 'error'}
					title={`${successfulImports || 'Aucune'}
					${pluralize('structure', successfulImports)}
					${pluralize('importée', successfulImports)}
					sur ${toImport.length}
					${pluralize('demandée', toImport.length)}.`}
				/>
			{/if}
			{#key insertResult}
				<div class="border-b border-gray-200 shadow">
					<table class="w-full divide-y divide-gray-300">
						<thead class="px-2 py-2">
							<th>Nom</th>
							<th>Ville</th>
							<th>Résultat</th>
						</thead>
						<tbody class="bg-white divide-y divide-gray-300">
							{#each insertResult as structure}
								<tr>
									<td class="px-2 py-2 ">
										<Text value={structure.struct.name} />
									</td>
									<td class="px-2 py-2 ">
										<Text value={structure.struct.city} />
									</td>
									<td class="px-2 py-2 ">
										{#if structure.error}
											<Text classNames="text-error" value={structure.error} />
										{:else}
											<span
												class="fr-icon-success-fill text-success"
												aria-hidden="true"
												style="margin: 0 50%;"
											/>
										{/if}
									</td>
								</tr>
							{/each}
							{#if insertInProgress}
								<tr>
									<td colspan="3">
										<i class="ri-loader-2-fill" style="margin: 0 50%;" />
									</td>
								</tr>
							{/if}
						</tbody>
					</table>
				</div>
			{/key}
		</div>
	{/if}
</div>
