<template>
  <div>
    <h1>QMK API Keyboard Status</h1>
    <input
      v-model="filter"
      class="keyboard-filter"
      placeholder="filter keyboards"
      @input="filterKeyboards"
    />
    <div v-show="loading" class="loading">
      <font-awesome-icon icon="atom" spin size="6x" fixed-width />
      <h2>Loading Data</h2>
    </div>
    <div v-if="!loading">
      <h5>Loaded in {{ loadtime / 1000 }} seconds</h5>
      <h3>Builds Failing ({{ sortedKeyboards.badlist.length }})</h3>
      <div class="build-status">
        <div
          v-for="k in sortedKeyboards.badlist"
          :key="sortedKeyboards.bad[k].key"
          :title="sortedKeyboards.bad[k].title"
          class="keyboard-status keyboard-failed"
          @click="showErrors(k)"
        >
          <a :href="sortedKeyboards.bad[k].url">
            {{ sortedKeyboards.bad[k].name }}
          </a>
          <br />
          {{ sortedKeyboards.bad[k].layout }}
        </div>
      </div>
      <h3>Builds Passing ({{ sortedKeyboards.goodlist.length }})</h3>
      <div class="build-status">
        <div
          v-for="k in sortedKeyboards.goodlist"
          :key="sortedKeyboards.good[k].key"
          :title="sortedKeyboards.good[k].title"
          class="keyboard-status keyboard-passed"
          @click="showErrors(k)"
        >
          <a :href="sortedKeyboards.good[k].url">
            {{ sortedKeyboards.good[k].name }}
          </a>
          <br />
          {{ sortedKeyboards.good[k].layout }}
        </div>
      </div>
      <div v-show="showErrorPane" class="error-pane" @click="hideErrors">
        <div class="error-title">
          <h4>Detailed Error Log</h4>
        </div>
        <div class="error-text">
          <pre>
         {{ errorLog }}
        </pre
          >
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios';
import reduce from 'lodash/reduce';
import keys from 'lodash/keys';

export default {
  name: 'ErrorReport',
  components: {},
  data: () => {
    return {
      loading: true,
      keyboards: {},
      sortedKeyboards: {
        good: {},
        bad: {},
        goodlist: [],
        badlist: []
      },
      buildLog: {},
      errorLog: '',
      showErrorPane: false,
      filter: '',
      loadtime: 0
    };
  },
  mounted() {
    const start = performance.now();
    axios
      .get('https://api.qmk.fm/v1/keyboards/build_log')
      .then(res => {
        if (res.status === 200) {
          this.buildLog = res.data;
          this.sortKeyboards();
        }
      })
      .then(() => {
        this.loading = false;
        this.loadtime = performance.now() - start;
      });
  },
  methods: {
    showErrors(key) {
      this.errorLog = this.buildLog[key].message;
      this.showErrorPane = true;
    },
    hideErrors() {
      this.showErrorPane = false;
    },
    filterKeyboards() {
      this.sortKeyboards();
    },
    sortKeyboards() {
      let obj = reduce(
        this.buildLog,
        (acc, value, key) => {
          let split = key.lastIndexOf('/');
          let name = key.slice(0, split);
          if (
            this.filter !== '' &&
            !name.toLowerCase().includes(this.filter.toLowerCase())
          ) {
            // filter out keyboards based on their name
            return acc;
          }
          let lastTested = new Date(value.last_tested * 1000);
          key = {
            name,
            key: key,
            layout: key.slice(split + 1, key.length),
            compiler_output: value.message,
            url: `https://github.com/qmk/qmk_firmware/tree/master/keyboards/${name}`
          };
          key.title = `Last Tested: ` + lastTested.toISOString();
          if (value.works) {
            acc.good[key.key] = key;
          } else {
            acc.bad[key.key] = key;
          }
          return acc;
        },
        { good: {}, bad: {}, goodlist: [], badlist: [] }
      );
      obj.goodlist = keys(obj.good).sort();
      obj.badlist = keys(obj.bad).sort();
      this.sortedKeyboards = obj;
    }
  }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.build-status {
  display: grid;
  grid-template: 1fr / 19% 19% 19% 19% 19%;
}
.keyboard-status {
  margin: 5px;
  padding: 5px;
  text-overflow: ellipsis;
  overflow: hidden;
  white-space: nowrap;
  border-radius: 8px;
}
.keyboard-passed {
  background: #2e8b57;
  color: #eee;
}
.keyboard-failed {
  background: #b22222;
  color: #eee;
  cursor: help;
}
.keyboard-passed > a {
  color: #eee;
}
.keyboard-failed > a {
  color: #eee;
}
.keyboard-filter {
  padding: 10px;
  width: 20%;
  font-size: 1.1rem;
}
.error-pane {
  position: fixed;
  top: 3%;
  display: grid;
  left: 0;
  right: 0;
  margin: auto;
  width: 80%;
  height: 90%;
  grid-template: 50px 1fr / 100%;
  background: #fffff0f5;
  overflow-y: scroll;
  border-radius: 5px;
}

.error-title {
  grid-column: 1;
  grid-row: 1;
}
.error-text {
  grid-column: 1;
  grid-row: 2;
  cursor: pointer;
}

@media (max-width: 640px) {
  .build-status {
    display: grid;
    grid-template: 1fr / 48% 48%;
  }
  .keyboard-filter {
    width: 80%;
  }
}
.loading {
  margin-top: 1em;
}
</style>
