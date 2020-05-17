<template>
<div>
  <!--media图标-->
  <el-row>
    <h3><i class="el-icon-picture"></i>Media</h3>
  </el-row>
  <!--按钮栏的导航-->
  <el-row>
    <div id="row-nav1">
      <el-upload
      action="http://localhost/api/media/upload"
      multiple
      :data="{upload_path: currentFolder}"
      :on-success="afterUpload"
      ref="upload">
        <el-button type="primary" icon="el-icon-upload">Upload</el-button>
      </el-upload><!--文件上传组件-->
    </div>
    <el-row id="row-nav2">
      <el-button-group>
        <el-button type="primary" size="medium" icon="el-icon-folder-add" @click="handleFolderAdd">Add Folder</el-button>
      </el-button-group>
      <el-button-group id="button-group">
        <el-button size="medium" icon="el-icon-refresh-right" @click="getFiles"></el-button>
      </el-button-group>
      <el-button-group id="button-group">
        <el-button size="medium" icon="el-icon-remove" @click="handleFilesMove" :disabled="files.length ? false : true">Move</el-button>
        <el-button size="medium" icon="el-icon-delete-solid" @click="handleFilesDelete" :disabled="files.length ? false : true">Delete</el-button>
      </el-button-group>
    </el-row>
  </el-row>
  <!--面包屑导航，当前目录-->
  <el-row id="row-bread">
    <el-breadcrumb separator-class="el-icon-arrow-right" class="breadcrumb">
      <el-breadcrumb-item @click.native="setCurrentPath(-1)" :class="'breadcrumb-item '+ (getCurrentPath().length ? 'not-current-folder' : 'current-folder')"><span class="breadcrumb-item-content">Media Library</span></el-breadcrumb-item>
      <el-breadcrumb-item v-for="(folder, i) in getCurrentPath()" :key="folder" @click.native="setCurrentPath(i)" :class="'breadcrumb-item '+ (getCurrentPath().length === i+1 ? 'current-folder' : 'not-current-folder')"><span class="breadcrumb-item-content">{{folder}}</span></el-breadcrumb-item>
    </el-breadcrumb>
  </el-row>
  <!--文件浏览窗口-->
  <div class="flex" v-loading="is_loading">
    <div id="left">
      <ul id="files">
        <el-card v-for="file in files" :key="file.name" @click.native="selectFile(file, $event)" @dblclick.native="openFile(file)" :class="'file ' + (isFileSelected(file) ? 'selected' : '')" :body-style="'padding:10px'" shadow="hover"> <!--body-style样式很重要。复制粘贴el-card元素就能看到效果-->
          <div v-if="fileIs(file, 'image')" class="image-div"> <!--本来想去掉这一层直接用el-image，然后把el-image的高度设置为70px，但是不知道为什么下方的文字的div的上方总是与el-image的下方隔着4px的高度-->
            <el-image :src="file.path" class="image" fit="cover" :preview-src-list="[file.path]"/>
          </div>
          <div v-else-if="fileIs(file, 'folder')" class="icon-div">
            <el-icon class="icon el-icon-folder"></el-icon>
          </div>
          <div v-else-if="fileIs(file, 'video')" class="icon-div"><!--从这个开始的2个图标需要改-->
            <el-icon class="icon el-video"></el-icon>
          </div>
          <div v-else-if="fileIs(file, 'audio')" class="icon-div">
            <el-icon class="icon el-audior"></el-icon>
          </div>
          <div v-else-if="fileIs(file, 'zip')" class="icon-div">
            <i class="mdi mdi-folder-zip-outline icon el-icon-zip"/>
          </div>
          <div v-else class="icon-div">
            <i class="mdi mdi-file-outline icon el-icon-file"/>
          </div>
          <div class="details">
            <h4>{{file.name}}</h4>
            <small v-if="!fileIs(file, 'folder')">
              <span>{{bytesToSize(file.size)}}</span>
            </small>
          </div>
        </el-card>
      </ul>
      <div id="no_files" v-if="files.length == 0">
        <h3><i class="mdi mdi-face el-icon-face"></i> 这个文件夹中没有文件。</h3>
      </div>
    </div>
    <!--文件预览窗口-->
    <div id="right">
      <div v-if="selectedFiles.length > 1" class="right-details right_none_selected">
        <i class="mdi mdi-format-list-bulleted icon el-icon-list"/>
        <p>{{ selectedFiles.length }} 选定的文件/文件夹</p>
      </div>

      <div v-else-if="selectedFiles.length == 1" class="right-details">
        <div class="detail-img">
          <el-image v-if="fileIs(selectedFile, 'image')" :src='selectedFile.path' fit='contain'></el-image>
          <i v-else-if="fileIs(selectedFile, 'zip')" class="mdi mdi-folder-zip-outline icon el-icon-zip"/><!--没有处理音频和视频的情况，后期要加上-->
          <el-icon v-else-if="fileIs(selectedFile, 'folder')" class="icon el-icon-folder"></el-icon>
          <i v-else class="mdi mdi-file-outline icon el-icon-file"/>
        </div>
        <div class="detail_info">
          <span>
            <h4>标题:</h4>
            <el-input v-model="modals.rename.new_filename" @change="renameFile"></el-input>
          </span>
          <span>
            <h4>类型:</h4>
            <p>{{selectedFile.type}}</p>
          </span>
          <template v-if="!fileIs(selectedFile, 'folder')">
            <span>
              <h4>大小:</h4>
              <p><span class="selected_file_size">{{bytesToSize(selectedFile.size)}}</span></p>
            </span>
            <span>
              <h4>公共URL:</h4>
              <p><a :href="selectedFile.path" target="_blank">点击这里</a></p>
            </span>
            <span>
              <h4>最近一次更改:</h4>
              <p>{{dateFilter(selectedFile.last_modified)}}</p>
            </span>
          </template>
        </div>  
      </div>

      <div v-else class="right-details right_none_selected">
        <i class="mdi mdi-cursor-default-outline icon el-icon-cursor"/><!--这个图标也要改，google icons-->
        <p>没有选择文件或文件夹</p>
      </div>
    </div>
  </div>
  <!--本应该有图片查看功能，借用elementUI el-image组件自带的图片查看功能，所以减少了工作-->
  <!--新建文件夹的模态框-->
  <el-dialog :visible.sync="addFolderFormVisible" class="new-folder">
    <template v-slot:title>
      <h4 class="head">
        <el-icon class="el-icon-folder-add"></el-icon>
        Add New Folder
      </h4> 
    </template>
    <el-input v-model="modals.add_folder.name" placeholder="请输入文件夹名称"></el-input>
    <template v-slot:footer>
      <span>
        <el-button @click="addFolderFormVisible=false">
          取消
        </el-button>
        <el-button @click="addFolder" type="primary">
          确定
        </el-button>
      </span>
    </template>
  </el-dialog>
  <!--删除文件的模态框-->
  <el-dialog :visible.sync="deleteFilesFormVisible" class="delete-file">
    <template v-slot:title>
      <h2 class="head">
        <el-icon class="el-icon-warning"></el-icon>
        Are you sure
      </h2>      
    </template>
    <h4>Are you sure you want to delete the following file(s)?</h4>
    <template v-slot:footer>
      <span>
        <el-button @click="deleteFilesFormVisible=false">
          取消
        </el-button>
        <el-button @click="deleteFiles" type="danger">
          确定
        </el-button>
      </span>
    </template>
  </el-dialog>
  <!--移动文件的模态框-->
  <el-dialog :visible.sync="moveFilesFormVisible" class="move-file">
    <template v-slot:title>
      <h2 class="head">
        <el-icon class="el-icon-remove"></el-icon>
        Move File/Folder
      </h2>      
    </template>
    <h4>Destination Folder</h4>
    <el-select v-model="modals.move_files.destination" clearable placeholder="请输入目标文件夹" class="destination-folder-select">
      <el-option v-if="currentFolder!=basePath" label="../" value="/../"></el-option>
      <template v-for="file in files">
        <el-option :key="file.name" v-if="fileIs(file, 'folder')&&!selectedFiles.includes(file)" :label="file.name" :value="currentFolder+'/'+file.name"></el-option>
      </template>
    </el-select>
    <template v-slot:footer>
      <span>
        <el-button @click="moveFilesFormVisible=false">
          取消
        </el-button>
        <el-button @click="moveFiles" type="warning">
          确定
        </el-button>
      </span>
    </template>
  </el-dialog>
