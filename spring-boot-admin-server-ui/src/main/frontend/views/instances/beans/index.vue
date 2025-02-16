<!--
  - Copyright 2014-2019 the original author or authors.
  -
  - Licensed under the Apache License, Version 2.0 (the "License");
  - you may not use this file except in compliance with the License.
  - You may obtain a copy of the License at
  -
  -     http://www.apache.org/licenses/LICENSE-2.0
  -
  - Unless required by applicable law or agreed to in writing, software
  - distributed under the License is distributed on an "AS IS" BASIS,
  - WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  - See the License for the specific language governing permissions and
  - limitations under the License.
  -->

<template>
  <section class="section" :class="{isLoading: isLoading}">
    <sba-alert v-if="error" :error="error" :title="$t('instances.beans.fetch_failed')" />

    <div class="field">
      <p class="control is-expanded has-icons-left">
        <input
          class="input"
          type="search"
          v-model="filter"
        >
        <span class="icon is-small is-left">
          <font-awesome-icon icon="filter" />
        </span>
      </p>
    </div>
    <template v-for="context in filteredContexts">
      <sba-panel :title="context.name" :header-sticks-below="['#navigation']" :key="context.name">
        <beans-list :beans="context.beans" :key="`${context.name}-beans`" />
      </sba-panel>
    </template>
  </section>
</template>

<script>
  import Instance from '@/services/instance';
  import {compareBy} from '@/utils/collections';
  import shortenClassname from '@/utils/shortenClassname';
  import BeansList from '@/views/instances/beans/beans-list';
  import isEmpty from 'lodash/isEmpty';
  import {VIEW_GROUP} from '../../index';
  import SbaPanel from '@/components/sba-panel';

  class Bean {
    constructor(name, bean) {
      Object.assign(this, bean);
      this.name = name;
      this.shortName = shortenClassname(this.name, 80);
      this.shortType = shortenClassname(this.type, 80);
    }
  }

  const flattenBeans = beans => {
    return Object.keys(beans)
      .map((key) => {
        return new Bean(key, beans[key]);
      });
  };

  const flattenContexts = beanData => {
    if (isEmpty(beanData.contexts)) {
      return [];
    }
    return Object.keys(beanData.contexts)
      .map((key) => ({
        beans: flattenBeans(beanData.contexts[key].beans),
        name: key,
        parent: beanData.contexts[key].parentId
      }));
  };

  export default {
    components: {SbaPanel, BeansList},
    props: {
      instance: {
        type: Instance,
        required: true
      }
    },
    data: () => ({
      isLoading: false,
      error: null,
      contexts: [],
      filter: '',
    }),
    computed: {
      filteredContexts() {
        const filterFn = this.getFilterFn();
        return this.contexts.map(ctx => ({
          ...ctx,
          beans: ctx.beans.filter(filterFn).sort(compareBy(bean => bean.name))
        }));
      }
    },
    methods: {
      getFilterFn() {
        if (!this.filter) {
          return () => true;
        }
        const regex = new RegExp(this.filter, 'i');
        return bean => (bean.name.match(regex) || bean.aliases.some(alias => alias.match(regex)));
      },
      async fetchBeans() {
        this.error = null;
        this.isLoading = true;
        try {
          const res = await this.instance.fetchBeans();
          this.contexts = flattenContexts(res.data);
        } catch (error) {
          console.warn('Fetching beans failed:', error);
          this.error = error;
        }
        this.isLoading = false;
      }
    },
    created() {
      this.fetchBeans();
    },
    install({viewRegistry}) {
      viewRegistry.addView({
        name: 'instances/beans',
        parent: 'instances',
        path: 'beans',
        label: 'instances.beans.label',
        group: VIEW_GROUP.INSIGHTS,
        component: this,
        order: 110,
        isEnabled: ({instance}) => instance.hasEndpoint('beans')
      });
    }
  }
</script>

