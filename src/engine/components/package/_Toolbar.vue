<template>
    <div>    
        <v-tooltip bottom>
            <v-btn color="primary darken-2" slot="activator" icon @click="packageDialog = !packageDialog"> 
                <v-icon dark>fa-archive</v-icon>
            </v-btn>
            <span>Package Manager</span>
        </v-tooltip>
        <v-dialog v-model="packageDialog" max-width="70%" max-height="81%" scrollable persistent>            
            <v-card>
                <v-card-title>
                    <span class="headline">Package Manager</span>                    
                    <v-spacer class="hidden-xs-only"></v-spacer>
                     <v-text-field 
                        prepend-icon="search"
                        label="Package name"
                        class="ma-0 pa-0 search-board"
                        single-line
                        clearable
                        hide-details
                        :append-outer-icon="searchText ? 'fa-chevron-circle-right' : ''"
                        v-model="searchText"></v-text-field>                    
                </v-card-title>
                <v-divider></v-divider>
                 <div ref="scrollArea" class="smooth-scrollbar">
                    <v-card-text>
                        <v-subheader>
                            Installed
                        </v-subheader>
                    <div>
                        
                    </div>

                    <v-divider></v-divider>
                    <v-subheader>
                        Online available
                    </v-subheader>

                    <div>

                        <v-list three-line>
            <template v-for="(item, index) in items">
              <v-subheader
                v-if="item.header"
                :key="item.header"
              >
                {{ item.header }}
              </v-subheader>
  
              <v-divider
                v-else-if="item.divider"
                :key="index"
                :inset="item.inset"
              ></v-divider>
  
              <v-list-tile
                v-else
                :key="item.title"
                avatar
                class="list-title"
              >
                <v-list-tile-avatar>
                  <img :src="item.avatar">
                </v-list-tile-avatar>
  
                <v-list-tile-content>
                  <v-list-tile-title v-html="item.title"></v-list-tile-title>
                  <v-list-tile-sub-title v-html="item.subtitle"></v-list-tile-sub-title>
                </v-list-tile-content>
                
                <v-list-tile-action>                  
                    <v-btn
                        icon fab small dark
                        class="primary"                                                    
                        @click="tobeinstall = '123456'; confirmInstallDialog = true"
                    >
                        <v-icon>fa-download</v-icon>
                    </v-btn>
                </v-list-tile-action>

              </v-list-tile>
              
            </template>
          </v-list>
                        
                    </div>

                    </v-card-text>
                </div>
                <v-card-actions>
                    <v-spacer></v-spacer>
                    <v-btn color="blue darken-1" flat @click.native="packageDialog = false">Close</v-btn>                    
                </v-card-actions>
            </v-card>
        </v-dialog>
        <v-dialog v-model="confirmRemoveDialog" persistent max-width="500px">
            <v-card>
                <v-card-title class="headline">Remove board confirmation</v-card-title>
                <v-card-text>Do you want to remove <strong>{{toberemove}}</strong>?</v-card-text>
                <v-card-actions>
                <v-spacer></v-spacer>
                <v-btn class="green--text darken-1" flat="flat" @click.native="confirmRemoveDialog = false">Cancel</v-btn>
                <v-btn class="green--text darken-1" flat="flat" @click.native="confirmRemoveDialog = false">OK</v-btn>
                </v-card-actions>
            </v-card>
        </v-dialog>
        <v-dialog v-model="confirmInstallDialog" persistent max-width='500px'>
            <v-card>
                <v-card-title class="headline">Install board confirmation</v-card-title>
                <v-card-text>Do you want to install <strong>{{tobeinstall}}</strong>?</v-card-text>
                <v-card-actions>
                <v-spacer></v-spacer>
                <v-btn class="green--text darken-1" flat="flat" @click.native="confirmInstallDialog = false">Cancel</v-btn>
                <v-btn class="green--text darken-1" flat="flat" @click.native="confirmInstallDialog = false;">OK</v-btn>
                </v-card-actions>
            </v-card>
        </v-dialog>
    </div>
</template>

<script>

const { shell } = require('electron');
const fs = require('fs');

import SmoothScrollbar from 'smooth-scrollbar'
const pg = require('smooth-scrollbar/plugins/overscroll');

let scrollbar;

import VWidget from '@/engine/views/VWidget';
import pm from '@/engine/PackageManager';
import util from '@/engine/utils';

export default {
    components: {
        VWidget
    },
    data () {
        return {    
            items: [
            ],
            packageDialog : false,
            confirmRemoveDialog : false,
            confirmInstallDialog : false,
            searchText : '', 
            scrollSettings: {
                alwaysShowTracks: false,
                plugins: {
                    overscroll : { 
                        enable: true,
                        effect: 'glow',
                        damping: 0.1,
                        maxOverscroll: 150,
                        glowColor: '#222a2d'
                    }
                },
            },
            isInstalling : false,

            installedPackage : null,//bm.boards().map(obj=>{ obj.status =  'READY'; return obj;}),
            localPackage : null,//bm.boards().map(obj=>{ obj.status =  'READY'; return obj;}),
            onlinePackageStatus : 'wait',
            onlinePackagePage : 0,
            onlinePackages : [],
            tobeinstall : '',
            toberemove : '',
            statusText : '',
            statusProgress : 0,

            scrollbar: null,
        }
    },
    methods:{        
        getPackageByName(name){
            return this.installedPackage.find(obj => { return obj.name == name});
        },
        getOnlinePackageByName(name){
            return this.onlinePackages.find(obj=>{ return obj.name == name});
        },
        openLink(url){
            shell.openExternal(url);
        },
        isOnline()
        {
            return window.navigator.onLine;
        },       
    },
    mounted(){
        let option = {
            damping: 0.1,
            thumbMinSize: 20,
            renderByPixels: true,
            alwaysShowTracks: false,
            continuousScrolling: true,
            delegateTo: null,
            plugins: {}
        }
        scrollbar = SmoothScrollbar.init(
            this.$refs.scrollArea,
            Object.assign({},{}, option, this.scrollSettings)
        )
        this.scrollbar = scrollbar;
    },
    destroyed() {
      scrollbar.destroy()
      scrollbar = null
      this.scrollbar = null
    },
    watch : {
        packageDialog : function(val){
            if(val){//on opening
                //this.listOnlineBoard();
            }
        }
    }
}
</script>
<style>
.list-title {
    background-color: white !important;
}
.list-title:hover {
    background: #EEE !important;
}

.smooth-scrollbar {
  width: 100%;
  height: 100%;
}
.search-board {
    width: 40px;
    margin-bottom: -10px !important;
}
</style>
