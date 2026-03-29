<script lang="ts">
	let firstName = $state('jh');
	let lastName = $state('kim');
	let fullNmae = $derived(firstName.concat(lastName));
	const src = 'https://picsum.photos/250/250';

	const doubledName = () => {
		firstName += firstName;
	};
</script>

<h1>Welcome {fullNmae}</h1>
<button type="button" onclick={doubledName}>click</button>
<img {src} alt="something" />
<input bind:value={firstName} />

<style>
	h1 {
		font-size: 48px;
	}

	button {
		padding: 4px 10px;
		border: 1px solid black;
		border-radius: 12px;
	}
</style>
