<template>
  <button @click="toggle()" class="btn btn-neutral w-16 h-16 fixed bottom-3 right-20 rounded-full z-10">
    <font-awesome-icon :icon="['fas', 'camera']" size="2x"/>
  </button>

  <section class="off-canvas">
    <input type="checkbox" v-model="open" class="modal-toggle"/>
    <div class="modal" role="dialog">
      <div class="modal-box">
        <h3 class="text-lg font-bold mb-3">Bug Reporting</h3>
        <img v-if="screenshot" :src="screenshot" alt="">
        <input v-else type="file" ref="test123" class="input input-file text-center w-full mb-3 hidden" @change="screenshot = fileToObjectURL($event.target.files[0])" accept="image/*">
        <button v-if="screenshot" class="btn btn-primary w-full mt-3" @click="screenshot = ''">zurücksetzen</button>
        <button v-else @click="test123.click()" class="btn btn-error text-white w-full">bild schießen</button>
        <label class="floating-label mt-6">
          <span>Deine Beschreibung für den Bug.</span>
          <textarea v-model="description" class="textarea w-full mt-3"></textarea>
        </label>
        <div class="modal-action flex justify-between">
          <label @click="open = false" class="btn">{{ $t('btn.close') }}</label>
          <label @click="capture()" v-if="!screenshot" class="btn btn-primary">Bild aus Clipboard</label>
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

const test123 = useTemplateRef('test123')
const open = useLocalStorage('modal-issue-reporting', false);
const screenshot = ref('');
const pb = usePocketBase();
const description = ref('')
const entry = ref({})

function fileToObjectURL(file) {
  // file: File|Blob
  return URL.createObjectURL(file);
}

// Get the last clipboard entry. Must be called from a user gesture in many browsers.
async function getLastClipboardEntry() {
  if (!navigator.clipboard) {
    throw new Error('Clipboard API not available in this environment.');
  }

  // If read() is supported, prefer it (can return files/blobs)
  const cb = navigator.clipboard;
  if (typeof cb.read === 'function') {
    try {
      const items = await cb.read(); // returns ClipboardItem[]
      if (!items || items.length === 0) return null;

      // Pick the last item (most recently placed in the clipboard)
      const lastItem = items[items.length - 1];

      // Prefer getting a file-like/blob type (image/video/other)
      for (const type of lastItem.types) {
        try {
          const blob = await lastItem.getType(type); // Blob
          // If it's textual, also return text
          if (type.startsWith('text/')) {
            const text = await blob.text();
            return { kind: 'text', mime: type, text, blob: null, fileName: null };
          }
          // For binary (images/videos/etc) return the blob and guess filename
          const ext = type.split('/').pop() || 'bin';
          const fileName = `clipboard-${Date.now()}.${ext}`;
          const file = new File([blob], fileName, { type: blob.type || type });
          return { kind: 'file', mime: type, file, blob, text: null, fileName };
        } catch (e) {
          // ignore this type and try next
        }
      }

      // If none of the types produced a usable blob, try text fallback via readText()
    } catch (err) {
      // read() may throw if permission denied or unsupported for file contents
      // fall through to readText fallback below
      // console.warn('clipboard.read() failed:', err);
    }
  }

  // Fallback: use readText() if available
  if (typeof cb.readText === 'function') {
    try {
      const text = await cb.readText();
      if (text === '') return null;
      return { kind: 'text', mime: 'text/plain', text, blob: null, fileName: null };
    } catch (err) {
      throw new Error('Unable to read clipboard text: ' + (err && err.message ? err.message : String(err)));
    }
  }

  throw new Error('No supported clipboard read method available.');
}

const reset = ()=>{
  screenshot.value = '';
  description.value = '';
}

const send = async() => {
  await pb.collection('issue_reporting').create({
    picture: entry.value.file,
    description: description.value
  })
  reset();
}

const capture = async () => {
  entry.value = await getLastClipboardEntry();
  if(entry.value.kind === 'file'){
    screenshot.value = fileToObjectURL(entry.value.file);
  }
}

const toggle = async() => {
  entry.value = await getLastClipboardEntry();

  if(entry.value.kind === 'file'){
    screenshot.value = fileToObjectURL(entry.value.file);
  }

  open.value = !open.value;
}

onMounted(() => {
  open.value = false;
});
</script>