<template>
    <v-card flat tile min-height="380" class="d-flex flex-column">
        <confirm ref="confirm"></confirm>
        <v-card-text
            v-if="!path"
            class="grow d-flex justify-center align-center grey--text"
            >Select a folder or a file</v-card-text>
        <v-card-text v-else-if="isFile" class="grow d-flex justify-center align-center">
            <v-list subheader v-if="metadata.length">
                <v-subheader>Metadata</v-subheader>
                <v-menu
                    v-model="newMetadataPopper"
                    :close-on-content-click="false"
                    :nudge-width="200"
                    offset-y
                    >
                    <template v-slot:activator="{ on, attrs }">
                        <v-btn v-if="path" icon v-bind="attrs" v-on="on" title="Create Metadata">
                            <v-icon>mdi-plus-circle</v-icon>
                        </v-btn>
                    </template>
                    <v-card>
                        <v-card-title>Create Metadata</v-card-title>
                        <v-card-text>
                            <v-form ref="mkMetaForm" lazy-validation v-model="validMkMetaForm">
                                <v-text-field :rules="fieldRules" required label="Name" v-model="newMetadataName" hide-details></v-text-field>
                                <v-text-field :rules="fieldRules" required label="Value" v-model="newMetadataValue" hide-details></v-text-field>
                                <v-text-field label="Unit" v-model="newMetadataUnit" hide-details></v-text-field>
                            </v-form>
                        </v-card-text>
                        <v-card-actions>
                            <div class="flex-grow-1"></div>
                            <v-btn @click="newMetadataPopper = false" depressed>Cancel</v-btn>
                            <v-btn
                                color="success"
                                :disabled="!validMkMetaForm"
                                depressed
                                @click="mkMetadata"
                                >Create Metadata</v-btn>
                        </v-card-actions>
                    </v-card>
                </v-menu>
                <v-list-item
                    v-for="item, index in metadata"
                    :key="index"
                    class="pl-0"
                    v-bind="attrs"
                    v-on="on"
                    >
                    <v-list-item-avatar class="ma-0">
                        <v-icon>{{ icons['other'] }}</v-icon>
                    </v-list-item-avatar>
                    <v-list-item-content class="py-2">
                        <v-list-item-title v-text="item.attname"></v-list-item-title>
                        <v-list-item-subtitle>{{ item.attvalue }}</v-list-item-subtitle>
                    </v-list-item-content>
                    <v-list-item-action>
                        <v-btn icon @click.stop="deleteItem(item)">
                            <v-icon color="grey lighten-1">mdi-delete-outline</v-icon>
                        </v-btn>
                        <v-btn icon v-if="false">
                            <v-icon color="grey lighten-1">mdi-information</v-icon>
                        </v-btn>
                    </v-list-item-action>
                </v-list-item>
            </v-list>

        </v-card-text>
        <v-card-text v-else-if="dirs.length || files.length" class="grow">
            <v-list subheader v-if="dirs.length">
                <v-subheader>Folders</v-subheader>
                <v-list-item
                    v-for="item in dirs"
                    :key="item.basename"
                    @click="changePath(item.path)"
                    class="pl-0"
                    >
                    <v-list-item-avatar class="ma-0">
                        <v-icon>mdi-folder-outline</v-icon>
                    </v-list-item-avatar>
                    <v-list-item-content class="py-2">
                        <v-list-item-title v-text="item.basename"></v-list-item-title>
                    </v-list-item-content>
                    <v-list-item-action>
                        <v-btn icon @click.stop="deleteItem(item)">
                            <v-icon color="grey lighten-1">mdi-delete-outline</v-icon>
                        </v-btn>
                        <v-btn icon v-if="false">
                            <v-icon color="grey lighten-1">mdi-information</v-icon>
                        </v-btn>
                    </v-list-item-action>
                </v-list-item>
            </v-list>
            <v-divider v-if="dirs.length && files.length"></v-divider>
            <v-list subheader v-if="files.length">
                <v-subheader>Files</v-subheader>
                <v-list-item
                    v-for="item in files"
                    :key="item.basename"
                    @click="changePath(item.path)"
                    class="pl-0"
                    >
                    <v-list-item-avatar class="ma-0">
                        <v-icon>{{ icons[item.extension.toLowerCase()] || icons['other'] }}</v-icon>
                    </v-list-item-avatar>

                    <v-list-item-content class="py-2">
                        <v-list-item-title v-text="item.basename"></v-list-item-title>
                        <v-list-item-subtitle>{{ formatBytes(item.size) }}</v-list-item-subtitle>
                    </v-list-item-content>

                    <v-list-item-action>
                        <v-btn icon @click.stop="deleteItem(item)">
                            <v-icon color="grey lighten-1">mdi-delete-outline</v-icon>
                        </v-btn>
                        <v-btn icon v-if="false">
                            <v-icon color="grey lighten-1">mdi-information</v-icon>
                        </v-btn>
                    </v-list-item-action>
                </v-list-item>
            </v-list>
        </v-card-text>
        <v-card-text
            v-else-if="filter"
            class="grow d-flex justify-center align-center grey--text py-5"
            >No files or folders found</v-card-text>
        <v-card-text
            v-else
            class="grow d-flex justify-center align-center grey--text py-5"
            >The folder is empty</v-card-text>
        <v-divider v-if="path"></v-divider>
        <v-toolbar v-if="false && path && isFile" dense flat class="shrink">
            <v-btn icon>
                <v-icon>mdi-download</v-icon>
            </v-btn>
        </v-toolbar>
        <v-toolbar v-if="path" dense flat class="shrink">
            <v-text-field
                solo
                flat
                hide-details
                label="Filter"
                v-model="filter"
                prepend-inner-icon="mdi-filter-outline"
                class="ml-n3"
                ></v-text-field>
            <v-btn icon v-if="false">
                <v-icon>mdi-eye-settings-outline</v-icon>
            </v-btn>
            <v-btn icon @click="load">
                <v-icon>mdi-refresh</v-icon>
            </v-btn>
        </v-toolbar>
    </v-card>
