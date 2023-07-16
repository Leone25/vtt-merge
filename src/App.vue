<script>
import draggable from 'vuedraggable';
import downArrow from './components/icons/downArrow.vue';
import info from './components/icons/info.vue';
import checkboxChecked from './components/icons/checkboxChecked.vue';
import checkboxUnchecked from './components/icons/checkboxUnchecked.vue';
import deleteIcon from './components/icons/deleteIcon.vue';

export default {
  data() {
    return {
      files: [],
      loading: false,
      idCounter: 0,
      mergeType: 'concat',
      globalOffset: 0,
      outputFilename: 'merged',
      removeComments: false,
      isDraggingOver: false
    }
  },
  components: {
    draggable,
    downArrow,
    info,
    checkboxChecked,
    checkboxUnchecked,
    deleteIcon
  },
  mounted() {
    document.body.addEventListener('dragover', (e) => {
      e.preventDefault();
      this.isDraggingOver = true;
    });
    document.body.addEventListener('dragend', (e) => {
      e.preventDefault();
      this.isDraggingOver = false;
    });
    document.body.addEventListener('dragcancel', (e) => {
      e.preventDefault();
      this.isDraggingOver = false;
    });
    document.body.addEventListener('drop', (e) => {
      e.preventDefault();
      this.isDraggingOver = false;
      this.loadFiles(e.dataTransfer.files);
    });
  },
  methods: {
    openFilePicker() {
      this.$refs.fileInput.click()
    },
    handleFileUpload() {
      this.loadFiles(this.$refs.fileInput.files);
    },
    loadFiles(fileList) {
      if (this.loading == true || fileList.length == 0) {
        return;
      }
      this.loading = true;
      let waits = [];
      Array.from(fileList).forEach((file) => {
        waits.push(this.readFile(file).then((json) => {
          this.files.push(json)
        }));
      });
      Promise.all(waits).then(() => {
        this.loading = false;
      });
    },
    readFile(file) {
      return new Promise((resolve, reject) => {
        const reader = new FileReader()
        reader.onload = (e) => {
          // parse WebVTT file
          let lines = e.target.result.split('\n');
          let parsed = []; // be aware, this does not include the header

          for (let i = 0; i < lines.length; i++) {
            function collectTillNewLine() {
              let result = [];
              while (i < lines.length && lines[i].trim() !== '') {
                result.push(lines[i]);
                i++;
              }
              return result;
            }

            if (i === 0 && lines[i].includes('WEBVTT')) {
              continue;
            }

            // region parsing
            if (lines[i].includes('REGION')) {
              let region = {};
              let regionLines = collectTillNewLine();
              for (let j = 0; j < regionLines.length; j++) {
                let line = regionLines[j];
                if (line.includes(':')) {
                  let split = line.split(':');
                  region[split[0].trim()] = split[1].trim();
                }
              }
              parsed.push({
                type: 'region',
                data: region
              });
              continue;
            }

            // note parsing
            if (lines[i].includes('NOTE')) {
              let noteLines = collectTillNewLine();
              noteLines[0] = noteLines[0].substring(5).trimStart();
              parsed.push({
                type: 'note',
                data: noteLines.join('\n')
              });
              continue;
            }

            // style parsing
            if (lines[i].includes('STYLE')) {
              let styleLines = collectTillNewLine();
              styleLines.shift();
              parsed.push({
                type: 'style',
                data: styleLines.join('\n')
              });
              continue;
            }

            // cue parsing
            if (lines[i].includes('-->')) {
              let id = null;
              if (lines[i-1] != undefined && lines[i-1].trim() !== '') {
                id = lines[i-1].trim();
              }
              let cueLines = collectTillNewLine();
              let firstLine = cueLines.shift();
              let arrowIndex = firstLine.indexOf('-->');
              let cueStart = firstLine.substring(0, arrowIndex).trim();
              let cueEnd = firstLine.substring(arrowIndex + 3).trim();
              let settingsIndex = cueEnd.indexOf(' ');
              let cueSettings = '';
              if (settingsIndex != -1) {
                cueEnd = cueEnd.substring(0, settingsIndex).trim();
                cueSettings = cueEnd.substring(settingsIndex + 1).trim();
              }
              function parseCueTime(time) {
                let split = time.split(':');
                let hours, minutes, seconds, milliseconds;
                if (split.length === 2) {
                  hours = 0;
                  minutes = parseInt(split[0]);
                  let splitAgain = split[1].split('.');
                  seconds = parseInt(splitAgain[0]);
                  milliseconds = parseInt(splitAgain[1]);
                } else {
                  hours = parseInt(split[0]);
                  minutes = parseInt(split[1]);
                  let splitAgain = split[2].split('.');
                  seconds = parseInt(splitAgain[0]);
                  milliseconds = parseInt(splitAgain[1]);
                }
                return (hours * 60 * 60 + minutes * 60 + seconds) * 1000 + milliseconds;
              }

              parsed.push({
                type: 'cue',
                data: cueLines.join('\n'),
                id: id,
                start: parseCueTime(cueStart),
                end: parseCueTime(cueEnd),
                settings: cueSettings
              });
              continue;
            }
          }
          resolve({
            name: file.name,
            data: parsed,
            offset: 0,
            id: this.idCounter++
          })
        }
        reader.onerror = (e) => {
          reject(e)
        }
        reader.readAsText(file)
      })
    },
    exportFile() {
      let output = [];

      if (this.mergeType == 'concat') {
        let offset = this.globalOffset;
        for (let i = 0; i < this.files.length; i++) {
          let file = this.files[i];
          let lastEnd = 0;
          offset += file.offset;
          file.data.forEach((d) => {
            if (d.type === 'cue') {
              output.push({
                ...d,
                start: d.start + offset,
                end: d.end + offset
              });
              lastEnd = d.end;
            } else {
              if (d.type === 'note' && this.removeComments) return;
              if (d.type === 'style')  output.unshift(d);
              else output.push(d);
            }
          });
          offset += lastEnd;
        }
      } else if (this.mergeType == 'overlap') {
        let sections = [];
        for (let i = 0; i < this.files.length; i++) {
          let file = this.files[i];
          let offset = file.offset + this.globalOffset;
          let collector = [];
          while (true) {
            let next = file.data.shift();
            if (next == undefined) {
              if (collector.length > 0) {
                sections.push({
                  start: undefined,
                  end: undefined,
                  data: collector
                });
              }
              break;
            };
            if (next.type === 'cue') {
              collector.push({
                ...next,
                start: next.start + offset,
                end: next.end + offset
              });
              sections.push({
                start: next.start + offset,
                end: next.end + offset,
                data: collector
              });
              collector = [];
            } else {
              if (next.type === 'note' && this.removeComments) continue;
              collector.push(next);
            }
          }
          if (collector.length > 0) {
            sections.push({
              start: undefined,
              end: undefined,
              data: collector
            });
          }
        }

        sections.sort((a, b) => { // sort by start time and then by end time
          if (a.start == undefined && b.start == undefined) return 0;
          if (a.start == undefined) return 1;
          if (b.start == undefined) return -1;
          if (a.start == b.start) {
            if (a.end == undefined && b.end == undefined) return 0;
            if (a.end == undefined) return 1;
            if (b.end == undefined) return -1;
            return a.end - b.end;
          }
          return a.start - b.start;
        });

        // merge everything back together
        sections.forEach((section) => {
          section.data.forEach((d) => {
            output.push(d);
          });
        });
      }

      let outputString = 'WEBVTT\n\n';

      output.forEach((d) => {
        if (d.type === 'cue') {
          if (d.id != null) 
            outputString += d.id + '\n';
          outputString += this.formatTime(d.start) + ' --> ' + this.formatTime(d.end);
          if (d.settings != '')
            outputString += ' ' + d.settings;
          outputString += '\n';
          outputString += d.data + '\n\n';
        } else if (d.type === 'region') {
          outputString += 'REGION\n';
          Object.keys(d.data).forEach((key) => {
            outputString += key + ': ' + d.data[key] + '\n';
          });
          outputString += '\n';
        } else if (d.type === 'note') {
          outputString += 'NOTE\n';
          outputString += d.data + '\n\n';
        } else if (d.type === 'style') {
          outputString += 'STYLE\n';
          outputString += d.data + '\n\n';
        }
      });

      let blob = new Blob([outputString], {type: 'text/vtt'});
      let url = URL.createObjectURL(blob);
      let a = document.createElement('a');
      a.href = url;
      a.download = this.outputFilename + '.vtt';
      a.click();
      URL.revokeObjectURL(url);
    },
    formatTime(time) {
      let milliseconds = time % 1000;
      time = (time - milliseconds) / 1000;
      let seconds = time % 60;
      time = (time - seconds) / 60;
      let minutes = time % 60;
      let hours = (time - minutes) / 60;
      return this.pad(hours) + ':' + this.pad(minutes) + ':' + this.pad(seconds) + '.' + this.pad(milliseconds, 3);
    },
    pad(num, size = 2) {
      let s = num + '';
      while (s.length < size) s = '0' + s;
      return s;
    }
  }
}


