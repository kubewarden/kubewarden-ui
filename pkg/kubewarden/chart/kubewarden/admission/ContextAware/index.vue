<script>
import { mapGetters } from 'vuex';
import { _CREATE, _VIEW } from '@shell/config/query-params';
import { removeAt } from '@shell/utils/array';

import { Banner } from '@components/Banner';
import Loading from '@shell/components/Loading';

import { isEmpty } from '@shell/utils/object';
import { KUBEWARDEN_APPS } from '../../../../types';
import { ARTIFACTHUB_PKG_ANNOTATION } from '../../../../plugins/kubewarden-class';
import Resource from './Resource';

export default {
  name: 'ContextAware',

  props: {
    mode: {
      type:    String,
      default: _CREATE
    },
    value: {
      type:     Object,
      required: true
    }
  },

  components: {
    Banner, Loading, Resource
  },

  async fetch() {
    await this.$store.dispatch(`${ this.currentProduct.inStore }/findAll`, { type: 'apigroup' });

    this.contextAwareResources = [];

    if ( !!this.value?.policy ) {
      this.contextAwareResources = this.value.policy?.spec?.contextAwareResources;
    }
  },

  data() {
    return { contextAwareResources: null };
  },

  computed: {
    ...mapGetters(['currentProduct']),

    apiGroups() {
      return this.$store.getters[`${ this.currentProduct.inStore }/all`]('apigroup');
    },

    /** Return the group ID and groupVersion for each apiGroup */
    apiGroupVersions() {
      const out = [];

      if ( !isEmpty(this.apiGroups) ) {
        this.apiGroups.forEach((group) => {
          for ( const version of group.versions ) {
            out.push({ groupName: group.id, groupVersion: version?.groupVersion });
          }
        });
      }

      return [...new Set(out)];
    },

    /** Determines if the policy originates from ArtifactHub or the kubewarden-defaults chart */
    disabledcontextAwareResources() {
      const annotations = this.value.policy?.metadata?.annotations;

      return !!annotations?.[ARTIFACTHUB_PKG_ANNOTATION] || annotations?.['meta.helm.sh/release-name'] === KUBEWARDEN_APPS.RANCHER_DEFAULTS;
    },

    isView() {
      return this.mode === _VIEW;
    },
  },

  methods: {
    addResource() {
      this.contextAwareResources.push({});
    },

    removeResource(index) {
      removeAt(this.contextAwareResources, index);
    }
  }
};
</script>

<template>
  <Loading v-if="$fetchState.pending" mode="relative" />
  <div v-else>
    <p>{{ t('kubewarden.policyConfig.contextAware.description') }}</p>
    <Banner
      data-testid="kw-policy-config-context-banner"
      class="mb-20"
      color="warning"
      :label="t('kubewarden.policyConfig.contextAware.warning')"
    />

    <div v-for="(resource, index) in contextAwareResources" :key="'filtered-resource-' + index">
      <Resource
        ref="lastResource"
        v-model="contextAwareResources[index]"
        :data-testid="`kw-policy-config-context-resource-${ index }`"
        :disabled="disabledcontextAwareResources"
        :mode="mode"
        :api-group-versions="apiGroupVersions"
      >
        <template v-if="!isView && !disabledcontextAwareResources" #removeResource>
          <button :data-testid="`kw-policy-config-context-remove-${ index }`" type="button" class="btn role-link p-0" @click="removeResource(index)">
            {{ t('kubewarden.policyConfig.contextAware.resource.remove') }}
          </button>
        </template>
      </Resource>
    </div>

    <button
      v-if="!isView && !disabledcontextAwareResources"
      data-testid="kw-policy-config-context-add"
      type="button"
      class="btn role-tertiary add"
      @click="addResource"
    >
      {{ t('kubewarden.policyConfig.contextAware.resource.add') }}
    </button>
  </div>
</template>