</div>
</template>

<script>
import Vue from 'vue'
import qs from 'qs'

export default {
  name: "MediaManager",
  props: {
    basePath: {
      type: String,
      default() {
        return ''
      }
    }
  },
  data() {
    return {
      addFolderFormVisible: false,
      deleteFilesFormVisible: false,
      moveFilesFormVisible: false,
      currentFolder: this.basePath,
      selectedFiles: [],
      files: [],
      is_loading: true,
      modals: {
        add_folder: {
          name: ''
        },
        move_files: {
          destination: ''
        },
        rename: {
          new_filename: ''
        }
      }
    }
  },
  mounted() {
    this.getFiles()
  },
  computed: { 
    selectedFile() {
      return this.selectedFiles[0]
    }
  },
  watch: {
    'selectedFile': function() {
        this.modals.rename.new_filename = this.selectedFile && this.selectedFile.name || ''
    }
  },
  methods: {
    getFiles() {
      let vm = this
      this.is_loading = true
      this.axios({
        url: 'api/media/files',
        method: 'post',
        data: {
          folder: this.currentFolder
        }
      }).then(response => {
        vm.files = []
        response.data.forEach(file => vm.files.push(file))
        vm.selectedFiles=[]
        if(vm.files.length > 0)
          vm.selectedFiles.push(vm.files[0])
        vm.is_loading=false
      })
    },
    selectFile(file, e) {
      if (!e.ctrlKey && !e.metaKey && !e.shiftKey) {
        this.selectedFiles = [];
      }      
      if (e.shiftKey && this.selectedFiles.length == 1) {
        let index = null;
        let start = 0;
        for (let i = 0; i < this.files.length; i++) {
            if (this.files[i] === this.selectedFile) {
                start = i;
                break;
            }
        }

        let end = 0;
        for (let i = 0; i < this.files.length; i++) {
            if (this.files[i] === file) {
                end = i;
                break;
            }
        }

        for (var i = start; i < end; i++) {
            index = this.selectedFiles.indexOf(this.files[i]);
            if (index === -1) {
                this.selectedFiles.push(this.files[i]);
            }
        }
      }

      let index = this.selectedFiles.indexOf(file);
      if (index === -1)
        this.selectedFiles.push(file);
      
      if(this.selectedFiles.length == 1) {
        let vm = this
        Vue.nextTick(() => {
          if (vm.fileIs(vm.selectedFile, 'video'))
            vm.$refs.videoplayer.load()//处理视频和音频，之后研究
          else if (vm.fileIs(vm.selectedFile, 'audio'))
            vm.$refs.audioplayer.load()
        })
      }
    },
    openFile(file) {
      if(file.type == 'folder') {
        this.currentFolder += ('/'+file.name)
        this.getFiles()
      }
    },
    isFileSelected(file) {
      return this.selectedFiles.includes(file)
    },
    fileIs(file, type) {
      if(typeof file === 'string') {
        if(type === 'image')
          return this.endsWithAny(['jpg', 'jpeg', 'png', 'bmp'], file)
      } else
        return file.type.includes(type)
    },
    getCurrentPath() {
      return this.currentFolder.replace(this.basePath, '').split('/').filter(value => value != '')
    },
    setCurrentPath(i) {
      if(i == -1)
        this.currentFolder = this.basePath
      else {
        let path = this.getCurrentPath()
        path.length = i + 1
        this.currentFolder = this.basePath + path.join('/')
      }

      this.getFiles()
    },
    endsWithAny(suffixes, string) {
      return suffixes.some(function (suffix) {
        return string.endsWith(suffix);
      });
    },
    handleFolderAdd() {
      this.addFolderFormVisible = true
      this.modals.add_folder.name = ''
    },
    addFolder() {
      let vm = this
      if(this.modals.add_folder.name == '')
        return
      this.axios({
        url: 'api/media/new_folder',
        method: 'post',
        data: {
          new_folder: this.currentFolder + '/' + this.modals.add_folder.name
        }
      }).then(response => {
        if(response.data.success == true) {
          vm.$notify({
            title: 'Success',
            message: this.modals.add_folder.name + '创建成功！',
            type: 'success'
          })
          vm.getFiles()
        } else {
          vm.$notify({
            title: 'Error',
            message: response.data.error,
            type: 'error'
          })
        }
        this.addFolderFormVisible = false
      })
    },
    handleFilesDelete() {
      this.deleteFilesFormVisible = true
    },
    deleteFiles() {
      let vm = this
      this.axios({
        url: 'api/media/delete_files',
        method: 'post',
        data: qs.stringify({
          path: this.currentFolder,
          files: this.selectedFiles,
        }, {
          arrayFormat: 'indices'
        })
      }).then(response => {
        if(response.data.success == true) {
          vm.$notify({
            title: 'Succcess',
            message: '文件删除成功！',
            type: 'success'
          })
          vm.getFiles()
        } else {
          vm.$notify({
            title: 'Error',
            message: response.data.error,
            type: 'error'
          })
        }
        vm.deleteFilesFormVisible = false
      })
    },
    handleFilesMove() {
      this.moveFilesFormVisible = true
      this.modals.move_files.destination = ''
    },
    moveFiles() {
      let vm = this
      if(this.modals.move_files.destination == '')
        return
      this.axios({
        url: 'api/media/move_files',
        method: 'post',
        data: qs.stringify({
          path: this.currentFolder,
          destination: this.modals.move_files.destination,
          files: this.selectedFiles
        }, {
          arrayFormat: 'indices'
        })
      }).then(response => {
        if(response.data.success == true) {
          vm.$notify({
            title: 'Succcess',
            message: '文件移动成功！',
            type: 'success'
          })
          vm.getFiles()
        } else {
          vm.$notify({
            title: 'Error',
            message: response.data.error,
            type: 'error'
          })
        }
        vm.moveFilesFormVisible = false
      })
    },
    renameFile() {
      let vm = this
      if(this.selectedFile.name == this.modals.rename.new_filename)
        return
      this.axios({
        url: 'api/media/rename_file',
        method: 'post',
        data: {
          folder_location: this.currentFolder,
          filename: this.selectedFile.name,
          new_filename: this.modals.rename.new_filename
        }
      }).then(response => {
        if(response.data.success == true) {
          vm.$notify({
            title: 'Success',
            message: '重命名成功！',
            type: 'success'
          })
          this.getFiles()
        } else
          vm.$notify({
            title: 'Error',
            message: response.data.error,
            type: 'error'
          })
      })
    },
    bytesToSize: function(bytes) {
				let sizes = ['Bytes', 'KB', 'MB', 'GB', 'TB']
				if (bytes == 0) return '0 Bytes'
				let i = parseInt(Math.floor(Math.log(bytes) / Math.log(1024)))
				return Math.round(bytes / Math.pow(1024, i), 2) + ' ' + sizes[i]
    },
    dateFilter: function(date) {
      if (!date)
        return null
      let newDate = new Date(date * 1000)

      let month = "0" + (newDate.getMonth() + 1)
      let minutes = "0" + newDate.getMinutes()
      let seconds = "0" + newDate.getSeconds()

      let dateFormated = newDate.getFullYear() + '-' + month.substr(-2) + '-' + newDate.getDate() + ' ' + newDate.getHours() + ':' + minutes.substr(-2) + ':' + seconds.substr(-2)

      return dateFormated
    },
    afterUpload(response, file) {
      if(response.success == true) { //后端返回什么，这个里就是什么，与axios不同。
        this.$notify({
          title: 'Success',
          message: file.name + '上传成功！',
          type: 'success'
        })
        this.getFiles()
      } else {
        this.$notify({
          title: 'Error',
          message: file.name + '上传失败！' + 'response.data.error',
          type: 'error'
        })
      }
      const index = this.$refs.upload.$data.uploadFiles.findIndex(value => value.name == file.name)
      this.$refs.upload.$data.uploadFiles.splice(index, 1)
    }
  }
}
</script>>

