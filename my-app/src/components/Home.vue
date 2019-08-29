<template>
  <div class="home mb-3 ml-3 mr-3">
  <b-row class="mb-3 pb-3">
    <b-col>
      <label>GitHub User</label><b-input v-model="gh_user"></b-input>
    </b-col>
    <b-col>
      <label>GitHub Repo</label><b-input v-model="gh_repo"></b-input>
    </b-col>
  </b-row>

  <b-row>
    <b-col cols="4" id="itemList">
      <h4>Items</h4>
      <b-input v-model="search" class="mb-3" placeholder="search"></b-input>
      <draggable
        class="dragArea list-group"
        :list="items"
        :group="{ name: 'people', pull: 'clone', put: false }"
        @change="log"
      >
        <div v-for="(item, url) in items" :key="url">
          <Item  :item="allItems[item.ref]">
          </Item>
        </div>
      </draggable>
    </b-col>

    <b-col cols="4" id="itemList">
      <h4>Activity</h4>
      <b-row>
        <b-col cols="8">
          <b-input v-model="activityName" class="mb-3" placeholder="search"></b-input>
        </b-col>
        <b-col cols="4">
          <b-button @click="saveActivity">Save</b-button>
        </b-col>
      </b-row>
      
      <draggable
        class="dragArea list-group"
        :list="itemList"
        group="people"
        @change="log"
      >
      <div

        v-for="(element, index) in itemList"
        :key="`act-${index}`"
      >
        <b-button variant="danger" size="sm" @click="removeItem(index)">x</b-button>
        <b-button variant="primary" size="sm" @click="moveItemUp(index)" v-if="index">^</b-button>
        <b-button variant="primary" size="sm" @click="moveItemDown(index)" v-if="index < itemList.length - 1">v</b-button>


        <Item  :item="allItems[element.ref]">
        </Item>
      </div>
    </draggable>
    </b-col>

    <b-col cols="4" id="activitySetList">
      <h4>Activity Set</h4>
        <b-row>
          <b-col cols="8">
            <b-input v-model="activitySetName" class="mb-3" placeholder="search"></b-input>
          </b-col>
          <b-col cols="4">
            <b-button :disabled="!userInfo.login" @click="pushToGitHub">Push</b-button>
          </b-col>
        </b-row>
        <b-row>

          <h6>activity set context</h6>
          <br>
        <vue-json-pretty
          :path="'res'"
          :data="formattedActivitySet.context"
          >
        </vue-json-pretty>

        <h6 class="mt-3">activity set schema</h6>
        <br>
        <vue-json-pretty
          :path="'res'"
          :data="formattedActivitySet.schema"
          >
        </vue-json-pretty>

        <b-card class="w-100 mt-3" v-for="(act, index) in activityList" :key="`appact_${index}`">
          <h5> {{act.name}} </h5>
          
          <h6 class="mt-3">activity context</h6>
          <br>
          <vue-json-pretty
            :path="'res'"
            :data="formatActivity(act).context"
            >
          </vue-json-pretty>

          <h6 class="mt-3">activity schema</h6>
          <br>
          <vue-json-pretty
            :path="'res'"
            :data="formatActivity(act).schema"
            >
          </vue-json-pretty>

        </b-card>
      </b-row>
    </b-col>
  </b-row>

  </div>


</template>

<style>
  .dragArea {
    min-height: 200px;
    padding: 10px;
    background: rgba(23, 162, 184, 0.15);
    border-radius: 5px;
    /* border-style: solid; */
  }

  .blah {
    border-width: 0;
  }

  #itemList {
    max-height: 80VH;
    overflow: scroll;
  }

  #itemList {
    max-height: 80VH;
    overflow: scroll;
  }

  #activitySetList {
    max-height: 80VH;
    overflow: scroll;
  }

</style>

<script>
import elasticlunr from 'elasticlunr';
import draggable from 'vuedraggable';
import VueJsonPretty from 'vue-json-pretty';
import _ from 'lodash';
import axios from 'axios';
import Item from './Item';
import items from '../assets/itemDump.json';

const index = elasticlunr(function el() {
  this.addField('question');
  this.addField('preamble');
  this.addField('altLabel');
  this.addField('prefLabel');
  this.setRef('id');
});

_.map(items, (item, i) => index.addDoc({ ...item, id: i }));

