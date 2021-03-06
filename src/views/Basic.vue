<template>
  <div class="basic">
    <el-form>
      <el-form-item>
        <div class="image-preview-wrapper">
          <div
            class="image-preview"
            :style="{'background-image': `url(${image})` }"
          />
          <div class="image-preview__actions">
            <el-upload
              ref="upload"
              action=""
              :on-change="onChange"
              :http-request="upload"
              :before-upload="onBeforeUpload"
              :multiple="false"
              :limit="1"
              :show-file-list="false"
            >
              <div slot="trigger">
                <el-button>Upload image</el-button>
              </div>
              <span
                v-if="basic.image.base64"
                class="remove-image"
                @click="onClearImage"
              >
                <i class="el-icon-circle-close" />
              </span>
            </el-upload>
            <el-row :gutter="20">
              <el-col :span="16">
                <el-input
                  v-model="imageUrl"
                  placeholder="http://"
                  clearable
                  @clear="onClearImageLink"
                />
              </el-col>
              <el-col :span="8">
                <el-button
                  style="width: 100%"
                  @click="onAddLink"
                >Add link</el-button>
              </el-col>
            </el-row>
            <div class="desc">
              <p>You can upload image or add the public link to image.</p>
              <p>The uploaded image will be converted to base64. The image should not be more than 10KB.</p>
              <el-popover
                placement="top"
                width="300"
                trigger="click"
                class="image-tips"
              >
                <p>Some email client, like Gmail and Outlook do not support or may not display embedded images on base64 at all.</p>
                <p>Use an image link instead of embedding it as base64.</p>
                <strong slot="reference">Image tips</strong>
              </el-popover>
            </div>
          </div>
        </div>
      </el-form-item>
      <el-row :gutter="20">
        <el-col
          v-for="(item, index) in basic.fields"
          :key="item.id"
          :span="12"
        >
          <field-item
            :is-addon-field="index > 5"
            :index="index"
          />
        </el-col>
      </el-row>
      <el-form-item>
        <el-button
          type="primary"
          style="width: 100%;"
          @click="onAddField"
        >Add custom field</el-button>
      </el-form-item>
    </el-form>
    <el-dialog
      title="Add new field"
      :visible.sync="showDialog"
    >
      <el-alert
        v-if="showAlert"
        title="Label is exist"
        type="error"
        :closable="false"
      />
      <el-form label-position="top">
        <el-row :gutter="20">
          <el-col :span="8">
            <el-form-item label="Label">
              <el-input
                ref="fieldName"
                v-model.trim="fieldName"
              />
            </el-form-item>
          </el-col>
          <el-col :span="8">
            <el-form-item label="Value">
              <el-input v-model.trim="filedValue" />
            </el-form-item>
          </el-col>
          <el-col :span="8">
            <el-form-item label="Type">
              <el-select
                v-model="filedType"
                style="width:100%;"
              >
                <el-option
                  v-for="item in attributes.types"
                  :key="item.value"
                  :value="item.value"
                  :label="item.label"
                />
              </el-select>
            </el-form-item>
          </el-col>
        </el-row>
        <el-form-item style="text-align: right;">
          <el-button
            type="primary"
            @click="addField"
          >Add field</el-button>
        </el-form-item>
      </el-form>
    </el-dialog>
  </div>
</template>

<script>
import { mapState } from 'vuex'
import { guid } from '../util/helpers'
import FieldItem from './FieldItem'

