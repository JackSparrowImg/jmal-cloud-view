<template>
  <div>

    <history-popover
      ref="historyPopover"
      style="right: 20px;top: 61px;position: absolute;"
      :transparent="true"
      :has-history-version.sync="hasHistoryVersion"
      :history-list-popover-visible.sync="historyListPopoverVisible"
      :history-operation-loading.sync="historyOperationLoading"
      :saved.sync="saved"
      @viewHistoryFile="viewHistoryFile"
      @recoverySuccess="recoverySuccess"
    >
    </history-popover>
    <div class="component-only-office">
      <div :id="this.id" class="placeholder"></div>
    </div>
  </div>
</template>

<script>

import api from "@/api/file-api"
import fileConfig from "@/utils/file-config";
import Bus from "@/assets/js/bus";
import SparkMD5 from 'spark-md5'
import HistoryPopover from "@/components/HistoryPopover/index.vue";

export default {
  name: "OnlyOfficeEditor",
  components: {HistoryPopover},
  props: {
    id: {
      type: String,
      default: () => {
        return "office_" + Math.round(Math.random() * 10000)
      }
    },
    fileUrl: {
      type: String,
      default: ''
    },
    code: {
      type: String,
      default: ''
    },
    shareId: {
      type: String,
      default: ''
    },
    file: {
      type: Object,
      default: function () {
        return {}
      }
    },
     readOnly: {
      type: Boolean,
      default: false
    },
    documentKey: Function
  },
  data() {
    return {
      docEditorConfig: {},
      docEditor: null,
      fileKey: '',
      saved: true,
      hasHistoryVersion: false,
      historyVersion: {metadata: {}},
      titleObserver: null,
      fileHistoryDateList: [],
      historyListPopoverVisible: false,
      historyPageIndex: 1,
      historyPageSize: 50,
      viewHistory: false,
      historyOperationLoading: false
    }
  },
  beforeDestroy() {
    this.destroyEditor()
  },
  computed: {
    fileType() {
      return this.getType(this.file.suffix)
    },
  },
  watch: {
    'file.id': {
      handler(id)  {
        if (!id) {
          return
        }
        let officeApiUrl = fileConfig.officeApiUrl()
        $J.loadScript($J.apiUrl(officeApiUrl), (e) => {
          if (e !== null) {
            this.$emit('onClose')
            Bus.$emit('loadFileFaild')
            return
          }
          if(this.$store.state.user.token && this.$store.state.user.userId === this.file.userId){
            api.getFileInfoById({id: this.file.id}).then(res => {
              this.file = res.data
              this.loadFile()
            })
          } else {
            api.getPublicFileInfoById({fileId: this.file.id, shareId: this.shareId}).then(res => {
              this.file = res.data
              this.loadFile()
            })
          }
        })
      },
      immediate: true,
    },
    hasHistoryVersion(value) {
      if (!value) {
        let parentDoc = document.querySelector('.component-only-office')
        let doc = parentDoc.getElementsByTagName('iframe')[0].contentWindow.document
        let versionBtn = doc.getElementById('box-doc-name-history-btn')
        if (versionBtn) {
          versionBtn.style.display = 'none'
        }
      }
    }
  },
  mounted() {
    Bus.$on('previewSaveAndClose', () => {
      this.requestClose()
    })
  },
  destroyed() {
    Bus.$off('previewSaveAndClose')
  },
  methods: {
    viewHistoryFile({historyInfo}) {
      const historyUrl = window.location.origin + fileConfig.previewHistoryUrl(historyInfo.id, this.$store.state.user.name, this.$store.state.user.token)
      this.onRequestHistoryData(null, historyInfo, historyUrl)
      this.historyListPopoverVisible = false

      let parentDoc = document.querySelector('.component-only-office')
      let doc = parentDoc.getElementsByTagName('iframe')[0].contentWindow.document
      const toolbar = doc.getElementById('box-doc-name')
      // add cancelPreview button
      if (!doc.getElementById('box-doc-name-cancel-preview')) {
        let newButton = document.createElement("button")
        newButton.setAttribute('id', 'box-doc-name-cancel-preview')
        newButton.classList.add('btn', 'btn-header')
        newButton.innerHTML = '取消预览'
        newButton.style.color = '#fff'
        newButton.style.width = '60px'
        newButton.addEventListener('click', this.cancelPreview)
        toolbar.prepend(newButton)
      }
      // update title
      this.updateTitle(doc, historyInfo)
    },
    updateTitle(doc, historyInfo) {
      this.historyVersion = historyInfo
      let docTitle = doc.getElementById('title-doc-name')

      // Monitor title changes
      if (this.titleObserver === null) {
        let that = this
        this.titleObserver = new MutationObserver(function(mutations) {
          let el = mutations[mutations.length-1].target
          setTimeout(() => {
            el.innerHTML = `历史版本: ${that.historyVersion.metadata.time}`
          }, 0)
        })
        let config = { childList: true, characterData: true }
        this.titleObserver.observe(docTitle, config)
      }
    },
    recoverySuccess({result}) {
      console.log('recoverySuccess', result)
      this.reloadDocument(`${result.data}-${SparkMD5.hash(this.file.id)}`)
      this.$message({message: '恢复成功',type: 'success'})
      this.historyListPopoverVisible = false
      this.historyOperationLoading = false
    },
    cancelPreview() {
      this.reloadDocument()
    },
    reloadDocument(key) {
      this.destroyEditor()
      this.docEditorConfig.document.key = key ? key : this.fileKey
      this.docEditor = new DocsAPI.DocEditor(this.id, this.docEditorConfig)
    },
    onRequestHistoryData(event, historyInfo, url) {
      this.docEditor.setHistoryData({
        "fileType": this.fileType,
        "key": `${new Date(historyInfo.metadata.time).getTime()}-${SparkMD5.hash(this.file.id)}`,
        "url":  url,
        "version": historyInfo.metadata.time,
      })
    },
    onRequestHistory() {
      this.docEditor.refreshHistory({
        "currentVersion": 'currentVersion',
        "history": []
      })
    },
    getType(type) {
      switch (type) {
        case 'doc':
          return 'docx'
        case 'word':
          return 'docx'
        case 'excel':
          return 'xlsx'
        case 'xls':
          return 'xlsx'
        case 'ppt':
          return 'pptx'
      }
      return type
    },
    async loadFile() {
      this.$nextTick(()=> {
        this.$refs.historyPopover.loadHistoryList(this.file.id)
      })
      this.destroyEditor()
      this.fileUrl = window.location.origin + fileConfig.previewUrl(this.$store.state.user.name, this.file, this.$store.getters.token)
      if (this.readOnly && window.shareId) {
        this.fileUrl = window.location.origin + fileConfig.publicPreviewUrl(this.file, window.shareId, this.$store.getters.shareToken)
      }
      this.fileKey = `${new Date(this.file.updateDate).getTime()}-${SparkMD5.hash(this.file.id)}`

      let callbackUrl = fileConfig.officeCallBackUrl(this.$store.getters.token, this.$store.getters.name, this.file.id)

      this.docEditorConfig = {
        document: {
          fileType: this.fileType,
          key: this.fileKey,
          title: this.file.name,
          url: this.fileUrl,
        },
        editorConfig: {
          mode: "edit",
          lang: "zh",
          user: {
            id: this.$store.state.user.userId,
            name: this.$store.state.user.name
          },
          customization: {
            autosave: false,
            comments: true,
            compactHeader: false,
            compactToolbar: false,
            compatibleFeatures: false,
            forcesave: false,
            help: false,
            hideRightMenu: false,
            hideRulers: false,
            submitForm: false,
            about: null,
            feedback: false
          },
          callbackUrl: callbackUrl,
        }
      }
      if (!this.$pc) {
        this.docEditorConfig.type = 'mobile'
      }
      if (this.readOnly) {
        this.docEditorConfig.editorConfig.mode = "view"
        this.docEditorConfig.editorConfig.callbackUrl = null
        if (!this.docEditorConfig.editorConfig.user.id) {
          let visitor = $J.getStorageInt("visitor")
          if (!visitor) {
            visitor = $J.randNum(1000, 99999)
            $J.setStorage("viewer", visitor)
          }
          this.docEditorConfig.editorConfig.user.id = "visitor_" + visitor
          this.docEditorConfig.editorConfig.user.name = "Visitor_" + visitor
        }
      }
      this.docEditorConfig.events = {
        "onAppReady": this.onAppReady,
        "onDocumentReady": this.onDocumentReady,
        "onDocumentStateChange": this.onDocumentStateChange,
        "onRequestHistory": this.onRequestHistory,
        "onRequestHistoryData": this.onRequestHistoryData,
      }
      this.$nextTick(() => {
        this.titleObserver = null
        this.docEditor = new DocsAPI.DocEditor(this.id, this.docEditorConfig)
      })
    },
    onClickHistoryVersion(event) {
      this.historyListPopoverVisible = !this.historyListPopoverVisible
      event.stopPropagation()
    },
    onAppReady() {
      this.$emit('onReady')
    },
    onDocumentReady() {
      let parentDoc = document.querySelector('.component-only-office')
      let doc = parentDoc.getElementsByTagName('iframe')[0].contentWindow.document

      if (!this.$pc) {
        parentDoc.style.top = '5rem'
        doc.querySelector('.navbar.main-navbar.navbar-with-logo').style.height = '0'
      }

      let logo = doc.querySelector('.extra .logo')
      // 隐藏logo,about
      logo.style.display = 'none'
      let about = doc.getElementById('left-btn-about')
      about.style.display = 'none'
      this.saveBtnDoc = doc.getElementById('slot-btn-dt-save').getElementsByTagName('button')[0]

      doc.getElementById('id-box-doc-name').style.paddingRight = '0'

      if (this.hasHistoryVersion && !doc.getElementById('box-doc-name-history-btn')) {
        const toolbar = doc.getElementById('box-doc-name')
        toolbar.style.justifyContent = 'right'
        let versionButton = document.createElement("button")
        versionButton.classList.add('btn', 'btn-header')
        versionButton.setAttribute('id', 'box-doc-name-history-btn')
        fetch(require("@/assets/img/history.svg"))
          .then(response => response.text())
          .then(data => {
            let parser = new DOMParser()
            let svgDoc = parser.parseFromString(data, "image/svg+xml")
            let svgElement = svgDoc.querySelector('svg')
            svgElement.setAttribute('style', 'width:16px;height:16px;')
            versionButton.appendChild(svgElement)
          })
        versionButton.addEventListener('click', this.onClickHistoryVersion)
        doc.addEventListener('click', this.$refs.historyPopover.onGlobalClick)
        toolbar.appendChild(versionButton)
      }
    },
    onDocumentStateChange() {
      this.saved = this.saveBtnDoc.classList.contains('disabled')
      this.$emit('onEdit', this.saved)
    },
    requestClose() {
      this.$emit('manualSave')
    },
    destroyEditor() {
      if (this.docEditor !== null) {
        this.docEditor.destroyEditor()
        this.docEditor = null
        this.titleObserver = null
      }
    },
  }
}
</script>
<style lang="scss" scoped>

</style>