</template>

<script>
import { formatBytes } from "./util";
import Confirm from "./Confirm.vue";

export default {
    props: {
        icons: Object,
        storage: String,
        path: String,
        endpoints: Object,
        axios: Function,
        refreshPending: Boolean
    },
    components: {
        Confirm
    },
    data() {
        return {
            items: [],
            //validEditMetaForm: true,
            validMkMetaForm: true,
            //editMetadataFormOpen: false,
            filter: "",
            newMetadataName: "",
            netMetadataValue: "",
            netMetadataUnit: "",
            newMetadataPopper: false,
            fieldRules: [
                v => !!v || 'This field is required',
            ],
        };
    },
    computed: {
        dirs() {
            return this.items.filter(
                item =>
                item.type === "dir" && item.basename.includes(this.filter)
            );
        },
        files() {
            return this.items.filter(
                item =>
                item.type === "file" && item.basename.includes(this.filter)
            );
        },
        metadata() {
            return this.items.filter(
                item =>
                item.type === "metadata" && item.attname.includes(this.filter)
            );
        },
        isDir() {
            return this.path[this.path.length - 1] === "/";
        },
        isFile() {
            return !this.isDir;
        }
    },
    methods: {
        formatBytes,
        changePath(path) {
            this.$emit("path-changed", path);
        },
        async load() {
            this.$emit("loading", true);
            if (this.isDir) {
                let url = this.endpoints.list.url
                    .replace(new RegExp("{storage}", "g"), this.storage)
                    .replace(new RegExp("{path}", "g"), this.path);

                let config = {
                    url,
                    method: this.endpoints.list.method || "get"
                };

                let response = await this.axios.request(config);
                this.items = response.data;
            } else {
                let url = this.endpoints.list.url
                    .replace(new RegExp("{storage}", "g"), this.storage)
                    .replace(new RegExp("{path}", "g"), this.path);

                let config = {
                    url,
                    method: this.endpoints.list.method || "get"
                };
                let response = await this.axios.request(config);
                this.items = response.data
                // TODO: load file
            }
            this.$emit("loading", false);
        },
        async deleteItem(item) {
            let confirmed = await this.$refs.confirm.open(
                "Delete",
                `Are you sure<br>you want to delete this ${
                    item.type === "dir" ? "folder" 
                        : item.type ==="file" ? "file"
                        : "metadata item"
                }?<br><em>${item.basename}</em>`
            );

            if (confirmed) {
                if (item.type != "metadata"){
                    this.$emit("loading", true);
                    let url = this.endpoints.delete.url
                        .replace(new RegExp("{storage}", "g"), this.storage)
                        .replace(new RegExp("{path}", "g"), item.path);

                    let config = {
                        url,
                        method: this.endpoints.delete.method || "post"
                    };

                    await this.axios.request(config);
                    this.$emit("file-deleted");
                    this.$emit("loading", false);
                } else {
                    this.$emit("loading", true);
                    let url = this.endpoints.rmmetadata.url
                        .replace(new RegExp("{storage}", "g"), this.storage)
                        .replace(new RegExp("{path}", "g"), item.path);

                    let config = {
                        url,
                        method: this.endpoints.rmmetadata.method || "delete",
                        data: { 
                            'attname': item.attname,
                            'attvalue': item.attvalue,
                            'attunit': item.attunit,
                        }
                    };

                    await this.axios.request(config);
                    this.$emit("file-deleted");
                    this.$emit("loading", false);
                }
            }
        },
        async mkMetadata() {
            this.validMkMetaForm = this.$refs.mkMetaForm.validate()
            if(this.validMkMetaForm) {
                this.$emit("loading", true);
                let url = this.endpoints.mkmetadata.url
                    .replace(new RegExp("{storage}", "g"), this.storage)
                    .replace(new RegExp("{path}", "g"), this.path);

                let config = {
                    url,
                    method: this.endpoints.mkmetadata.method || "post",
                    data: { 
                        'attname': this.newMetadataName,
                        'attvalue': this.newMetadataValue,
                        'attunit': this.newMetadataUnit,
                    }
                };

                await this.axios.request(config);
                this.newMetadataPopper = false;
                this.newMetadataName = "";
                this.newMetadataValue = "";
                this.$emit("file-deleted");
                this.$emit("loading", false);
            }
        },
        /*
        //async editMetadata(index,item) {
            this.validEditMetaForm = this.$refs['editMetaForm' + index][0].validate()
            console.log(this.$refs['editMetaForm' + index][0])
            if(this.validEditMetaForm) {
                this.$emit("loading", true);
                let url = this.endpoints.editmetadata.url
                    .replace(new RegExp("{storage}", "g"), this.storage)
                    .replace(new RegExp("{path}", "g"), this.path);

                let config = {
                    url,
                    method: this.endpoints.editmetadata.method || "post"
                };

                await this.axios.request(config);
                this.$emit("metadata-created", this.newMetadataName);
                this.newMetadataPopper = false;
                this.newMetadataName = "";
                this.newMetadataValue = "";
                this.$emit("loading", false);
            }
        }
         */
    },
    watch: {
        async path() {
            this.items = [];
            await this.load();
        },
        async refreshPending() {
            if (this.refreshPending) {
                await this.load();
                this.$emit("refreshed");
            }
        }
    }
};
</script>

<style lang="scss" scoped>
.v-card {
    height: 100%;
}
</style>
