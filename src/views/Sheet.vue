<template lang="pug">
  .DataSheet-ct
    DataSheet(:records="mergedRecords")
    DataSheetLog(:records="mergedRecords" @commit="onCommit" @upload="onUpload")
</template>

<script>
import { mapState, mapActions, mapMutations } from 'vuex';
import { uniqueNamesGenerator } from 'unique-names-generator';

import DataSheet from '@/components/DataSheet.vue';
import DataSheetLog from '@/components/DataSheetLog.vue';

export default {
  components: {
    DataSheet,
    DataSheetLog,
  },
  props: {
    srcRef: {
      type: String,
      default: 'master',
    },
    dstRef: {
      type: String,
      default: null,
    },
  },
  computed: {
    ...mapState(['records', 'diffs']),
    keyedDiffs () {
      return this.diffs.reduce((accum, item) => {
        accum[item._id] = item;
        return accum;
      }, {});
    },
    keyedRecords () {
      return this.records.reduce((accum, item) => {
        const { _id, ...value } = item
        accum[_id] = {
          _id,
          status: null,
          value,
        };
        return accum;
      }, {});
    },
    mergedRecords () {
      const diffsKeys = Object.keys(this.keyedDiffs);
      const recordsKeys = Object.keys(this.keyedRecords);
      const mergedKeys = new Set([...diffsKeys, ...recordsKeys]);

      const merged = Array.from(mergedKeys).reduce((accum, key) => {
        accum[key] = {
          ...this.keyedRecords[key],
          ...this.keyedDiffs[key],
        };
        return accum;
      }, {});
      return Object.values(merged);
    },
  },
  watch: {
    srcRef: {
      immediate: true,
      handler: 'fetch',
    },
    dstRef: 'fetch',
  },
  methods: {
    ...mapActions([
      'getRecords',
      'getDiffs',
      'merge',
      'import',
    ]),
    ...mapMutations({
      resetSheet: 'RESET_SHEET',
    }),
    async fetch () {
      try {
        this.resetSheet();
        await this.getRecords(this.srcRef);
        if (this.dstRef) {
          const { srcRef, dstRef } = this
          await this.getDiffs({ srcRef, dstRef });
        }
      } catch (err) {
        console.error(err.message);
      }
    },
    async onCommit (msg) {
      const srcRef = this.srcRef;
      const dstRef = this.dstRef;

      try {
        await this.merge({ srcRef, dstRef });
        this.$router.push(`/${srcRef}`);
      } catch (err) {
        console.error(err.message);
      }
    },
    async onUpload (file) {
      const srcRef = this.dstRef || this.srcRef;
      const branch = this.generateBranchName();

      try {
        await this.import({ srcRef, file, branch });
        this.$router.push(`/${srcRef}/compare/${branch}`);
      } catch (err) {
        console.error(err.message);
      }
    },
    generateBranchName () {
      return uniqueNamesGenerator({ separator: '-' });
    },
  },
};
</script>

<style lang="postcss">
.DataSheet-ct {
  @apply border-t overflow-hidden;
  display: grid;
  grid-template-columns: 1fr 20rem;
}
</style>