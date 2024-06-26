<template>
    <v-card v-if="localDashboardData" height="100%">
        <v-row align="center">
            <v-col cols="auto" class="pr-0">
                <v-card-title>{{ localDashboardData.title }}</v-card-title>
            </v-col>
            <v-col v-if="AASData?.idShort" cols="auto" class="pl-0">
                <v-chip size="x-small" color="primary" label border>{{ 'AAS: ' + AASData?.idShort }}</v-chip>
            </v-col>
            <v-spacer></v-spacer>
            <v-col cols="auto" density="compact" justify="end" class="py-0">
                <!-- Edit Button -->
                <DashboardEditElement :aasData="AASData" :dashboardData="localDashboardData" :chartData="newChartOptions" @updateElement="updateDashboardElement"></DashboardEditElement>
                <!-- Sync Button -->
                <v-btn v-if="isLinkedSegment" icon variant="plain" size="small" @click="changeSync()">
                    <span :class="syncStatus ? 'custom-loader' : ''">
                        <v-icon light>mdi-cached</v-icon>
                    </span>
                </v-btn>
                <!-- Visibility Button -->
                <v-btn @click="updateVisibility()" :icon="localDashboardData.visibility ? 'mdi-eye' : 'mdi-eye-off' " variant="plain" size="small"></v-btn>
                <!-- Delete Button -->
                <v-btn @click="deleteDialog = true" icon="mdi-delete" variant="plain" size="small"></v-btn>
            </v-col>
        </v-row>
        <template v-if="AASData && Object.keys(AASData).length > 0">
            <TimeSeriesData v-if="checkSemanticId('https://admin-shell.io/idta/TimeSeries/1/1')" :submodelElementData="AASData" :configData="localDashboardData" :editDialog="false" :loadTrigger="trigger"></TimeSeriesData>
        </template>
    </v-card>
    <!-- Dialog for deleting a dashboard element -->
    <v-dialog v-model="deleteDialog" width="450px">
        <v-card>
            <v-card-title>{{ 'Are you sure?' }}</v-card-title>
            <v-divider></v-divider>
            <v-card-actions>
                <v-spacer></v-spacer>
                <v-btn @click="deleteDialog = false">{{ 'cancel' }}</v-btn>
                <v-btn variant="flat" class="text-buttonText" color="error" @click="deleteElement(localDashboardData.id)">{{ 'delete' }}</v-btn>
            </v-card-actions>
        </v-card>
    </v-dialog>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import { useEnvStore } from '@/store/EnvironmentStore';
import DashboardHandling from '@/mixins/DashboardHandling';
import TimeSeriesData from '../SubmodelPlugins/TimeSeriesData.vue';
import DashboardEditElement from '../Dashboard/DashboardEditElement.vue';

export default defineComponent({
    name: 'DashboardElement',
    components: {
        DashboardHandling,
        TimeSeriesData,
        DashboardEditElement,
    },
    mixins: [DashboardHandling],
    props: ['dashboardData', 'globalSyncStatus'],
    emits: ['deleteElement', 'updateElement'],

    setup() {
        const envStore = useEnvStore();

        return {
            envStore, // EnvironmentStore Object
        };
    },

    data() {
        return {
            localDashboardData: {} as any,
            AASData: {} as any,
            isClicked: false,
            deleteDialog: false,
            newChartOptions: {} as any,
            trigger: false,
            syncStatus: false,
            timeout: null as number | null,
        }
    },

    mounted() {
        // console.log('Dashboard Element Data from MongoDB: ', this.dashboardData);
        this.localDashboardData = { ...this.dashboardData };
        this.fetchAASData();
    },

    beforeUnmount() {
        if (this.timeout) {
            window.clearTimeout(this.timeout);
            this.timeout = null;
        }
    },

    watch: {
        globalSyncStatus() {
            if (this.globalSyncStatus) {
                this.requestData();
                this.syncStatus = true;
            } else {
                this.syncStatus = false;
                if (this.timeout) {
                    console.log('Clear Timeout')
                    window.clearTimeout(this.timeout);
                    this.timeout = null;
                }
            }
        }
    },

    computed: {
        isLinkedSegment() {
            if (this.localDashboardData.configObject?.segment?.semanticId?.keys?.[0]?.value === 'https://admin-shell.io/idta/TimeSeries/Segments/LinkedSegment/1/1') {
                return true;
            }
            return false;
        }
    },

    methods: {
        fetchAASData() {
            let path = this.localDashboardData.endpoint;
            let context = 'retrieving dashboard element data';
            let disableMessage = true;
            this.getRequest(path, context, disableMessage).then((response: any) => {
                if (response.success) { // execute if the Request was successful
                    // console.log(response.data)
                    response.data.path = this.localDashboardData.endpoint; // add the path to the SubmodelElement Data
                    this.AASData = response.data;
                } else { // execute if the Request failed
                    console.error(response);
                }
            });

        },
        // Function to check if the SemanticID of a SubmodelElement matches the given SemanticID
        checkSemanticId(semanticId: string): boolean {
            return this.localDashboardData.configObject.semanticId === semanticId;
        },

        updateVisibility() {
            this.localDashboardData.visibility = !this.localDashboardData.visibility;
            this.$emit("updateElement", this.localDashboardData);
            // console.log(data)
            this.updateElement(this.localDashboardData);
        },

        async deleteElement(elementId: any) {
            // console.log(elementId)
            let deletedId =  await this.deleteSingle(elementId)
            if(deletedId) {
                this.$emit("deleteElement", deletedId);
            }
            this.deleteDialog = false;
        },

        updateDashboardElement(element: any) {
            console.log('Updated Element: ', element, this.localDashboardData);
            if (element) {
                // check if the element moved to another group
                // console.log(element.group.groupId, this.localDashboardData.group.groupId)
                if (element.group.groupId !== this.localDashboardData.group.groupId) {
                    this.$emit("deleteElement", this.localDashboardData.id);
                }
                // update the localDashboardData with the new Element
                this.localDashboardData = element;
                // trigger rerender of the TimeSeriesData Component
                this.trigger = !this.trigger;
            }
        },

        changeSync() {
            this.syncStatus = !this.syncStatus;
            if (this.syncStatus) {
                this.requestData();
            } else {
                if (this.timeout) {
                    window.clearTimeout(this.timeout);
                    this.timeout = null;
                }
            }
        },

        requestData() {
            // create a timeout to toggle the trigger every 5 seconds
            this.timeout = window.setTimeout(() => {
                if (this.syncStatus) {
                    this.trigger = !this.trigger;
                    this.requestData();
                }
            }, 5000);
        },
    },
})
</script>

<style>
.custom-loader {
    animation: loader 1s infinite;
    display: flex;
}

@-moz-keyframes loader {
    from {
        transform: rotate(0);
    }

    to {
        transform: rotate(360deg);
    }
}

@-webkit-keyframes loader {
    from {
        transform: rotate(0);
    }

    to {
        transform: rotate(360deg);
    }
}

@-o-keyframes loader {
    from {
        transform: rotate(0);
    }

    to {
        transform: rotate(360deg);
    }
}

@keyframes loader {
    from {
        transform: rotate(0);
    }

    to {
        transform: rotate(360deg);
    }
}
</style>