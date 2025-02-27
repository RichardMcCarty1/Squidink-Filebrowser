
<template>
  <div class="item"
  role="button"
  tabindex="0"
  :draggable="isDraggable"
  @dragstart="dragStart"
  @dragover="dragOver"
  @drop="drop"
  @click="itemClick"
  @dblclick="dblclick"
  @touchstart="touchstart"
  :data-dir="isDir"
  :aria-label="name"
  :title="name"
  :aria-selected="isSelected">
    <div>
      <ContextMenu ref="menu">
        <template slot-scope="{ contextData }">
          <ContextMenuItem @click.native="sendToCont(contextData)" >
            {{shouldDisplaySquid(contextData)}}
          </ContextMenuItem>
        </template>
      </ContextMenu>
    <div>
      <img v-if="type==='image' && isThumbsEnabled && !isSharing" v-lazy="thumbnailUrl">
      <i v-else class="material-icons">{{ icon }}</i>
    </div>
    <div>
      <p class="name">{{ name }}</p>

      <p v-if="isDir" class="size" data-order="-1">&mdash;</p>
      <p v-else class="size" :data-order="humanSize()">{{ humanSize() }}</p>

      <p class="modified">
        <time :datetime="modified">{{ humanTime() }}</time>
      </p>
    </div>
    </div>
  </div>
</template>

