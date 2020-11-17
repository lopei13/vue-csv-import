<template>
  <div class="vue-csv-uploader">
    <div class="form">
      <div class="vue-csv-uploader-part-one">
        <div
          class="form-check form-group csv-import-checkbox"
          v-if="headers === null"
        >
          <slot
            name="hasHeaders"
            :headers="hasHeaders"
            :toggle="toggleHasHeaders"
          >
            <input
              :class="checkboxClass"
              type="checkbox"
              :id="makeId('hasHeaders')"
              :value="hasHeaders"
              @change="toggleHasHeaders"
            />
            <label class="form-check-label" :for="makeId('hasHeaders')">
              File Has Headers
            </label>
          </slot>
        </div>
        <div class="form-group csv-import-file">
          <input
            ref="csv"
            type="file"
            @change.prevent="validFileMimeType"
            :class="inputClass"
            name="csv"
          />
          <slot name="error" v-if="showErrorMessage">
            <div class="invalid-feedback d-block">File type is invalid</div>
          </slot>
          <p v-if="loadingFile">{{ loadMessage }}</p>
        </div>
        <div class="form-group">
          <slot name="next" :load="load" v-if="!autoMatchFields">
            <button
              type="submit"
              :disabled="disabledNextButton"
              :class="buttonClass"
              @click.prevent="load"
            >
              {{ loadBtnText }}
            </button>
          </slot>
        </div>
      </div>
      <div class="vue-csv-uploader-part-two">
        <div class="vue-csv-mapping" v-if="sample && !autoMatchFields">
          <table :class="tableClass">
            <slot name="thead">
              <thead>
                <tr>
                  <th>Field</th>
                  <th>CSV Column</th>
                </tr>
              </thead>
            </slot>
            <tbody>
              <tr v-for="(field, key) in fieldsToMap" :key="key">
                <td>{{ field.label }}</td>
                <td>
                  <select
                    :class="tableSelectClass"
                    :name="`csv_uploader_map_${key}`"
                    v-model="map[field.key]"
                  >
                    <option :value="null" v-if="canIgnore">Ignore</option>
                    <option
                      v-for="(column, key) in firstRow"
                      :key="key"
                      :value="key"
                    >
                      {{ column }}
                    </option>
                  </select>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { drop, every, forEach, get, isArray, map, set } from 'lodash'
import Papa from 'papaparse'
import mimeTypes from 'mime-types'

export default {
  props: {
    value: Array,
    mapFields: {
      required: true
    },
    callback: {
      type: Function,
      default: () => ({})
    },
    catch: {
      type: Function,
      default: () => ({})
    },
    finally: {
      type: Function,
      default: () => ({})
    },
    parseConfig: {
      type: Object,
      default () {
        return {}
      }
    },
    headers: {
      default: null
    },
    loadBtnText: {
      type: String,
      default: 'Next'
    },
    submitBtnText: {
      type: String,
      default: 'Submit'
    },
    autoMatchFields: {
      type: Boolean,
      default: false
    },
    autoMatchIgnoreCase: {
      type: Boolean,
      default: false
    },
    tableClass: {
      type: String,
      default: 'table'
    },
    checkboxClass: {
      type: String,
      default: 'form-check-input'
    },
    buttonClass: {
      type: String,
      default: 'btn btn-primary'
    },
    inputClass: {
      type: String,
      default: 'form-control-file'
    },
    validation: {
      type: Boolean,
      default: true
    },
    fileMimeTypes: {
      type: Array,
      default: () => {
        return [
          'text/csv',
          'text/x-csv',
          'application/vnd.ms-excel',
          'text/plain'
        ]
      }
    },
    tableSelectClass: {
      type: String,
      default: 'form-control'
    },
    canIgnore: {
      type: Boolean,
      default: false
    },
    loadMessage: {
      default: 'Loading....'
    }
  },
  data: () => ({
    form: {
      csv: null
    },
    fieldsToMap: [],
    map: {},
    hasHeaders: true,
    csv: null,
    sample: null,
    isValidFileMimeType: false,
    fileSelected: false,
    loadingFile: false
  }),
  created () {
    this.hasHeaders = this.headers

    if (isArray(this.mapFields)) {
      this.fieldsToMap = map(this.mapFields, (item) => {
        return {
          key: item,
          label: item
        }
      })
    } else {
      this.fieldsToMap = map(this.mapFields, (label, key) => {
        return {
          key: key,
          label: label
        }
      })
    }
  },
  methods: {
    submit () {
      const _this = this
      this.form.csv = this.buildMappedCsv()
      this.$emit('input', this.form.csv)
      _this.callback(this.form.csv)
    },
    buildMappedCsv () {
      const _this = this

      let csv = this.hasHeaders ? drop(this.csv) : this.csv

      return map(csv, (row) => {
        let newRow = {}

        forEach(_this.map, (column, field) => {
          set(newRow, field, get(row, column))
        })

        return newRow
      })
    },
    validFileMimeType () {
      this.loadingFile = true
      let file = this.$refs.csv.files[0]
      const mimeType =
        file.type === '' ? mimeTypes.lookup(file.name) : file.type

      if (file) {
        this.fileSelected = true
        this.isValidFileMimeType = this.validation
          ? this.validateMimeType(mimeType)
          : true
        if (this.isValidFileMimeType && this.autoMatchFields) {
          this.load()
        }
      } else {
        this.isValidFileMimeType = !this.validation
        this.fileSelected = false
      }
      this.loadingFile = false
    },
    validateMimeType (type) {
      return this.fileMimeTypes.indexOf(type) > -1
    },
    load () {
      const _this = this

      this.readFile((output) => {
        _this.sample = get(
          Papa.parse(output, { preview: 2, skipEmptyLines: true }),
          'data'
        )
        _this.csv = get(Papa.parse(output, { skipEmptyLines: true }), 'data')
      })
    },
    readFile (callback) {
      let file = this.$refs.csv.files[0]

      if (file) {
        let reader = new FileReader()
        reader.readAsText(file, 'UTF-8')
        reader.onload = function (evt) {
          callback(evt.target.result)
        }
        reader.onerror = function () {}
      }
    },
    toggleHasHeaders () {
      this.hasHeaders = !this.hasHeaders
    },
    makeId (id) {
      return `${id}${this._uid}`
    }
  },
  watch: {
    map: {
      deep: true,
      handler: function (newVal) {
        if (!this.url) {
          let hasAllKeys = Array.isArray(this.mapFields)
            ? every(this.mapFields, function (item) {
              return newVal.hasOwnProperty(item)
            })
            : every(this.mapFields, function (item, key) {
              return newVal.hasOwnProperty(key)
            })

          if (hasAllKeys) {
            this.submit()
          }
        }
      }
    },
    sample (newVal, oldVal) {
      if (this.autoMatchFields) {
        if (newVal !== null) {
          this.fieldsToMap.forEach((field) => {
            newVal[0].forEach((columnName, index) => {
              if (this.autoMatchIgnoreCase === true) {
                if (
                  field.label.toLowerCase().trim() ===
                  columnName.toLowerCase().trim()
                ) {
                  this.$set(this.map, field.key, index)
                }
              } else {
                if (field.label.trim() === columnName.trim()) {
                  this.$set(this.map, field.key, index)
                }
              }
            })
          })
        }
      }
    }
  },
  computed: {
    firstRow () {
      return get(this, 'sample.0')
    },
    showErrorMessage () {
      return this.fileSelected && !this.isValidFileMimeType
    },
    disabledNextButton () {
      return !this.isValidFileMimeType
    }
  }
}
</script>