<style scoped>
#button-group {
  margin-left: 10px
}

#row-nav1 {
  padding: 20px 20px 10px;
  background: #DCDFE6;
  text-align: center
}

#row-nav2 {
  padding: 10px 20px 20px;
  background: #DCDFE6
}

/*面包屑导航*/
#row-bread {
  padding: 5px 20px;
  background: #EBEEF5
}

#row-bread .breadcrumb {
  font-size: 9px;
}

#row-bread .breadcrumb .breadcrumb-item .breadcrumb-item-content {
  cursor: pointer
}

#row-bread .breadcrumb .breadcrumb-item.not-current-folder .breadcrumb-item-content {
  color: #4DA7E8
}

#row-bread .breadcrumb .breadcrumb-item.not-current-folder .breadcrumb-item-content:hover {
  color: #606266
}

/*文件浏览窗口样式*/
.flex {
  border: 1px solid #E0E0E0;   
  border-top: 0px;

  display: flex;/*这两行必须，控制了div内部可以换行*/
  flex-wrap: wrap;
}

.flex #left{
  flex: 4;/*这行必须，跟上面的flex的display呼应*/
  position: relative;

  min-height: 500px; /*控制着大框的最小的高度*/
  width: 100%;
  user-select: none; /*用户在大框内不能选中*/
}