<script>
import { baseURL, enableThumbs } from '@/utils/constants'
import store from '@/store'
import { mapMutations, mapGetters, mapState } from 'vuex'
import filesize from 'filesize'
import moment from 'moment'
import { files as api } from '@/api'
import * as upload  from '@/utils/upload'
import ContextMenu from './ContextMenu';
import ContextMenuItem from './ContextMenuItem';
const axios = require('axios');
export default {
  name: 'item',
  components: {
      ContextMenu,
      ContextMenuItem,
    },
  data: function () {
    return {
      touches: 0
    }
  },
  props: ['name', 'isDir', 'url', 'type', 'size', 'modified', 'index'],
  computed: {
    ...mapState(['user', 'selected', 'req', 'jwt']),
    ...mapGetters(['selectedCount', 'isSharing']),
    singleClick () {
      if (this.isSharing) return false
      return this.user.singleClick
    },
    isSelected () {
      return (this.selected.indexOf(this.index) !== -1)
    },
    icon () {
      if (this.isDir) return 'folder'
      if (this.type === 'image') return 'insert_photo'
      if (this.type === 'audio') return 'volume_up'
      if (this.type === 'video') return 'movie'
      return 'insert_drive_file'
    },
    isDraggable () {
      return !this.isSharing && this.user.perm.rename
    },
    canDrop () {
      if (!this.isDir || this.isSharing) return false

      for (let i of this.selected) {
        if (this.req.items[i].url === this.url) {
          return false
        }
      }

      return true
    },
    thumbnailUrl () {
      const path = this.url.replace(/^\/files\//, '')
      return `${baseURL}/api/preview/thumb/${path}?auth=${this.jwt}&inline=true`
    },
    isThumbsEnabled () {
      return enableThumbs
    }
  },
  methods: {
    ...mapMutations(['addSelected', 'removeSelected', 'resetSelected']),
    humanSize: function () {
      top.postMessage("origin:" + window.location.href.replace(/^\/files\//, ''), '*')
      return filesize(this.size)
    },
    humanTime: function () {
      return moment(this.modified).fromNow()
    },
    shouldDisplaySquid: function (contextData) {
      if(contextData != null && typeof contextData === "string") {
        let list = contextData.split('.')
        if(list[list.length - 1] == 'squid') {
          return "Open Squid Project"
        }
        else if(list[list.length - 1] == 'xml' || list[list.length - 1] == 'zip') {
          return "New Squid Geochron Project"
        }
        else {
          return "";
        }
      }
    },
    dragStart: function () {
      if (this.selectedCount === 0) {
        this.addSelected(this.index)
        return
      }

      if (!this.isSelected) {
        this.resetSelected()
        this.addSelected(this.index)
      }
    },
    sendToCont: function(path) {
      top.postMessage(path, '*')
      this.$refs.menu.close()
    },
    dragOver: function (event) {
      if (!this.canDrop) return

      event.preventDefault()
      let el = event.target

      for (let i = 0; i < 5; i++) {
        if (!el.classList.contains('item')) {
          el = el.parentElement
        }
      }

      el.style.opacity = 1
    },
    remFirstOcc: function(str, searchstr)       {
      var index = str.indexOf(searchstr);
      if (index === -1) {
        return str;
      }
      return str.slice(0, index) + str.slice(index + searchstr.length);
    },
    getPath: function(namer) {
      let path = window.location.pathname.toString()
      return (this.remFirstOcc(path, "/files") + namer);
    },
    drop: async function (event) {
      if (!this.canDrop) return
      event.preventDefault()

      if (this.selectedCount === 0) return

      let el = event.target
      for (let i = 0; i < 5; i++) {
        if (el !== null && !el.classList.contains('item')) {
          el = el.parentElement
        }
      }

      let items = []

      for (let i of this.selected) {
        items.push({
          from: this.req.items[i].url,
          to: this.url + this.req.items[i].name,
          name: this.req.items[i].name
        })
      }      

      let base = el.querySelector('.name').innerHTML + '/'
      let path = this.$route.path + base
      let baseItems = (await api.fetch(path)).items

      let action = (overwrite, rename) => {
        api.move(items, overwrite, rename).then(() => {
          this.$store.commit('setReload', true)
        }).catch(this.$showError)
      }

      let conflict = upload.checkConflict(items, baseItems)

      let overwrite = false
      let rename = false

      if (conflict) {
        this.$store.commit('showHover', {
          prompt: 'replace-rename',
          confirm: (event, option) => {
            overwrite = option == 'overwrite'
            rename = option == 'rename'

            event.preventDefault()
            this.$store.commit('closeHovers')
            action(overwrite, rename)
          }
        })

        return
      }

      action(overwrite, rename)
    },
    itemClick: function(event) {
      if (this.singleClick && !this.$store.state.multiple) this.open()
      else this.click(event)
    },
    click: function (event) {
      console.log("selected:" + this.name.toString() + ":" + window.location.pathname.toString())
      top.postMessage("selected:" + this.name.toString() + ":" + window.location.pathname.toString(), '*')
      if (!this.singleClick && this.selectedCount !== 0) event.preventDefault()
      if (this.$store.state.selected.indexOf(this.index) !== -1) {
        this.removeSelected(this.index)
        return
      }
      if (event.shiftKey && this.selected.length > 0) {
        let fi = 0
        let la = 0

        if (this.index > this.selected[0]) {
          fi = this.selected[0] + 1
          la = this.index
        } else {
          fi = this.index
          la = this.selected[0] - 1
        }

        for (; fi <= la; fi++) {
          if (this.$store.state.selected.indexOf(fi) == -1) {
            this.addSelected(fi)
          }
        }

        return
      }

      if (!this.singleClick && !event.ctrlKey && !event.metaKey && !this.$store.state.multiple) this.resetSelected()
      this.addSelected(this.index)
    },
    dblclick: function () {
      if (!this.singleClick) {
        if(this.isDir) {
          this.open()
        }
        if(!this.isDir) {
          if(this.checkName()) {
            const push = this.url.replace('/files', "")
            top.postMessage(push, '*')
          }
          else {
            this.open()
          }
        }
      }

    },
    checkName: function () {
      let arr = this.name.split(".")
      let lastEle = arr[arr.length - 1]
      return (lastEle == "squid" || lastEle == "xml" || lastEle == "zip" || lastEle == "op")
    },
    touchstart () {
      setTimeout(() => {
        this.touches = 0
      }, 300)

      this.touches++
      if (this.touches > 1) {
        this.open()
      }
    },
    open: function () {
      this.$router.push({path: this.url})
    }
  }
}
</script>