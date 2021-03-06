<template>
  <div class="typeahead-container">
    <div class="input-wrapper">
      <input
        type="text"
        placeholder="University"
        autocomplete="off"
        v-model="query"
        @keydown.down="down"
        @keydown.up="up"
        @keydown.tab="hit"
        @keydown.enter="processEnter"
        @input="update"
        @keydown.esc="reset"
        @blur="reset"
        @focus="update"
        :readonly="submitted" >
      <span
        class="fa"
        :class="typeaheadIndicatorClass"/>
    </div>
    <div class="uni-list-container">
      <div
        class="list-wrapper"
        v-show="hasItems">
        <div class="caret"/>
        <ul>
          <li
            v-for="(item, $item) in items"
            :key="item.name"
            :class="activeClass($item)"
            @mousedown="hit"
            @mousemove="setActive($item)">
            <span v-text="item.name"/>
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>

<script>
import VueTypeahead from 'vue-typeahead';
import universityList from './universities.json';
import createTrie from 'autosuggest-trie';

const universities = universityList.map(uni => ({ name: uni }));
const splitByHyphen = /\s+|-/;
const trie = createTrie(universities, 'name', { splitRegex: splitByHyphen });

export default {
  extends: VueTypeahead,
  props: ['submitted'],
  data() {
    return {
      src: '/findunis',
      selected: ''
    };
  },
  methods: {
    processEnter() {
      if (this.current !== -1) {
        this.onHit(this.items[this.current]);
      } else {
        this.$emit('pressed:enter');
      }
    },
    // The callback function which is triggered when the user hits on an item
    // (required)
    onHit(item) {
      this.reset();
      if (item) {
        this.query = item.name;
      }
    },
    reset() {
      this.items = [];
      this.current = -1;
      this.loading = false;
    },
    fetch() {
      let results;
      const trimmedQuery = this.query.trim();
      if (trimmedQuery === '') {
        results = [];
      } else {
        results = trie.getMatches(trimmedQuery, { limit: 15, splitRegex: splitByHyphen });
        const ranks = {};
        const normalizedQuery = trimmedQuery.split(splitByHyphen).join(' ').toLowerCase();
        for (const result of results) {
          const normalizedResult = result.name.split(splitByHyphen).join(' ').toLowerCase();
          if (normalizedResult === normalizedQuery) {
            ranks[result.name] = 3;
          } else if (normalizedResult.startsWith(normalizedQuery)) {
            ranks[result.name] = 2;
          } else if (normalizedResult.includes(normalizedQuery)) {
            ranks[result.name] = 1;
          } else {
            ranks[result.name] = 0;
          }
        }
        results = Object.keys(ranks)
          .sort((a, b) => ranks[b] - ranks[a]).slice(0, 4)
          .map(result => ({ name: result }));
        if (results.length === 1 && results[0].name === trimmedQuery) {
          results = [];
        }
      }
      return Promise.resolve({ data: results });
    }
  },
  computed: {
    typeaheadIndicatorClass() {
      if (this.query.trim() === '') {
        return ['icon-graduation-cap'];
      } else if (this.query.length < 8) {
        return ['icon-attention-circled'];
      } else {
        return ['icon-ok-circled'];
      }
    }
  },
  watch: {
    query(val) {
      this.$emit('update:query', val);
    }
  }
};
</script>
