<template>
  <q-page class="">
    <gltf-component/>
  </q-page>
</template>

<script lang="ts">
import GltfComponent from 'components/GltfComponent.vue';
import { Vue, Options } from 'vue-class-component';

@Options({
  components: { GltfComponent },
})
export default class PageIndex extends Vue {}
</script>
