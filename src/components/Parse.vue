<template>
  <div class="container">
    <h1 class="title is-1">Conversor archivo CSV</h1>

    <div class="columns">
      <div class="column">
        <div class="container first-column">
          <div class="file is-boxed is-centered"
               v-bind:class="{'is-info': !loadedFile, 'is-primary': loadedFile}">
            <label class="file-label">
              <input class="file-input"
                     type="file"
                     name="resume"
                     id="fileInput"
                     @change="upload">
              <span class="file-cta">
              <span class="file-label" id="file-name">
                Selecciona archivo CSV...
              </span>
            </span>
            </label>
          </div>
        </div>


        <div class="columns container">
          <div class="column">

            <p>
              Esta aplicación web convierte el archivo CSV de <a href="https://www.endesaclientes.com">Endesa Clientes</a>
              en un formato que puede leer el simulador de la <a href="https://facturaluz2.cnmc.es">CNMC</a>.
              <hr>
              Tras la conversión, se te mostrará en el panel de la derecha el archivo a descargar. Simplemente pulsa en
              <strong>Guardar</strong> y el archivo se descargará para que pueda ser introducido en el simulador sin ningún error.
              <hr>
              En principio, ninguna edición es necesaria, adaptándose al modelo que se puede encontrar <a href="https://facturaluz2.cnmc.es/ejemplo.csv">aquí</a>.
              No obstante, pulsando en el botón <strong>Editar</strong> se podrán realizar modificaciones en el archivo
              de forma manual; no es recomendado si no sabes nada de archivos CSV y como funcionan.
              <hr>
              Ningún dato que se introduzca aquí se almacena. Es únicamente una aplicación que hice para mi propio uso
              y que he decidido compartir con aquel que la necesite. El código fuente de la aplicación se puede encontrar
              en mi <a href="https://github.com/gloaysa/parse-csv">repositorio de gitHub</a>, si te interesan estas cosas.
            </p>

          </div>
          <div class="column">
            <div class="control">
          <textarea
            class="textarea"
            v-bind:class="{'is-warning': isEditable}"
            rows = '20'
            :readonly="!isEditable"
            v-model='doc'>
          </textarea>
            </div>
            <button
              class="button is-large is-primary"
              @click='save'
              :disabled="!loadedFile"
            >
              Guardar
            </button>
            <button
              class="button is-large is-warning"
              @click='isEditable = !isEditable'
              :disabled="!loadedFile"
              v-show="!isEditable">
              Editar
            </button>
            <button
              class="button is-large is-info"
              @click='isEditable = !isEditable'
              v-show="isEditable">
              Terminar edición
            </button>
          </div>
        </div>
      </div>


    </div>

  </div>

</template>

<script>
  import Papa from 'papaparse'
  import Blob from 'blob'
  import FileSaver from 'file-saver'

  export default {
    name: 'parse',
    data () {
      return {
        doc: null,
        loadedFile: false,
        isEditable: false
      }
    },
    methods: {
      upload (e) {
        const that = this
        const fileToLoad = event.target.files[0]
        const reader = new FileReader()
        document.getElementById('file-name').innerHTML = event.target.files[0].name
        this.loadedFile = true
        reader.onload = fileLoadedEvent => {
          Papa.parse(fileLoadedEvent.target.result, {
            header: false,
            delimiter: ';',
            skipEmptyLines: true,
            complete (results) {
              const HEADER = ['CUPS', 'Fecha', 'Hora', 'Consumo_kWh', 'Metodo_obtencion']

              var data = results.data.map((dataHour) => {
                dataHour = dataHour.map((dataHour) => {
                  return dataHour.trim()
                })

                // Delete precio & consumo por horas
                dataHour.splice(3, 2)
                // Add CUPS to the beginning of columns
                dataHour.unshift(that.cups)
                // Convert from hW to kWh
                var kWh = parseInt(dataHour[3]) / 1000
                dataHour[3] = kWh.toString().replace(/\./g, ',')
                // Convert date
                dataHour[1] = tranformDate(dataHour[1])
                // Add metodo de obtencion
                dataHour.push('R')

                return dataHour
              })
              data.unshift(HEADER)
              function tranformDate (date) {
                return date.replace(/(\d{4})-(\d{1,2})-(\d{1,2})/, function (match, y, m, d) {
                  var newDate = d + '/' + m + '/' + y
                  return newDate.toString()
                })
              }
              that.doc = Papa.unparse(data)
            },
            beforeFirstChunk (chunk) {
              let lines = chunk.split('\n')
              that.cups = lines[0].split(',').pop().trim()
              that.from = lines[1].split(',').pop().trim()
              that.to = lines[2].split(',').pop().trim()
              let lastLine = lines.length - 2
              lines.splice(lastLine, 1)
              lines.splice(0, 7)
              return lines.join('\n').replace(/,/g, ';')
            },
            error (errors) {
              console.log('error', errors)
            }
          })
        }
        reader.readAsText(fileToLoad)
      },
      save () {
        const blob = new Blob([this.parseJSONtoCSV()], { type: 'text/csv' })
        FileSaver.saveAs(blob, 'Consumo desde ' + this.from + ' hasta ' + this.to + '.csv')
      },
      parseJSONtoCSV () {
        return this.doc
      }
    }
  }
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
  .first-column {
    margin-bottom: 20px;
  }
</style>