export default {
  name: '',

  components: {
    FieldItem
  },

  data () {
    return {
      fieldName: '',
      filedValue: '',
      filedType: 'text',
      fileList: '',
      fileBase64: '',
      imageLink: '',
      showDialog: false,
      showAlert: false,
      isLt10: false
    }
  },

  computed: {
    ...mapState(['attributes', 'basic']),
    image () {
      return this.basic.image.link
        ? this.basic.image.link
        : this.basic.image.base64
    },
    imageUrl: {
      get () {
        return this.basic.image.link
      },
      set (v) {
        this.$store.dispatch('updateImage', { link: v })
      }
    }
  },

  created () {
    this.$ga.page(this.$router)
  },

  methods: {
    addField () {
      const newFiled = {
        name: this.fieldName,
        value: this.filedValue,
        type: this.filedType,
        id: guid()
      }

      this.$store.dispatch('addField', newFiled)
      this.fieldName = ''
      this.filedValue = ''
      this.showDialog = false
    },
    onAddField () {
      this.showDialog = true
      this.$nextTick(() => {
        this.$refs.fieldName.focus()
      })
    },
    onAddLink () {
      this.$store.dispatch('updateImage', { link: this.imageLink })
    },
    onClearImage () {
      this.$refs.upload.clearFiles()
      this.fileBase64 = ''
      this.$store.dispatch('updateImage', { base64: this.fileBase64 })
    },
    onClearImageLink () {
      this.imageLink = ''
      this.onAddLink()
    },
    onBeforeUpload (file) {
      const isJPG = file.type === 'image/jpeg'
      const isPNG = file.type === 'image/png'
      this.isLt10 = file.size < 10000

      if (!this.isLt10) {
        this.$message({
          message: 'Warning, uploaded file is more than 10KB, and will be compressed.',
          type: 'warning'
        })
      }
      if (!isJPG && !isPNG) {
        this.$message({
          message: 'Error, uploaded file should be a .jpg or .png.',
          type: 'error'
        })
      }

      return true
    },
    onChange (file, fileList) {
      this.fileList = fileList
    },
    async upload (data) {
      this.fileBase64 = await this.getBase64ImageFromBlob(data.file)
      if (!this.isLt10) {
        this.fileBase64 = await this.compressImage(this.fileBase64)
      }
      this.$store.dispatch('updateImage', { base64: this.fileBase64 })
    },
    async compressImage (base64) {
      const canvas = document.createElement('canvas')
      const img = document.createElement('img')

      return new Promise((resolve, reject) => {
        img.onload = function () {
          let width = img.width
          let height = img.height

          const maxHeight = 200
          const maxWidth = 200

          if (width > height) {
            if (width > maxWidth) {
              height = Math.round(height *= maxWidth / width)
              width = maxWidth
            }
          } else {
            if (height > maxHeight) {
              width = Math.round(width *= maxHeight / height)
              height = maxHeight
            }
          }

          canvas.width = width
          canvas.height = height

          const ctx = canvas.getContext('2d')
          ctx.drawImage(img, 0, 0, width, height)

          resolve(canvas.toDataURL('image/jpeg', 0.7))
        }

        img.onerror = function (err) {
          reject(err)
        }

        img.src = base64
      })
    },
    getBase64ImageFromBlob (blob) {
      return new Promise((resolve, reject) => {
        const reader = new FileReader()

        reader.addEventListener(
          'load',
          function () {
            resolve(reader.result)
          },
          false
        )

        reader.onerror = (err) => {
          return reject(new Error(err))
        }
        reader.readAsDataURL(blob)
      })
    }
  }
}
</script>

<style lang="scss">
@import '../assets/scss/variables';

.image-preview-wrapper {
  display: flex;
  margin-top: 20px;
  .image-preview {
    margin-right: 30px;
    width: 100px;
    height: 100px;
    background-color: #eee;
    border-radius: 100%;
    background-size: 100px;
    background-position: center center;
    flex-shrink: 0;
    &__actions {
    }
  }
  .remove-image {
    padding: 10px;
    cursor: pointer;
    &:hover {
      i {
        color: #909399;
      }
    }
    i {
      color: #c0c4cc;
    }
  }

  .el-button {
    height: 40px;
  }
  .el-upload {
    display: inline-block;
    margin-bottom: 20px;
  }
}
.remove-field {
  display: inline-block;
  cursor: pointer;
}
.edit {
  i {
    color: $color-primary;
  }
  &:hover {
    i {
      color: darken($color-primary, 20%);
    }
  }
}
.delete {
  i {
    color: red;
  }
  &:hover {
    i {
      color: darken(red, 20%);
    }
  }
}
.image-tips {
  display: block;
  text-align: right;
  .el-popover__reference {
    cursor: pointer;
  }
}
</style>
