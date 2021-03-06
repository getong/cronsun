<style scope>
  .clearfix:after {content:""; clear:both; display:table;}
  .kill-proc-btn { color:red;cursor: pointer;}
</style>
<template>
  <div>
    <div class="clearfix" style="margin-bottom: 20px;">
      <router-link class="ui left floated button" to="/job">{{$L('view job list')}}</router-link>
      <button class="ui left floated icon button" v-on:click="refresh"><i class="refresh icon"></i></button>
      <router-link class="ui right floated primary button" to="/job/create"><i class="add to calendar icon"></i> {{$L('create job')}}</router-link>
    </div>
    <form class="ui form" v-bind:class="{loading:loading}" v-on:submit.prevent>
      <div class="field">
        <label>{{$L('job ID')}}</label>
        <input type="text" ref="ids" v-model:value="ids" :placeholder="$L('multiple IDs can separated by commas')"/>
      </div>
      <div class="field">
        <label>{{$L('select groups')}}</label>
        <Dropdown :title="$L('select groups')" v-bind:items="prefetchs.groups" v-on:change="changeGroup" :selected="groups" :multiple="true"/>
      </div>
      <div class="field">
        <label>{{$L('select nodes')}}</label>
        <Dropdown :title="$L('select nodes')" v-bind:items="$store.getters.dropdownNodes" v-on:change="changeNodes" :selected="nodes" :multiple="true"/>
      </div>
      <div class="field">
        <button class="fluid ui button" type="button" v-on:click="submit">{{$L('submit query')}}</button>
      </div>
    </form>
    <table class="ui hover blue table" v-if="executings.length > 0">
      <thead>
        <tr>
          <th class="center aligned">{{$L('job ID')}}</th>
          <th width="200px" class="center aligned">{{$L('job group')}}</th>
          <th class="center aligned">{{$L('node')}}</th>
          <th class="center aligned">{{$L('process ID')}}</th>
          <th class="center aligned">{{$L('starting time')}}</th>
          <th class="center aligned">{{$L('operation')}}</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(proc, index) in executings">
          <td class="center aligned"><router-link :to="'/job/edit/'+proc.group+'/'+proc.jobId">{{proc.jobId}}</router-link></td>
          <td class="center aligned">{{proc.group}}</td>
          <td class="center aligned">{{$store.getters.hostshows(proc.nodeId)}}</td>
          <td class="center aligned">{{proc.id}}</td>
          <td class="center aligned">{{proc.time}}</td>
          <td class="center aligned"><a class="kill-proc-btn" v-on:click="killProc(proc, index)">{{$L('kill process')}}</a></td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script>
import Dropdown from './basic/Dropdown.vue';
import {split} from '../libraries/functions';

export default {
  name: 'job-executing',
  data(){
    return {
      prefetchs: {groups: []},
      loading: false,
      groups: [],
      ids: '',
      nodes: [],
      executings: []
    }
  },
  
  mounted(){
    var vm = this;
    this.groups = split(this.$route.query.groups, ',');
    this.nodes = split(this.$route.query.nodes, ',');
    this.ids = this.$route.query.ids || '';

    this.$rest.GET('job/groups').onsucceed(200, (resp)=>{
      !resp.includes('default') && resp.unshift('default');
      vm.prefetchs.groups = resp;
      this.fetchList(this.buildQuery());
    }).do();
  },

  watch: {
    '$route': function(){
      this.groups = split(this.$route.query.groups, ',');
      this.nodes = split(this.$route.query.nodes, ',');
      this.ids = this.$route.query.ids || '';
      this.fetchList(this.buildQuery());
    }
  },

  methods: {
    changeGroup(val, text){
      this.groups = split(val, ',');
    },

    changeNodes(val){
      this.nodes = split(val, ',');
    },

    submit(){
      this.$router.push('/job/executing?'+this.buildQuery());
    },

    killProc(proc, index) {
      if (!confirm(this.$L('whether to kill the process'))) return;
      var vm = this
      var params = []
      params.push('node='+proc.nodeId)
      params.push('group='+proc.group)
      params.push('job='+proc.jobId)
      params.push('pid='+proc.id)
      this.$rest.DELETE('job/executing?' + params.join('&'))
      .onsucceed(200, (resp) => vm.$bus.$emit('success', vm.$L('command has been sent to the node')))
      .onfailed((resp) => vm.$bus.$emit('error', resp))
      .do();
    },

    buildQuery(){
      var params = [];
      if (this.groups && this.groups.length > 0) params.push('groups='+this.groups.join(','));
      if (this.nodes && this.nodes.length > 0) params.push('nodes='+this.nodes.join(','));
      if (this.ids) params.push('ids='+this.ids);
      return params.join('&');
    },

    fetchList(query){
      var vm = this;
      this.loading = true;
      this.$rest.GET('job/executing?'+query).
      onsucceed(200, (resp)=>{
        vm.executings = resp;
        vm.$nextTick(()=>{
          $(vm.$el).find('table .ui.dropdown').dropdown();
        });
      }).
      onend(()=>{vm.loading = false}).
      do();
    },

    refresh(){
      this.fetchList(this.buildQuery());
    }
  },

  components: {
    Dropdown
  }
}
</script>
