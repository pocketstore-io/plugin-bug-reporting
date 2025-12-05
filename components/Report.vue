<template>
  <button class="btn btn-primary w-16 h-16 rounded-full">Test</button>

  <section class="off-canvas">
    <input type="checkbox" v-model="open" class="modal-toggle"/>
    <div class="modal" role="dialog">
      <div class="modal-box">
        <h3 class="text-lg font-bold">Hello!</h3>
        <p class="py-4">This modal works with a hidden checkbox!</p>
        <div class="modal-action">
          <label for="my_modal_6" class="btn">Close!</label>
        </div>
      </div>
    </div>
  </section>
</template>
<script setup lang="ts">
const open = useLocalStorage('modal-bug-reporting', false);

onMounted(()=>{
  open.value = false;
});
</script>