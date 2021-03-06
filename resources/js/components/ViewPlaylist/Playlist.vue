<template>
    <v-container v-if="!isLoading" v-show="p5Loaded">
        <div v-show="aTag">
            <a ref="exportButton" :href="exportHref" :download="exportTitle"></a>
        </div>
        <v-row justify="center">
            <v-col cols="12" md="6">
                <div class="d-flex justify-end">
                    <v-btn
                            v-show="!exporting"
                            class="export-button black--text rounded-xl"
                            color="#ffffff"
                            large
                            @click="exportPlaylist"
                    >
                        Export
                    </v-btn>
                </div>

                <h1 class="text-center white--text playlist-title"> {{ playlist.info.playlistTitle }}</h1>
                <v-card>

                    <v-list two-line>
                        <template v-for="(track, index) in playlist.tracks">
                            <v-row class="track-container">
                                <v-col>
                                    <v-subheader class="track-name">
                                        {{ track.name }}
                                    </v-subheader>
                                    <v-list-item>
                                        {{ track.artist }}
                                    </v-list-item>
                                </v-col>
                                <v-col>
                                    <div class="d-flex justify-center" :id="index" :ref="'canvas' + index"></div>
                                </v-col>
                            </v-row>
                            <v-divider></v-divider>
                        </template>
                    </v-list>

                </v-card>
            </v-col>
        </v-row>
    </v-container>
</template>

<script>
    import Loading from '@/js/components/ViewPlaylist/Loading'

    export default {
        props: [
            'playlist',
            'analysis',
            'isLoading',
        ],

        data() {
            return {
                aTag: null,
                exportHref: null,
                exportTitle: null,
                p5Loaded: false,
                exporting: false
            }
        },

        computed: {
            playlistTracks() {

            }
        },
        watch: {
            analysis() {
                const playlistLength = Object.keys(this.playlist.tracks).length;
                let script = [];
                let number = [];

                for (var i = 0; i < playlistLength; i++) {
                    script[i] = function (sketch) {
                        let angle = 0;
                        sketch.setup = _ => {
                            sketch.angleMode(sketch.DEGREES);
                            sketch.createCanvas(100, 100)
                        }
                        sketch.draw = _ => {
                            sketch.noLoop()
                            let value = sketch.analysis['danceability'] * 100;
                            let max = 100;
                            let colour = (value/max) * 255;
                            sketch.background(255, 255, 255);
                            angle = sketch.map((sketch.analysis['danceability']) * 100, 0, 100, 0, 180) - 180;
                            sketch.stroke(0, colour, 0);
                            sketch.strokeWeight(2.7);
                            sketch.translate(50, sketch.height);
                            sketch.branch(40);
                        }

                        sketch.branch = (len) => {
                            sketch.line(0, 0, 0, -len);
                            sketch.translate(0, -len);
                            if (len > 4) {
                                sketch.push();
                                sketch.rotate(angle);
                                sketch.branch(len * 0.6);
                                sketch.pop();
                                sketch.push();
                                sketch.rotate(-angle);
                                sketch.branch(len * 0.6);
                                sketch.pop();
                            }
                        }
                    }

                    const p5 = require('p5');
                    number[i] = new p5(script[i], i.toString())
                    number[i].analysis = this.analysis[i];
                }

                this.p5Loaded = true;
            }
        },

        methods: {
            exportPlaylist() {
                this.exporting = true;
                this.$emit('isExporting', this.exporting);
                const playlistId = this.$route.params.playlistId;
                let canvases = [];
                let dataUrls = [];

                for (let ref in this.$refs) {
                    if ((ref).match(/canvas/)) {
                        canvases.push(this.$refs[ref][0].childNodes[0]);
                    }
                }

                canvases.forEach(function (item, index) {
                    dataUrls.push(item.toDataURL('image/png'));
                })

                window.axios.post('/api/playlists/' + playlistId + '/export', {
                    headers: {
                        contentType: 'application/pdf'
                    },
                    dataUrls: dataUrls,
                }, {
                    responseType: 'arraybuffer'
                }).then(response => {
                    let blob = new Blob([response.data], {type: 'application/pdf'})
                    this.aTag = true;
                    this.exportHref = window.URL.createObjectURL(blob);
                    this.exportTitle = this.playlist.info.playlistTitle;
                    const exportButton = this.$refs.exportButton;
                    setTimeout(() => {
                        this.exporting = false;
                        this.$emit('isExporting', this.exporting)
                        exportButton.click();
                    }, 2000);
                })
            },
        },
        components: {
            Loading
        }
    }
</script>

<style scoped>
    .playlist-title {
        padding: 20px;
        background-color: #707372;
    }

    .export-button {
        margin-bottom: 10px;
    }

    /* TODO Do this in a Vue way */
    @media (max-width: 425px) {
        .track-name {
            height: 100px;
        }
        .track-container {
            align-items: center;
        }
    }
</style>
