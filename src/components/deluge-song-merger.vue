
<template>
  <div>
    <h4>Deluge Song Merger</h4>
    <p class="lead">Merge multiple Deluge (Synthstrom) songs into one single song</p>

    <div v-if="state.error">
      {{state.error}}
    </div>

    <div class="form-group">
      <label for="songFiles">Song Files</label>
      <input type="file" id="songFiles" name="files[]" multiple @change="onFileSelect" class="form-control-file">
      <small class="form-text text-muted">
        eg SONG000.XML and SONG001.XML
      </small>
    </div>
    <div v-if="state.files.length > 1" class="small mb-4">
      <div v-for="file in state.files" :key="file.name">
        {{file.name}}
        {{file.size}}
        {{file.type}}
      </div>
    </div>

    <div class="form-group">
      <label for="mergeinto">Merge into</label>
      <input type="file" id="mergeinto" name="mergeinto" @change="onMergeintoSelect" class="form-control-file">
      <small class="form-text text-muted">
        eg SONG000.XML or SONG001.XML
      </small>
    </div>

    <div class="form-group">
      <div v-for="(op,opidx) in ops" :key="opidx" class="form-check">
        <input v-model="op.active" type="checkbox" :id="'op-'+opidx" class="form-check-input">
        <label :for="'op-'+opidx" class="form-check-label">{{op.name}}</label>
      </div>
    </div>

    <button @click="mergeFiles" class="btn btn-primary">mergeFiles</button>

    <div v-if="state.output" class="mt-4 p-4 bg-light">
      <div>
        <a :href="state.outputDataURL" :download="state.outputFilename">save to disk</a>
        <small class="text-muted ml-1">(might not work for large files)</small>
      </div>

      <div class="mt-2">
        <textarea v-model="state.output" readonly="readonly" class="form-control" style="min-height:9rem"></textarea>
      </div>
    </div>

    <div class="small text-muted mt-4">
      * Works only for files written by firmware 3.0.0 or above
    </div>
  </div>
</template>

<script>

var SELECTOR_INSTRUMENT_CHILDS = 'song > instruments > sound, song > instruments > kit';
var SELECTOR_INSTRUMENT_CLIPS = 'song > sessionClips > instrumentClip';
var SELECTOR_SESSION_CLIPS = 'song > sessionClips';
var SELECTOR_INSTRUMENTS = 'song > instruments';

var NUMBER_OF_SECTIONS = 12;

export default {
  name: 'deluge-song-merger',

  data: function() {
    return {
      ops: [
        {
          name: 'mute all clips',
          active: true,
          fn: function(files) {
            for(var i = 0, len = files.length; i < len; i++) {
              files[i].doc.querySelectorAll(SELECTOR_INSTRUMENT_CLIPS).forEach(function(node){
                node.setAttribute('isPlaying', '0');
              });
            }
          }
        },
        {
          name: 'clips to one section per song',
          active: false,
          fn: function(files) {
            for(var i = 0, len = files.length; i < len; i++) {
              files[i].doc.querySelectorAll(SELECTOR_INSTRUMENT_CLIPS).forEach(function(node){
                node.setAttribute('section', i % NUMBER_OF_SECTIONS);
              });
            }
          }
        },
      ],

      state: {
        files: [],
        mergeInto: null,

        error: null,

        output: null,
        outputDataURL: null,
        outputFilename: null
      }
    };
  },

  methods: {
    onMergeintoSelect: function($event) {
      var file = $event.target.files[0];
      this.state.error = null;
      this.state.mergeInto = null;
      readAndParseXMLFile(file).then((file) => {
        this.state.mergeInto = file;
      }).catch((err) => {
        console.error(err.stack||err);
        this.state.error = err;
      });
    },

    onFileSelect: function($event) {
      this.state.error = null;
      this.state.files.splice(0, this.state.files.length);

      var files = $event.target.files;
      var promises = [];
      for(let i = 0, len = files.length, f; i < len, f = files[i]; i++) {
        promises.push(readAndParseXMLFile(f));
      }

      Promise.all(promises).then((files) => {
        this.state.files.splice.apply(this.state.files, [0, this.state.files.length].concat(files));
      }).catch((err) => {
        console.error(err.stack||err);
        this.state.error = err;
      });
    },

    mergeFiles: function() {
      var files = this.state.files;
      var mergeInto = this.state.mergeInto;

      for(let i = 0, len = this.ops.length, op; i < len; i++) {
        op = this.ops[i];
        if(op.active) {
          op.fn.call(this, files, mergeInto);
        }
      }

      var instrumentChilds = [];
      var instrumentClips = [];
      for(let i = 0, len = files.length, f; i < len, f = files[i]; i++) {
        instrumentChilds.push.apply(instrumentChilds, f.doc.querySelectorAll(SELECTOR_INSTRUMENT_CHILDS));
        instrumentClips.push.apply(instrumentClips, f.doc.querySelectorAll(SELECTOR_INSTRUMENT_CLIPS));
      }

      var instruments = mergeInto.doc.querySelector(SELECTOR_INSTRUMENTS);
      var sessionClips = mergeInto.doc.querySelector(SELECTOR_SESSION_CLIPS);

      while(instruments.childNodes.length) {
        instruments.removeChild(instruments.firstChild);
      }
      while(sessionClips.childNodes.length) {
        sessionClips.removeChild(sessionClips.firstChild);
      }

      for(let i = 0, len = instrumentChilds.length; i < len; i++) {
        instruments.appendChild(instrumentChilds[i]);
      }
      for(let i = 0, len = instrumentClips.length; i < len; i++) {
        sessionClips.appendChild(instrumentClips[i]);
      }

      console.log(mergeInto.doc);

      var serializer = new XMLSerializer();
      var output = serializer.serializeToString(mergeInto.doc);
      this.state.output = output;
      this.state.outputDataURL = 'data:text/xml;charset=utf-8,' + encodeURIComponent(output);
      this.state.outputFilename = 'SONG.XML';
    },
  }
}

function readAndParseXMLFile(theFile) {
  return new Promise(function(resolve, reject) {
    var reader = new FileReader();

    reader.onload = (e) => {
      var content = e.target.result;

      var parser = new DOMParser();
      try { // dunno if this actually throws
        var doc = parser.parseFromString(content, 'text/xml');
      } catch(ex) {
        return reject(ex);
      }
      console.log(doc);

      resolve({
        name: theFile.name,
        size: theFile.size,
        type: theFile.type,
        doc: doc,
      });
    };

    reader.readAsText(theFile);
  });
}

</script>
