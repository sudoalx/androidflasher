<script setup>
import Swal from 'sweetalert2';
import disableVerifyVbmeta from 'vbmeta-disabler';
import { ref } from 'vue';
import { useDeviceStore } from '@/stores/devices';

import ProgressBar from '@/components/ProgressBar.vue';
import FlashLog from '@/components/FlashLog.vue';

const deviceStore = useDeviceStore();
const device = deviceStore.device;
const latestLine = ref('');
const progress = ref(0);
const data = deviceStore.flashObject.files;

async function startFlashing() {
  progress.value = 0;
  let curr_progress = -1;
  for (const item of data) {
    curr_progress++;
    if (item.slot != null) { 
      console.log("Switching slot")
      await device.runCommand(`set_active:${item.slot}`);
    }
    latestLine.value = `Flashing ${item.filename}...`;
    if (data.options?.disableVerity && item.partition == 'vbmeta') {
      const vbmeta = await disableVerifyVbmeta(item.blob);
      await device.flashBlob(item.partition, vbmeta, (t) => {
        // take progress.value every iteration and add progress to it
        progress.value = curr_progress + t;
      })
      continue;
    }
    await device.flashBlob(item.partition, item.blob, (t) => {
      console.log(t)
      // take progress.value every iteration and add progress to it
      progress.value = curr_progress + t;
    })
  }
  if (data.options?.cleanFlash) {
    latestLine.value = 'Erasing userdata...';
    await deviceStore.device.runCommand(`erase:userdata`);
  }
}

startFlashing().then(() => {
  Swal.fire({
    title: 'Flashing completed',
    text: 'Congratulations! You have successfully flashed your device!',
    icon: 'success',
    confirmButtonText: 'Reboot'
  }).then((result) => {
    if (result.isConfirmed) {
      deviceStore.device.reboot()
    }
  })
})

</script>

<template>
    <main>
      <o-icon icon="bolt" size="large" />
      <FlashLog :latest-line="latestLine" :opacity-decrease="0.3" :maxLines="6"/>
      <ProgressBar :progress="progress" :parts="data.length" style="width: 80%;"/>
    </main>
</template>

<style scoped>
main {
  background-color: #f5f5f5;
  border-radius: 25px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: space-evenly;
}
</style>