</script>

<template>
<div class="loading" v-if="loading"></div>
<input type="file" ref="fileInput" @change="handleFileUpload" class="hidden" multiple accept=".vtt">
<div class="drag-n-drop-prompt" :class="{'drag-over': isDraggingOver}" v-if="files.length == 0">
  <div class="drag-n-drop-prompt__text">
    <h1>Drag and drop your WebVTT files here</h1>
    <p>or</p>
    <button class="button large" @click="openFilePicker">Select files</button>
  </div>
</div>
<div class="split-view" v-else>
  <draggable v-model="files" item-key="id" class="files" :class="{'drag-over': isDraggingOver}">
    <template #header>
      <div class="info" v-if="files.length > 1"><info class="icon" />You can drag&drop to rearrange the files</div>
    </template>
    <template #item="{element}">
      <div class="file" :title="element.name">
        <h3>{{ element.name }}</h3>
        <div class="form-group">
          <label for="offset">Offset</label>
          <div class="input-group">
            <div class="text" @click="element.offset -= 1">-</div>
            <input type="number" id="global-offset" v-model="element.offset" step="1" class="center">
            <div class="text small">ms</div>
            <div class="text" @click="element.offset += 1">+</div>
          </div>
        </div>
        <div class="delete-button" @click="files.splice(files.findIndex(files => files.id == element.id), 1)">
          <deleteIcon />
        </div>
      </div>
    </template>
    <template #footer>
      <div class="button" @click="openFilePicker">+ Add</div>
    </template>
  </draggable>
  <div class="export-settings">
    <div class="settings">
      <h1> Export Settings </h1>
      <div class="form-group">
        <label for="merge-type">Merge Type</label>
        <div class="input-group">
          <select id="merge-type" v-model="mergeType" ref="mergeTypeSelect">
            <option value="overlap">Overlap</option>
            <option value="concat">Concat</option>
          </select>
        </div>
        <div class="info" v-if="mergeType=='overlap'"><info class="icon" />Will keep all the times identical to the original files</div>
        <div class="info" v-if="mergeType=='concat'"><info class="icon" />Will append each file to the end of the previous one and add offset to the files so that they do not overlay</div>
      </div>
      <div class="form-group">
        <label for="global-offset">Global Offset</label>
        <div class="input-group">
          <div class="text" @click="globalOffset -= 1">-</div>
          <input type="number" id="global-offset" v-model="globalOffset" step="1" class="center">
          <div class="text small">ms</div>
          <div class="text" @click="globalOffset += 1">+</div>
        </div>
        <div class="info"><info class="icon" />Will add this offset to all the files</div>
      </div>
      <div class="form-group">
        <label for="output-filename">Output Filename</label>
        <div class="input-group">
          <input type="text" id="output-filename" v-model="outputFilename">
          <div class="text small">.vtt</div>
        </div>
      </div>
      <div class="form-group">
        <label for="remove-comments">Remove Comments</label>
        <div class="input-group">
          <checkboxChecked v-if="removeComments" @click="removeComments = false" />
          <checkboxUnchecked v-else @click="removeComments = true" />
        </div>
        <div class="info"><info class="icon" />Will remove all the comments from the output file</div>
      </div>
      <div class="form-group">
        <div class="info"><info class="icon" />Find out more about this project on <a href="https://github.com/leone25/vtt-merge"> Github</a></div>
      </div>
    </div>
    <div @click="exportFile" class="button">Save -></div>
  </div>
