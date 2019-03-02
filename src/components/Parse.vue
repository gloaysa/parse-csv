<template>
  <div class="parse">
    <h1>Conversor archivo CSV</h1>
    <input
      id="fileInput"
      type="file"
      @change="upload">
    <button
      @click='save'
      type='button'
      download >
      Save
    </button>
    <div class="body">
      <div class="entry">
        <textarea
          class="entry-result"
          v-model='doc'
          placeholder="Type here">
        </textarea>
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
        cups: null,
        from: null,
        to: null
      }
    },
    methods: {
      upload (e) {
        const that = this
        const fileToLoad = event.target.files[0]
        const reader = new FileReader()
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
  .body {
    display: flex;
    justify-content: center;
    margin-top: 30px;
  }

  .entry {
    width: 40%;
  }

  .entry-result {
    width: 100%;
    height: 50vh;
  }


</style>
