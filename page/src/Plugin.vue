<template>
  <div>
    <h2 class="mb-4">SR-IOV i915 Driver</h2>
    <v-skeleton-loader v-if="loading" :loading="true" type="card, card" />
    <v-card v-else-if="!sriovSupported" class="mb-4 pa-0">
      <v-card-title>SR-IOV Status</v-card-title>
      <v-card-text class="pa-4">
        <v-alert type="warning" variant="tonal">
          SR-IOV is not active or not supported on this system (you have to reboot once after the first installation).
        </v-alert>
      </v-card-text>
    </v-card>
    <div v-else style="margin-bottom: 80px">
      <v-card class="mb-4 pa-0">
        <v-card-title>VFS Configuration</v-card-title>
        <v-card-text class="pa-4">
          <v-select
            v-model="selectedVfs"
            :items="vfsOptions"
            label="Number of Virtual Functions"
            variant="outlined"
            density="comfortable"
            hide-details
          />
        </v-card-text>
        <v-divider />
        <v-card-actions class="pa-4">
          <v-spacer />
          <v-btn color="onPrimary" :loading="updating" @click="updateVfs">
            Update
          </v-btn>
        </v-card-actions>
      </v-card>
      <v-card class="mb-4 pa-0">
        <v-card-title>Installed Driver</v-card-title>
        <v-card-text class="pa-4">
          <div
            v-if="!driverInfo || !driverInfo.package"
            class="text-center text-grey py-4"
          >
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
    <v-snackbar v-model="snackbar.show" :color="snackbar.color" timeout="6000">
      {{ snackbar.text }}
    </v-snackbar>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';

const PLUGIN_NAME = 'sriov-driver';

const loading = ref(true);
const updating = ref(false);
const sriovSupported = ref(false);

const driverInfo = ref(null);
const settings = ref({ vfs_number: 1 });

const selectedVfs = ref(1);
const vfsOptions = [0, 1, 2, 3, 4, 5, 6, 7];

const initError = ref(null);

const snackbar = ref({ show: false, text: '', color: 'success' });
const showSnackbarError = (text, errorText = '') => {
  const message = errorText ? `${text}: ${errorText}` : text;
  snackbar.value = { show: true, text: message, color: 'error' };
};
const showSnackbarSuccess = (text) => {
  snackbar.value = { show: true, text, color: 'success' };
};

const getAuthHeaders = () => ({
  Authorization: 'Bearer ' + localStorage.getItem('authToken'),
});

const fetchDriverInfo = async () => {
  try {
    const res = await fetch(`/api/v1/mos/plugins/driver/${PLUGIN_NAME}`, {
      headers: getAuthHeaders(),
    });

    if (res.ok) driverInfo.value = await res.json();
    else driverInfo.value = null;
  } catch (e) {
    console.error('Failed to fetch driver info:', e);
    driverInfo.value = null;
  }
};

const fetchSettings = async () => {
  try {
    const res = await fetch(`/api/v1/mos/plugins/settings/${PLUGIN_NAME}`, {
      headers: getAuthHeaders(),
    });

    if (res.ok) {
      const data = await res.json();
      settings.value = data;
      selectedVfs.value = data.vfs_number ?? 1;
    }
  } catch (e) {
    console.error('Failed to fetch settings:', e);
  }
};

const checkSriovSupport = async () => {
  try {
    const res = await fetch('/api/v1/mos/plugins/query', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: 'sriov',
        args: ['check'],
        timeout: 10,
        parse_json: true,
      }),
    });

    if (!res.ok) {
      sriovSupported.value = false;
      return;
    }

    const data = await res.json();
    sriovSupported.value = data.success === true;
  } catch (e) {
    console.error('Failed to check SR-IOV support:', e);
    sriovSupported.value = false;
  }
};

const updateVfs = async () => {
  updating.value = true;
  try {
    const queryRes = await fetch('/api/v1/mos/plugins/query', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: 'sriov',
        args: ['set_vfs', selectedVfs.value],
        timeout: 10,
        parse_json: true,
      }),
    });

    if (!queryRes.ok) {
      showSnackbarError(`Failed to execute command (HTTP ${queryRes.status})`);
      return;
    }

    const queryData = await queryRes.json();

    if (!queryData.success) {
      const msg =
        queryData.output?.trim() ||
        `SR-IOV command failed (exit code ${queryData.exit_code ?? '-'})`;
      showSnackbarError(msg);
      return;
    }

    const settingsRes = await fetch(`/api/v1/mos/plugins/settings/${PLUGIN_NAME}`, {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        vfs_number: selectedVfs.value,
      }),
    });

    if (!settingsRes.ok) {
      showSnackbarError(`Failed to save settings (HTTP ${settingsRes.status})`);
      return;
    }

    settings.value.vfs_number = selectedVfs.value;
    await Promise.all([fetchDriverInfo(), fetchSettings()]);
    showSnackbarSuccess('VFs updated successfully');
  } catch (e) {
    console.error('Failed to update VFS:', e);
    showSnackbarError('Update failed', String(e));
  } finally {
    updating.value = false;
  }
};

onMounted(async () => {
  loading.value = true;
  initError.value = null;

  try {
    await checkSriovSupport();

    if (sriovSupported.value) {
      await Promise.all([fetchDriverInfo(), fetchSettings()]);
    } else {
      driverInfo.value = null;
    }
  } catch (e) {
    console.error('Failed to initialize:', e);
    initError.value = e;
    showSnackbarError('Initialization failed', String(e));
  } finally {
    loading.value = false;
  }
});
</script>