export default {
  name: 'Home',
  props: {
    userInfo: {
      type: Object,
    },
  },
  components: {
    Item,
    draggable,
    VueJsonPretty,
  },
  data() {
    return {
      index,
      search: 'sleep',
      allItems: items,
      itemList: [],
      activityList: [],
      activityName: 'new_activity',
      activitySetName: 'new_activity_set',
      activitSetDetails: {
        prefLabel: '',
        altLabel: '',
        description: '',
        about: '',
        image: '',
      },
      gh_user: 'akeshavan',
      gh_repo: 'openhumans-mindlogger-survey',
    };
  },
  computed: {
    items() {
      return this.index.search(this.search);
    },
    formattedActivitySet() {
      return this.formatActivitySet();
    },
  },
  methods: {
    log(evt) {
      window.console.log(evt);
      this.$forceUpdate();
    },

    saveActivity() {
      const activity = { name: this.activityName, items: this.itemList };
      this.activityList.push(activity);
    },

    formatActivitySet() {
      const context = {
        '@context': {
          '@version': 1.1,
          activity_path: `https://raw.githubusercontent.com/${this.gh_user}/${this.gh_repo}/master/activities/`,
        },
      };

      const schema = {
        '@context': [
          'https://raw.githubusercontent.com/ReproNim/schema-standardization/master/contexts/generic.jsonld',
          `https://raw.githubusercontent.com/${this.gh_user}/${this.gh_repo}/master/activitySets/${this.activitySetName}/${this.activitySetName}_context.jsonld`,
        ],
        '@type': 'https://raw.githubusercontent.com/ReproNim/schema-standardization/master/schemas/ActivitySet.jsonld',
        'schema:schemaVersion': '0.0.1',
        'schema:version': '0.0.1',
        '@id': this.activitySetName,
        'skos:prefLabel': this.activitSetDetails.prefLabel,
        'skos:altLabel': this.activitSetDetails.altLabel,
        'schema:description': this.activitSetDetails.description,
        'schema:about': this.activitSetDetails.about,
        'schema:image': this.activitSetDetails.image,
        ui: {
          order: [],
          visibility: {},
        },
        variableMap: [],
      };

      _.map(this.activityList, (act) => {
        context['@context'][act.name] = {
          '@id': `activity_path:${act.name}/${act.name}.jsonld`,
          '@type': '@id',
        };

        schema.ui.order.push(act.name);
        schema.ui.visibility[act.name] = true;
        schema.variableMap.push({
          variableName: act.name,
          isAbout: act.name,
        });
      });

      return { context, schema };
    },

    formatActivity(act) {
      const schema = {
        '@context': [
          'https://raw.githubusercontent.com/ReproNim/schema-standardization/master/contexts/generic.jsonld',
          `https://raw.githubusercontent.com/${this.gh_user}/${this.gh_repo}/master/activities/${act.name}/${act.name}_context.jsonld`,
        ],
        '@type': 'https://raw.githubusercontent.com/ReproNim/schema-standardization/master/schemas/Activity.jsonld',
        'schema:schemaVersion': '0.0.1',
        'schema:version': '0.0.1',
        '@id': act.name,
        'skos:prefLabel': '',
        'skos:altLabel': '',
        'schema:description': '',
        preamble: '',
        scoringLogic: {
        },
        ui: {
          order: [],
          visibility: {},
        },
        variableMap: [],
      };

      const context = {
        '@context': {
          '@version': 1.1,
        },
      };

      _.map(act.items, (item) => {
        const name = this.allItems[item.ref]['skos:altLabel'];
        context['@context'][name] = {
          '@id': this.allItems[item.ref].url,
          '@type': '@id',
        };

        schema.ui.order.push(name);
        schema.ui.visibility[name] = true;
        schema.variableMap.push({
          variableName: name,
          isAbout: name,
        });
      });

      return { schema, context };
    },

    async getFileContents(path) {
      const token = this.userInfo.token;
      const url = `https://api.github.com/repos/${this.gh_user}/${this.gh_repo}/contents/${path}`;
      return axios.get(url,
        {
          headers: {
            Authorization: `token ${token}`,
          },
        });
    },

    async updateFileContents(path, sha, content) {
      const token = this.userInfo.token;
      const url = `https://api.github.com/repos/${this.gh_user}/${this.gh_repo}/contents/${path}`;
      const data = {
        message: `creating file ${path}`,
        content: btoa(JSON.stringify(content)),
      };

      if (sha) {
        data.sha = sha;
      }

      return axios({
        url,
        method: 'put',
        data,
        headers: {
          Authorization: `token ${token}`,
        },
      });
    },

    async pushLDToGitHub(path, content) {
      return this.getFileContents(path).then((resp) => {
        this.updateFileContents(path, resp.data.sha, content);
      }).catch(() => {
        this.updateFileContents(path, null, content);
      });
    },

    async pushSchemaAndContext(schemaPath, schema, contextPath, context) {
      return this.pushLDToGitHub(schemaPath, schema).then(() => {
        setTimeout(() => this.pushLDToGitHub(contextPath, context), 2000);
      });
    },

    pushToGitHub() {
      // eslint-disable-next-line
      console.log('hello, pushing...');

      const schemaPath = `activitySets/${this.activitySetName}/${this.activitySetName}_schema.jsonld`;
      const contextPath = `activitySets/${this.activitySetName}/${this.activitySetName}_context.jsonld`;

      const { context, schema } = this.formatActivitySet();

      this.pushSchemaAndContext(schemaPath, schema, contextPath, context);


      _.map(this.activityList, (act, idx) => {
        const { contextAct, schemaAct } = this.formatActivity(act);
        const schemaActPath = `activities/${act.name}/${act.name}_schema.jsonld`;
        const contextActPath = `activities/${act.name}/${act.name}_context.jsonld`;

        setTimeout(() =>
          this.pushSchemaAndContext(schemaActPath, schemaAct, contextActPath, contextAct),
          (idx + 2) * 3500);
      });
    },
    // eslint-disable-next-line
    removeItem(index) {
      // TODO: remove an item from itemList by its index
    },
    // eslint-disable-next-line
    moveItemUp(index) {
      // TODO: move an item up
    },
    // eslint-disable-next-line
    moveItemDown(index) {
      // TODO: move an item down
    },
  },
};
</script>

<!-- Add 'scoped' attribute to limit CSS to this component only -->
<style scoped>
h1, h2 {
  font-weight: normal;
}
/* ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
} */
a {
  color: #42b983;
}
.home {
  text-align: left;
}

.home h1 {
  text-align: center;
}

</style>