.flex #left #files {/*依然是必不可少的，控制el-card的高度不填充满整个父元素 */
  margin:0px;
  display: flex; /*跟上面的display同理*/
  flex-wrap: wrap;
  padding: 10px;
}

.flex #left #files .file {
  flex:1; /*跟上面的flex同理*/
  margin:10px;

  width:100%;
  min-width:120px;/*很关键，控制着每个文件框的最小和最大宽度*/
  max-width:150px;
  cursor:pointer
}

.flex #left #files .file .icon-div { /*控制icon水平居中，且尺寸和上层的div大小一样，因为没找到让icon垂直居中的方式 */
  text-align: center;
  height: 70px
}

.flex #left #files .file .icon-div .icon {
  font-size: 70px;

  padding: 0px /*这句不加也行，但是若icon没有填满div就必须加上合适的padding值*/
}

.flex #left #files .file .image-div {/*控制image外层的div高度固定，不然下方的字和别的card不一样高*/
  height: 70px;
}

.flex #left #files .file .image-div .image {/*控制el-image的div占满上层的div，不然image可能不居中*/
  height: 100%;
}

.flex #left #files .file .details {
  text-align:center;
}

.flex #left #files .file .details h4 {
  margin-top: 0px;
  margin-bottom: 2px;

  font-size:14px;
  overflow: hidden;
  font-weight: 500;

}

