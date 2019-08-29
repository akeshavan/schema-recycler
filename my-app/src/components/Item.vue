<template>
  <b-card class="mt-1 mb-1 item">
    <!-- <SurveyItem :item="{'@id': item.url}" /> -->
    <b-badge>{{item.ui ? item.ui.inputType : null}}</b-badge>
    <!-- {{item}} -->
    <InputSelector
      v-if="status === 'ready'"
      :inputType="item.ui.inputType"
      :readOnly="false"
      :title="title"
      :valueConstraints="valueConstraints"
      :selected_language="'en'"
      :preamble="item.preamble"
      :showPassOptions="{}"
      :responses={}
    />
    <div v-else>
      {{status}}
    </div>
  </b-card>
</template>

<style>
  .item {
    cursor: pointer;
  }
</style>

<script>
// import SurveyItem from '@bit/akeshavan.mindlogger-web.survey-item';
import jsonld from 'jsonld/dist/jsonld.min';
import InputSelector from '@bit/akeshavan.mindlogger-web.input-selector';


export default {
  name: 'itemView',
  props: {
    item: {
      type: Object,
    },
  },
  components: {
    InputSelector,
  },
  computed: {
    title() {
      return this.item.question ? this.item.question.en || this.item.question : '';
    },
    valueConstraints() {
      if (this.status === 'ready') {
        return this.expanded[0]['https://schema.repronim.org/valueconstraints'][0];
      }
      return [];
    },
  },
  data() {
    return {
      status: 'loading',
      expanded: [],
    };
  },
  methods: {
    async expandLD(item) {
      return jsonld.expand(item);
    },
  },
  mounted() {
    this.expandLD(this.item).then((resp) => {
      this.expanded = resp;
      const val = this.expanded[0]['https://schema.repronim.org/valueconstraints'][0];
      // eslint-disable-next-line
      console.log('VAL IS', val);
      if (val['@id']) {
        this.status = 'loading';
        this.expandLD(val['@id']).then((resp1) => {
          this.expanded[0]['https://schema.repronim.org/valueconstraints'] = resp1;
          this.status = 'ready';
          // eslint-disable-next-line
          console.log('THEN', this.expanded[0]['https://schema.repronim.org/valueconstraints']);
        });
      } else {
        this.status = 'ready';
      }
    }).catch(() => {
      this.status = 'error';
    });
  },
};
</script>