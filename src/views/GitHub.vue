<template>
    <div>
        <v-container v-if="isLoading">
            <v-layout align-center justify-center>
                        <v-progress-circular
                                :rotate="360"
                                :size="400"
                                :width="150"
                                :value="donePercent"
                                color="teal"
                        >
                            {{ donePercent }}
                        </v-progress-circular>
            </v-layout>
        </v-container>
        <template v-else>
            <Tab v-bind:tabTitles=tabs v-bind:toolbarTitle="title"></Tab>
        </template>
    </div>
</template>

<script>
    import Tab from "@/components/Tab";
    import DataTable from "@/components/DataTable";
    import {HEADERS1} from "../constants";
    import GithubCodeQualityByFile from "@/tabs/GithubCodeQualityByFile";
    import GithubCodeQualityByAuthor from "@/tabs/GithubCodeQualityByAuthor";
    import GitHubDetails from "@/tabs/GitHubDetails";

    export default {
        name: "Github",
        components: {Tab},
        data() {
            return {
                isLoading: true,
                interval: {},
                donePercent: 0,
                titlePrefix: "Github Statistics",
                title: null,
                resources: {},
                slug: this.$route.params.slug,
                tabKeys: {
                    commits: 'Commits', codeQualityByFiles: 'Code Quality By Files', details: 'Details',
                    codeQualityByAuthors: 'Code Quality By Authors'
                },
                tabs: {}
            }
        },
        methods: {
            getCommitsAPI: function () {
                let hack = this.slug.split('/');
                return process.env.VUE_APP_GITHUB_BASE + process.env.VUE_APP_GITHUB_COMMIT_VIEW
                    + hack[1];
            },
            getAllFilesComplexityAPI: function () {
                return process.env.VUE_APP_GITHUB_BASE + process.env.VUE_APP_GITHUB_COMPLEXITY_ALL_FILES
                    + this.slug;
            },
            getDetailsAPI: function () {
                let hack = this.slug.split('/');
                return process.env.VUE_APP_GITHUB_BASE + process.env.VUE_APP_GITHUB_LIST_REPO_VIEW
                    + hack[1];
            },
            getAllAuthorsComplexityAPI: function () {
                return process.env.VUE_APP_GITHUB_BASE + process.env.VUE_APP_GITHUB_COMPLEXITY_ALL_AUTHORS
                    + this.slug;
            },
            prepareCommitsDataForRender: function (data) {
                if (data === null) {
                    return [];
                } else {
                    return data;
                }
            },
            prepareFilesComplexityDataForRender: function (data) {
                const LEAD = "fileName";
                const SKIP_KEY = "repositoryName";
                let renderDataObj = {};
                renderDataObj['data'] = [];
                if (data === null) {
                    return [];
                } else {
                    data.forEach(function (node, nodeIndex) {
                        delete node[SKIP_KEY];
                        let renderData = {};
                        for (let key in node) {
                            if (node.hasOwnProperty(key)) {
                                renderData[key] = node[key];
                                if (key === LEAD) {
                                    renderData['leadValue'] = key;
                                }
                            }
                        }
                        renderDataObj['data'].push(renderData);
                    });
                }
                return renderDataObj;
            },
            prepareDetailsDataForRender: function (data) {
                const EXTRACT_KEY = "collaborators";
                if (data === null) {
                    return [];
                } else {
                    data = data[EXTRACT_KEY];
                    return data;
                }
            },
            prepareAuthorsComplexityDataForRender: function (data) {
                const LEAD = "author";
                const SKIP_KEY = "repositoryName";
                let renderDataObj = {};
                renderDataObj['data'] = [];
                if (data === null) {
                    return renderDataObj;
                } else {
                    data.forEach(function (node, nodeIndex) {
                        delete node[SKIP_KEY];
                        let renderData = {};
                        for (let key in node) {
                            if (node.hasOwnProperty(key)) {
                                renderData[key] = node[key];
                                if (key === LEAD) {
                                    renderData['leadValue'] = key;
                                }
                            }
                        }
                        renderDataObj['data'].push(renderData);
                    });
                }
                return renderDataObj;
            },
            fillTabData: function (data, component, headers, title, leadColumn) {
                return {data, component, headers, title, leadColumn};
            },
            addToProgress: function () {
                return 100.0 / Object.keys(this.tabKeys).length;
            },
        }
        ,
        created() {
        }
        ,
        mounted() {
            // Form Page Title
            this.title = `${this.titlePrefix} - ${this.slug}`;

            // Initialize resources
            this.resources[this.tabKeys['commits']] = this.$resource(this.getCommitsAPI());
            this.resources[this.tabKeys['codeQualityByFiles']] = this.$resource(this.getAllFilesComplexityAPI());
            this.resources[this.tabKeys['codeQualityByAuthors']] = this.$resource(this.getAllAuthorsComplexityAPI());
            this.resources[this.tabKeys['details']] = this.$resource(this.getDetailsAPI());
            // Progress Bar
            this.interval = setInterval(() => {
                if (this.donePercent >= 100.0) {
                    this.isLoading = false;
                    return (this.donePercent = 0.0)
                }
            }, 1000);

            // Get GitHub data
            this.resources[this.tabKeys['commits']].get({}).then(response => {
                this.tabs[this.tabKeys['commits']] = this.fillTabData(
                    this.prepareCommitsDataForRender(response.body), DataTable, HEADERS1, null);
            }).catch((error) => {
                this.fillTabData(
                    this.prepareCommitsDataForRender(null), DataTable, HEADERS1, null);
                console.log(error);
            }).finally(() => {
                this.donePercent += this.addToProgress();
            });
            this.resources[this.tabKeys['codeQualityByFiles']].get({}).then(response => {
                const COMPLEXITY_KEY = "metric";
                let auxData = this.prepareFilesComplexityDataForRender(response.body);
                let tableData = [];
                let headers = [];
                let leadValue = '';
                auxData.data.forEach(function (node, _index) {
                    let tmp = {};
                    leadValue = node.leadValue;
                    tmp[leadValue] = node[leadValue];
                    delete node.leadValue; // pop key
                    for (let key in node[COMPLEXITY_KEY]) {
                        if (node[COMPLEXITY_KEY].hasOwnProperty(key)) {
                            tmp[key] = node[COMPLEXITY_KEY][key];
                        }
                    }
                    delete node[COMPLEXITY_KEY]; // pop key
                    tableData.push(tmp);
                });
                Object.keys(tableData[0]).forEach(function (key) {
                    headers.push({
                        text: key.charAt(0).toUpperCase() + key.slice(1),
                        value: key
                    });
                });
                this.tabs[this.tabKeys['codeQualityByFiles']] = this.fillTabData(tableData
                    , GithubCodeQualityByFile, headers, "By Files", leadValue);
            }).catch((error) => {
                this.fillTabData(
                    this.prepareFilesComplexityDataForRender(null), GithubCodeQualityByFile, null, null);
                console.log(error)
            }).finally(() => {
                this.donePercent += this.addToProgress();
            });
            this.resources[this.tabKeys['codeQualityByAuthors']].get({}).then(response => {
                const COMPLEXITY_KEY = "metric";
                let auxData = this.prepareAuthorsComplexityDataForRender(response.body);
                let tableData = [];
                let headers = [];
                let leadValue = '';
                auxData.data.forEach(function (node, _index) {
                    let tmp = {};
                    leadValue = node.leadValue;
                    tmp[leadValue] = node[leadValue];
                    delete node.leadValue; // pop key
                    for (let key in node[COMPLEXITY_KEY]) {
                        if (node[COMPLEXITY_KEY].hasOwnProperty(key)) {
                            tmp[key] = node[COMPLEXITY_KEY][key];
                        }
                    }
                    delete node[COMPLEXITY_KEY]; // pop key
                    tableData.push(tmp);
                });
                Object.keys(tableData[0]).forEach(function (key) {
                    headers.push({
                        text: key.charAt(0).toUpperCase() + key.slice(1),
                        value: key
                    });
                });
                this.tabs[this.tabKeys['codeQualityByAuthors']] = this.fillTabData(tableData
                    , GithubCodeQualityByAuthor, headers, "By Authors", leadValue);
            }).catch((error) => {
                this.fillTabData(
                    this.prepareAuthorsComplexityDataForRender(null), GithubCodeQualityByAuthor, null, null);
                console.log(error)
            }).finally(() => {
                this.donePercent += this.addToProgress();
            });
            this.resources[this.tabKeys['details']].get({}).then(response => {
                this.tabs[this.tabKeys['details']] = this.fillTabData(
                    this.prepareDetailsDataForRender(response.body), GitHubDetails,
                    [{text: 'Member Name', value: 'name'}], "Repository Details", null);
            }).catch((error) => {
                this.fillTabData(
                    this.prepareDetailsDataForRender(null), GitHubDetails,
                    [{text: 'Member Name', value: 'name'}], "Repository Details", null);
                console.log(error)
            }).finally(() => {
                this.donePercent += this.addToProgress();
            });
        },
        beforeDestroy: function () {
            clearInterval(this.interval);
        }
    }

</script>