.flex #left #files .file .details small {
  font-size: 11px;
  position: relative;
  top: -3px;
  color: #aaa
}


.flex #left #files .file.selected, .flex #left #files .file.selected:hover {
  background: linear-gradient(to top,rgba(193,219,252,1),rgba(220,235,252,1))!important;
  border-color: rgba(125, 162, 206, 1)
}

.flex #left #files .file:hover
{
  background: linear-gradient(to top,rgba(235,243,253,1),rgba(250,251,253,1))!important;
  border-color: rgba(184, 214, 251, 1)
}

.flex #left #no_files {
  margin-top: 200px;
  margin-bottom:75px;

  color:#949494;

  text-align: center
}

/*文件预览窗口*/
.flex #right{
  flex:1;
  border-left:1px solid #f1f1f1;
}

.flex #right .right-details {
  display:block
}

.flex #right .right-details .el-image {
  /*width: 100%  加上这句过小的图片会放大，失真，但会布满屏幕*/
}

.flex #right .right-details .icon {
  display: block; /*这句必须，阴影布满整个div*/
  font-size: 70px;
  background: #f9f9f9;
  width: 100%;/*有这行就必须有下一行*/
  box-sizing: border-box;
  text-align: center;

  padding: 50px /*这句也必须，上下居中需要*/
}

.flex #right .right-details.right_none_selected p {
  margin: 0px; /*这行必须，默认的p的margin样式不是这个*/

  text-align: center;
  color: #bbb;
  border-bottom: 1px solid #f1f1f1;
  text-align: center;

  padding:10px
}

.flex #right .right-details .detail_info {
  padding: 10px
}

.flex #right .right-details .detail_info span {
  display:block;
  clear:both
}

.flex #right .right-details .detail_info h4 {
  margin: 3px 8px 0 0;

  float:left;/*float css样式的使用能使两行变一行，具体原理不清楚*/
  color:#bbb;
  font-size:12px;
  font-weight:400;

  padding-bottom:2px
}

.flex #right .right-details .detail_info p {
  margin: 3px 0 10px;

  float:left;
  color:#444;
  font-size:12px;
  font-weight:400;

  padding-bottom:3px
}

.flex #right .right-details .detail_info a {
  color:#4DA7E8
}

.flex #right .right-details .detail_info .selected_file_size {
  padding-top:0
}

/*新建文件的模态框*/
.new-folder .head {
  margin: 0 0 -10px;/*这个技巧很重要，可以忽略父元素的padding*/
  color: #409EFF
}

/*删除文件的模态框*/
.delete-file .head {
  margin: 0 0 -10px;
  color: #F56C6C
}

/*move文件的模态框*/
.move-file .head {
  margin: 0 0 -10px;
  color: #E6A23C
}

.move-file .destination-folder-select {
  width: 100%
}
</style>>