</div>
</template>

<style>
body, html {
	background-color: black;
	color: #fbfbfb;
	font-family: 'Roboto Mono', monospace;
  height: 100%;
  width: 100%;
  margin: 0;
}
a {
	color: #2677eb;
}
a:hover {
	color: #5c8acd;
}
a:visited {
	color: #1650a5;
}
a:active {
	color: #bbb;
}
#app {
	width: 100%;
  height: 100%;
}
.hidden {
  display: none;
}
* {
  scrollbar-width: thin;
  scrollbar-color: #aaa transparent;
}

/* Chrome, Edge, and Safari */
*::-webkit-scrollbar {
  width: 8px;
}

*::-webkit-scrollbar-track {
  background: transparent;
}

*::-webkit-scrollbar-thumb {
  background-color: #aaa;
  border-radius: 10px;
  border: 4px solid transparent;
}

.file {
  background-color: #1a1a1a;
  border-radius: 5px;
  margin: 10px;
  padding: 10px 30px;
  position: relative;
}

.file h3 {
  overflow: hidden;
  white-space: nowrap;
}

.file .delete-button {
  position: absolute;
  right: 0;
  bottom: 0;
  height: 24px;
  width: 24px;
  cursor: pointer;
  border-radius: 5px 0 5px 0;
  background: #222;
  padding: 10px;
}

