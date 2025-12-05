<template>
  <button @click="toggle()" class="btn btn-neutral w-16 h-16 fixed bottom-3 right-20 rounded-full z-10">
    <font-awesome-icon :icon="['fas', 'bell']" size="2x"/>
  </button>

  <section class="off-canvas">
    <input type="checkbox" v-model="open" class="modal-toggle"/>
    <div class="modal" role="dialog">
      <div class="modal-box">
        <h3 class="text-lg font-bold mb-3">Bug Reporting</h3>
        <img :src="screenshot" alt="">
        <label class="floating-label mt-3">
          <span>Deine Beschreibung für den Bug.</span>
          <textarea v-model="description" class="textarea w-full mt-3"></textarea>
        </label>
        <div class="modal-action flex justify-between">
          <label @click="open = false" class="btn">{{ $t('btn.close') }}</label>
          <label @click="send()" class="btn btn-primary">Bericht abschicken</label>
        </div>
      </div>
      <label class="modal-backdrop" @click="open = false">Close</label>
    </div>
  </section>
</template>
<script setup lang="ts">
import {useLocalStorage} from "@vueuse/core";
import {FontAwesomeIcon} from "@fortawesome/vue-fontawesome";
import {usePocketBase} from "@/utils/pocketbase";

const open = useLocalStorage('modal-bug-reporting', false);
const screenshot = ref(null);
const pb = usePocketBase();
const description = ref('')

function dataUrlToFile(dataUrl, filename = "file", fallbackMime = "application/octet-stream") {
  // If input contains the data: prefix, split header and base64 payload
  const hasPrefix = dataUrl.startsWith("data:");
  let mime;
  let base64;

  if (hasPrefix) {
    const parts = dataUrl.split(",");
    if (parts.length !== 2) throw new Error("Invalid data URL");
    base64 = parts[1];
    const mimeMatch = parts[0].match(/data:([^;]+);base64/);
    mime = mimeMatch ? mimeMatch[1] : fallbackMime;
  } else {
    // raw base64 (no prefix)
    base64 = dataUrl;
    mime = fallbackMime;
  }

  const binary = atob(base64);
  const len = binary.length;
  const u8 = new Uint8Array(len);
  for (let i = 0; i < len; i++) {
    u8[i] = binary.charCodeAt(i);
  }

  return new File([u8], filename, { type: mime });
}

const captureScreenshot = async () => {
  try {
    const stream = await navigator.mediaDevices.getDisplayMedia({
      video: {mediaSource: "screen"}
    });

    const video = document.createElement("video");
    video.srcObject = stream;

    await video.play();

    const canvas = document.createElement("canvas");
    canvas.width = video.videoWidth;
    canvas.height = video.videoHeight;

    const ctx = canvas.getContext("2d");
    ctx.drawImage(video, 0, 0, canvas.width, canvas.height);

    // Stop screen sharing
    stream.getTracks().forEach(track => track.stop());

    screenshot.value = canvas.toDataURL("image/png");
  } catch (err) {
    console.error("Screenshot failed:", err);
  }
};

const toggle = () => {
  captureScreenshot();

  open.value = !open.value;
}

const send = async () => {
  await pb.collection('bug_reports').create({
    description: description.value,
    picture: new File([dataUrlToFile(screenshot.value)], 'screenshot.png')
  })
  open.value = false;
}

onMounted(() => {
  open.value = false;
});
</script>