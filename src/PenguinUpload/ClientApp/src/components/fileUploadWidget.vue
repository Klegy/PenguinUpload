<template>
  <div class="file-upload-widget">
    <div class="container">
      <v-layout row>
        <v-flex xs12 lg10 offset-lg1>
          <div>
            <v-btn primary @click.native="viewMyFiles">View my files</v-btn>
          </div>
          <div class="center">
            <div class="upload-here" @drop.stop.prevent="handleDragDropUpload" @dragenter.stop.prevent @dragleave.stop.prevent @dragover.stop.prevent>
              <!--<a class="target">Drag and drop or click to upload files</a>-->
              <h5>Upload Files</h5>
              <v-layout class="upload-settings-section" row>
                <v-flex xs10 offset-xs1 md6 offset-md3>
                  <v-text-field
                    label="Upload Destination"
                    v-model="uploadDestination"
                  ></v-text-field>
                </v-flex>
              </v-layout>
              <p>Drag and drop or click</p>
              <div class="upload-area-padding" @click="browseForFiles">
              </div>
            </div>
          </div>
        </v-flex>
      </v-layout>
      <v-layout row>
        <v-flex xs12 lg10 offset-lg1>
          <div class="upload-progress-indicators">
              <!--<md-spinner md-size="60" :md-progress="progressIndicator.value" class="md-warn"></md-spinner>
                  <p>{{ progressMessage }}</p>-->
              <v-list two-line subheader>
                <!--Uploading file-->
                <v-subheader v-if="progressIndicators.length > 0">Uploading</v-subheader>
                <v-list-item :key="ix" v-for="(item, ix) in progressIndicators">
                  <v-list-tile avatar>
                    <v-list-tile-avatar>
                      <v-icon>cloud_queue</v-icon>
                    </v-list-tile-avatar>
                    <v-list-tile-content>
                      <v-list-tile-title>{{ item.name }}</v-list-tile-title>
                      <template v-if="!item.error">
                        <v-list-tile-sub-title>{{ (item.value < 100) ? `Uploading... (${item.value}%)` : 'Uploaded, Processing...' }}</v-list-tile-sub-title>
                      </template>
                      <template v-else>
                        <v-list-tile-sub-title>{{ 'Upload error: ' + item.message }}</v-list-tile-sub-title>
                      </template>
                    </v-list-tile-content>
                    <v-list-tile-action>
                      <v-btn icon ripple @click.native="cancelUpload(item)">
                        <v-icon class="grey--text text--lighten-1">cancel</v-icon>
                      </v-btn>
                    </v-list-tile-action>
                  </v-list-tile>
                  <v-divider inset></v-divider>
                </v-list-item>
                <!--Upload completed files-->
                <v-subheader v-if="completedFiles.length > 0">Completed</v-subheader>
                <v-list-item :key="ix" v-for="(item, ix) in completedFiles" @click="visitUrl(item.downloadPage)">
                  <v-list-tile avatar>
                    <v-list-tile-avatar>
                      <v-icon>cloud_done</v-icon>
                    </v-list-tile-avatar>
                    <v-list-tile-content>
                      <v-list-tile-title>{{ item.name }}</v-list-tile-title>
                      <v-list-tile-sub-title>Upload Complete!</v-list-tile-sub-title>
                    </v-list-tile-content>
                    <v-list-tile-action>
                      <v-btn icon ripple>
                        <v-icon class="grey--text text--lighten-1">done</v-icon>
                      </v-btn>
                    </v-list-tile-action>
                  </v-list-tile>
                  <v-divider inset></v-divider>
                </v-list-item>
              </v-list>
              <v-btn v-if="completedFiles.length > 0" @click.native="completedFiles = []">Clear All</v-btn>
            </div>
        </v-flex>
      </v-layout>
    </div>
    <input type="file" class="invisible" ref="browse" @change="onFilesUploaded" multiple />
  </div>
</template>
<script>
  import axios from 'axios'

  let axiosRequestConfig = {
    validateStatus: function (status) {
      return status >= 200 && status < 500
    }
  }

  export default {
    data() {
      return {
        progressIndicators: [],
        /* schema:
        {
          value: number [0-100],
          fileRef: object [reference to file that is uploading],
          name: string [name of file],
          xhr: object [xhr object reference],
          error: bool
          message: string or null [an error message]
        }
        */
        completedFiles: [],
        /* schema:
         */
        uploadDestination: '/'
      }
    },
    computed: {
      uploadingFiles: function () {
        return this.progressIndicators.length
      },
      uploading: function () {
        return this.uploadingFiles > 0
      }
    },
    methods: {
      viewMyFiles: function () {
        this.$router.push('/f')
      },
      browseForFiles: function () {
        this.$refs.browse.click()
      },
      onFilesUploaded: function (e) {
        let browse = this.$refs.browse
        let fileCount = browse.files.length
        for (let i = 0; i < fileCount; i++) {
          let f = browse.files[i]
          let progress = {
            value: 0,
            fileRef: f,
            name: f.name
          }
          this.progressIndicators.push(progress)
          this.uploadFile(f, progress)
        }
      },
      handleDragDropUpload: function (e) {
        for (var i = 0; i < e.dataTransfer.files.length; i++) {
          var f = e.dataTransfer.files[i];
          let progress = {
            value: 0,
            fileRef: f,
            name: f.name,
            error: false,
            message: ''
          }
          this.progressIndicators.push(progress)
          this.uploadFile(f, progress)
        }
      },
      cancelUpload: function (progress) {
        let vm = this
        progress.xhr.abort()
        vm.progressIndicators.splice(vm.progressIndicators.indexOf(progress), 1)
      },
      uploadFile: function (file, progress) {
        let vm = this
        let xhr = new XMLHttpRequest()
        progress.xhr = xhr
        xhr.open("POST", "/api/upload")
        xhr.onload = function () {
          if (xhr.status === 200) {
            // upload complete
            // console.log('upload complete', progress.name)
            // dequeue the uploading file
            vm.progressIndicators.splice(vm.progressIndicators.indexOf(progress), 1)
            // console.log(xhr.responseText)
            let response = JSON.parse(xhr.responseText)
            vm.completedFiles.push({
              name: progress.name,
              downloadPage: '/#/d/' + response.fileId
              // downloadPage: response.downloadPage // get download page from server response
            })
          } else {
            // update progress indicator
            progress.error = true
            // progress.message = xhr.responseText
            // reactive update
            vm.$set(progress, 'message', xhr.responseText)
          }
        }
        xhr.upload.onprogress = function (e) {
          if (e.lengthComputable) {
            progress.value = Math.floor((e.loaded / e.total) * 100)
          }
        }
        let form = new FormData()
        form.append("apikey", this.$store.getters.auth_data.key)
        form.append("dir", vm.uploadDestination)
        form.append("file", file)
        xhr.send(form)
      },
      visitUrl: function (u) {
        window.open(u, '_blank')
      }
    }
  }
</script>
<style scoped>
  .upload-area-padding {
    margin: 20px;
    padding: 14%;
    background: #f7f7f7;
  }
  
  .upload-progress-indicators {
    text-align: center;
  }
  
  .upload-progress-bar {
    padding: 5px;
  }
  
  .upload-settings-section {
    text-align: center;
  }
</style>