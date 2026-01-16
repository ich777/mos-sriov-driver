<template>
  <div>
    <h2 class="mb-4">SR-IOV i915 Driver</h2>

    <v-skeleton-loader
      v-if="loading"
      :loading="true"
      type="card"
    />

    <v-card v-else class="pa-0">
      <v-card-title>Installed Driver</v-card-title>

      <v-card-text class="pa-4">
        <div v-if="!driverInfo || !driverInfo.package" class="text-center text-grey py-4">
          No driver installed
        </div>

        <v-row v-else dense>
          <v-col cols="6" md="3">
            <div class="text-caption text-medium-emphasis">
              <strong>Plugin</strong>
            </div>
            <div class="text-body-2">
              {{ driverInfo.plugin || '-' }}
            </div>
          </v-col>

          <v-col cols="6" md="3">
            <div class="text-caption text-medium-emphasis">
              <strong>Kernel</strong>
            </div>
            <div class="text-body-2">
              {{ driverInfo.kernel || '-' }}
            </div>
          </v-col>

          <v-col cols="12" md="6">
            <div class="text-caption text-medium-emphasis">
              <strong>Package</strong>
            </div>
            <div class="text-body-2" style="word-break: break-all">
              {{ driverInfo.package || '-' }}
            </div>
          </v-col>
        </v-row>
      </v-card-text>
    </v-card>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';

const PLUGIN_NAME = 'sriov-driver';

const loading = ref(true);
const driverInfo = ref(null);

const getAuthHeaders = () => ({
  Authorization: 'Bearer ' + localStorage.getItem('authToken'),
});

const fetchDriverInfo = async () => {
  try {
    const res = await fetch(
      `/api/v1/mos/plugins/driver/${PLUGIN_NAME}`,
      {
        headers: getAuthHeaders(),
      }
    );

    if (res.ok) {
      driverInfo.value = await res.json();
    } else {
      driverInfo.value = null;
    }
  } catch (e) {
    console.error('Failed to fetch driver info:', e);
    driverInfo.value = null;
  }
};

onMounted(async () => {
  await fetchDriverInfo();
  loading.value = false;
});
</script>
