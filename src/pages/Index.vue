<template>
  <q-page class="row items-center justify-evenly">
    <div class="row">
      <div class="q-pa-md">
        <q-card style="max-width: 500px; min-width: 300px">
          <q-card-section>
            <div class="text-h6">Общее</div>
          </q-card-section>
          <q-separator inset />
          <q-card-section class="q-pt-none">
            <div class="row q-pt-sm">
              <q-select filled v-model="resultSet.fromVersion" :options="fromVersions" label="С версии" @input="calc" style="width: 100%" class="q-pb-sm" />
            </div>
            <div class="row">
              <q-select filled v-model="resultSet.toVersion" :options="toVersions" label="На версию" @input="calc" style="width: 100%" class="q-pb-sm" />
            </div>
            <div class="row">
              <q-input v-model="resultSet.serversCount" label="Кол-во серверов" stack-label @input="calc" style="width: 100%" class="q-pb-sm" />
            </div>
            <div class="row">
              <q-checkbox v-model="resultSet.testConvert" label="С тестовой конвертацией" @input="calc" :disable="true" />
            </div>
            <div class="row">
              <q-checkbox v-model="resultSet.sync1C" label="Синхронизация с 1С" @input="calc" />
            </div>
            <div class="row">
              <q-checkbox v-model="resultSet.syncAD" label="Синхронизация с AD" @input="calc" />
            </div>
            <div class="row">
              <q-checkbox v-model="resultSet.hasNOMAD" label="Есть мобильные приложения" @input="calc" />
            </div>
            <div class="row">
              <q-checkbox v-model="resultSet.otherVendorDev" label="Есть другие ТР" @input="calc" />
            </div>
            <div class="row">
              <q-select filled multiple
                        @input="calc"
                        v-model="resultSet.vendorStandardModules"
                        :options="vendorStandardModules"
                        label="ТР вендора"
                        stack-label
                        :disable="!resultSet.otherVendorDev"
                        style="width: 100%; max-width: 500px" />
            </div>
          </q-card-section>

          <q-separator inset />

          <q-card-section>
            <div class="text-h6">Разработка</div>
          </q-card-section>
          <q-separator inset />
          <div class="q-gutter-sm">
            <q-toggle
              v-model="resultSet.onlyDefaultDev"
              label="Только коробка"
              @input="calc"
            />
          </div>
          <q-card-section class="q-pt-none">
            <div class="row">
              <q-checkbox v-model="resultSet.hasVendorDev" label="Есть заказная разработка от вендора" @input="calc" :disable="resultSet.onlyDefaultDev" />
            </div>
            <div class="row">
              <q-checkbox v-model="resultSet.hasTaskDev" label="Дорабатывалась задача на согласование по регламенту" @input="calc" :disable="resultSet.onlyDefaultDev" />
            </div>
            <div class="row">
              <q-checkbox v-model="resultSet.hasDocumentTypesDev" label="Дорабатывались типы документов" @input="calc" :disable="resultSet.onlyDefaultDev" />
            </div>
            <div class="row">
              <q-select filled
                        @input="calc"
                        :disable="resultSet.onlyDefaultDev"
                        v-model="resultSet.developmentDifficulty"
                        :options="developmentLevels"
                        label="Степень сложности разработки"
                        stack-label
                        style="width: 100%; max-width: 500px; min-width: 200px" />
            </div>
          </q-card-section>

          <q-separator inset />
          <q-card-section class="q-pt-none">
            <div class="q-pt-md row">
              <div class="col">
                Итого (часов): {{ hours }}
              </div>
              <div class="col">
                <q-btn class="float-right" color="primary" label="Ссылка" @click="copyLink" :disable="buttonDisabled" />
              </div>
            </div>
          </q-card-section>
        </q-card>
      </div>
    </div>
  </q-page>
</template>

<script lang="ts">
import { defineComponent } from '@vue/composition-api'
import { copyToClipboard, Loading } from 'quasar'
import axios from 'axios'