.drag-n-drop-prompt {
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  outline: 20px dashed #082a5b;
  outline-offset: 40px;
  transition: outline-offset 0.3s cubic-bezier(0.68, -0.55, 0.27, 1.55);
}

.drag-n-drop-prompt.drag-over {
  outline-offset: -40px;
}

.drag-n-drop-prompt__text {
  text-align: center;
}

.split-view {
  display: grid;
  width: 100%;
  height: 100%;
  grid-template-columns: minmax(0, 1fr) max(200px, min(400px, 50%));
}

@media screen and (max-width: 800px) {
  .split-view {
    grid-template-columns: 100%;
    grid-template-rows: 50% 50%;
  }
}

.files {
  overflow-y: auto;
  transition: background-color 0.3s linear;
}

.files.drag-over {
  background-color: #111;
}

.export-settings {
  background: #1a1a1a;
  display: grid;
  grid-template-rows: 1fr auto;
  width: 100%;
}

.settings {
  overflow-y: auto; 
  padding: 20px;
}

.settings h1 {
  margin: 10px;
}

.form-group {
  margin: 25px 10px;
}

select, input {
  background-color: #101010;
  border: none;
  padding: 10px;
  color: #fbfbfb;
  width: 100%;
}

.input-group {
  overflow: hidden;
  border-radius: 5px;
  width: 100%;
  max-width: 200px;
  margin: 10px;
  display: flex;
}

.input-group .text {
  background-color: #101010;
  padding: 10px;
  color: #fbfbfb;
}

.input-group .text.small {
  color: #aaa;
  font-size: 12px;
  display: flex;
  align-items: center;
}

input::-webkit-outer-spin-button,
input::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}

input[type=number] {
  -moz-appearance: textfield;
}

.info {
  color: #aaa;
  font-size: 12px;
  margin: 5px 0;
  padding-left: 30px;
  position: relative;
}

.info .icon {
  position: absolute;
  left: 0;
  top: 0;
}

.center {
  text-align: center;
}

.button {
	background-color: #082a5b;
	border: none;
	padding: 23px 20px;
	margin: 10px;
	border-radius: 5px;
	color: #fbfbfb;
	text-align: center;
}
.button.large {
  font-size: 1.5em;
  font-weight: 400;
  padding: 20px 30px;
}
.loading {
	width: 100%;
	height: 100%;
	background-color: rgba(20, 20, 20, 0.7);
	position: fixed;
	top: 0;
	left: 0;
}
.loading:after {
	content: '';
	height: 100px;
	width: 100px;
	border-radius: 100px;
	border: solid 15px;
	border-color: white transparent;
	position: absolute;
	top: 0;
	bottom: 0;
	left: 0;
	right: 0;
	margin: auto;
	animation-name: rotate;
	animation-iteration-count: infinite;
	animation-duration: 1s;
}
@keyframes rotate {
	from {
		transform: rotate(0deg) scale(1);
		border-color: white transparent;
	}
	50% {
		transform: rotate(180deg) scale(0.9);
		border-color: #082a5b transparent;
	}
	to {
		transform: rotate(360deg);
		border-color: white transparent;
	}
}
</style>
