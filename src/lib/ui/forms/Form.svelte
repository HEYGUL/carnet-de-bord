<script lang="ts">
	import { onDestroy, setContext } from 'svelte';

	import { writable } from 'svelte/store';
	import { createForm, key } from 'svelte-forms-lib';
	import type { ObjectSchema } from 'yup';
	import { dev } from '$app/env';
	import DebugForm from './DebugForm.svelte';
	import type { ObjectShape } from 'yup/lib/object';

	export let initialValues: Record<string, unknown>;
	export let validationSchema: ObjectSchema<ObjectShape>;
	export let onSubmit: (values: Record<string, unknown>) => void;

	const formHandler = createForm({ initialValues, validationSchema, onSubmit });

	const {
		errors,
		form,
		touched,
		modified,
		handleSubmit,
		isValid,
		isValidating,
		isSubmitting,
		isModified,
		handleReset,
		updateField,
		updateValidateField,
	} = formHandler;

	let formRef: HTMLFormElement;

	const isSubmitted = writable(false);

	setContext(key, { ...formHandler, isSubmitted });

	const unsubscribeValid = isValid.subscribe((valid) => {
		if (!valid && $isSubmitted) {
			focusError();
		}
	});
	function submitHandler(e: Event) {
		$isSubmitted = true;

		handleSubmit(e);
	}

	onDestroy(unsubscribeValid);

	function focusError() {
		// If errors we scroll to the first error
		// and focus the input
		const [key] = Object.entries($errors).find(([, value]) => Boolean(value)) ?? [];
		if (key && formRef[key]) {
			formRef[key].focus();
			const labelEl: HTMLLabelElement = formRef.querySelector(`label[for=${formRef[key].id}]`);
			if (labelEl) {
				labelEl.scrollIntoView({ behavior: 'smooth' });
			} else {
				formRef[key].scrollIntoView({ behavior: 'smooth' });
			}
		}
	}

	$: {
		formHandler.updateInitialValues(initialValues);
	}
</script>

<form bind:this={formRef} on:submit|preventDefault={submitHandler} novalidate class={$$props.class}>
	<slot
		isValid={$isValid}
		isSubmitting={$isSubmitting}
		isSubmitted={$isSubmitted}
		isValidating={$isValidating}
		isModified={$isModified}
		touched={$touched}
		modified={$modified}
		errors={$errors}
		form={$form}
		formStore={form}
		{handleReset}
		{updateField}
		{updateValidateField}
	/>
	{#if dev}
		<DebugForm />
	{/if}
</form>