export default defineComponent({
  name: 'PageIndex',
  data () {
    return {
      resultSet: {
        onlyDefaultDev: true,
        sync1C: false,
        syncAD: false,
        hasNOMAD: false,
        otherVendorDev: false,
        testConvert: true,
        serversCount: 1,
        fromVersion: '',
        toVersion: '',
        vendorStandardModules: [],
        hasVendorDev: false,
        hasTaskDev: false,
        hasDocumentTypesDev: false,
        developmentDifficulty: ''
      },
      hours: 0,
      versions: [
        '2.8', '2.9', '3.0', '3.1', '3.2', '3.3', '3.4', '3.5', '3.6', '4.0', '4.1'
      ],
      vendorStandardModules: [
        'Планирование проектов', 'Выполнение заданий через почту', 'HR-процессы', 'Командировки', 'Agile-доски'
      ],
      developmentLevels: [
        'Мелкие изменения косметического характера',
        'Дополнение коробочных механизмов',
        'Локальная, продуманная доработка или переработка коробки',
        'Доработки, местами носящие непродуманный характер',
        'Существенные изменения, противоречащие коробочным механизмам'
      ],
      buttonDisabled: true
    }
  },
  computed: {
    fromVersions () {
      const v: string[] = this.versions
      return v.slice(0, v.length - 1)
    },
    toVersions () {
      const v: string[] = this.versions
      const from: string = this.resultSet.fromVersion
      return from === '' ? [] : v.slice(v.indexOf(from) + 1, v.length)
    }
  },
  methods: {
    calc () {
      if (this.resultSet.toVersion === '' || this.resultSet.fromVersion === '') {
        return
      }
      this.buttonDisabled = false
      let cHours = 0
      // Сколько шагов конвертации
      let convertSteps = this.versions.indexOf(this.resultSet.toVersion) - this.versions.indexOf(this.resultSet.fromVersion)
      convertSteps = Math.floor(convertSteps / 2) + (convertSteps % 2)
      // Полагаем 1 шаг конвертации = 8 часам
      convertSteps = convertSteps * 8
      // Дополнительный сервер еще +4 часа
      const convertBase = convertSteps + (4 * (this.resultSet.serversCount - 1))
      cHours += this.resultSet.testConvert ? 2 * convertBase : convertBase
      // Считаем, что 1С\AD это +2 часа на перенастройку коннектора
      cHours += this.resultSet.sync1C ? 2 : 0
      cHours += this.resultSet.syncAD ? 2 : 0
      cHours += this.resultSet.hasNOMAD ? 2 : 0
      // Каждый ТР +16 часов
      cHours += this.resultSet.otherVendorDev ? this.resultSet.vendorStandardModules.length * 16 : 0
      // Разработка и адаптация
      if (!this.resultSet.onlyDefaultDev) {
        const devDifficulty = (this.developmentLevels.indexOf(this.resultSet.developmentDifficulty) * 20) / 100 + 1.27
        let devHours = this.resultSet.hasVendorDev ? 20 : 0
        devHours += this.resultSet.hasTaskDev ? 40 : 0
        devHours += this.resultSet.hasDocumentTypesDev ? 30 : 0
        cHours += Math.floor(devHours * devDifficulty)
      }
      this.hours = cHours
    },
    copyLink () {
      this.buttonDisabled = true
      Loading.show()
      const link = window.location.href + '?data=' + JSON.stringify(this.resultSet)
      axios.post('https://dev-io.ru/l/api/addLink', {
        link: link
        // eslint-disable-next-line camelcase
      }).then((response: { data: { short_id: string } }) => {
        copyToClipboard('https://dev-io.ru/l/' + response.data.short_id)
          .then(() => {
            console.log(link)
            Loading.hide()
            this.$q.notify({
              message: 'Ссылка скопирована в буфер обмена, вы можете отправить её для просмотра расчёта. Помните, что трудозатраты, расчитанные данным калькулятором являются минимальными, и подлежат пересмотру в соответствии с реальной ситуацией.',
              color: 'primary',
              multiLine: true,
              progress: true,
              timeout: 30000,
              actions: [
                { label: 'ОК', color: 'white', handler: () => { /* ... */ } }
              ]
            })
          })
          .catch((e) => {
            console.error(e)
            Loading.hide()
          })
      }).catch((e: any) => {
        console.error(e)
        Loading.hide()
      })
    }
  },
  mounted () {
    const queryString = window.location.hash
      .replace('/#/', '')
      .replace('/#', '')
      .replace('#/', '')
      .replace('#/?data=', '')
      .replace('?data=', '')
    if (queryString !== '') {
      Object.assign(this.resultSet, JSON.parse(decodeURIComponent(queryString)))
    }
    this.calc()
  }
})
</script>
