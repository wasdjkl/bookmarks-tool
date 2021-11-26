<template>
  <div class="item">
    <div class="id">
      {{ source }}<br />
      <img :src="source.icon" alt="favicon" />{{ source.title }} <br />
      {{ source.url }}<br />
      {{ source.folder }} - {{ source.addDate }} <br />
    </div>
  </div>
</template>

<script>
export default {
  name: "item",
  props: {
    source: {
      type: Object,
      default() {
        return {};
      },
    },
  },
};
</script>

<style scoped>
.item {
  display: flex;
  flex-direction: column;
  border-bottom: 1px solid lightgrey;
  padding: 1em;
}
</style>
