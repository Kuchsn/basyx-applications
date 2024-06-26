<template>
    <v-dialog max-width="700px" height="auto" v-model="editDialog">
        <!-- Edit Button -->
        <template v-slot:activator="{ props: activatorProps }">
            <v-btn v-bind="activatorProps" icon="mdi-pencil" variant="plain" size="small"></v-btn>
        </template>
        <template v-slot:default="{ isActive }">
            <v-card>
                <v-row align="center" class="py-2 px-3">
                    <v-col>
                        <v-card-title>Edit</v-card-title>
                    </v-col>
                    <v-col cols="auto">
                        <v-text-field v-model="elementOrder" label="Order" type="number" density="compact" variant="outlined" style="width: 100px" hide-details></v-text-field>
                    </v-col>
                </v-row>
                <v-divider></v-divider>
                <v-card-text style="max-height: 800px; overflow-y: auto">
                    <v-text-field v-model="localTitle" label="Title" variant="outlined" class="px-4 pb-3" density="compact" hide-details="auto" clearable></v-text-field>
                    <!-- Combobox for selecting the group or adding a new group -->
                    <v-combobox v-model="elementGroup" v-model:search="searchGroup" :items="groups" item-title="groupName" item-value="groupId" :hide-no-data="false" label="Group" variant="outlined" class="px-4 pb-3" density="compact" hide-details="auto" @keyup.enter="createGroup($event.target.value)">
                        <template v-slot:no-data>
                            <v-list-item>
                                <v-list-item-title>
                                    No group matching "<strong>{{ searchGroup }}</strong>". Press
                                    <kbd>enter</kbd> to create a new one
                                </v-list-item-title>
                            </v-list-item>
                        </template>
                    </v-combobox>
                    <template v-if="aasData && Object.keys(aasData).length > 0">
                        <TimeSeriesData v-if="checkSemanticId('https://admin-shell.io/idta/TimeSeries/1/1')" class="py-2" :submodelElementData="aasData" :configData="dashboardData" :editDialog="true" @newOptions="setNewOptions"></TimeSeriesData>
                    </template>
                </v-card-text>
                <v-divider></v-divider>
                <v-card-actions>
                    <v-btn text="Close" @click="isActive.value = false"></v-btn>
                    <v-spacer></v-spacer>
                    <v-btn @click="saveConfig()" text="Save" color="primary" class="text-buttonText" variant="flat"></v-btn>
                </v-card-actions>
            </v-card>
        </template>
    </v-dialog>
</template>
<script lang="ts">
    import { defineComponent } from 'vue';
    import { useEnvStore } from '@/store/EnvironmentStore';
    import DashboardHandling from '@/mixins/DashboardHandling';
    import TimeSeriesData from '../SubmodelPlugins/TimeSeriesData.vue';

    export default defineComponent({
    name: 'DashboardEditElement',
    components: {
        TimeSeriesData,
        DashboardHandling,
    },
    mixins: [DashboardHandling],
    props: ['aasData', 'dashboardData'],

    setup() {
        const envStore = useEnvStore();

        return {
            envStore, // EnvironmentStore Object
        };
    },

    data() {
        return {
            editDialog: false,
            groups: [] as Array<any>,
            searchGroup: '',
            localTitle: "",
            elementGroup: {} as any,
            elementOrder: Number,
            newChartOptions: {} as any,
        }
    },

    watch: {
        editDialog() {
            if (this.editDialog) {
                // console.log('Dashboard Data', this.dashboardData);
                this.getAvailableGroups();
                this.localTitle = this.dashboardData.title;
                this.elementOrder = this.dashboardData.order;
                this.elementGroup = { ...this.dashboardData.group };
                // Set the current chart options to the new chart options
                this.newChartOptions = { ...this.dashboardData.configObject };
            }
        }
    },

    methods: {
        checkSemanticId(semanticId: string): boolean {
            // console.log(this.dashboardData)
            return this.dashboardData.configObject.semanticId === semanticId;
        },

        async getAvailableGroups() {
            this.groups = await this.getGroups();
        },

        async saveConfig() {
            let newData = { ...this.dashboardData };
            newData.title = this.localTitle;
            newData.group = this.elementGroup;
            newData.order = this.elementOrder;
            newData.configObject = this.newChartOptions;
            // console.log('Save updated Element: ', newData);
            let updateddElement = await this.updateElement(newData);
            // console.log('Updated Element: ', updateddElement);
            if (updateddElement) this.editDialog = false;
            this.$emit('updateElement', updateddElement);
        },

        createGroup(event: string) {
            // console.log('Create Group: ', event);
            const newGroupName = event;
            const newGroupId = this.UUID();
            this.groups.push({ groupName: newGroupName, groupId: newGroupId });
            this.elementGroup = { groupName: newGroupName, groupId: newGroupId };
        },            

        setNewOptions(data: any) {
            // console.log('New Options: ', data, this.newChartOptions);
            this.newChartOptions = { ...this.newChartOptions, ...data };
        },
    }
})
</script>